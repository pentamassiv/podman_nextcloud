# podman_nextcloud
Short description on how to install and run Nextcloud with rootless podman in a local network with SSL. You need to have podman already installed.

## Edit firewall rules
Allow access to port 443
```
sudo ufw allow 443/tcp
```
Check the status of ufw
```
sudo ufw status verbose
```

## Add SSL Encryption

Write a Dockerfile to run a script after starting the container
```
cat > Dockerfile << EOF
FROM docker.io/nextcloud:latest
COPY setssl.sh /usr/local/bin/
ENV NEXTCLOUD_UPDATE=1
RUN /usr/local/bin/setssl.sh admin@nextcloud nextcloud
EOF
```

Add the script to enable SSL
```
cat > setssl.sh << EOF
# setssl.sh
# USAGE: setssl.sh <email> <domain>

echo 'SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
SSLProtocol All -SSLv2 -SSLv3
SSLHonorCipherOrder On
Header always set Strict-Transport-Security "max-age=63072000; includeSubdomains"
Header always set X-Frame-Options DENY
Header always set X-Content-Type-Options nosniff
SSLCompression off
SSLSessionTickets Off' > /etc/apache2/conf-available/ssl-params.conf
echo "<IfModule mod_ssl.c>
        <VirtualHost _default_:443>
                ServerAdmin $2
                ServerName $1
" > /etc/apache2/sites-available/default-ssl.conf
echo '
                DocumentRoot /var/www/html

                ErrorLog ${APACHE_LOG_DIR}/error.log
                CustomLog ${APACHE_LOG_DIR}/access.log combined

                SSLEngine on

                SSLCertificateFile    /etc/ssl/nextcloud/cert.pem
                SSLCertificateKeyFile /etc/ssl/nextcloud/key.pem

                <FilesMatch "\.(cgi|shtml|phtml|php)$">
                                SSLOptions +StdEnvVars
                </FilesMatch>
                <Directory /usr/lib/cgi-bin>
                                SSLOptions +StdEnvVars
                </Directory>
        </VirtualHost>
</IfModule>' >> /etc/apache2/sites-available/default-ssl.conf
a2enmod ssl >/dev/null
a2ensite default-ssl >/dev/null
a2enconf ssl-params >/dev/null
EOF
```

Make script executable
```
sudo chmod +x setssl.sh
```

## Allow binding to lower ports for non-root users
Add 'net.ipv4.ip_unprivileged_port_start=443' to /etc/sysctl.conf

## Create a CA and self sign the certificates
Follow this tutorial: https://stackoverflow.com/a/66558789. 
Copy the CA_cert.pem file to each client that is supposed to trust the certificate

## Set up podman
Create a new network
```
podman network create nextcloud-net
```

List all networks
```
podman network ls
```

Create the volumes
```
podman volume create nextcloud-app
podman volume create nextcloud-data
podman volume create nextcloud-db
podman volume create nextcloud-ssl
```

List volumes
```
podman volume ls
```

Create files containing the secret passwords (and then enter the password in the file)
```
touch DB_USER_PASSWORD
touch DB_ROOT_PASSWORD
touch NC_PASSWORD
```

Create secrets for the passwords
```
podman secret create DB_USER_PASSWORD DB_USER_PASSWORD
podman secret create DB_ROOT_PASSWORD DB_ROOT_PASSWORD
podman secret create NC_PASSWORD NC_PASSWORD
```

Start MariaDB
```
podman run --detach \
  --env MYSQL_DATABASE=nextcloud \
  --env MYSQL_USER=nextcloud \
  --env MYSQL_PASSWORD_FILE=/run/secrets/DB_USER_PASSWORD \
  --env MYSQL_ROOT_PASSWORD_FILE=/run/secrets/DB_ROOT_PASSWORD \
  --secret DB_USER_PASSWORD \
  --secret DB_ROOT_PASSWORD \
  --volume nextcloud-db:/var/lib/mysql \
  --network nextcloud-net \
  --restart on-failure \
  --name nextcloud-db \
  docker.io/library/mariadb:latest
```
  
Deploy Nextcloud
```
podman build --tag nextcloud-ssl .
podman run --detach \
  --env MYSQL_HOST=nextcloud-db.dns.podman \
  --env MYSQL_DATABASE=nextcloud \
  --env MYSQL_USER=nextcloud \
  --env MYSQL_PASSWORD_FILE=/run/secrets/DB_USER_PASSWORD \
  --env NEXTCLOUD_ADMIN_USER=NC_ADMIN \
  --env NEXTCLOUD_ADMIN_PASSWORD_FILE=/run/secrets/NC_PASSWORD \
  --env NEXTCLOUD_DATA_DIR=/var/www/html/data \
  --secret DB_USER_PASSWORD \
  --secret NC_PASSWORD \
  --volume nextcloud-app:/var/www/html \
  --volume nextcloud-data:/var/www/html/data \
  --volume nextcloud-ssl:/etc/ssl/nextcloud \
  --network nextcloud-net \
  --restart on-failure \
  --name nextcloud-ssl \
  --publish 443:443 \
  localhost/nextcloud-ssl:latest
```

Check running containers
```
podman container ls
```

## Add trusted domains
Copy file to host machine
```
podman cp nextcloud-ssl:/var/www/html/config/config.php .
```

Edit file
```
nano config.php
```

Change to:
```
'trusted_domains' => 
  array (
    0 => 'localhost',
    1 => 'nextcloud',
    2 => '192.168.178.*',
    3 => '[fe80::56e7:cdc1:2f07:c432]',
  ),
```

Copy file back
```
podman cp ./config.php nextcloud-ssl:/var/www/html/config/
```

Fix ownership of file
```
podman exec -it nextcloud-ssl bash
chown --reference=config/autoconfig.php config/config.php
exit
```

Remove config file
```
rm config.php
```

## Have systemd start the containers when booting the device

Automatically generate the service files
```
podman generate systemd --new --name nextcloud-db -f
podman generate systemd --new --name nextcloud-ssl -f
```

Add dependency to nextcloud-db to nextcloud service so it always starts after the db container
```
nano container-nextcloud-ssl.service
```
Add the following lines:
```
After=nextcloud-db.service
Requires=nextcloud-db.service
```

Move service files to correct directory
```
mkdir -p ~/.config/systemd/user/
mv -v container-nextcloud-db.service ~/.config/systemd/user/
mv -v container-nextcloud-ssl.service ~/.config/systemd/user/
```

Have systemd check for new services
```
systemctl --user daemon-reload
sudo systemctl daemon-reload
```

Enable the services
```
systemctl --user enable container-nextcloud-db.service
systemctl --user enable container-nextcloud-ssl.service
```

Check their status
```
systemctl --user status container-nextcloud-db.service
systemctl --user status container-nextcloud-ssl.service
```

Maybe stop and remove existing containers
```
podman container stop nextcloud-db
podman container rm nextcloud-db
podman container stop nextcloud-ssl
podman container rm nextcloud-ssl
```

Reboot and check if everything works
```
sudo reboot
podman container list
```
