#### 1. Запустить nginx на нестандартном порту 3-мя разными способами:
1. Переключатели setsebool. Делается командой
```
setsebool -P nis_enabled 1
```
Пример: 
```
[root@packages ~]# sed -i 's/80/5080/g' /etc/nginx/conf.d/default.conf
[root@packages ~]# service nginx restart
Redirecting to /bin/systemctl restart nginx.service
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@packages ~]# setsebool -P nis_enabled 1
[root@packages ~]# service nginx start
Redirecting to /bin/systemctl start nginx.service
[root@packages ~]# service nginx status
Redirecting to /bin/systemctl status nginx.service
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2021-01-13 10:27:34 UTC; 3s ago
     Docs: http://nginx.org/en/docs/
  Process: 1090 ExecStop=/bin/kill -s TERM $MAINPID (code=exited, status=0/SUCCESS)
  Process: 1147 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf (code=exited, status=0/SUCCESS)
 Main PID: 1148 (nginx)
   CGroup: /system.slice/nginx.service
           ├─1148 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
           └─1149 nginx: worker process

Jan 13 10:27:34 packages systemd[1]: Starting nginx - high performance web server...
Jan 13 10:27:34 packages systemd[1]: Can't open PID file /var/run/nginx.pid (yet?) after start: No such file or...ectory
Jan 13 10:27:34 packages systemd[1]: Started nginx - high performance web server.
Hint: Some lines were ellipsized, use -l to show in full.
```

2. Добавление нестандартного порта в имеющийся тип. Делается командой 
```
semanage port -a -t http_port_t -p tcp 5080
```
Пример: 
```
[root@packages ~]# setsebool -P nis_enabled 0
[root@packages ~]# service nginx restart
Redirecting to /bin/systemctl restart nginx.service
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@packages ~]# semanage port -a -t http_port_t -p tcp 5080
[root@packages ~]# service nginx start
Redirecting to /bin/systemctl start nginx.service
[root@packages ~]# service nginx status
Redirecting to /bin/systemctl status nginx.service
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2021-01-13 10:32:04 UTC; 2s ago
     Docs: http://nginx.org/en/docs/
  Process: 1182 ExecStop=/bin/kill -s TERM $MAINPID (code=exited, status=0/SUCCESS)
  Process: 1271 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf (code=exited, status=0/SUCCESS)
 Main PID: 1272 (nginx)
   CGroup: /system.slice/nginx.service
           ├─1272 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
           └─1273 nginx: worker process

Jan 13 10:32:04 packages systemd[1]: Starting nginx - high performance web server...
Jan 13 10:32:04 packages systemd[1]: Can't open PID file /var/run/nginx.pid (yet?) after start: No such file or...ectory
Jan 13 10:32:04 packages systemd[1]: Started nginx - high performance web server.
Hint: Some lines were ellipsized, use -l to show in full.

```

3. Формирование и установка модуля SELinux. Делается командой 
```
ausearch -c 'nginx' --raw | audit2allow -M my-nginx
```
и последующим подключением этого модуля с помощью команды semodule. Пример: 

```
[root@packages ~]# semanage port -d -t http_port_t -p tcp 5080
[root@packages ~]# service nginx restart
Redirecting to /bin/systemctl restart nginx.service
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@packages ~]# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i my-nginx.pp

[root@packages ~]# semodule -i my-nginx.pp
[root@packages ~]# service nginx start
Redirecting to /bin/systemctl start nginx.service
[root@packages ~]# service nginx status
Redirecting to /bin/systemctl status nginx.service
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2021-01-13 10:52:46 UTC; 4s ago
     Docs: http://nginx.org/en/docs/
  Process: 1307 ExecStop=/bin/kill -s TERM $MAINPID (code=exited, status=0/SUCCESS)
  Process: 1349 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf (code=exited, status=0/SUCCESS)
 Main PID: 1350 (nginx)
   CGroup: /system.slice/nginx.service
           ├─1350 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
           └─1351 nginx: worker process

Jan 13 10:52:46 packages systemd[1]: Starting nginx - high performance web server...
Jan 13 10:52:46 packages systemd[1]: Can't open PID file /var/run/nginx.pid (yet?) after start: No such file or...ectory
Jan 13 10:52:46 packages systemd[1]: Started nginx - high performance web server.
Hint: Some lines were ellipsized, use -l to show in full.
```

#### 2. Обеспечить работоспособность приложения при включенном selinux.

После первой провалившейся попытки добавить домен в зону анализ логов с помощью audit2why даёт следующий совет: 

```
Was caused by:
The boolean named_write_master_zones was set incorrectly.
Description:
Allow named to write master zones
```

Исправляем эту ошибку: 

```		
[root@ns01 ~]# setsebool -P named_write_master_zones 1
```

Тем не менее, этого оказывается недостаточно. Продолжаем анализ. audit2why предлагает следующий вариант решения: 

```
type=AVC msg=audit(1610542691.694:5535): avc:  denied  { create } for  pid=16325 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
```

Попробуем сначала решить проблему другими способами. 

1. Прямое изменение контекста. 

