# Security

## Database encryption

Sensitive data is stored in the database, in an encrypted form. You should set the configuration option `access_key_encryption` in configuration file to enable Access Keys encryption. It must be generated by command:

```bash
head -c32 /dev/urandom | base64
```

----

## Encrypted connection

For security reasons, Semaphore **should not be used** over unencrypted HTTP!

Why use encrypted connections? See: [Article](https://www.cloudflare.com/learning/ssl/why-use-https/)

Options you have:

* VPN
* Reverse Proxy with SSL


## VPN

You can use a Client-to-Site VPN, that terminates on the Semaphore server, to encrypt & secure the connection.

## Reverse Proxy with SSL

Semaphore doesn't support SSL/TLS on its own.

You need to use a reverse proxy like
[NGINX](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/) /
[Apache](https://httpd.apache.org/docs/current/howto/reverse_proxy.html) /
[HAProxy](https://www.haproxy.com/documentation/hapee/latest/onepage/)
 in front of Semaphore to serve secure connections.

### NGINX

Configuration example:

```yaml
server {
  listen 443 ssl;
  server_name  _;

  # add Strict-Transport-Security to prevent man in the middle attacks
  add_header Strict-Transport-Security "max-age=31536000" always;

  # SSL
  ssl_certificate /etc/nginx/cert/cert.pem;
  ssl_certificate_key /etc/nginx/cert/privkey.pem;

  # Recommendations from 
  # https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html
  ssl_protocols TLSv1.1 TLSv1.2;
  ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';
  ssl_prefer_server_ciphers on;
  ssl_session_cache shared:SSL:10m;

  # required to avoid HTTP 411: see Issue #1486 
  # (https://github.com/docker/docker/issues/1486)
  chunked_transfer_encoding on;

  location / {
    proxy_pass http://127.0.0.1:3000/;
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_buffering off;
    proxy_request_buffering off;
  }

  location /api/ws {
    proxy_pass http://127.0.0.1:3000/api/ws;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Origin "";
  }
}
```

### Apache

```
<VirtualHost *:443>

    ServerName yourapp.test

    ServerAdmin webmaster@localhost
	
    SSLEngine on
    SSLCertificateFile /path/to/www_yoursite_com.crt
    SSLCertificateKeyFile /path/to/www_yoursite_com.key

    <Location />
        ProxyPass http://127.0.0.1:3000/
        ProxyPassReverse http://127.0.0.1:3000/
    </Location>

    <Location /api/ws>
        ProxyPass ws://127.0.0.1:3000/api/ws/
    </Location>
</VirtualHost>
```

### Others

If you want to use any other reverse proxy - make sure to also forward websocket connections on the `/api/ws` route!
