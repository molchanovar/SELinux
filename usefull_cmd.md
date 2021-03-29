#### Просмотр статуса SELinux: 
 - sestatus    (много инфы) 
 - setenforce  (изменяет режим SELinux
 - getenforce  (текущий режим работы)

 ```
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      31
 ```

#### Просмотр установленных пакетов:
```
rpm -qa | grep selinux
libselinux-2.5-15.el7.x86_64
libselinux-python-2.5-15.el7.x86_64
selinux-policy-3.13.1-266.el7.noarch
selinux-policy-targeted-3.13.1-266.el7.noarch
libselinux-utils-2.5-15.el7.x86_64
```

#### Чистка лога
```
cat /dev/null > /var/log/audit/audit.log
```

**Конфиг** - vim /etc/selinux/config

#### Просмотр контекста
ls -lZ /var/www
```
[root@bash ~]# ls -lZ /var/www     
drwxr-xr-x. root root system_u:object_r:httpd_sys_script_exec_t:s0 cgi-bin
drwxr-xr-x. root root system_u:object_r:httpd_sys_content_t:s0 html
```

#### Просмотр домена процесса
ps auxZ
```
[root@bash ~]# ps auxZ | grep httpd
system_u:system_r:httpd_t:s0    root      2068  0.5  2.0 224080  5024 ?        Ss   09:19   0:00 /usr/sbin/httpd -DFOREGROUND
system_u:system_r:httpd_t:s0    apache    2069  0.0  1.2 224080  2924 ?        S    09:19   0:00 /usr/sbin/httpd -DFOREGROUND
system_u:system_r:httpd_t:s0    apache    2070  0.0  1.2 224080  2924 ?        S    09:19   0:00 /usr/sbin/httpd -DFOREGROUND
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 root 2075 0.0  0.2 12500 684 pts/1 S+ 09:20   0:00 grep --color=auto httpd
```
#### Список разрешенных портов:
ALL: 
semanage port -l

HTTP:
semanage port -l | grep -w http_port_t
```
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
```
semanage port -a -t http_port_t -p tcp 8111  - добавить порт 8111
semanage port -d -t http_port -p tcp 8111    - Удалить порт

#### Логирование
less /var/log/audit/audit.log
Или **sealert** - покажет только ошибки 
(sudo yum install setroubleshoot + sealert -a /var/log/audit/audit.log)

#### Модули
semodule --list-modules=full     - Установленные модули
semodule -l                      - Список всех активных модулей
semodule -r <module_name>        - Удалить модуль 


#### Флаги
getsebool -a                     - Состояние флагов
setsebool -P <флаг> on/off       - Изменить состояние флага
(setsebool -P httpd_can_network_connect on)
