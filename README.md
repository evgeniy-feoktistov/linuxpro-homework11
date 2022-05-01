# linuxpro-homework11

## Домашнее задание
### Практика с SELinux

**Цель**:
Тренируем умение работать с SELinux: диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений, если это требуется.

**Описание/Пошаговая инструкция выполнения домашнего задания:**

1. **Запустить nginx на нестандартном порту 3-мя разными способами:**
- [переключатели setsebool;](#1)
- [добавление нестандартного порта в имеющийся тип;](#2)
- [формирование и установка модуля SELinux.](#3)

2. **Обеспечить работоспособность приложения при включенном selinux.**
- [развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;](#21)
- [выяснить причину неработоспособности механизма обновления зоны (см. README);](#22)
- [предложить решение (или решения) для данной проблемы;](#23)

***

И так, имеет: Запущенная VM с настроенным **nginx** на порт **4881**. Служба не стартанула, так как SELinux заблокировал работу на не стандартном порту.
```
selinux: Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
Заходим на сервер
```
$ vagrant ssh
[vagrant@selinux ~]$
```
Проверим, включен ли firewalld на сервере
```
[vagrant@selinux ~]$ systemctl status firewalld.service
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
```
Выключен.
Проверим корректность конфигурации **nginx** под **root**
```
[root@selinux vagrant]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
Проверим режим работы SELinux
```
[root@selinux vagrant]# getenforce
Enforcing
```
Данный режим показывает, что SELinux будет блокировать запрещенную активность.

## Запустить nginx на нестандартном порту 3-мя разными способами

<a name="1"/>
</a>

### Разрешим в SELinux работу **nginx** на порту TCP 4881 c помощью переключателей **setsebool**
Смотрим, что у нас в логе:
```
[root@selinux vagrant]# less /var/log/audit/audit.log
...
type=AVC msg=audit(1650548718.172:807): avc:  denied  { name_bind } for  pid=2826 comm="nginx" src=4881 scontext=system_u:system_r:htt$
type=SYSCALL msg=audit(1650548718.172:807): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=55c6a2235858 a2=10 a3=7ffc804c8ea0 it$
type=PROCTITLE msg=audit(1650548718.172:807): proctitle=2F7573722F7362696E2F6E67696E78002D74
...
```
Запоминаем таймштамп 1650548718.172:807 когда был **denied** и посмотрим, что покажет утилита audit2why
```
[root@selinux vagrant]# grep 1650548718.172:807 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1650548718.172:807): avc:  denied  { name_bind } for  pid=2826 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1
```
Утилита audit2why покажет почему трафик блокируется. Исходя из вывода утилиты, мы видим, что нам нужно поменять параметр nis_enabled.
Включим параметр nis_enabled и перезапустим nginx:
```
[root@selinux vagrant]# setsebool -P nis_enabled 1
[root@selinux vagrant]# systemctl restart nginx
[root@selinux vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2022-04-22 10:44:01 UTC; 6s ago
  Process: 1973 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 1971 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 1970 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 1975 (nginx)
   CGroup: /system.slice/nginx.service
           ├─1975 nginx: master process /usr/sbin/nginx
           └─1976 nginx: worker process

Apr 22 10:44:01 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Apr 22 10:44:01 selinux nginx[1971]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Apr 22 10:44:01 selinux nginx[1971]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Apr 22 10:44:01 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
nginx - работает. Можем проверить
```
[root@selinux vagrant]# curl http://127.0.0.1:4881
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
  <title>Welcome to CentOS</title>
  <style rel="stylesheet" type="text/css">

        html {
...
```
Проверить статус параметра можно с помощью команды: **getsebool -a | grep nis_enabled**
```
[root@selinux vagrant]# getsebool -a | grep nis_enabled
nis_enabled --> on
```
Выключим параметр, и проверим статус **nginx**
```
[root@selinux vagrant]# setsebool -P nis_enabled 0
[root@selinux vagrant]# systemctl restart nginx.service
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
Не запускается.
```
```

<a name="2"/>
</a>

### Теперь разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип:
Поиск имеющегося типа, для http трафика:
```
[root@selinux vagrant]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
Добавим порт в тип http_port_t:
```
[root@selinux vagrant]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
[root@selinux vagrant]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux vagrant]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
Теперь присутствует в списке.
Перезапустим **nginx** и проверим результат
```
[root@selinux vagrant]# systemctl restart nginx.service
[root@selinux vagrant]# systemctl status nginx.service
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2022-04-22 11:31:18 UTC; 5s ago
  Process: 2383 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2381 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 2380 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2385 (nginx)
   CGroup: /system.slice/nginx.service
           ├─2385 nginx: master process /usr/sbin/nginx
           └─2387 nginx: worker process

Apr 22 11:31:18 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Apr 22 11:31:18 selinux nginx[2381]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Apr 22 11:31:18 selinux nginx[2381]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Apr 22 11:31:18 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
Проверим что выдает **curl**
```
[root@selinux vagrant]# curl http://127.0.0.1:4881
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
  <title>Welcome to CentOS</title>
  <style rel="stylesheet" type="text/css">
  ...
```
Все работает.
Удалим нестандартный порт из имеющегося типа и перезапустим **nginx**
```
[root@selinux vagrant]# semanage port -d -t http_port_t -p tcp 4881
[root@selinux vagrant]# systemctl restart nginx.service
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
**nginx** опять не стартует.

<a name="3"/>
</a>

### Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux:

Посмотрим, что мы видим в логе, после неудачной попытке запуска **nginx**
```
[root@selinux vagrant]# grep nginx /var/log/audit/audit.log
...
type=SERVICE_START msg=audit(1650627236.082:571): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
type=AVC msg=audit(1650627372.166:572): avc:  denied  { name_bind } for  pid=2419 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1650627372.166:572): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=55b94f19a858 a2=10 a3=7fff459570b0 items=0 ppid=1 pid=2419 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1650627372.167:573): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
...
```
Воспользуемся утилитой **audit2allow** для того, чтобы на основе логов **SELinux** сделать модуль, разрешающий работу **nginx** на нестандартном порту:
```
[root@selinux vagrant]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
```
Audit2allow сформировал модуль, и сообщил нам команду, с помощью которой можно применить данный модуль. Выполним ее.
```
[root@selinux vagrant]# semodule -i nginx.pp
```
Проверим работу **nginx**
```
[root@selinux vagrant]# systemctl restart nginx.service
[root@selinux vagrant]# systemctl status nginx.service
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2022-04-22 11:40:19 UTC; 11s ago
  Process: 2461 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2459 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 2458 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2463 (nginx)
   CGroup: /system.slice/nginx.service
           ├─2463 nginx: master process /usr/sbin/nginx
           └─2465 nginx: worker process

Apr 22 11:40:19 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Apr 22 11:40:19 selinux nginx[2459]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Apr 22 11:40:19 selinux nginx[2459]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Apr 22 11:40:19 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
Как видим, nginx запустился без ошибок. При использовании модуля изменения сохранятся после перезагрузки.
Просмотр всех установленных модулей:
```
[root@selinux vagrant]# semodule -l
abrt    1.4.1
accountsd       1.1.0
acct    1.6.0
afs     1.9.0
...
[root@selinux vagrant]# semodule -l | grep nginx
nginx   1.0
```
Для удаления модуля воспользуемся командой:
```
[root@selinux vagrant]# semodule -r nginx
libsemanage.semanage_direct_remove_key: Removing last nginx module (no other nginx module exists at another priority).
```
Done.

## Обеспечить работоспособность приложения при включенном selinux.

<a name="21"/>
</a>

### Развернуть приложенный стенд

Выполним клонирование репозитория:
```
$ git clone https://github.com/mbfx/otus-linux-adm.git
Cloning into 'otus-linux-adm'...
remote: Enumerating objects: 542, done.
remote: Counting objects: 100% (440/440), done.
remote: Compressing objects: 100% (295/295), done.
remote: Total 542 (delta 118), reused 381 (delta 69), pack-reused 102
Receiving objects: 100% (542/542), 1.38 MiB | 1.30 MiB/s, done.
Resolving deltas: 100% (133/133), done.
```
Переходим в папку **selinux_dns_problems**
```
$ cd otus-linux-adm/selinux_dns_problems
```
Развернём 2 ВМ с помощью vagrant: **vagrant up**
```
selinux_dns_problems$ vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)
```
Подключимся к клиенту: **vagrant ssh client**
```
selinux_dns_problems$ vagrant ssh client
Last login: Fri Apr 22 13:01:24 2022 from 10.0.2.2
###############################
### Welcome to the DNS lab! ###
###############################

- Use this client to test the enviroment
- with dig or nslookup. Ex:
    dig @192.168.50.10 ns01.dns.lab

- nsupdate is available in the ddns.lab zone. Ex:
    nsupdate -k /etc/named.zonetransfer.key
    server 192.168.50.10
    zone ddns.lab
    update add www.ddns.lab. 60 A 192.168.50.15
    send

- rndc is also available to manage the servers
    rndc -c ~/rndc.conf reload

###############################
### Enjoy! ####################
###############################
[vagrant@client ~]$
```
Попробуем внести изменения в зону: **nsupdate -k /etc/named.zonetransfer.key**
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
```

<a name="22"/>
</a>

Изменения внести не получилось. Давайте посмотрим логи SELinux и попробуем понять в чём может быть проблема.
```
[vagrant@client ~]$ sudo -i
[root@client ~]# cat /var/log/audit/audit.log | audit2why
[root@client ~]#
```
Тут мы видим, что на клиенте отсутствуют ошибки. Не закрывая сессию на клиенте, подключимся к серверу ns01 и проверим логи SELinux:
```
selinux_dns_problems$ vagrant ssh ns01
Last login: Fri Apr 22 12:59:24 2022 from 10.0.2.2
[vagrant@ns01 ~]$ sudo -i
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1651156915.018:2964): avc:  denied  { create } for  pid=5133 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
```
В логах мы видим, что ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t. Проверим данную проблему в каталоге /etc/named
```
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```
Тут мы также видим, что контекст безопасности неправильный. Проблема заключается в том, что конфигурационные файлы лежат в другом каталоге. Посмотреть в каком каталоги  должны лежать, файлы, чтобы на них распространялись правильные политики SELinux можно с помощью команды:
```
[root@ns01 ~]# sudo semanage fcontext -l | grep named
/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0
...
```

<a name="23"/>
</a>

Изменим тип контекста безопасности для каталога /etc/named:
```
[root@ns01 ~]# sudo chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```
Попробуем снова внести изменения с клиента:
```
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
```
проверим, что отдает ДНС после внесения данных:
```
[root@client ~]# dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.9 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 30426
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.               3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Thu Apr 28 15:30:04 UTC 2022
;; MSG SIZE  rcvd: 96
```
Видим, что изменения применились. Попробуем перезагрузить хосты и ещё раз сделать запрос с помощью dig:
```
[vagrant@client ~]$ sudo -i
[root@client ~]# dig @192.168.50.10 www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.9 <<>> @192.168.50.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 8330
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.               3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Sun May 01 08:01:02 UTC 2022
;; MSG SIZE  rcvd: 96
```
Всё правильно. После перезагрузки настройки сохранились.
