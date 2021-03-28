# SELinux

Цель: Тренируем умение работать с SELinux: диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений, если это требуется.
1. Запустить nginx на нестандартном порту 3-мя разными способами:
- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.


Сдать README с описанием каждого решения (скриншоты и демонстрация приветствуются).


## Часть I. Запустить nginx на нестандартном порту 3-мя разными способами:

Ставим пакеты для работы с SELinux: 
```
yum install policycoreutils-python setroubleshoot setools-console
```

1. **policycoreutils-python** – базовый набор утилит для работы с SELinux
2. **setroubleshoot** — инструмент, который анализирует сообщения об отказах AVC, сравнивает их со значениями в своей базе данных, а затем в удобочитаемом виде выводит сообщения об отказе с пояснениями и предлагает возможные варианты решения проблемы.
3. **setools-console** - предоставляет инструменты для мониторинга логов аудита, политики запросов и управления контекстом файлов.

Ставим nginx + настраиваем его на прослушивание нестандартного порта (например, **8111**): 
Репозиторий + установка + правка конфига + попытка запуска:
```
cat >/etc/yum.repos.d/nginx.repo <<EOF
[nginx]
name =nginx repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
EOF

yum -y install nginx
sed -i '2s/listen       80;/listen       8111;/g' /etc/nginx/conf.d/default.conf
systemctl start nginx
```

Запуск не происходит: 
```
[root@bash ~]# systemctl start nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@bash ~]# systemctl status nginx
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Вс 2021-03-28 12:42:49 UTC; 4s ago
     Docs: http://nginx.org/en/docs/
  Process: 13752 ExecStop=/bin/sh -c /bin/kill -s TERM $(/bin/cat /var/run/nginx.pid) (code=exited, status=0/SUCCESS)
  Process: 13783 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf (code=exited, status=1/FAILURE)
 Main PID: 13589 (code=exited, status=0/SUCCESS)

мар 28 12:42:49 bash systemd[1]: Starting nginx - high performance web server...
мар 28 12:42:49 bash nginx[13783]: nginx: [emerg] bind() to 0.0.0.0:8111 failed (13: Permission denied)
мар 28 12:42:49 bash systemd[1]: nginx.service: control process exited, code=exited status=1
мар 28 12:42:49 bash systemd[1]: Failed to start nginx - high performance web server.
мар 28 12:42:49 bash systemd[1]: Unit nginx.service entered failed state.
мар 28 12:42:49 bash systemd[1]: nginx.service failed.
[root@bash ~]# 
```

Понимаем что причина в SELinux, запускаем `sealert` и смотрим отчет об ошибках:  
```
[root@bash ~]# sealert -a /var/log/audit/audit.log
```

<details>
  <summary>Вывод sealert:</summary>

```
[root@bash ~]# sealert -a /var/log/audit/audit.log
100% done
found 2 alerts in /var/log/audit/audit.log
--------------------------------------------------------------------------------

SELinux запрещает /usr/sbin/nginx доступ name_bind к tcp_socket port 8000.

*****  Модуль catchall предлагает (точность 100.)  ***************************

Если вы считаете, что nginx должно быть разрешено name_bind доступ к port 8000 tcp_socket по умолчанию.
То рекомендуется создать отчет об ошибке.
Чтобы разрешить доступ, можно создать локальный модуль политики.
Сделать
allow this access for now by executing:
# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
# semodule -i my-nginx.pp


Дополнительные сведения:
Исходный контекст             system_u:system_r:httpd_t:s0
Целевой контекст              system_u:object_r:soundd_port_t:s0
Целевые объекты               port 8000 [ tcp_socket ]
Источник                      nginx
Путь к источнику              /usr/sbin/nginx
Порт                          8000
Узел                          <Unknown>
Исходные пакеты RPM           nginx-1.18.0-2.el7.ngx.x86_64
Целевые пакеты RPM            
Пакет регламента              selinux-policy-3.13.1-268.el7_9.2.noarch
SELinux активен               True
Тип регламента                targeted
Режим                         Enforcing
Имя узла                      bash
Платформа                     Linux bash 3.10.0-1127.el7.x86_64 #1 SMP Tue Mar
                              31 23:36:51 UTC 2020 x86_64 x86_64
Счетчик уведомлений           4
Впервые обнаружено            2021-03-28 10:51:56 UTC
В последний раз               2021-03-28 11:47:42 UTC
Локальный ID                  72853173-5e76-4dca-a183-5a0d91fd5c07

Построчный вывод сообщений аудита
type=AVC msg=audit(1616932062.708:497): avc:  denied  { name_bind } for  pid=13500 comm="nginx" src=8000 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:soundd_port_t:s0 tclass=tcp_socket permissive=0


type=SYSCALL msg=audit(1616932062.708:497): arch=x86_64 syscall=bind success=no exit=EACCES a0=6 a1=555e5b303e28 a2=10 a3=7fff567bf630 items=0 ppid=1 pid=13500 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm=nginx exe=/usr/sbin/nginx subj=system_u:system_r:httpd_t:s0 key=(null)

Hash: nginx,httpd_t,soundd_port_t,tcp_socket,name_bind

--------------------------------------------------------------------------------

SELinux запрещает /usr/sbin/nginx доступ name_bind к tcp_socket port 8111.

*****  Модуль bind_ports предлагает (точность 92.2)  *************************

Если вы хотите разрешить /usr/sbin/nginx для привязки к сетевому порту $PORT_ЧИСЛО
То you need to modify the port type.
Сделать
# semanage port -a -t PORT_TYPE -p tcp 8111
    где PORT_TYPE может принимать значения: http_cache_port_t, http_port_t, jboss_management_port_t, jboss_messaging_port_t, ntop_port_t, puppet_port_t.

*****  Модуль catchall_boolean предлагает (точность 7.83)  *******************

Если хотите allow nis to enabled
То вы должны сообщить SELinux об этом, включив переключатель «nis_enabled».

Сделать
setsebool -P nis_enabled 1

*****  Модуль catchall предлагает (точность 1.41)  ***************************

Если вы считаете, что nginx должно быть разрешено name_bind доступ к port 8111 tcp_socket по умолчанию.
То рекомендуется создать отчет об ошибке.
Чтобы разрешить доступ, можно создать локальный модуль политики.
Сделать
allow this access for now by executing:
# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
# semodule -i my-nginx.pp


Дополнительные сведения:
Исходный контекст             system_u:system_r:httpd_t:s0
Целевой контекст              system_u:object_r:unreserved_port_t:s0
Целевые объекты               port 8111 [ tcp_socket ]
Источник                      nginx
Путь к источнику              /usr/sbin/nginx
Порт                          8111
Узел                          <Unknown>
Исходные пакеты RPM           nginx-1.18.0-2.el7.ngx.x86_64
Целевые пакеты RPM            
Пакет регламента              selinux-policy-3.13.1-268.el7_9.2.noarch
SELinux активен               True
Тип регламента                targeted
Режим                         Enforcing
Имя узла                      bash
Платформа                     Linux bash 3.10.0-1127.el7.x86_64 #1 SMP Tue Mar
                              31 23:36:51 UTC 2020 x86_64 x86_64
Счетчик уведомлений           3
Впервые обнаружено            2021-03-28 11:55:56 UTC
В последний раз               2021-03-28 12:42:49 UTC
Локальный ID                  02e12b2d-a42a-4386-b066-b72be02da937

Построчный вывод сообщений аудита
type=AVC msg=audit(1616935369.886:570): avc:  denied  { name_bind } for  pid=13783 comm="nginx" src=8111 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0


type=SYSCALL msg=audit(1616935369.886:570): arch=x86_64 syscall=bind success=no exit=EACCES a0=6 a1=55cbd0b8be28 a2=10 a3=7fff9d8d59f0 items=0 ppid=1 pid=13783 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm=nginx exe=/usr/sbin/nginx subj=system_u:system_r:httpd_t:s0 key=(null)

Hash: nginx,httpd_t,unreserved_port_t,tcp_socket,name_bind

```
</details>

