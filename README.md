# Simple setup of released simplelogin app with [docker](https://hub.docker.com/r/simplelogin/app/tags):

## Server prerequisites:

1. Setup your own server with hostname set to your domain where you will be hosting simplelogin webapp (sl.example.com) and add entry in '/etc/hosts':
```127.0.0.1    sl.example.com  sl```

2. Make sure everything is up to date:
```sudo apt update && sudo apt upgrade -y```

3. enable these firewall rules
```
sudo ufw allow 25
sudo ufw allow 80
sudo ufw allow 443
```
4. enable firewall
```
sudo ufw enable
```

5. Install dnsutils
```
sudo apt install dnsutils
```

6. Create SL folders
```
mkdir sl sl/pgp sl/db sl/upload
```

7. Generate Domain Keys
```
openssl genrsa -traditional -out dkim.key 1024
```
The '-traditional' flag is needed for openssl version 3 (check version with 'openssl version')
```
openssl rsa -in dkim.key -pubout -out dkim.pub.key
```

# DNS records
----------------------------------------------------------------------------------------
1. A-record that points sl.example.com. to my server IP
 This needs to be setup at your domain. If you use DynDNS, make sure your WAN adress is updated correctly.

2. MX-record that points example.com. to sl.example.com. with priority 10
Also setup at your domain. Each registrar is different so please refer to to their documentation

3. TXT-record for dkim._domainkey.example.com.
```
sed "s/-----BEGIN PUBLIC KEY-----/v=DKIM1; k=rsa; p=/g" $(pwd)/dkim.pub.key | sed 's/-----END PUBLIC KEY-----//g' |tr -d '\n' | awk 1
```
Copy the output of that command into the DKIM record on your domain.

4. TXT-record for example.com.
```
v=spf1 mx ~all
```

5. TXT-record for _dmarc.example.com.
```
v=DMARC1; p=quarantine; adkim=r; aspf=r
```

6. VERIFY!
```
dig @1.1.1.1 example.com <type>
nslookup -type= example.com
```
----------------------------------------------------------------------------------------
## Preparing Docker environment
1. Install Docker
```
sudo curl -fsSL https://get.docker.com | sh
```

2. Docker Network
```
sudo docker network create -d bridge \
    --subnet=10.0.0.0/24 \
    --gateway=10.0.0.1 \
    sl-network
```

3. Postgres db container
```
sudo docker run -d \
    --name sl-db \
    -e POSTGRES_PASSWORD=MySuperStrongPassword \
    -e POSTGRES_USER=dbuser \
    -e POSTGRES_DB=simplelogin \
    -p 127.0.0.1:5432:5432 \
    -v $(pwd)/sl/db:/var/lib/postgresql/data \
    --restart always \
    --network="sl-network" \
    postgres:13
```

