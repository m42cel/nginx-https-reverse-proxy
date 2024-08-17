# nginx HTTPS Reverse Proxy
A minimal docker-compose and nginx configuration to enable https for other applications, using only the official nginx image.

Many published docker images don't offer an out-of-the-box way to enable https and configure the server certificate and private key.
An easy solution is to use an nginx reverse proxy that handles SSL and proxy requests to the application.

## How To
1. In the application's directory, create `nginx` directory and place adapted `nginx.conf` there (see example below).
2. In the `nginx` directory, create an `ssl` directory and place the server certificate and private key there. If the private key uses a password, put the password in a password file.
3. Extend the `docker-compose.yaml` and include the `nginx` service. Adapt the marked lines (see example below).

### Directory structure
```shell
.
├── docker-compose.yaml
└── nginx
    ├── nginx.conf
    └── ssl
        ├── mayapp.crt # Your app's server certificate
        ├── mayapp.key # Your app's private key
        └── mayapp.pass # (Optional) A file containing the private key's password
```

### nginx.conf
```nginx
worker_processes auto;

events { worker_connections 1024; }

http {
  server {
      listen              443 ssl;
      ssl_certificate     /etc/nginx/ssl/mayapp.crt; # Adapt to .crt file name
      ssl_certificate_key /etc/nginx/ssl/mayapp.key; # Adapt to .key file name
      ssl_password_file   /etc/nginx/ssl/mayapp.pass; # (Optional) Only needed if the private key is password-protected

      location / {
          proxy_pass http://myapp:8080; # Adapt to service name defined in docker-compose.yaml and the port the application runs on
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Real-IP  $remote_addr;
          proxy_set_header Host $host;
      }
  }

  server { # (Optional) Redirect http requests to https
      listen 80;
      return 307 https://$host$request_uri;
  }
}
```

### docker-compose.yaml
```yaml
services:
  nginx:
    image: nginx:latest
    container_name: myapp_nginx # Adapt to name of your app
    restart: unless-stopped
    volumes:
      - ./nginx/ssl:/etc/nginx/ssl
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
    ports:
      - "192.168.0.42:443:443" # Adapt IP on which the container shoud listen on, or remove IP part if it should listen on all interfaces
      - "192.168.0.42:80:80" # (Optional) Only needed if http should be redirected to https
    depends_on:
    - myapp # Adapt to name of your app
  myapp: # Your http-only app that should be proxied
    image: hello-world:latest
```