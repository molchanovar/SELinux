# SELinux

Цель: Тренируем умение работать с SELinux: диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений, если это требуется.
1. Запустить nginx на нестандартном порту 3-мя разными способами:
- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.


Сдать README с описанием каждого решения (скриншоты и демонстрация приветствуются).


### Часть I Подготовка

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



#### Troubleshooting 




