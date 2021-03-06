# LB03
## Inhaltsverzeichnis

- [LB03](#lb03)
  - [Inhaltsverzeichnis](#inhaltsverzeichnis)
  - [Einleitung](#einleitung)
    - [Benötigte Vagrant-System-Änderungen](#benötigte-vagrant-system-änderungen)
  - [Realisierung](#realisierung)
    - [Vorbereitung](#vorbereitung)
      - [Wichtig!](#wichtig)
    - [docker-compose.yml](#docker-composeyml)
      - [MySQL](#mysql)
      - [PHP](#php)
      - [Apache](#apache)
      - [Redis](#redis)
      - [phpMyAdmin](#phpmyadmin)
    - [Apache-Konfiguration](#apache-konfiguration)
    - [MySQL-Konfiguration](#mysql-konfiguration)
    - [PHP-Konfiguration](#php-konfiguration)
    - [phpMyAdmin-Konfiguration](#phpmyadmin-konfiguration)
    - [Docker-Compose starten](#docker-compose-starten)
  - [Testen](#testen)
    - [Problem](#problem)
  - [Quellenangaben](#quellenangaben)

## Einleitung
Für diese LB möchte ich eine Docker-Compose erstellen, welche eine MySQL-Datenbank per MyPHP ins LAN verfügbar stellt. Als Grundlage für Docker nehme ich eine Kopie der VM, welche Herr Berger in seinem M300 Github zur Verfügung stellte.

### Benötigte Vagrant-System-Änderungen
Um die Komposition besser testen zu können, nehme ich die VM in mein LAN auf, was bedeutet, dass es die IP-Adresse *192.168.0.45 /24* bekommt. Vor der Abgabe werde ich dies wieder auf die verlangte Adresse *192.168.60.101 /24* ändern.
<br>

Ebenso füge ich diese Provision ins Vagrantfile ein, um meine Ordnerstruktur und Files immer verfügbar zu haben:
```shell
  # Shell Provision
  config.vm.provision "shell", inline: <<-SHELL 
  
   apt-get update
   apt-get install docker-compose
   apt-get install -y python-pip
   pip --upgrade pip
   pip install docker-compose

   mkdir compose-projekt
   mkdir compose-projekt/docker
   mkdir compose-projekt/docker/apache
   mkdir compose-projekt/docker/apache/certs
   mkdir compose-projekt/docker/mysql
   mkdir compose-projekt/docker/mysql/data
   mkdir compose-projekt/docker/php
   mkdir compose-projekt/docker/www
   sudo wget -P ./compose-projekt/docker https://raw.githubusercontent.com/Ben1702/M300_LB_Stutz/main/lb3/docker-compose.yml
   sudo wget -P ./compose-projekt/docker/apache https://raw.githubusercontent.com/Ben1702/M300_LB_Stutz/main/lb3/my_vhost.conf
   sudo wget -P ./compose-projekt/docker/www https://raw.githubusercontent.com/Ben1702/M300_LB_Stutz/main/lb3/info.php
   sudo wget -P ./compose-projekt/docker/php https://raw.githubusercontent.com/Ben1702/M300_LB_Stutz/main/lb3/php.ini

  SHELL
```

## Realisierung
### Vorbereitung
Als erstes erstelle ich auf der VM folgende Ordnerstruktur im Home-Ordner:
```shell
compose-projekt/docker
docker/apache/certs
docker/mysql/data
docker/php
docker/www
```
Wie unter den [Vagrant-System-Änderungen](#benötigte-vagrant-system-änderungen) bereits zu sehen war, wird mein im Repository bereits erstelltes docker-compose-File in den Docker-Ordner kopiert. Also so:
```shell
compose-projekt/docker/docker-compose.yml
```
<br>

#### Wichtig!
Um das Docker-Compose.yml auf der Version 3.7  auf der VM von Herrn Berger laufen lassen zu können, muss Docker-Compose Version 1.29.1 / die neueste Version installiert sein. Da der normale *apt-get*-Installer nur eine veraltete Verion installiert, mache ich nach der normalen *apt-get*-Installation folgendes:

```shell
sudo apt-get install python-pip
sudo pip --upgrade pip
sudo pip install docker-compose
```
Ich installiere zuerst den alternativ-Installer pip, aktualisiere ihn und installiere (eigentlich aktualisiere) danach damit Docker-Compose.

### docker-compose.yml
Alle folgenden Abschnitte werden in das Docker-Compose-File eingetragen. Das File operiert auf Version 3.7.

#### MySQL
Für den MYSQL-Container erstelle ich folgenden Eintrag unter services:
```docker
  mysql:
    container_name: "mysql"
    image: mysql:latest
    environment:
      - MYSQL_ROOT_PASSWORD=Welcome20
      - MYSQL_USER=admin
      - MYSQL_PASSWORD=Welcome20
    ports:
      - '3306:3306'
    volumes:
      - ./docker/mysql/data:/mysql/data
```
Dies erstellt ein MySQL-Container, welcher bereits mit den Admin-Anmeldedaten gefüttert wurde. Anmelden kann man sich mit dem User **Admin** und dem Passwort **Welcome20**.

#### PHP
Für PHP selbst besteht dieser Eintrag:
```docker
  php:  
    container_name: "php"
    image: bitnami/php-fpm:latest
    depends_on:
      - redis
    volumes:
      - ./docker/www:/app:delegated
      - ./docker/php/php.ini:/opt/php/etc/conf.d/php.ini:ro
```
Hierfür benutze ich ein leicht angepasstes php-image, *php-fpm*, vom alternativen Image-Anbieter Bitnami. FPM bietet nützliche Features für alle möglichen PHP-Seiten.
Ebenso hat es eine Dependenz an Redis, einem Datenstruktur-Dienst.

#### Apache
```docker
apache:
    container_name: "apache"
    image: httpd:latest
    ports:
      - '81:81'
      - '8443:8443'
    depends_on:
      - php
    volumes:
      - ./docker/www:/app:delegated
      - ./docker/apache/my_vhost.conf:/vhosts/myapp.conf:ro
      - ./docker/apache/certs:/certs
```
Apache wird recht standardmässig implementiert, einfach mit der Dependenz auf PHP und leicht veränderten Ports. <br>
"my_vhost.conf" wird als Konfigurationsfile für Apache dienen.

#### Redis
```docker
redis:
    container_name: "redis"
    image: redis:latest
    environment:
      - REDIS_PASSWORD: Welcome20
```
Redis ist wie bereits in PHP gesagt ein Datenstruktur-Dienst für Server. Dieser wird äusserst Standardmässig aufgebaut, nur mit einem Passwort versehen.

#### phpMyAdmin
```docker
phpmyadmin:
    container_name: "phpmyadmin"
    image: phpmyadmin:latest
    depends_on:
      - mysql
    ports:
      - '80:80'
      - '443:443'
    environment:
      - DATABASE_HOST=host.docker.internal
```
Auch phpMyAdmin wird ziemlich dem Standard getreu aufgebaut, nur mit einer Dependenz auf MySQL.

Das Docker-Compose-File kann nun geschlossen und gespeichert werden.
### Apache-Konfiguration
Als nächstes muss Apache konfiguriert werden. Dafür wird als erstes ein SSL-Zertifikat benötigt, welches durch diesen Command erstellt werden kann:
```shell
openssl req -x509 -newkey rsa:4096 -sha256 -nodes -keyout server.key -out server.crt -subj "/CN=dstack.local" -days 3650
```
Das Zertifikat wird am besten gleich im Ordner *docker/apache/certs* erstellt, da es dort abgespeichert werden soll.

Nun an die eigentliche Konfig. Dafür wird unter *docker/apache* das File **my_vhost.conf** erstellt, mit folgendem Inhalt:
```apache
<VirtualHost *:8080>
  DocumentRoot "/app"
  ProxyPassMatch ^/(.*\.php(/.*)?)$ fcgi://php:9000/app/$1
  <Directory "/app">
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
    DirectoryIndex index.html index.php
  </Directory>
</VirtualHost>

<VirtualHost *:8443>
  SSLEngine on  
  SSLCipherSuite ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP:+eNULL  
  SSLCertificateFile "/certs/server.crt"  
  SSLCertificateKeyFile "/certs/server.key"  

  DocumentRoot "/app"
  ProxyPassMatch ^/(.*\.php(/.*)?)$ fcgi://php:9000/app/$1
  <Directory "/app">
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
    DirectoryIndex index.html index.php
  </Directory>
</VirtualHost>
```
Dies erstellt die Konfigurationen für beide supporteten Ports.
Diese Konfig-datei ist ebenfalls hier im Repository zu finden.

Als letztes für Apache wird noch im Ordner *docker/www* die Datei **info.php** erstellt, welche folgenden Inhalt hat:
```php
<?php
phpinfo();
?>
```
Datei ebenfalls im Repository vorhanden.
### MySQL-Konfiguration
MySQL benötigt nur den bereits erstellten, leeren Ordner *docker/mysql/data*. Den Rest übernimmt das Compose-File.

### PHP-Konfiguration
Als PHP-Konfig wird im Ordner *docker/php* die Datei **php.ini** mit diesem Inhalt erstellt:
```php
display_errors = On
expose_php = off

max_execution_time = 360
max_input_time = 360
memory_limit = 256M
upload_max_filesize = 1G
post_max_size = 1G

opcache.enable = 1
opcache.revalidate_freq = 2
opcache.validate_timestamps = 1
opcache.interned_strings_buffer = 32
opcache.memory_consumption = 256

extension=imagick.so
zend_extension = "/opt/bitnami/php/lib/php/extensions/xdebug.so"

[Xdebug]
xdebug.remote_autostart=1
xdebug.remote_enable=1
xdebug.default_enable=0
xdebug.remote_host=host.docker.internal
xdebug.remote_port=9000
xdebug.remote_connect_back=0
xdebug.profiler_enable=0
xdebug.remote_log="/tmp/xdebug.log"
```
Diese Konfig bestimmt Limitationen des Servers und Standorten von benötigten Dateien.<br>
Auch diese Konfig ist im Repository zu finden.

### phpMyAdmin-Konfiguration
phpMmyAdmin benötigt ebenfalls keine zusätzliche Konfiguration und sollte nach Aufstarten der Container auf *[IP-Adresse]:80* / *[IP-Adresse]:443* erreichbar sein.

### Docker-Compose starten
Um die Komposition zu starten, muss folgender Command eingegeben werden:
```shell
docker-compose up -d
```
Stimmt für das Compose-Programm alles im File, sollte folgender Output kommen:
![Docker-Compose-ouptut](Dokumentation-data/docker-compose1.png)

## Testen
Um auf phpMyAdmin zu kommen, muss ich nur die IP-Adresse der VM in einen Browser eingeben:

![phpMyAdmin-Login](Dokumentation-data/phpMyAdmin1.png)
Das Compose-File funktioniert also schon mal wie gewollt.
### Problem
Nun bin ich auf ein Problem gestossen:
Wenn ich versuche, mich in PHPMyAdmin anzumelden, kommen folgende Fehlermeldungen:

![phpMyAdmin-Fehler](Dokumentation-data/phpmyadmin-fehler.jpeg)

Ich habe ein paar Fälle im Internet gefunden, aber keiner, welcher mir helfen konnte. Leider ist mir die Projektzeit ausgegangen, bevor ich dies lösen konnte.

## Quellenangaben
Die Scripts und die generelle Vorgehensweise wurde grösstenteils von dieser Seite inspiriert:

[https://marketmix.com/de/docker-apache-mysql-php-und-phpmyadmin-im-container-verbund/](https://marketmix.com/de/docker-apache-mysql-php-und-phpmyadmin-im-container-verbund/)