4. test db
```
sudo docker exec -it sl-db psql -U dbuser simplelogin
```
5. Install and configure postfix
```
sudo apt install postfix postfix-pgsql -y
```
```
sudo nano /etc/postfix/main.cf
```
-------------------------------------------------------------------------
```
# POSTFIX config file, adapted for SimpleLogin
smtpd_banner = $myhostname ESMTP $mail_name (Ubuntu)
biff = no

# appending .domain is the MUA's job.
append_dot_mydomain = no

# Uncomment the next line to generate "delayed mail" warnings
#delay_warning_time = 4h

readme_directory = no

# See http://www.postfix.org/COMPATIBILITY_README.html -- default to 2 on
# fresh installs.
compatibility_level = 2

# TLS parameters
smtpd_tls_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
smtpd_tls_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
smtp_tls_security_level = may
smtpd_tls_security_level = may

# See /usr/share/doc/postfix/TLS_README.gz in the postfix-doc package for
# information on enabling SSL in the smtp client.

alias_maps = hash:/etc/aliases
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 10.0.0.0/24

# Set your domain here
mydestination =
myhostname = sl.example.com
mydomain = example.com
myorigin = example.com

relay_domains = pgsql:/etc/postfix/pgsql-relay-domains.cf
transport_maps = pgsql:/etc/postfix/pgsql-transport-maps.cf

# HELO restrictions
smtpd_delay_reject = yes
smtpd_helo_required = yes
smtpd_helo_restrictions =
    permit_mynetworks,
    reject_non_fqdn_helo_hostname,
    reject_invalid_helo_hostname,
    permit

# Sender restrictions:
smtpd_sender_restrictions =
    permit_mynetworks,
    reject_non_fqdn_sender,
    reject_unknown_sender_domain,
    permit

# Recipient restrictions:
smtpd_recipient_restrictions =
   reject_unauth_pipelining,
   reject_non_fqdn_recipient,
   reject_unknown_recipient_domain,
   permit_mynetworks,
   reject_unauth_destination,
   reject_rbl_client zen.spamhaus.org=127.0.0.[2..11],
   reject_rbl_client bl.spamcop.net=127.0.0.2,
   permit
```
-------------------------------------------------------------------------
```
sudo nano /etc/postfix/pgsql-relay-domains.cf
```
-------------------------------------------------------------------------
```
# postgres config
hosts = localhost
user = dbuser
password = MySuperStrongPassword
dbname = simplelogin

query = SELECT domain FROM custom_domain WHERE domain='%s' AND verified=true
    UNION SELECT domain FROM public_domain WHERE domain='%s'
    UNION SELECT '%s' WHERE '%s' = 'example.com' LIMIT 1;
```
-------------------------------------------------------------------------
```
sudo nano /etc/postfix/pgsql-transport-maps.cf
```
-------------------------------------------------------------------------
```
# postgres config
hosts = localhost
user = dbuser
password = MySuperStrongPassword
dbname = simplelogin

# forward to smtp:127.0.0.1:20381 for custom domain AND email domain
query = SELECT 'smtp:127.0.0.1:20381' FROM custom_domain WHERE domain = '%s' AND verified=true
    UNION SELECT 'smtp:127.0.0.1:20381' FROM public_domain WHERE domain = '%s'
    UNION SELECT 'smtp:127.0.0.1:20381' WHERE '%s' = 'example.com' LIMIT 1;
```
-------------------------------------------------------------------------

6. Restart Postfix
```
sudo systemctl restart postfix
```
7. Generate Flask secret
```
sudo apt install pwgen
```

8. Copy output of command into Flask Secret
```
pwgen -B -s -y 64 -N 1
```

9. Create environment configuration for hosting SimpleLogin via docker
```
nano $(pwd)/simplelogin.env
```
-------------------------------------------------------------------------
```
# WebApp URL
URL=http://sl.example.com

# domain used to create alias
EMAIL_DOMAIN=example.com

# transactional email is sent from this email address
SUPPORT_EMAIL=support@example.com

# custom domain needs to point to these MX servers
EMAIL_SERVERS_WITH_PRIORITY=[(10, "sl.example.com.")]

# By default, new aliases must end with ".{random_word}". This is to avoid a person taking all "nice" aliases.
# this option doesn't make sense in self-hosted. Set this variable to disable this option.
DISABLE_ALIAS_SUFFIX=1

# the DKIM private key used to compute DKIM-Signature
DKIM_PRIVATE_KEY_PATH=/dkim.key

# DB Connection
DB_URI=postgresql://dbuser:MySuperStrongPassword@sl-db:5432/simplelogin

FLASK_SECRET=*copy output of pwgen -B -s -y 64 -N 1*

GNUPGHOME=/sl/pgp

LOCAL_FILE_UPLOAD=1

POSTFIX_SERVER=10.0.0.1

```
-------------------------------------------------------------------------


