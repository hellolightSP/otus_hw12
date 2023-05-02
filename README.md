**Домашнее задание**

Практика с SELinux

**Цель**

Тренируем умение работать с SELinux: диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений, если это требуется.

**Что нужно сделать**

1. Запустить nginx на нестандартном порту 3-мя разными способами:
- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.

2. Обеспечить работоспособность приложения при включенном selinux.
- развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
- выяснить причину неработоспособности механизма обновления зоны (см. README);
- предложить решение (или решения) для данной проблемы;
- выбрать одно из решений для реализации, предварительно обосновав выбор;
- реализовать выбранное решение и продемонстрировать его работоспособность.

**Выполнение**

**Все команды, выполненные в консоли, записаны утилитой script [otus_hw12_p1](https://github.com/hellolightSP/otus_hw12/blob/main/otus_hw12_p1)**

**1. Запуск nginx на нестандартном порту 3-мя разными способами**

Проверим, что в ОС отключен файервол: systemctl status firewalld
```
[root@selinux ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
```

Проверим что конфигурация nginx настроена без ошибок: nginx -t
```
[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Проверим режим работы SELinux: getenforce 
```
[root@selinux ~]# getenforce
 Enforcing
```

Данный режим означает, что SELinux будет блокировать запрещенную активность.
Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool
Находим в логах (/var/log/audit/audit.log) информацию о блокировании порта
```
[root@selinux ~]# cat /var/log/audit/audit.log | grep 4881
type=AVC msg=audit(1683030425.762:785): avc:  denied  { name_bind } for  pid=2729 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```

Копируем время, в которое был записан этот лог, и, с помощью утилиты audit2why
```
[root@selinux ~]# yum -y install policycoreutils-python

[root@selinux ~]# grep 1683030425.762:785 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1683030425.762:785): avc:  denied  { name_bind } for  pid=2729 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1

        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled


        Allow access by executing:
        # setsebool -P nis_enabled 1
```

Утилита audit2why покажет почему трафик блокируется. Исходя из вывода утилиты, мы видим, что нам нужно поменять параметр nis_enabled. 
Включим параметр nis_enabled и перезапустим nginx: setsebool -P nis_enabled on
```
[root@selinux ~]# setsebool -P nis_enabled on
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2023-05-02 12:34:42 UTC; 5s ago
  Process: 2894 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2892 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 2891 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2896 (nginx)
   CGroup: /system.slice/nginx.service
           ├─2896 nginx: master process /usr/sbin/nginx
           └─2898 nginx: worker process
``` 

Также можно проверить работу nginx из браузера. Заходим в любой браузер на хосте и переходим по адресу http://127.0.0.1:4881

Проверить статус параметра можно с помощью команды: getsebool -a | grep nis_enabled
```
[root@selinux ~]# getsebool -a | grep nis_enabled
nis_enabled --> on
```

Вернём запрет работы nginx на порту 4881 обратно. Для этого отключим nis_enabled: setsebool -P nis_enabled off
После отключения nis_enabled служба nginx снова не запустится.

**Теперь разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип:**

Поиск имеющегося типа, для http трафика: semanage port -l | grep http
```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

Добавим порт в тип http_port_t: semanage port -a -t http_port_t -p tcp 4881
```
[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep  http_port_t
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
```

Теперь перезапустим службу nginx и проверим её работу: systemctl restart nginx
```
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2023-05-02 12:39:17 UTC; 4s ago
  Process: 2932 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2930 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 2929 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2934 (nginx)
   CGroup: /system.slice/nginx.service
           ├─2934 nginx: master process /usr/sbin/nginx
           └─2935 nginx: worker process
```

Также можно проверить работу nginx из браузера. Заходим в любой браузер на хосте и переходим по адресу http://127.0.0.1:4881


Удалить нестандартный порт из имеющегося типа можно с помощью команды: semanage port -d -t http_port_t -p tcp 4881
```
[root@selinux ~]# semanage port -d -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep  http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.

[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Tue 2023-05-02 12:41:06 UTC; 29s ago
...
```

**Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux:**

Попробуем снова запустить nginx: systemctl start nginx
```
[root@selinux ~]# systemctl start nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```

Nginx не запуститься, так как SELinux продолжает его блокировать. Посмотрим логи SELinux, которые относятся к nginx: 
```
[root@selinux ~]# grep nginx /var/log/audit/audit.log
...
type=AVC msg=audit(1683031266.277:857): avc:  denied  { name_bind } for  pid=2955 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1683031266.277:857): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=55cbbc887878 a2=10 a3=7fff3ef2c7c0 items=0 ppid=1 pid=2955 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1683031266.277:858): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
```

Воспользуемся утилитой audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту: 
grep nginx /var/log/audit/audit.log | audit2allow -M nginx

```
[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
```

Audit2allow сформировал модуль, и сообщил нам команду, с помощью которой можно применить данный модуль: semodule -i nginx.pp

```
[root@selinux ~]# semodule -i nginx.pp
```

Попробуем снова запустить nginx: systemctl start nginx
```
[root@selinux ~]# systemctl start nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2023-05-02 12:44:03 UTC; 3s ago
  Process: 2984 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2982 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 2981 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2986 (nginx)
   CGroup: /system.slice/nginx.service
           ├─2986 nginx: master process /usr/sbin/nginx
           └─2987 nginx: worker process
```

После добавления модуля nginx запустился без ошибок. При использовании модуля изменения сохранятся после перезагрузки. 

Просмотр всех установленных модулей: semodule -l
Для удаления модуля воспользуемся командой: semodule -r nginx

```
[root@selinux ~]# semodule -r nginx
libsemanage.semanage_direct_remove_key: Removing last nginx module (no other nginx module exists at another priority).
```

**2.Обеспечение работоспособности приложения при включенном SELinux**


Выполним клонирование репозитория: git clone https://github.com/mbfx/otus-linux-adm.git
```
git clone https://github.com/mbfx/otus-linux-adm.git
Cloning into 'otus-linux-adm'...
remote: Enumerating objects: 542, done.
remote: Counting objects: 100% (440/440), done.
remote: Compressing objects: 100% (295/295), done.
remote: Total 542 (delta 118), reused 381 (delta 69), pack-reused 102
Receiving objects: 100% (542/542), 1.38 MiB | 3.65 MiB/s, done.
Resolving deltas: 100% (133/133), done.
```
Перейдём в каталог со стендом: cd otus-linux-adm/selinux_dns_problems

Развернём 2 ВМ с помощью vagrant: vagrant up

После того, как стенд развернется, проверим ВМ с помощью команды: vagrant status
```
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
```

Подключимся к клиенту: vagrant ssh client
Попробуем внести изменения в зону: nsupdate -k /etc/named.zonetransfer.key
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.56.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.56.15
> send
update failed: SERVFAIL
> quit
```

Изменения внести не получилось. Давайте посмотрим логи SELinux, чтобы понять в чём может быть проблема.
Для этого воспользуемся утилитой audit2why: cat /var/log/audit/audit.log | audit2why
```
[vagrant@client ~]$ sudo -i
[root@client ~]# cat /var/log/audit/audit.log | audit2why
[root@client ~]# 
```
Тут мы видим, что на клиенте отсутствуют ошибки. 

Не закрывая сессию на клиенте, подключимся к серверу ns01 и проверим логи SELinux:
```
➜ selinux_dns_problems (master) ✔ vagrant ssh ns01 
Last login: Tue Nov 16 09:58:37 2021 from 10.0.2.2
[vagrant@ns01 ~]$ sudo -i 
[root@ns01 ~]# 
[root@ns01 ~]# 
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1683032642.511:1908): avc:  denied  { create } for  pid=5154 comm="isc-worker0000" name="named.ddns.lab.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.


    Was caused by:
        Missing type enforcement (TE) allow rule.


        You can use audit2allow to generate a loadable module to allow this access.
