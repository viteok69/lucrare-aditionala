# Lucrare-aditionala

> Realizat de studentul: Badia Victor \
> Grupa: I2301
> \
> Verificat de Mihail Croitor

## Scopul Lucrării

Scopul acestei lucrări este de a învăța administrarea serverelor Web care funcționează în containere.
> În producție, crearea copiilor de rezervă este adesea realizată cu instrumente specializate, însă în această lucrare se analizează utilizarea managerului de sarcini cron.
Pentru a facilita colectarea jurnalelor, o practică standard este redirecționarea jurnalelor către fluxurile standard STDOUT și STDERR.

## Pregătire

> Verificați dacă aveți instalat și pornit Docker Desktop.
Creați folderul additional. În acesta se va desfășura întreaga lucrare. Creați folderele database - pentru baza de date, files - pentru stocarea configurațiilor și site - în acest folder va fi amplasat site-ul.
Lucrarea de laborator se desfășoară cu conexiune la rețeaua Internet, deoarece imaginile sunt descărcate din depozitul Docker Hub.
![image](https://github.com/user-attachments/assets/ef663858-15e7-4029-a29a-28da7d5c185a)

## Executare

Descărcați CMS Wordpress și dezarhivați-l în folderul site. În folderul site ar trebui să apară folderul wordpress cu codul sursă al site-ului.
![image](https://github.com/user-attachments/assets/acda81b9-64ab-490c-aaa6-d5760cc120a1)

## Container Apache HTTP Server

Pentru început, vom crea un fișier de configurare pentru Apache HTTP Server. Pentru aceasta, executați următoarele comenzi în consolă:

```bash
# comanda descarcă imaginea httpd și lansează un container pe baza acesteia cu numele httpd
docker run -d --name httpd  httpd:2.4

# copiem fișierul de configurare din container în folderul .\files\httpd
docker cp httpd:/usr/local/apache2/conf/httpd.conf .\files\httpd\httpd.conf

# oprim containerul httpd
docker stop httpd

# ștergem containerul
docker rm httpd
```
![image](https://github.com/user-attachments/assets/1ed67b8c-c767-49bd-a957-80c94a826c31)

![image](https://github.com/user-attachments/assets/e91b5e3c-ac2b-4e47-8154-714b015f6528)

![image](https://github.com/user-attachments/assets/72d38788-8a4f-44ef-859f-48afbace5106)

![image](https://github.com/user-attachments/assets/6e889aa2-bfff-44ec-a46c-45ff9c98fa2a)

În fișierul creat .\files\httpd\httpd.conf, de-comentați liniile care conțin extensiile mod_proxy.so, mod_proxy_http.so, mod_proxy_fcgi.so.
![image](https://github.com/user-attachments/assets/1b96c16c-a98e-4703-add5-c375f015a29c)

Găsiți în fișierul de configurare declarația parametrului ServerName. Sub acesta, adăugați următoarele linii:

```bash
# definirea numelui de domeniu al site-ului
ServerName wordpress.localhost:80
# redirecționarea cererilor php către containerul php-fpm
ProxyPassMatch ^/(.*\.php(/.*)?)$ fcgi://php-fpm:9000/var/www/html/$1
# fișierul index
DirectoryIndex /index.php index.php
```
![image](https://github.com/user-attachments/assets/fed2d514-ee2d-4ece-943d-70b2ad2d1567)

De asemenea, găsiți definiția parametrului DocumentRoot și setați-i valoarea /var/www/html, la fel ca în linia următoare parametrului.
![image](https://github.com/user-attachments/assets/b057711e-07de-443f-affd-b08919ff79b2)

Creați fișierul Dockerfile.httpd cu următorul conținut:

```bash
FROM httpd:2.4
RUN apt update && apt upgrade -y
COPY ./files/httpd/httpd.conf /usr/local/apache2/conf/httpd.conf
```
![image](https://github.com/user-attachments/assets/71c8ccc6-1aed-44d4-b579-3458b5175bc2)

## Container PHP-FPM

Creați fișierul Dockerfile.php-fpm cu următorul conținut:

```bash
FROM php:7.4-fpm

RUN apt-get update && apt-get upgrade -y && apt-get install -y \
    libfreetype6-dev \
    libjpeg62-turbo-dev \
    libpng-dev
RUN docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-configure pdo_mysql \
    && docker-php-ext-install -j$(nproc) gd mysqli
```
![image](https://github.com/user-attachments/assets/dd44f1c0-0d60-40b6-951c-c7d313ebe6f2)

## Container MariaDB

Creați fișierul Dockerfile.mariadb cu următorul conținut:

```bash
FROM mariadb:10.8
RUN apt-get update && apt-get upgrade -y
```
![image](https://github.com/user-attachments/assets/43743693-4a73-440a-aaa8-c86e16164db2)

## Construirea soluției

Creați fișierul docker-compose.yml cu următorul conținut:

```bash
version: '3.9'

services:
  httpd:
    build:
      context: ./
      dockerfile: Dockerfile.httpd
    networks:
      - internal
    ports:
      - "80:80"
    volumes:
      - "./site/wordpress/:/var/www/html/"
  php-fpm:
    build:
      context: ./
      dockerfile: Dockerfile.php-fpm
    networks:
      - internal
    volumes:
      - "./site/wordpress/:/var/www/html/"
  mariadb:
    build: 
      context: ./
      dockerfile: Dockerfile.mariadb
    networks:
      - internal
    environment:
     MARIADB_DATABASE: sample
     MARIADB_USER: sampleuser
     MARIADB_PASSWORD: samplepassword
     MARIADB_ROOT_PASSWORD: rootpassword
    volumes:
      - "./database/:/var/lib/mysql"
networks:
  internal: {}
```
![image](https://github.com/user-attachments/assets/3b69ddc8-f983-4eb9-aa6d-148741d2f105)

Acest fișier declară structura a trei containere: http ca punct de intrare, containerul php-fpm și containerul cu baza de date. Pentru interacțiunea containerelor se declară și rețeaua internal cu setările implicite.

## Serviciul Cron

Pentru realizarea copiilor de rezervă vom folosi containerul cron, care:
- la fiecare 24 de ore creează o copie de rezervă a bazei de date CMS;
- în fiecare luni creează o copie de rezervă a directorului CMS;
- la fiecare 24 de ore șterge copiile de rezervă care au fost create acum 30 de zile.
- În fiecare minut scrie în jurnal mesajul alive, \<username>.

Pentru aceasta, în folderul ./files/ creați folderul cron. În folderul ./files/cron/ creați folderul scripts. În directorul rădăcină creați folderul backups, iar în acesta mysql, site.

# mesaj de stare














