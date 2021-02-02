# Overview
Agenda:
 - Creation of Docker file
 - Creation of an image using Dockerfile
 - Using Let us encrypt certificate of secure web server

### Prerequisites
- One Linux vm
- Docker Installed
- Docker-compose 

 Open the Zip file and unzip it in /opt/
```
- [ ] move to dir node_project
```
cd  /opt/node_project
```
- [ ] create an image
```
docker build -t node-demo .
```
- [ ] check  images
```
docker images
```
- [ ] test image
```
docker run --name node-demo -p 80:8080 -d node-demo
```
- [ ] Remove all unused images and containers
```
docker system prune -a
```

- [ ] create dir for nginx configuartion
```
mkdir nginx-conf
```
- [ ] create a file
```
vi nginx-conf/nginx.conf
server {
        listen 80;
        listen [::]:80;

        root /var/www/html;
        index index.html index.htm index.nginx-debian.html;

        server_name <domainname>;

        location / {
                proxy_pass http://nodejs:8080;
        }

        location ~ /.well-known/acme-challenge {
                allow all;
                root /var/www/html;
        }
}
```
- [ ] create nginx metric file 
```
vi sub_status.conf
server {
        listen *:81;
        location /metrics {
                           stub_status on;
        }
}
```
Now we will create build (image using docker compose along with container)
Docker compose is another to create images,run images ,create volumes etc

- [ ] create docker-compose.yaml file
```
vi ~/node_project/docker-compose.ym

version: '2'

services:
  nodejs:
    build:
      context: .
      dockerfile: Dockerfile
    image: nodejs
    container_name: nodejs
    restart: unless-stopped
    networks:
      - app-network

  webserver:
    image: nginx:mainline-alpine
    container_name: webserver
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - web-root:/var/www/html
      - ./nginx-conf:/etc/nginx/conf.d
      - certbot-etc:/etc/letsencrypt
      - certbot-var:/var/lib/letsencrypt
    depends_on:
      - nodejs
    networks:
      - app-network

  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot-etc:/etc/letsencrypt
      - certbot-var:/var/lib/letsencrypt
      - web-root:/var/www/html
    depends_on:
      - webserver
    command: certonly --webroot --webroot-path=/var/www/html --email <email address> --agree-tos --no-eff-email --staging -d <your domain anme>

volumes:
  certbot-etc:
  certbot-var:
  web-root:
    driver: local
    driver_opts:
      type: none
      device: /opt/node_project/views/
      o: bind

networks:
  app-network:
    driver: bridge
```

- [ ] run this docker-compose.yaml file
```
docker-compose up -d
```
- [ ] check all contaner created
```
docker-compose ps
```
- [ ] check all volumes created
```
docker volume ls
```
- [ ] to check logs
``` 
docker-compose logs webserver
```
- [ ] run this again if your webserver is working fine 
```
docker-compose up --force-recreate --no-deps certbot
```

- [ ] stop all services
```
docker-compose stop webserver
```
- [ ] create a directory
```
mkdir /opt/node_project/dhparam
```
- [ ] create key
```
sudo openssl dhparam -out /opt/node_project/dhparam/dhparam-2048.pem 2048
```

**NOTE**: once the key is created modify nginx.conf

```
server {
        listen 80;
        listen [::]:80;
        server_name <domainname>;

        location ~ /.well-known/acme-challenge {
          allow all;
          root /var/www/html;
        }

        location / {
                rewrite ^ https://$host$request_uri? permanent;
        }
}

server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name <domainname>;

        server_tokens off;

        ssl_certificate /etc/letsencrypt/live/<domainname>/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/<domainname>/privkey.pem;

        ssl_buffer_size 8k;

        ssl_dhparam /etc/ssl/certs/dhparam-2048.pem;

        ssl_protocols TLSv1.2 TLSv1.1 TLSv1;
        ssl_prefer_server_ciphers on;

        ssl_ciphers ECDH+AESGCM:ECDH+AES256:ECDH+AES128:DH+3DES:!ADH:!AECDH:!MD5;

        ssl_ecdh_curve secp384r1;
        ssl_session_tickets off;

        ssl_stapling on;
        ssl_stapling_verify on;
        resolver 8.8.8.8;

        location / {
                try_files $uri @nodejs;
        }

        location @nodejs {
                proxy_pass http://nodejs:8080;
                add_header X-Frame-Options "SAMEORIGIN" always;
                add_header X-XSS-Protection "1; mode=block" always;
                add_header X-Content-Type-Options "nosniff" always;
                add_header Referrer-Policy "no-referrer-when-downgrade" always;
                add_header Content-Security-Policy "default-src * data: 'unsafe-eval' 'unsafe-inline'" always;
                # add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
                # enable strict transport security only if you understand the implications
        }

        root /var/www/html;
        index index.html index.htm index.nginx-debian.html;
}
```

- [ ] modify docker-compose.yaml block as well
```
nano docker-compose.yml
...
 webserver:
    image: nginx:latest
    container_name: webserver
    restart: unless-stopped
    ports:
      - "80:80"
      **- "443:443"**
    volumes:
      - web-root:/var/www/html
      - ./nginx-conf:/etc/nginx/conf.d
      - certbot-etc:/etc/letsencrypt
      - certbot-var:/var/lib/letsencrypt
      **- dhparam:/etc/ssl/certs**
    depends_on:
      - nodejs
    networks:
      - app-network
...
volumes:
  ...
  dhparam:
    driver: local
    driver_opts:
      type: none
      device: /opt/node_project/dhparam/
      o: bind
```
- [ ] Recreate the webserver service
```
docker-compose up -d --force-recreate --no-deps webserver
```

- [ ] Now check your website 



### Set Up prometheus
- [ ] create prometheus config file
```
vi prometheus.yml
scrape_configs:
  # Scrape Prometheus itself every 5 seconds.
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
  # Scrape the Node Exporter every 5 seconds.
  - job_name: 'node'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9100']
 # Scrape the Nginx 
  - job_name: 'nginx'
    scrape_interval: 5s
    static_configs:
     - targets:
        - <IPADDRESS>:9113
```
- [ ] use below command to have prometheus
```
docker run -d -p 9090:9090 -v /opt/node_project/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus --config.file=/etc/prometheus/prometheus.yml
```
 - [ ]  install nginx  exporter
```
docker run -d  -p 9113:9113 nginx/nginx-prometheus-expor
ter:0.8.0 -nginx.scrape-uri http://<ipaddress>:81/metrics
```
