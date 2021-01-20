# SElinux

# 1. Запустить nginx на нестандартном порту 3-мя разными способами:
- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.

Настрооим виртуалку
Установим все необходимые пакеты: nginx, net-tools, setools-console, policycoreutils-python
```ruby
[root@selinux vagrant]# sudo systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2021-01-20 13:11:51 UTC; 11s ago
  Process: 27230 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 27227 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 27226 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 27232 (nginx)
 ```
 Видим что nginx запущен смотрим на каком он порту
 ```ruby
 [root@selinux vagrant]# netstat -tulpn | grep nginx
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      27232/nginx: master
tcp6       0      0 :::80                   :::*                    LISTEN      27232/nginx: master
```
# переключатели setsebool
правим конфиг nginx, сменив порт на нестандартный
```ruby
[root@selinux vagrant]# cat /etc/nginx/nginx.conf
 include /etc/nginx/conf.d/*.conf;
    server {
        listen       3435 default_server;
        listen       [::]:3435 default_server;
        server_name  _;
        root         /usr/share/nginx/html;
```
Смотрим логи. т.к сервис не стартует после смены портов
```ruby
type=AVC msg=audit(1611149166.771:1554): avc:  denied  { name_bind } for  pid=27408 comm="nginx" src=3435 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

Was caused by:
The boolean nis_enabled was set incorrectly.
Description:
Allow nis to enabled

Allow access by executing:
# setsebool -P nis_enabled 1
                
```
Применим предложенное решение 
```ruby
[root@selinux vagrant]# setsebool -P nis_enabled 1
[root@selinux vagrant]# systemctl restart nginx
[root@selinux vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2021-01-20 13:33:40 UTC; 5s ago
  Process: 27431 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 27429 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 27428 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 27433 (nginx)
   CGroup: /system.slice/nginx.service
           ├─27433 nginx: master process /usr/sbin/nginx
           └─27434 nginx: worker process
```
как видим все работает
```ruby
[root@selinux vagrant]# netstat -tulpn | grep nginx
tcp        0      0 0.0.0.0:3435            0.0.0.0:*               LISTEN      27433/nginx: master
tcp6       0      0 :::3435                 :::*                    LISTEN      27433/nginx: master
```
 # добавление нестандартного порта в имеющийся тип
 Вернем setsebool -P nis_enabled 1 обратно в 0, снова сломав наш nginx на порту 3435
 ```ruby
 [root@selinux vagrant]# setsebool -P nis_enabled 0
 [root@selinux vagrant]# systemctl restart nginx
 [root@selinux vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Wed 2021-01-20 13:37:13 UTC; 2s ago
  Process: 27431 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 27454 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 27453 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 27433 (code=exited, status=0/SUCCESS)
 ```
 как видно выше всё опять сломалось. необходимо выяснить на каком порту можно держать веб сервер
 ```ruby
 [root@selinux vagrant]# semanage port -l | grep http_port
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
```
Видно что нашего порта 3435 нету в данном списке, занчит надо добавить
```ruby
[root@selinux vagrant]# semanage port -a -t http_port_t -p tcp 3435
```
проверяем, и как видно все гуд
```ruby
[root@selinux vagrant]# systemctl restart nginx
[root@selinux vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2021-01-20 13:41:56 UTC; 14s ago
  Process: 27494 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 27491 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 27490 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 27496 (nginx)
   CGroup: /system.slice/nginx.service
           ├─27496 nginx: master process /usr/sbin/nginx
           └─27497 nginx: worker process
[root@selinux vagrant]# semanage port -l | grep http_port
http_port_t                    tcp      3435, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
```
# формирование и установка модуля SELinux.
Откатим изменения
```ruby
[root@selinux vagrant]# semanage port -d -t http_port_t -p tcp 3435
```
Теперь посмотрим, какой модуль надо доустановить.
```ruby
[root@selinux vagrant]# whereis nginx
nginx: /usr/sbin/nginx /usr/lib64/nginx /etc/nginx /usr/share/nginx /usr/share/man/man3/nginx.3pm.gz /usr/share/man/man8/nginx.8.gz
[root@selinux vagrant]# ls -Z /usr/sbin/nginx
-rwxr-xr-x. root root system_u:object_r:httpd_exec_t:s0 /usr/sbin/nginx
[root@selinux vagrant]# audit2allow -M httpd_add --debug < /var/log/audit/audit.log
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i httpd_add.pp
[root@selinux vagrant]# systemctl restart nginx
[root@selinux vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2021-01-20 13:48:21 UTC; 1s ago

[root@selinux vagrant]# netstat -tulpn | grep nginx
tcp        0      0 0.0.0.0:3435            0.0.0.0:*               LISTEN      27549/nginx: master
tcp6       0      0 :::3435                 :::*                    LISTEN      27549/nginx: master

````
как видно наш nginx  прекрасно работает на нашем выбранном порту


