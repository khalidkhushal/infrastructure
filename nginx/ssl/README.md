### SSL with NGINX-CertBot

Run all services with docker/docker compose.

add file at `$HOME/nginx/nginx.conf` with following content:

````
events {
    worker_connections  1024;
}

http {
    server_tokens off;
    charset utf-8;

    server {
        listen 80 default_server;

        server_name _;

        location / {
            proxy_pass http://<SERVICE_NAME>:<SERVICE_PORT>/;
        }
    }
}
````
write and run `nginx.yaml` for docker compose:
````
services:
    nginx:
        container_name: nginx
        restart: unless-stopped
        image: nginx
        ports:
            - 80:80
            - 443:443
        volumes:
            - $HOME/nginx/nginx.conf:/etc/nginx/nginx.conf #replace $HOME with dir
````
RUN: ` docker compose nginx.yaml up -d `

You shoould see the service on PORT 80 of your public IP.

### AUTOMATIC SSL with CertBot

Change `$HOME/nginx/nginx.conf` to:
````
    server {
        listen 80 default_server;

        .
        .
        .

        location ~ /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }
    }
````
Add volumes for `root /var/www/certbot` to work for acme_challenges, final `nginx.yaml` will be:
````
services:
    nginx:
        container_name: nginx
        restart: unless-stopped
        image: nginx
        ports:
            - 80:80
            - 443:443
        volumes:
            - $HOME/nginx/nginx.conf:/etc/nginx/nginx.conf #replace $HOME with dir
            - $HOME/certbot/conf:/etc/letsencrypt
            - $HOME/certbot/www:/var/www/certbot
````


RUN: `docker compose -f nginx.yaml up -d`

Add `certbot` for cert generation:

````
services:
    nginx:
        container_name: nginx
        restart: unless-stopped
        image: nginx
        ports:
            - 80:80
            - 443:443
        volumes:
            #replace $HOME with dir
            - $HOME/nginx/nginx.conf:/etc/nginx/nginx.conf
            - $HOME/certbot/conf:/etc/letsencrypt
            - $HOME/certbot/www:/var/www/certbot
    certbot:
        image: certbot/certbot
        container_name: certbot
        volumes: 
        - ./certbot/conf:/etc/letsencrypt
        - ./certbot/www:/var/www/certbot
        command: certonly --webroot -w /var/www/certbot --force-renewal --email {email} -d {domain1} -d {domain2} --agree-tos
````
See `docker logs certbot` if the cert generation process is successful.

````
Note:
You can add --dry-run in command section certbot in nginx.yaml so that the failure limit do not reach before successfull cert creation. As we have max 5 failures limit for cloudflare.
````

Now, change `$HOME/nginx/nginx.conf` for redirecting request to port 443 for SSL:
````
events {
    worker_connections  1024;
}

http {
    server_tokens off;
    charset utf-8;

    # always redirect to https
    server {
        listen 80 default_server;
        listen      [::]:80 default_server;

        server_name _;

        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        # use the certificates
        ssl_certificate     /etc/letsencrypt/live/{domain}/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/{domain}/privkey.pem;
        server_name {domain};
        root /var/www/html;
        index index.php index.html index.htm;

        location / {
            proxy_pass http://<SERVICE_NAME>:<PORT>//;
        }

        location ~ /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }
    }
}
````

TADA!, You should be able to access the domain with ssl.

### Automated renewal with crontab

RUN: `crontab -e`

Add a new line and enter following:
`0 5 1 */2 *  docker compose -f /path/to/nginx.yaml up certbot`

This will restart the certbot container after every 2 months, cert expiry is 3 months. 