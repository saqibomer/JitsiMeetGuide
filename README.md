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
server {
      listen 80;
      listen [::]:80;
      server_name  meetings.mydomain.com;
      rewrite ^ https://$http_host$request_uri? permanent;	# force redirect http to https
  }
  server {
      listen 443 ssl http2;
      listen   [::]:443 ssl http2;
      server_name meetings.mydomain.com;

      ssl on;
      ssl_certificate /home/ssl/my.ssl.crt;            # path to your cacert.pem
      ssl_certificate_key /home/ssl/key;	# path to your privkey.pem

      # global SSL options with Perfect Forward Secrecy (PFS) high strength ciphers
      # first. PFS ciphers are those which start with ECDHE which means (EC)DHE
      # which stands for (Elliptic Curve) Diffie-Hellman Ephemeral.
      ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
      ssl_protocols TLSv1.2 TLSv1.3;
      ssl_session_timeout 1d;
      ssl_session_cache shared:MozSSL:10m;  # about 40000 sessions
      resolver 8.8.8.8;
      ssl_stapling on;
      ssl_stapling_verify on;

      # this are optional but recommended Security Headers
      # thats the HSTS Header - it will enforce that all connections regarding this host and the subdomains will only used with encryption
      add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
      # this tells the browser that when click on links in the chat / pad, the referrer is only set when the link points to hosts site and encrypted
      add_header Referrer-Policy strict-origin;
      # this tells the browser that jitsi can't be embedded in a Frame
      add_header X-Frame-Options "DENY";
      add_header X-Content-Type-Options nosniff;
      add_header X-XSS-Protection "1; mode=block";
      add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; img-src 'self'; style-src 'self' 'unsafe-inline'; font-src 'self'; object-src 'none'; base-uri 'none'; frame-ancestors 'none'; form-action 'none'; block-all-mixed-content";
      # List of Browser-Features which are allowed / denied for this Site
    add_header Feature-Policy "geolocation 'none'; camera 'self'; microphone 'self'; speaker 'self'; autoplay 'none'; battery 'none'; accelerometer 'none'; autoplay 'none'; payment 'none';";


      root /srv/meetings.mydomain.com;
      index index.html;
    
      location ~ ^/(?!(http-bind|external_api\.|xmpp-websocket))([a-zA-Z0-9=_äÄöÖüÜß\?\-]+)$ {
          rewrite ^/(.*)$ / break;
      }
      # BOSH
      location /http-bind {
          proxy_pass      http://localhost:5280/http-bind;
          proxy_set_header X-Forwarded-For $remote_addr;
          proxy_set_header Host $http_host;
      }
      # xmpp websockets
      location /xmpp-websocket {
          proxy_pass http://localhost:5280;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "upgrade";
          proxy_set_header Host $host;
          tcp_nodelay on;
      }
 }
 ```
 Hit command + c or Control + c to exit. Save before exiting. 

## Restart 

Once installation is finished restart nginx, daemon and videobridge
```sudo systemctl restart nginx```
```sudo systemctl daemon-reload```
```sudo systemctl restart jitsi-videobridge2```
## Test
Open your domain in webbrowser. If everything works well you should see jitsimeet.

