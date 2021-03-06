# SELinux

Тренируем умение работать с SELinux: диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений, если это требуется.
1. Запустить nginx на нестандартном порту 3-мя разными способами:
- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.


2. Обеспечить работоспособность приложения при включенном selinux.
- Развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems
- Выяснить причину неработоспособности механизма обновления зоны (см. README);
- Предложить решение (или решения) для данной проблемы;
- Выбрать одно из решений для реализации, предварительно обосновав выбор;
- Реализовать выбранное решение и продемонстрировать его работоспособность. К сдаче:
- README с анализом причины неработоспособности, возможными способами решения и обоснованием выбора одного из них;
- Исправленный стенд или демонстрация работоспособной системы скриншотами и описанием.


Сдать README с описанием каждого решения (скриншоты и демонстрация приветствуются).


## Часть I. Запустить nginx на нестандартном порту 3-мя разными способами.

Ставим пакеты для работы с SELinux: 
```
yum install policycoreutils-python setroubleshoot setools-console
```

1. **policycoreutils-python** – базовый набор утилит для работы с SELinux
2. **setroubleshoot** — инструмент, который анализирует сообщения об отказах AVC, сравнивает их со значениями в своей базе данных, а затем в удобочитаемом виде выводит сообщения об отказе с пояснениями и предлагает возможные варианты решения проблемы.
3. **setools-console** - предоставляет инструменты для мониторинга логов аудита, политики запросов и управления контекстом файлов.

Ставим nginx + настраиваем его на прослушивание нестандартного порта (для примера взят порт **8111**): 
Репозиторий + установка + правка конфига + попытка запуска:
```
cat >/etc/yum.repos.d/nginx.repo <<EOF
[nginx]
name=nginx repo
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
Счетчик уведомлений           7
Впервые обнаружено            2021-03-28 11:55:56 UTC
В последний раз               2021-03-28 16:07:05 UTC
Локальный ID                  68113768-1fd1-44bb-a0ef-6bba5a3be024

Построчный вывод сообщений аудита
type=AVC msg=audit(1616947625.514:648): avc:  denied  { name_bind } for  pid=14369 comm="nginx" src=8111 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0


type=SYSCALL msg=audit(1616947625.514:648): arch=x86_64 syscall=bind success=no exit=EACCES a0=6 a1=55e412393e28 a2=10 a3=7ffd23a12d80 items=0 ppid=1 pid=14369 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm=nginx exe=/usr/sbin/nginx subj=system_u:system_r:httpd_t:s0 key=(null)

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

Проверяем работу Nginx и убеждаемся что все ОК: 
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
В конце удаляем модуль из списка:
```
[root@bash ~]# semodule -r my-nginx
```


## Часть II. Обеспечить работоспособность приложения при включенном selinux.
<details>
  <summary>Задача:</summary>
   
```
  Инженер настроил следующую схему:

    ns01 - DNS-сервер (192.168.50.10);
    client - клиентская рабочая станция (192.168.50.15).

При попытке удаленно (с рабочей станции) внести изменения в зону ddns.lab происходит следующее:

[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
>

Инженер перепроверил содержимое конфигурационных файлов и, убедившись, что с ними всё в порядке, предположил, что данная ошибка связана с SELinux.
```
</details>


Разворачиваем [стенд](https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems) `vagrant up` и заходим на клиент `vagrant ssh client`. 
Выполняем запрос на обновление зоны DNS: 
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
>
```
Ошибки в логе SELinux на DNS Server'е (/var/log/audit/audit.log):
```
[root@ns01 ~]# cat /var/log/audit/audit.log | grep denied
type=AVC msg=audit(1616979951.515:2004): avc:  denied  { create } for  pid=5181 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0
```

Используем утилиту `sealert` для анализа ошибок в логе: 
```
[root@ns01 ~]# sealert -a /var/log/audit/audit.log
100% done
found 1 alerts in /var/log/audit/audit.log
--------------------------------------------------------------------------------

Traceback (most recent call last):
  File "/bin/sealert", line 510, in on_analyzer_state_change
    self.output_results()
  File "/bin/sealert", line 524, in output_results
    print siginfo.format_text()
UnicodeEncodeError: 'ascii' codec can't encode characters in position 8-16: ordinal not in range(128)
```

Далее идем в `/var/log/messages`:
```
[root@ns01 ~]# tail -n 20  /var/log/messages | grep audit2allow
Mar 29 01:05:55 localhost python: SELinux is preventing /usr/sbin/named from create access on the file named.ddns.lab.view1.jnl.#012#012*****  Plugin catchall_labels (83.8 confidence) suggests   *******************#012#012If you want to allow named to have create access on the named.ddns.lab.view1.jnl file#012Then you need to change the label on named.ddns.lab.view1.jnl#012Do#012# semanage fcontext -a -t FILE_TYPE 'named.ddns.lab.view1.jnl'#012where FILE_TYPE is one of the following: dnssec_trigger_var_run_t, ipa_var_lib_t, krb5_host_rcache_t, krb5_keytab_t, named_cache_t, named_log_t, named_tmp_t, named_var_run_t, named_zone_t.#012Then execute:#012restorecon -v 'named.ddns.lab.view1.jnl'#012#012#012*****  Plugin catchall (17.1 confidence) suggests   **************************#012#012If you believe that named should be allowed create access on the named.ddns.lab.view1.jnl file by default.#012Then you should report this as a bug.#012You can generate a local policy module to allow this access.#012Do#012allow this access for now by executing:#012# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000#012# semodule -i my-iscworker0000.pp#012
```

Находим подсказку `ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000 #012# semodule -i my-iscworker0000.pp`. 
Добавляем модуль:
```
[root@ns01 ~]# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000                                     
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i my-iscworker0000.pp

[root@ns01 ~]# semodule -i my-iscworker0000.pp
[root@ns01 ~]# semodule --list-modules=full | grep my-iscworker
400 my-iscworker0000  pp 
```
Пробуем перезапустить `nsupdate`, но снова получаем ту же ошибку `update failed: SERVFAIL`. Проверяем логи: 
```
[root@ns01 ~]# cat /var/log/audit/audit.log | grep denied
type=AVC msg=audit(1616979951.515:2004): avc:  denied  { create } for  pid=5181 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0
type=AVC msg=audit(1616981098.970:2007): avc:  denied  { write } for  pid=5181 comm="isc-worker0000" path="/etc/named/dynamic/named.ddns.lab.view1.jnl" dev="sda1" ino=5055621 scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0
type=AVC msg=audit(1616981324.828:2008): avc:  denied  { write } for  pid=5181 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" dev="sda1" ino=5055621 scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0
```
Обращаем внимание, что причина `denied` изменилась с `{ create }` на `{ write }`. Идем по указанному пути `path="/etc/named/dynamic/named.ddns.lab.view1.jnl"`
```
[root@ns01 ~]# cd /etc/named/dynamic/  
[root@ns01 dynamic]# ls
named.ddns.lab  named.ddns.lab.view1  named.ddns.lab.view1.jnl
```
Видим что файл пустой. Пробуем его удалить `rm named.ddns.lab.view1.jnl` и перезапускаем сервис DNS `systemctl restart named`

Снова пробуем обновить DNS запись:
```
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
```
Обновленно успешно. 
<details>
  <summary>Пробуем обратиться к сервису DNS:</summary>

```
[root@client ~]# dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.4 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 4847
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10

;; Query time: 3 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Mon Mar 29 01:52:41 UTC 2021
;; MSG SIZE  rcvd: 96
```
</details>

Итог: Selinux блокировал доступ к обновлению файлов для сервиса DNS и файлам к которым обращается DNS. Возможные решения - скомпилировать модуль SELinux или измененить контекст безопасности.
