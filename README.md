# JitsiMeetGuide
SSH to your server.

# Uninstallation
If you already have installed [JitsiMeet](http://jitsi.github.io/) uninstall it using below command
```
sudo apt purge jigasi jitsi-meet jitsi-meet-web-config jitsi-meet-prosody jitsi-meet-turnserver jitsi-meet-web jicofo jitsi-videobridge2
```

Make sure you have ```root``` or ```sudo``` access

## Install required packages
```
sudo apt update
sudo apt install gnupg2 nginx-full curl wget
apt install apt-transport-https
sudo apt-add-repository universe
sudo apt update
```
## Install JitsiMeet

Decide a domain name for you server like ```meetings.mydomain.com```

Setup domain by running following command
```
sudo nano /etc/hosts
```
And add following in hosts file

```
127.0.1.1 meetings.mydomain.com jitsimeet
127.0.0.1 localhost
x.x.x.x meetings.mydomain.com
# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
```
Where ```x.x.x.x``` is your public IP.

Add Fully Qualified Domain Name (FQDN)
```sudo hostnamectl set-hostname meetings.mydomain.com```

Test setup using following command
``` ping "$(hostname)" ```
If setup is successfull you will see ping messages. 

## Add the Prosody package
```echo deb http://packages.prosody.im/debian $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list
wget https://prosody.im/files/prosody-debian-packages.key -O- | sudo apt-key add -
```

## Add the Jitsi package
```curl https://download.jitsi.org/jitsi-key.gpg.key | sudo sh -c 'gpg --dearmor > /usr/share/keyrings/jitsi-keyring.gpg'
echo 'deb [signed-by=/usr/share/keyrings/jitsi-keyring.gpg] https://download.jitsi.org stable/' | sudo tee /etc/apt/sources.list.d/jitsi-stable.list > /dev/null
```

Update all package sources
```sudo apt update```

## Configure Firewall
```
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 10000/udp
sudo ufw allow 22/tcp
sudo ufw allow 3478/udp
sudo ufw allow 5349/tcp
sudo ufw enable
```

## Install JitsiMeet
Before you install jitsi you need to decide you will use self signed certificate or use your own ssl certificates. We will be using our own certificates. 
Create a directory anywhere on system. I will create at /home/ssl
```cd /home
mkdire ssl
```
Now upload ```.crt``` and ```.key``` file on ```/home/ssl/``` using any FTP client or via SSH. Once ssl certifcate is uploaded run following command

```sudo apt install jitsi-meet```

This will open a setup console. The first thing it will ask is hostname. Add you domain name ```meetings.mydomain.com``` press down arrow key to select ```<Ok>``` and press enter. 
Next it will ask for ssl certificates. You can use first option for self signing certificates or use your own certificate. Press down arrow key to highlight ```I want to use my own certificate``` and press enter.

First enter path for ```.key``` press enter and in next step enter path for ```.crt``` files which in our case it was ```/home/ssl/my.ssl.key``` and ```/home/ssl/my.ssl.crt```


## Configure Nginx
We need to add/update site in nginx

First create a site 
```sudo nano /etc/nginx/sites-available/meetings.mydomain.com```
Add following configuration. Make sure you update ssl path and server name. 

```
server_names_hash_bucket_size 64;

types {
# nginx's default mime.types doesn't include a mapping for wasm
    application/wasm     wasm;
}
server {
      listen 80;
      listen [::]:80;
      server_name  meetings.mydomain.com;
      rewrite ^ https://$http_host$request_uri? permanent;      # force redirect http to https
  }
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name meetings.mydomain.com;

    # Mozilla Guideline v5.4, nginx 1.17.7, OpenSSL 1.1.1d, intermediate configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:10m;  # about 40000 sessions
    ssl_session_tickets off;

    add_header Strict-Transport-Security "max-age=63072000" always;
    set $prefix "";

	ssl_certificate /home/dev-user/ssl/meetings.mydomain.com.crt;            # path to your cacert.pem
     ssl_certificate_key /home/dev-user/ssl/meetings.mydomain.com.key;

    root /usr/share/jitsi-meet;

    # ssi on with javascript for multidomain variables in config.js
    ssi on;
    ssi_types application/x-javascript application/javascript;

    index index.html index.htm;
    error_page 404 /static/404.html;

    gzip on;
    gzip_types text/plain text/css application/javascript application/json image/x-icon application/octet-stream application/wasm;
    gzip_vary on;
    gzip_proxied no-cache no-store private expired auth;
    gzip_min_length 512;

    location = /config.js {
        alias /etc/jitsi/meet/meetings.mydomain.com-config.js;
    }

    location = /external_api.js {
        alias /usr/share/jitsi-meet/libs/external_api.min.js;
    }

    # ensure all static content can always be found first
    location ~ ^/(libs|css|static|images|fonts|lang|sounds|connection_optimization|.well-known)/(.*)$
    {
        add_header 'Access-Control-Allow-Origin' '*';
        alias /usr/share/jitsi-meet/$1/$2;

        # cache all versioned files
        if ($arg_v) {
            expires 1y;
        }
    }

    # BOSH
    location = /http-bind {
        proxy_pass http://127.0.0.1:5280/http-bind?prefix=$prefix&$args;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header Host $http_host;
    }

    # xmpp websockets
    location = /xmpp-websocket {
        proxy_pass http://127.0.0.1:5280/xmpp-websocket?prefix=$prefix&$args;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $http_host;
        tcp_nodelay on;
    }

    # colibri (JVB) websockets for jvb1
    location ~ ^/colibri-ws/default-id/(.*) {
        proxy_pass http://127.0.0.1:9090/colibri-ws/default-id/$1$is_args$args;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        tcp_nodelay on;
    }

    # load test minimal client, uncomment when used
    #location ~ ^/_load-test/([^/?&:'"]+)$ {
    #    rewrite ^/_load-test/(.*)$ /load-test/index.html break;
    #}
    #location ~ ^/_load-test/libs/(.*)$ {
    #    add_header 'Access-Control-Allow-Origin' '*';
    #    alias /usr/share/jitsi-meet/load-test/libs/$1;
    #}

    location ~ ^/([^/?&:'"]+)$ {
        try_files $uri @root_path;
    }

    location @root_path {
        rewrite ^/(.*)$ / break;
    }

    location ~ ^/([^/?&:'"]+)/config.js$
    {
        set $subdomain "$1.";
        set $subdir "$1/";

        alias /etc/jitsi/meet/meetings.mydomain.com-config.js;
    }

    # BOSH for subdomains
    location ~ ^/([^/?&:'"]+)/http-bind {
        set $subdomain "$1.";
        set $subdir "$1/";
        set $prefix "$1";

        rewrite ^/(.*)$ /http-bind;
    }

    # websockets for subdomains
    location ~ ^/([^/?&:'"]+)/xmpp-websocket {
        set $subdomain "$1.";
        set $subdir "$1/";
        set $prefix "$1";

        rewrite ^/(.*)$ /xmpp-websocket;
    }

    # Anything that didn't match above, and isn't a real file, assume it's a room name and redirect to /
    location ~ ^/([^/?&:'"]+)/(.*)$ {
        set $subdomain "$1.";
        set $subdir "$1/";
        rewrite ^/([^/?&:'"]+)/(.*)$ /$2;
    }
}

 ```
 Make sure you rename domain name to you own domain in above configurations.
 
 Hit command + c or Control + c to exit. Save before exiting. 

## Restart 

Once installation is finished restart nginx, daemon and videobridge
```sudo systemctl restart nginx```
```sudo systemctl daemon-reload```
```sudo systemctl restart jitsi-videobridge2```
## Test
Open your domain in webbrowser. If everything works well you should see jitsimeet.