### Переключатели setsebool:

Видим одно из решений предложенных SELinux: 
```
SELinux запрещает /usr/sbin/nginx доступ name_bind к tcp_socket port 8111.

Если хотите allow nis to enabled
То вы должны сообщить SELinux об этом, включив переключатель «nis_enabled».

setsebool -P nis_enabled 1
```
Переключаем setsebool (с флагом -P изменения вносятся на постоянной основе) и запускаем NGINX 
```
[root@bash ~]# setsebool -P nis_enabled 1
[root@bash ~]# systemctl restart nginx
[root@bash ~]# systemctl status nginx

[root@bash ~]# systemctl status nginx
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Вс 2021-03-28 13:05:59 UTC; 11s ago
  Process: 13752 ExecStop=/bin/sh -c /bin/kill -s TERM $(/bin/cat /var/run/nginx.pid) (code=exited, status=0/SUCCESS)
  Process: 13897 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf (code=exited, status=0/SUCCESS)
 Main PID: 13898 (nginx)
   CGroup: /system.slice/nginx.service
           ├─13898 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
           └─13899 nginx: worker process
   
[root@bash ~]# getsebool -a | grep nis             
nis_enabled --> on
varnishd_connect_any --> off
```
Nginx запустился. 
Возвращаем значение `setsebool -P nis_enabled 0`


### Добавление нестандартного порта в имеющийся тип:
Проверяем разрешенные порты: 
```
[root@bash ~]# semanage port -l | grep -w http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
[root@bash ~]# 
```
Добавляем нестандартный порт 8111 в список `http_port_t`:
```
[root@bash ~]# semanage port -a -t http_port_t -p tcp 8111
[root@bash ~]# semanage port -l | grep -w http_port_t
http_port_t                    tcp      8111, 80, 81, 443, 488, 8008, 8009, 8443, 9000
[root@bash ~]# 
```

Проверяем работу Nginx: 
```
[root@bash ~]# systemctl status nginx
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Вс 2021-03-28 15:48:48 UTC; 2s ago
     Docs: http://nginx.org/en/docs/
  Process: 14000 ExecStop=/bin/sh -c /bin/kill -s TERM $(/bin/cat /var/run/nginx.pid) (code=exited, status=0/SUCCESS)
  Process: 14004 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf (code=exited, status=0/SUCCESS)
 Main PID: 14005 (nginx)

```
Удаляем порт из разрешенных:
```
[root@bash ~]# semanage port -d -t http_port -p tcp 8111
```

### Формирование и установка модуля SELinux:
Формирование модуля SELinux утилитой audit2allow: 
```
[root@bash ~]# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
```
Применяем созданный модуль и смотрим вывод списка модулей: 

```
[root@bash ~]# semodule -i my-nginx.pp
[root@bash ~]# semodule --list-modules=full | grep my-nginx
400 my-nginx          pp
```
Проверяем что Nginx запустился: 
```
[root@bash ~]# systemctl restart nginx
[root@bash ~]# systemctl status nginx
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Вс 2021-03-28 16:00:53 UTC; 2s ago
     Docs: http://nginx.org/en/docs/
  Process: 14225 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf (code=exited, status=0/SUCCESS)
 Main PID: 14226 (nginx)
   CGroup: /system.slice/nginx.service
           ├─14226 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
           └─14227 nginx: worker process
```

#### Troubleshooting 