10. db migration (replace simplelogin/app:4.6.2-beta with latest version in [Docker](https://hub.docker.com/r/simplelogin/app/tags)
```
sudo docker run --rm \
    --name sl-migration \
    -v $(pwd)/sl:/sl \
    -v $(pwd)/sl/upload:/code/static/upload \
    -v $(pwd)/dkim.key:/dkim.key \
    -v $(pwd)/dkim.pub.key:/dkim.pub.key \
    -v $(pwd)/simplelogin.env:/code/.env \
    --network="sl-network" \
    simplelogin/app:4.6.2-beta alembic upgrade head
```
11. init data
```
sudo docker run --rm \
    --name sl-init \
    -v $(pwd)/sl:/sl \
    -v $(pwd)/simplelogin.env:/code/.env \
    -v $(pwd)/dkim.key:/dkim.key \
    -v $(pwd)/dkim.pub.key:/dkim.pub.key \
    --network="sl-network" \
    simplelogin/app:4.6.2-beta python init_app.py
```
12. webapp
```
sudo docker run -d \
    --name sl-app \
    -v $(pwd)/sl:/sl \
    -v $(pwd)/sl/upload:/code/static/upload \
    -v $(pwd)/simplelogin.env:/code/.env \
    -v $(pwd)/dkim.key:/dkim.key \
    -v $(pwd)/dkim.pub.key:/dkim.pub.key \
    -p 127.0.0.1:7777:7777 \
    --restart always \
    --network="sl-network" \
    simplelogin/app:4.6.2-beta
```
13. email handler
```
sudo docker run -d \
    --name sl-email \
    -v $(pwd)/sl:/sl \
    -v $(pwd)/sl/upload:/code/static/upload \
    -v $(pwd)/simplelogin.env:/code/.env \
    -v $(pwd)/dkim.key:/dkim.key \
    -v $(pwd)/dkim.pub.key:/dkim.pub.key \
    -p 127.0.0.1:20381:20381 \
    --restart always \
    --network="sl-network" \
    simplelogin/app:4.6.2-beta python email_handler.py
```
14. job runner
```
sudo docker run -d \
    --name sl-job-runner \
    -v $(pwd)/sl:/sl \
    -v $(pwd)/sl/upload:/code/static/upload \
    -v $(pwd)/simplelogin.env:/code/.env \
    -v $(pwd)/dkim.key:/dkim.key \
    -v $(pwd)/dkim.pub.key:/dkim.pub.key \
    --restart always \
    --network="sl-network" \
    simplelogin/app:4.6.2-beta python job_runner.py
```
15. nginx
```
sudo apt install nginx
sudo nano /etc/nginx/sites-enabled/simplelogin
```
-------------------------------------------------------------------------
```
server {
    server_name  sl.example.com;

    location / {
        proxy_pass http://localhost:7777;
    }
}
```
-------------------------------------------------------------------------

16. reload nginx
```
sudo systemctl reload nginx
```
## SSL certificate from Let's Encrypt
```
sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo certbot --nginx
```
```
sudo nano /etc/postfix/main.cf
```
-------------------------------------------------------------------------
```
smtpd_tls_cert_file = /etc/letsencrypt/live/sl.example.com/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/sl.example.com/privkey.pem
```
-------------------------------------------------------------------------
```
nano simplelogin.env
```
-------------------------------------------------------------------------
```
URL=https://sl.example.com
```
```
sudo docker restart sl-db sl-app sl-email sl-job-runner
```
-------------------------------------------------------------------------

### Enable HSTS
```
sudo nano /etc/nginx/sites-enabled/simplelogin
```
-------------------------------------------------------------------------
```
add_header Strict-Transport-Security "max-age: 31536000; includeSubDomains" always;
```
-------------------------------------------------------------------------

**reload nginx**
```
sudo systemctl reload nginx
```

# make account premium
```
sudo docker exec -it sl-db psql -U hoyohayo simplelogin
UPDATE users SET lifetime = TRUE;
exit
```
# disable registrations in simplelogin.env
```
DISABLE_REGISTRATION=1
DISABLE_ONBOARDING=true
```
# restart web-app
```
sudo docker restart sl-app
```