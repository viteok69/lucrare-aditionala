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

În folderul ./files/cron/scripts/ creați fișierul 01_alive.sh cu următorul conținut:

```bash
#!/bin/sh
echo "alive ${USERNAME}" > /proc/1/fd/1
```
![image](https://github.com/user-attachments/assets/e3c74277-ea3c-45c3-9e7e-51eca7457d6c)

Acest script afișează mesajul alive ${USERNAME}.

# copierea de rezervă a site-ului

În folderul ./files/cron/scripts/ creați fișierul 02_backupsite.sh cu următorul conținut:

```bash
#!/bin/sh

echo "[backup] create site backup" \
    > /proc/1/fd/1 \
    2> /proc/1/fd/2
tar czfv /var/backups/site/www_$(date +\%Y\%m\%d).tar.gz /var/www/html
echo "[backup] site backup done" \
    >/proc/1/fd/1 \
    2> /proc/1/fd/2
```
![image](https://github.com/user-attachments/assets/356860ca-d6d0-411e-a3c3-5b6a5d0040b1)

Acest script arhivează folderul /var/www/html și salvează arhiva în /var/backups/site/.

# copierea de rezervă a bazei de date

În folderul ./files/cron/scripts/ creați fișierul 03_mysqldump.sh cu următorul conținut:

```bash
#!/bin/sh

echo "[backup] create mysql dump of ${MARIADB_DATABASE} database" \
    > /proc/1/fd/1
mysqldump -u ${MARIADB_USER} --password=${MARIADB_PASSWORD} -v -h mariadb ${MARIADB_DATABASE} \
    | gzip -c > /var/backups/mysql/${MARIADB_DATABASE}_$(date +\%F_\%T).sql.gz 2> /proc/1/fd/1
echo "[backup] sql dump created" \
    > /proc/1/fd/1
```
![image](https://github.com/user-attachments/assets/e071d7e6-06f7-4c7b-ae2d-ce868065f179)

# ștergerea fișierelor vechi

În folderul ./files/cron/scripts/ creați fișierul 04_clean.sh cu următorul conținut:

```bash
#!/bin/sh

echo "[backup] remove old backups" \
    > /proc/1/fd/1 \
    2> /proc/1/fd/2
find /var/backups/mysql -type f -mtime +30 -delete \
    > /proc/1/fd/1 \
    2> /proc/1/fd/2
find /var/backups/site -type f -mtime +30 -delete \
    > /proc/1/fd/1 \
    2> /proc/1/fd/2
echo "[backup] done" \
    > /proc/1/fd/1 \
    2> /proc/1/fd/2
```
![image](https://github.com/user-attachments/assets/bdffe683-4db5-4900-a597-26210f26a68f)

# pregătirea cron

În folderul ./files/cron/scripts/ creați fișierul environment.sh cu următorul conținut:

```bash
#!/bin/sh

env >> /etc/environment

# execute CMD
echo "Start cron" >/proc/1/fd/1 2>/proc/1/fd/2
echo "$@"
exec "$@"
```
![image](https://github.com/user-attachments/assets/e6fd120e-4b00-44d9-a1f1-5deafb410ba2)

În folderul ./files/cron/ creați fișierul crontab cu următorul conținut:

```bash
# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
  *  *  *  *  * /scripts/01_alive.sh > /dev/null
  *  *  *  *  * /scripts/02_backupsite.sh > /dev/null
  *  *  *  *  * /scripts/03_mysqldump.sh > /dev/null
  *  *  *  *  * /scripts/04_clean.sh > /dev/null
# Don't remove the empty line at the end of this file. It is required to run the cron job
```
![image](https://github.com/user-attachments/assets/8b8dada8-d7c0-436f-8b62-4fafac924082)

#crearea containerului cron

Creați în directorul rădăcină fișierul Dockerfile.cron cu următorul conținut:

```bash
FROM debian:latest

RUN apt update && apt -y upgrade && apt install -y cron mariadb-client

COPY ./files/cron/crontab /etc/cron.d/crontab
COPY ./files/cron/scripts/ /scripts/

RUN crontab /etc/cron.d/crontab

ENTRYPOINT [ "/scripts/environment.sh" ]
CMD [ "cron", "-f" ]
```
![image](https://github.com/user-attachments/assets/76e94008-a944-4d8d-b9fe-be3686cae74c)

Editați fișierul docker-compose.yml, adăugând după definiția serviciului mariadb următoarele linii:

```bash
  cron:
    build:
      context: ./
      dockerfile: Dockerfile.cron
    environment:
      USERNAME: <nume prenume>
      MARIADB_DATABASE: sample
      MARIADB_USER: sampleuser
      MARIADB_PASSWORD: samplepassword
    volumes:
      - "./backups/:/var/backups/"
      - "./site/wordpress/:/var/www/html/"
    networks:
      - internal
```
Înlocuiți <nume prenume> cu numele și prenumele dvs.

![image](https://github.com/user-attachments/assets/9a62880c-46e2-4ada-bafa-27c1a99deef7)



## Lansare și testare

În folderul lucrării de laborator, deschideți consola și executați comanda:

```bash
docker-compose build
```
![image](https://github.com/user-attachments/assets/6157b56f-e2a2-4138-a7a0-8b72e1f1b9ff)

![image](https://github.com/user-attachments/assets/8a739b52-4749-4e90-b54c-64da04f502be)

Pe baza definițiilor create, docker va construi imaginile serviciilor. Câte secunde a durat construirea proiectului?

> Construirea proiectului a durat 62.1 secunde.

Executați comanda:

```bash
docker-compose up -d
```
![image](https://github.com/user-attachments/assets/f1bbfc4e-be8e-4e1a-86e5-143e5d60a745)

Pe baza imaginilor, se vor lansa containerele. Deschideți în browser pagina: http://wordpress.localhost și realizați instalarea site-ului. Rețineți că containerele se văd între ele după nume, astfel încât, la instalarea site-ului, trebuie să specificați pentru serverul bazei de date numele gazdei egal cu numele containerului, adică mariadb. Numele utilizatorului bazei de date, parola acestuia și numele bazei de date le puteți lua din fișierul docker-compose.yml.
![image](https://github.com/user-attachments/assets/c7484ecb-d9c7-4295-a7d1-0bc1e19feee3)

Citiți jurnalele fiecărui container. Pentru aceasta, executați comanda:

```bash
docker logs <numele containerului>
```

```bash
docker logs additional-cron-1
```
![image](https://github.com/user-attachments/assets/88a2d0ef-67ef-47e4-a5aa-42abc277932f)

```bash
docker logs additional-httpd-1
```
![image](https://github.com/user-attachments/assets/ec0b0b9a-f402-4571-877a-51de7f16f17d)

```bash
docker logs additional-php-fpm-1
```
![image](https://github.com/user-attachments/assets/52aea036-6bab-4738-93cb-c2eec893098b)

```bash
docker logs additional-mariadb-1
```
![image](https://github.com/user-attachments/assets/dd8ba8bf-9d60-4c02-aba0-1f625496f070)

Executați următoarele comenzi în ordine:
```bash
# opriți containerele
docker-compose down
# ștergeți containerele
docker-compose rm
```
![image](https://github.com/user-attachments/assets/76a61086-d2eb-4e48-9b76-8da1f5209f20)

Verificați dacă site-ul http://wordpress.localhost se deschide. Porniți din nou clusterul de containere:

```bash
docker-compose up -d
```
![image](https://github.com/user-attachments/assets/7364351d-8331-495e-ace2-8023981bbc92)

și verificați funcționalitatea site-ului.
![image](https://github.com/user-attachments/assets/29f1fdd9-00ec-4197-b521-2b519adc78a5)

Așteptați 2-3 minute și verificați ce se află în directoarele ./backups/mysql/ și ./backups/site/.
![image](https://github.com/user-attachments/assets/e9e285cf-e54e-4718-812e-47b81df37d2e)

![image](https://github.com/user-attachments/assets/6a3febe5-808a-4691-b86d-6a940cd87c0c)

Opriți containerele și modificați fișierul ./files/cron/crontab astfel încât:
- în fiecare zi la ora 1:00 să se creeze o copie de rezervă a bazei de date CMS (sql dump);
- în fiecare luni să se creeze o copie de rezervă a directorului CMS;
- în fiecare zi la ora 2:00 să se șteargă copiile de rezervă care au fost create acum 30 de zile.
![image](https://github.com/user-attachments/assets/b6b5af5f-4a14-4cee-98c6-d0b6ced533a9)

![image](https://github.com/user-attachments/assets/443af084-c4f7-4b7d-aaf1-f087851bb5a9)

Răspundeți la întrebări:

- De ce este necesar să se creeze un utilizator al sistemului pentru fiecare site?

Securitate: Izolează fiecare site. Dacă unul e compromis, celelalte sunt protejate. Atacatorul nu obține acces la tot serverul.
Privilegiu Minim: Fiecare site are doar permisiunile strict necesare.
  
- În ce cazuri serverul Web trebuie să aibă acces complet la directoarele (folderul) site-ului?

Niciodata "acces complet" (rwx) la tot site-ul.
Serverul web (Apache/Nginx) are nevoie doar de citire pentru majoritatea fișierelor.
Permisiuni de scriere sunt necesare doar pentru directoare specifice unde aplicația trebuie să scrie (ex: upload-uri, cache, loguri). Acestea sunt gestionate de procesele aplicației (PHP-FPM), nu direct de serverul web.
  
- Ce înseamnă comanda chmod -R 0755 /home/www/anydir?

chmod: Schimbă permisiunile.
-R: Schimbă recursiv (și subdirectoarele/fișierele).
0755: Setează permisiunile:
Proprietar (0755 = 7): Citire, Scriere, Execuție (rwx)
Grup (0755 = 5): Citire, Execuție (r-x)
Alții (0755 = 5): Citire, Execuție (r-x)
/home/www/anydir: Directorul țintă.

- În scripturile shell, fiecare comandă se termină cu linia > /proc/1/fd/1. Ce înseamnă aceasta?

Redirecționarea Output-ului: Aceasta ia ieșirea standard a unei comenzi (ceea ce ar fi afișat pe ecran) și o trimite către o destinație specifică.
/proc/1/fd/1: În context Docker, /proc/1/fd/1 reprezintă fluxul standard de ieșire (STDOUT) al procesului principal al containerului (care rulează ca PID 1).
Concluzie: Toate mesajele și erorile generate de scripturile tale shell sunt redirectionate către STDOUT-ul containerului, făcându-le vizibile prin docker logs <nume_container>. Aceasta este o practică standard pentru a colecta jurnalele din containere.
