```
[root@ns01 vagrant]# ls -dZ /etc/named/dynamic/
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   /etc/named/dynamic/
[root@ns01 vagrant]# chcon -R system_u:object_r:named_zone_t:s0 /etc/named/dynamic/
```

Проверяем работу: 
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab 60 A 192.168.50.15
> send
> quit
[root@client vagrant]# nslookup www.ddns.lab 192.168.50.10
Server:         192.168.50.10
Address:        192.168.50.10#53

Name:   www.ddns.lab
Address: 192.168.50.15
```

Получилось! 

2. Перенос файлов в другую директорию с другим контекстом. 

```
[root@ns01 vagrant]# restorecon -R /etc/named/dynamic/
[root@ns01 vagrant]# ls -dZ /etc/named/dynamic/
drw-rwx---. root named system_u:object_r:etc_t:s0       /etc/named/dynamic/
[root@ns01 vagrant]# mv /etc/named/dynamic/* /var/named/dynamic/
[root@ns01 vagrant]# sed -i 's#/etc/named/dynamic#/var/named/dynamic#g' /etc/named.conf
[root@ns01 vagrant]# service named restart
Redirecting to /bin/systemctl restart named.service
[root@ns01 vagrant]# ls -Z /var/named/dynamic/
-rw-r--r--. named named system_u:object_r:named_cache_t:s0 default.mkeys
-rw-r--r--. named named system_u:object_r:named_cache_t:s0 default.mkeys.jnl
-rw-rw----. named named system_u:object_r:etc_t:s0       named.ddns.lab
-rw-rw----. named named system_u:object_r:etc_t:s0       named.ddns.lab.view1
-rw-r--r--. named named system_u:object_r:etc_t:s0       named.ddns.lab.view1.jnl
-rw-r--r--. named named system_u:object_r:named_cache_t:s0 view1.mkeys
-rw-r--r--. named named system_u:object_r:named_cache_t:s0 view1.mkeys.jnl
[root@ns01 vagrant]# restorecon -R /var/named/dynamic/
[root@ns01 vagrant]# ls -Z /var/named/dynamic/
-rw-r--r--. named named system_u:object_r:named_cache_t:s0 default.mkeys
-rw-r--r--. named named system_u:object_r:named_cache_t:s0 default.mkeys.jnl
-rw-rw----. named named system_u:object_r:named_cache_t:s0 named.ddns.lab
-rw-rw----. named named system_u:object_r:named_cache_t:s0 named.ddns.lab.view1
-rw-r--r--. named named system_u:object_r:named_cache_t:s0 named.ddns.lab.view1.jnl
-rw-r--r--. named named system_u:object_r:named_cache_t:s0 view1.mkeys
-rw-r--r--. named named system_u:object_r:named_cache_t:s0 view1.mkeys.jnl
```

Проверяем: 

```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab 60 A 192.168.50.14
> send
> quit
[vagrant@client ~]$ nslookup www.ddns.lab
Server:         192.168.50.10
Address:        192.168.50.10#53

Name:   www.ddns.lab
Address: 192.168.50.15
Name:   www.ddns.lab
Address: 192.168.50.14
```

Этот способ было бы целесообразнее использовать при изначальной настройке, в этом случае не пришлось бы наследовать (с помощью команды restorecon) контексты перенесённых файлов 
от директории /var/named/dynamic, куда их перенесли; они были бы автоматически установлены верно. 

3. Пользуемся советом утилиты audit2why. 
```
[root@ns01 vagrant]# mv /var/named/dynamic/named.ddns.lab* /etc/named/dynamic/
[root@ns01 vagrant]# sed -i 's#/var/named/dynamic/named#/etc/named/dynamic/named#g' /etc/named.conf
[root@ns01 vagrant]# restorecon -R /etc/named/dynamic/
[root@ns01 vagrant]# ls -Z /etc/named/dynamic/
-rw-rw----. named named system_u:object_r:etc_t:s0       named.ddns.lab
-rw-r--r--. named named system_u:object_r:etc_t:s0       named.ddns.lab.view1
-rw-r--r--. named named system_u:object_r:etc_t:s0       named.ddns.lab.view1.jnl
[root@ns01 vagrant]# service named restart
Redirecting to /bin/systemctl restart named.service
[root@ns01 vagrant]# echo > /var/log/audit/audit.log
```

Генерим ошибку: 
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
>  update add www.ddns.lab 60 A 192.168.50.13
> send
update failed: SERVFAIL
> quit
```

Создаём правило: 
```
[root@ns01 vagrant]# audit2allow -M my-named < /var/log/audit/audit.log
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i my-named.pp

[root@ns01 vagrant]# semodule -i my-named.pp
```

Тестируем: 
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab 60 A 192.168.50.13
> send
> quit
[vagrant@client ~]$ nslookup www.ddns.lab
Server:         192.168.50.10
Address:        192.168.50.10#53

Name:   www.ddns.lab
Address: 192.168.50.14
Name:   www.ddns.lab
Address: 192.168.50.13
Name:   www.ddns.lab
Address: 192.168.50.15
```

Из перечисленных способов самым простым в реализации выглядит первый. Его я и реализовал в прилагаемом playbook. 

