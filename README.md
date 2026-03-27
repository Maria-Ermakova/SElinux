# SElinux
### 3 ways

## Задание: Запустить Nginx на нестандартном порту 3-мя разными способами:

#### на ВМ 

[root@localhost nginx]# cat /etc/os-release
NAME="AlmaLinux"
VERSION="9.7 (Moss Jungle Cat)"

[root@localhost nginx]# getenforce
Enforcing

#### работает сервис nginx на стандартном 80 порту

[root@localhost nginx]# ss -tulpan | grep nginx

tcp   LISTEN    0      511             0.0.0.0:80         0.0.0.0:*     users:(("nginx",pid=6922,fd=6),("nginx",pid=6921,fd=6))

tcp   LISTEN    0      511                [::]:80            [::]:*     users:(("nginx",pid=6922,fd=7),("nginx",pid=6921,fd=7))

#### после замены порта на 4880 в конфиге /etc/nginx/nginx.conf получаем ошибку при запуске сервиса

[root@localhost nginx]# nginx -t

nginx: the configuration file /etc/nginx/nginx.conf syntax is ok

nginx: configuration file /etc/nginx/nginx.conf test is successful

[root@localhost nginx]# systemctl reload nginx

[root@localhost nginx]# systemctl restart nginx

Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.

## Решения:
### 1)переключатели setsebool;

#### отправим аудитлог на анализ в утилиту audit2why

[root@localhost nginx]# audit2why < /var/log/audit/audit.log

type=AVC msg=audit(1774531192.836:333): avc:  denied  { name_bind } for  pid=7253 comm="nginx" src=4880 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
        
Was caused by:
The boolean nis_enabled was set incorrectly.
Description:
Allow nis to enabled

Allow access by executing:
#setsebool -P nis_enabled 1
        
#### так и поступим: установим параметризованную политику - включим параметр (-P постоянно) NIS (Network Information Service)
[root@localhost nginx]# setsebool -P nis_enabled 1

#### и проверим работу сервиса

[root@localhost nginx]# systemctl restart nginx

[root@localhost nginx]# ss -tulpan | grep nginx

tcp   LISTEN 0      511             0.0.0.0:4880      0.0.0.0:*     users:(("nginx",pid=7292,fd=6),("nginx",pid=7291,fd=6))

curl http://192.168.56.20:4880   получаем страницу nginx

curl http://localhost:4880   получаем страницу nginx

из браузера хостовой ОС получаем страницу nginx

### 2)добавление нестандартного порта в имеющийся тип;

#### ngninx использует для работы http_port_t поэтому найдём все разрешенные порты для этого типа 

[root@localhost nginx]# ps auxZ | grep nginx

system_u:system_r:httpd_t:s0    root        7413  0.0  0.1  15932  2212 ?        Ss   16:43   0:00 nginx: master process /usr/sbin/nginx

system_u:system_r:httpd_t:s0    nginx       7414  0.0  0.2  16356  3984 ?        S    16:43   0:00 nginx: worker process

unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 root 7416 0.0  0.1 221672 2504 pts/2 S+ 16:44   0:00 grep --color=auto nginx

[root@localhost nginx]# sesearch -A -s httpd_t | grep http_port

allow httpd_t http_port_t:tcp_socket name_bind;

allow httpd_t http_port_t:tcp_socket name_connect; [ httpd_can_network_relay ]:True

allow httpd_t http_port_t:tcp_socket name_connect; [ httpd_graceful_shutdown ]:True

allow httpd_t http_port_t:udp_socket name_bind;

#### посмотрим список всех разрешенных портов

[root@localhost nginx]# semanage port -l | grep http

http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010

http_cache_port_t              udp      3130

http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000

pegasus_http_port_t            tcp      5988

pegasus_https_port_t           tcp      5989

#### нужно добавить порт 4880
[root@localhost nginx]# semanage port -a -t http_port_t -p tcp 4880

#### и проверим работу сервиса

[root@localhost nginx]# systemctl restart nginx

[root@localhost nginx]# ss -tulpan | grep nginx

tcp   LISTEN 0      511             0.0.0.0:4880      0.0.0.0:*     users:(("nginx",pid=7292,fd=6),("nginx",pid=7291,fd=6))

curl http://192.168.56.20:4880   получаем страницу nginx

curl http://localhost:4880   получаем страницу nginx

из браузера хостовой ОС получаем страницу nginx

### 3)формирование и установка модуля SELinux.

#### очистим аудитлог и создадим только нужное его наполнение

[root@localhost ~]# echo > /var/log/audit/audit.log

[root@localhost ~]# systemctl restart nginx

Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.

#### отправим аудитлог в утилиту audit2allow, которая создаст модуль SElinux для сервиса на основе данных из лога 

[root@localhost ~]# audit2allow -M nginx_add < /var/log/audit/audit.log

******************** IMPORTANT ***********************

To make this policy package active, execute:

semodule -i nginx_add.pp

#### инициализируем новый модуль

[root@localhost ~]# semodule -i nginx_add.pp

#### и проверим работу сервиса

[root@localhost nginx]# systemctl restart nginx

[root@localhost nginx]# ss -tulpan | grep nginx

tcp   LISTEN 0      511             0.0.0.0:4880      0.0.0.0:*     users:(("nginx",pid=7292,fd=6),("nginx",pid=7291,fd=6))

curl http://192.168.56.20:4880   получаем страницу nginx

curl http://localhost:4880   получаем страницу nginx

из браузера хостовой ОС получаем страницу nginx

![SELinux_nginx4880](https://github.com/user-attachments/assets/2b6ace34-732b-4e23-9a2b-fd834bcb2558)

