# Benötigte Ports
=> 443
=> 80
=> 22

# Nginx Server Konfigurieren (PHP, SFTP, SSL)
## Mit Docker Continainer verbinden
`docker exec -it container_name bash`

## Change Root Passwort
`passwd root`

## Benutzer hinzufügen
`adduser sftpuser`

## Install SSH-Server
1. `apt-get update`
2. `apt-get upgrade`
3. `apt-get install openssh-server`
4. `apt-get install nano`

## Configure SSH-Server
1. `nano /etc/ssh/sshd_config` -> Uncomment "Port 22"
```
Match User sftpuser
        ForceCommand internal-sftp
        AllowTcpForwarding no
        X11Forwarding no
```

2. `/etc/init.d/ssh restart`

## Install PHP
1. `apt-get update`
2. `apt-get upgrade -y`

## Repository für PHP 8.4 herunterladen
```
apt-get -y install lsb-release ca-certificates curl apt-transport-https wget gnupg2
wget https://packages.sury.org/php/apt.gpg -O /usr/share/keyrings/deb.sury.org-php.gpg
echo "deb [signed-by=/usr/share/keyrings/deb.sury.org-php.gpg] https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list
```
2. `apt-get update`

## Nginx installieren
`apt-get install nginx -y`

## PHP 8.4 und FPM installieren
`apt-get install php8.4 php8.4-fpm php8.4-cli php8.4-mbstring php8.4-xml php8.4-curl php8.4-zip -y`

# PHP-Status überprüfen oder starten
1. `php-fpm8.4 -F`
2. optional: `systemctl enable php8.4-fpm`

## Ordner erstellen /var/www/html
`mkdir -p /var/www/html`

## Nginx konfigurieren (/usr/share/nginx/html/conf.d/default.conf) und diesen Teil einfügen
`nano /usr/share/nginx/html/conf.d/default.conf`
```
server {
    listen 80;
    listen [::]:80;
    server_name localhost;

    access_log  /var/www/logs/host.access.log;

    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name localhost;

    access_log  /var/www/logs/host.access.log;

    ssl_certificate /etc/nginx/ssl/selfsigned.crt;
    ssl_certificate_key /etc/nginx/ssl/selfsigned.key;

    root /var/www/html;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

## Eine Datei unter /etc/nginx/snippets/fastcgi-php.conf erstellen und das rein
`mkdir /etc/nginx/snippets`

`nano /etc/nginx/snippets/fastcgi-php.conf`
```
# Standard FastCGI-Konfiguration für PHP

# Teilt URI in Skript + Pfadinfo auf
fastcgi_split_path_info ^(.+?\.php)(/.*)$;

# Wenn Datei nicht existiert, gib 404 zurück
try_files $fastcgi_script_name =404;

# Setze PATH_INFO für PHP-FPM
set $path_info $fastcgi_path_info;
fastcgi_param PATH_INFO $path_info;

# Setze das "index.php" als Index
fastcgi_index index.php;

# Lade die allgemeinen FastCGI-Parameter
include fastcgi_params;

# WICHTIG: PHP-FPM braucht diesen Pfad zur Datei
fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
```

## Configure PHP-FPM
`nano /etc/php/8.4/fpm/pool.d/www.conf` #Ändere zu "listen = 127.0.0.1:9000"

## mysqli und pdo installieren
1. `apt update`
2. `apt install php8.4-mysqli php8.4-pdo php8.4-mysql php8.4-cli php8.4-common`
3. `apt install php8.4-bz2`

## Installation prüfen
1. `php -m | grep -E 'mysqli|pdo'`
2. `php -m | grep bz2`


## php.ini anpassen (/etc/php/8.4/fpm/php.ini) und (/etc/php/8.4/cli/php.ini)
`nano /etc/php/8.4/fpm/php.ini`

`nano /etc/php/8.4/cli/php.ini`
```
memory_limit = 512M

upload_max_filesize = 512M

post_max_size = 1024M

max_execution_time = 300

date.timezone = Europe/Berlin

extension=mysqli
...
extension=pdo_firebird
extension=pdo_mysql
extension=pdo_odbc
extension=pdo_pgsql
extension=pdo_sqlite
```

## SSL Einrichten, dafür den ordner ssl davor erstellen
1. `mkdir -p /etc/nginx/ssl`
2. `cd /etc/nginx/ssl`
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/selfsigned.key \
  -out /etc/nginx/ssl/selfsigned.crt \
  -subj "/C=DE/ST=Test/L=Local/O=Dev/CN=localhost"
```
3. `cd /`

## Access-Log einrichten
`mkdir -p /var/www/logs/`

## Berechtigungen anpassen

`chown sftpuser:sftpuser /var/www/html`

# Setze die Berechtigungen für den Ordner
`chmod 775 /var/www/html`

## Nginx neustarten
`/etc/init.d/nginx restart`

## Nginx testen
`nginx -t`

## Alles neustarten
1. `/etc/init.d/ssh restart`
2. `/etc/init.d/php8.4-fpm restart`

## Füge das automatische Starten vopn ssh und PHP ein
1. `nano docker-entrypoint.sh`
2. ```
   `/etc/init.d/ssh restart`
   `/etc/init.d/php8.4-fpm restart`
``` 