```

В логах мы видим, что ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t.
Проверим данную проблему в каталоге /etc/named:
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
Тут мы также видим, что контекст безопасности неправильный. Проблема заключается в том, что конфигурационные файлы лежат в другом каталоге. Посмотреть в каком каталоги должны лежать, файлы, чтобы на них распространялись правильные политики SELinux можно с помощью команды: sudo semanage fcontext -l | grep named
```
[root@ns01 ~]# sudo semanage fcontext -l | grep named
/etc/rndc.*              regular file       system_u:object_r:named_conf_t:s0 
/var/named(/.*)?         all files          system_u:object_r:named_zone_t:s0 
...
```

Изменим тип контекста безопасности для каталога /etc/named: sudo chcon -R -t named_zone_t /etc/named
```
[root@ns01 ~]# sudo chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# 
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
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.56.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.56.15
> send
> quit 
[vagrant@client ~]$ 
[vagrant@client ~]$ dig @192.168.56.10 www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.13 <<>> @192.168.56.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 63187
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	192.168.56.15

;; AUTHORITY SECTION:
ddns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10

;; Query time: 0 msec
;; SERVER: 192.168.56.10#53(192.168.56.10)
;; WHEN: Tue May 02 13:06:39 UTC 2023
;; MSG SIZE  rcvd: 96
```

Видим, что изменения применились. Попробуем перезагрузить хосты и ещё раз сделать запрос с помощью dig: 
```
[vagrant@client ~]$ dig @192.168.56.10 www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.13 <<>> @192.168.56.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 49736
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	192.168.56.15

;; AUTHORITY SECTION:
ddns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10

;; Query time: 1 msec
;; SERVER: 192.168.56.10#53(192.168.56.10)
;; WHEN: Tue May 02 13:08:12 UTC 2023
;; MSG SIZE  rcvd: 96
```

Всё правильно. После перезагрузки настройки сохранились. 
Для того, чтобы вернуть правила обратно, можно ввести команду: restorecon -v -R /etc/named

```
[root@ns01 ~]# restorecon -v -R /etc/named
restorecon reset /etc/named context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.dns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.dns.lab.view1 context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic context unconfined_u:object_r:named_zone_t:s0->unconfined_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1 context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.jnl context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.newdns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.50.168.192.rev context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
```
