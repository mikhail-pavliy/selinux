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
# 2. Обеспечить работоспособность приложения при включенном selinux.
- Развернуть приложенный стенд
https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems
- Выяснить причину неработоспособности механизма обновления зоны (см. README);
- Предложить решение (или решения) для данной проблемы;
- Выбрать одно из решений для реализации, предварительно обосновав выбор;
- Реализовать выбранное решение и продемонстрировать его работоспособность.
1. Запустим приложенный по ссылке стенд, дождемся конфигурации обеих машин
2. Зачистим на обеих машинах /var/log/audit/audit.log
```ruby 
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
[root@client ~]# audit2why < /var/log/audit/audit.log
Nothing to do
```
Вроде со стороны клента исправлять ничего не надо, глянем наш сервер
```ruby
[root@ns01 vagrant]# audit2why < /var/log/audit/audit.log
type=AVC msg=audit(1611393220.125:2438): avc:  denied  { create } for  pid=28826 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
```
Как видим, доступ заблокирован, в соответствии с отсутствием правила Type Enforcement. Также посмотрим вывод утилиты sealert:
```ruby
[root@ns01 vagrant]# sealert -a /var/log/audit/audit.log
100% done
found 1 alerts in /var/log/audit/audit.log
--------------------------------------------------------------------------------

SELinux is preventing /usr/sbin/named from create access on the file named.ddns.lab.view1.jnl.

*****  Plugin catchall_labels (83.8 confidence) suggests   *******************

If you want to allow named to have create access on the named.ddns.lab.view1.jnl file
Then you need to change the label on named.ddns.lab.view1.jnl
Do
# semanage fcontext -a -t FILE_TYPE 'named.ddns.lab.view1.jnl'
where FILE_TYPE is one of the following: dnssec_trigger_var_run_t, ipa_var_lib_t, krb5_host_rcache_t, krb5_keytab_t, named_cache_t, named_log_t, named_tmp_t, named_var_run_t, named_zone_t.
Then execute:
restorecon -v 'named.ddns.lab.view1.jnl'


*****  Plugin catchall (17.1 confidence) suggests   **************************

If you believe that named should be allowed create access on the named.ddns.lab.view1.jnl file by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000
# semodule -i my-iscworker0000.pp
```
как видно выше SElinux запрещает доступ /usr/sbin/named на создание файла named.ddns.lab.view1.jnl. и предлагает два решения проблемы.
Воспользуемся вторым споособом т.к. создание можулей через audit2allow черввато для ИБ

Из файла /etc/named.conf получим распололжение файла зоны ddns.lab
```ruby
 [root@ns01 vagrant]# cat /etc/named.conf
  // labs ddns zone
    zone "ddns.lab" {
        type master;
        allow-transfer { key "zonetransfer.key"; };
        allow-update { key "zonetransfer.key"; };
        file "/etc/named/dynamic/named.ddns.lab.view1";
````
как видно файл распаложен file "/etc/named/dynamic/named.ddns.lab.view1"
Расммотрим его в контектсте безопасности
```ruby
[root@ns01 vagrant]# ll -Z /etc/named/dynamic/named.ddns.lab.view1
-rw-rw----. named named system_u:object_r:etc_t:s0       /etc/named/dynamic/named.ddns.lab.view1
```
Видим, что тип etc_t, необходимо поменять на named_cache_t согласно документации RedHat
```ruby
[root@ns01 vagrant]# semanage fcontext -a -t named_cache_t '/etc/named/dynamic(/.*)?'
[root@ns01 vagrant]# restorecon -R -v /etc/named/dynamic/
restorecon reset /etc/named/dynamic context unconfined_u:object_r:etc_t:s0->unconfined_u:object_r:named_cache_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab context system_u:object_r:etc_t:s0->system_u:object_r:named_cache_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1 context system_u:object_r:etc_t:s0->system_u:object_r:named_cache_t:s0
```
Переключаемся на нашего клиента и проверям, снова сменив зону
```ruby
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
>
> quit
```

 
