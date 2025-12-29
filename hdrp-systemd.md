## SYSTEMD SERVICES for application RUNTIME process

***/etc/systemd/system/gunicorn.service***         

```
[Unit]
Description=Gunicorn instance to serve Hiringdog
After=network.target

[Service]
User=ubuntu
Group=ubuntu
WorkingDirectory=/srv/hdrp-staging/current

Environment="PATH=/srv/hdrp-staging/shared/venv/bin"
EnvironmentFile=/srv/hdrp-staging/shared/env/staging.env
Environment="PYTHONUNBUFFERED=1"


ExecStart=/srv/hdrp-staging/shared/venv/bin/gunicorn \
     --config /srv/hdrp-staging/current/gunicorn.conf.py \
     --pid /run/gunicorn/gunicorn.pid \
     hdrpbackend.wsgi:application


ExecReload=/bin/kill -s USR2 $MAINPID
ExecStop=/bin/kill -s TERM $MAINPID

PIDFile=/run/gunicorn/gunicorn.pid
RuntimeDirectory=gunicorn
RuntimeDirectoryMode=0755
Restart=always

# Redirecting standard output and error to log files
#StandardOutput=append:/var/log/hiringdog/gunicorn.log
#StandardError=append:/var/log/hiringdog/gunicorn_error.log

# Set limits
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target

```

***/etc/systemd/system/celery.service***         

```
[Unit]
Description=Celery Service
After=network.target

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/srv/hdrp-staging/current

Environment="PATH=/srv/hdrp-staging/shared/venv/bin"
EnvironmentFile=/srv/hdrp-staging/shared/env/staging.env
Environment="PYTHONUNBUFFERED=1"

ExecStart=/srv/hdrp-staging/shared/venv/bin/celery -A hdrpbackend worker --loglevel=info

# Gracefully stop with SIGTERM
KillSignal=SIGTERM
TimeoutStopSec=300

Restart=always
RestartSec=3
TimeoutSec=300
LimitNOFILE=65536

# Log to journal for easier access to logs
StandardOutput=append:/srv/hdrp-staging/shared/logs/celery.log
StandardError=inherit

[Install]
WantedBy=multi-user.target
```
***cat /etc/systemd/system/celery-beat.service***                                                                                  
```
[Unit]
Description=Celery Beat Service
After=network.target

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/srv/hdrp-staging/current

Environment="PATH=/srv/hdrp-staging/shared/venv/bin"
EnvironmentFile=/srv/hdrp-staging/shared/env/staging.env
Environment="PYTHONUNBUFFERED=1"

ExecStart=/srv/hdrp-staging/shared/venv/bin/celery -A hdrpbackend beat \
  --scheduler django_celery_beat.schedulers.DatabaseScheduler \
  --loglevel=info

Restart=always
RestartSec=3
TimeoutSec=300
LimitNOFILE=65536

StandardOutput=append:/srv/hdrp-staging/shared/logs/celery_beat.log
StandardError=inherit

[Install]
WantedBy=multi-user.target
```
***UNIX Socket***
Instead of binding Gunicorn to a TCP port, it binds to a Unix socket (/run/gunicorn/gunicorn.sock).
Nginx proxies requests to this socket.
Benefits:
- Faster than TCP localhost. lower latency 
- No open port exposed. 
- Clear boundary: Nginx ⇄ Gunicorn via socket.
***cat /etc/nginx/sites-available/hiringdogbackend***                                    
```
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 444;  # Close connection for requests to the IP
}

server {
    if ($host = staging.hdrplatform.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot
   
    listen 80;
    server_name staging.hdrplatform.com;
    return 404; # managed by Certbot


}

server {
    server_name staging.hdrplatform.com;

    location / {
        proxy_pass http://unix:/run/gunicorn/hdrp.sock;  # Change port if needed
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    client_max_body_size 25M;
    error_log /srv/hdrp-staging/shared/logs/nginx/hiringdogbackend_error.log;
    access_log /srv/hdrp-staging/shared/logs/nginx/hiringdogbackend_access.log;

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/staging.hdrplatform.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/staging.hdrplatform.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
```

### Test and Reload: 
` sudo nginx -t && sudo systemctl reload nginx`

- Check if the Socket and PID Are Active: ` ls -l /run/gunicorn/gunicorn.sock `
- View the PID file (if needed): ` cat /run/gunicorn/gunicorn.pid `


### Fix Permission Issues (If Nginx can't access the socket) 
Give socket to group www-data (Nginx user): 
``` 
sudo chown ubuntu:www-data /run/gunicorn/gunicorn.sock
```
Ensure group (Nginx) can read/write:
```
sudo chmod 660 /run/gunicorn/gunicorn.sock
```
- Test Your Deployment ` https://api.hdiplatform.in `
- Access Log:
```
sudo tail -f /var/log/nginx/hiringdogbackend_access.log
```
- Error Log: 
```
sudo tail -f /var/log/nginx/hiringdogbackend_error.log
```
