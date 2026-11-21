# A single origin, through nginx

## The browser only talks to the web container

```
Browser ──> web container (nginx in production, Vite in development) ──/api/*──> api:3000
```

The web container serves the frontend and relays every `/api/*` request to the API, without the `/api` prefix:
`/api/users/me` reaches the API as `/users/me`. The API is never published outside Docker.

- **No CORS**: the frontend and the API share one origin, so the browser has no cross-origin request to allow, and the
  session cookie stays `SameSite=Strict`.
- **No API URL to configure in the frontend**: it always calls `/api`, whatever the environment.
- **Development reproduces the same setup** with Vite's proxy, so nothing behaves differently once deployed.

## Development: Vite's proxy

```ts
server: {
    // Internal port: Docker maps it to WEB_PORT on the host, so there is no reason to change it
    port: 5173,
    // Fails instead of silently switching to another port, which the Docker mapping wouldn't reach
    strictPort: true,
    // The API is only reachable inside Docker: the browser always talks to the web origin
    proxy: {
        '/api': {
            target: 'http://api:3000',
            rewrite: (path) => path.replace(/^\/api/, ''),
        },
    },
},
```

The API's address is its Compose service name, hardcoded: it only resolves inside the network Compose creates.

## Production: `nginx.conf`

```nginx
server {
  # Internal port: Docker maps it to WEB_PORT on the host, so there is no reason to change it
  listen 80;

  root /usr/share/nginx/html;
  index index.html;

  # Docker's DNS. The API's IP changes whenever its container is recreated: with its address in a variable, nginx
  # resolves it again (at most every 10 s) instead of keeping the IP found at startup. It also starts without the API
  resolver 127.0.0.11 valid=10s;
  set $api http://api:3000;

  location / {
    try_files $uri $uri/ /index.html;
  }

  # The API is only reachable inside Docker: the browser always talks to this origin
  location /api/ {
    # proxy_pass doesn't strip the prefix itself when its address is a variable
    rewrite ^/api/(.*)$ /$1 break;
    proxy_pass $api;
    # Appends the caller's IP, as every proxy should: the API reads the client's IP from this header (trustProxy)
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}
```

- **A plain `nginx.conf` with hardcoded values**, copied to `/etc/nginx/conf.d/default.conf`. No template filled from
  environment variables at startup: nothing in it differs between environments.
- **Every unknown path serves `index.html`**: the frontend's router handles the URLs, so reloading `/transactions` must
  return the application, not a 404.
- **The API's address goes through Docker's DNS on every request.** Inside a Compose network, `api` is a name that
  Docker's DNS server (`127.0.0.11`) turns into the container's current IP. By default, nginx looks the name up once, at
  startup, and keeps that IP forever. Two things then break: nginx refuses to start if the API isn't running yet, and
  once the API container is recreated (a deployment) with a new IP, nginx keeps sending requests to the old one. Putting
  the address in a variable, with a `resolver`, makes nginx look it up again as needed, at most every 10 seconds. The
  only cost: `proxy_pass` with a variable no longer strips the `/api` prefix by itself, hence the `rewrite`.
- **`X-Forwarded-For` carries the client's IP to the API**, which only sees nginx otherwise: without it, every request
  would come from the same address, and a rate limit would block everyone at once. The API trusts that header
  (`trustProxy`), which is only safe because the API is reachable through nginx alone. A proxy in front of nginx (a
  tunnel, a CDN) usually adds the client's IP first, and nginx appends its own caller after it.
