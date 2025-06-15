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







