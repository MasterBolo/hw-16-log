
# Домашняя работа: Основы сбора и хранения логов


	Цель работы: Научиться проектировать централизованный сбор логов.

	Что нужно сделать?

	1. В Vagrant развернуть две машины web и log
	2. на web настраиваем nginx
	3. на log настраиваем центральный лог сервер на любой системе на выбор
		- jornald;
		- rsyslog;
		- elk.
	4. Настраиваем аудит, следящий за изменением конфигов nginx
	
	5. Все настройки сделать с помощью Ansible и добавить запуск Ansible playbook из Vagrantfile.

Все критические логи с web должны собираться и локально и удаленно.
Все логи с nginx должны уходить на удаленный сервер (локально только критичные).
Логи аудита должны также уходить на удаленную систему.
  
# Выполнение

## Разворачиваем две машины web и log

Создаем в домашней директории Vagrantfile, собираю стенд на основе двух машин:
ubuntu-22.04 ( машина web) almalinux -9 (машина log). Листинг для файла Vagrant берём и прилагаемой методички.

## Настраиваем nginx на сервере web
 
Дальнейший разбор настроек будем проводить на примере playbook файла. Все необходимые файлы конфигурации для настройки
кладём в созданную директорию files. Создадим в домашней директории папку Ansible, в данной папке создадим фаил web.yml и дополняем его tasks(задачи).

	Для правильной работы с логами необходимо настроить время. Мой часовой пояс - Asia/Yekaterinburg. 

Cоздадим задачу настройки тайм-зоны:
```
tasks: 
    - name: set timezone to Asia/Yekaterinburg
      timezone:
        name: Asia/Yekaterinburg

````
	Создадим задачу установки nginx :
```
- name: update
      apt:
        update_cache=yes
      tags:
        - update apt

    - name: NGINX | Install NGINX
      apt:
        name: nginx
        state: latest
      tags:
        -nginx-package
```
	Для сбора логов nginx как локально так и для отправки на удаленный сервер (log) дополним стандартную
конфигурацию nginx следующими строками:
````
        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;
        error_log syslog:server=192.168.56.13:514,tag=nginx_error;
        access_log syslog:server=192.168.56.13:514,tag=nginx_access,severity=info combined;
````
	Создадим задачу настройки nginx путем замены стандартного файла конфигурации нужной нам конфигурацией в директории nginx :

```
- name: NGINX | Create NGINX config file from template
      template:
        src: files/web/nginx.conf
        dest: /etc/nginx/nginx.conf
      notify:
        - reload nginx
      tags:
        - nginx-configuration
```
	Для локального сбора логов сервера web и отправки логов на удаленный сервер log выбираем систему rsyslog.
В фаил /etc/rsyslog.d/all.conf добавим следующую строку:
````
*.* @@192.168.56.13:514
````
Создадим задачу настройки rsyslog путем добавления файла all.conf  в директорию /etc/rsyslog.d/:
	
```
- name: rsyslog | configure rsyslog
      template:
        src: files/web/all.conf
        dest: /etc/rsyslog.d/all.conf
      notify:
        - restart rsyslog
      tags:
        - rsyslog-configuration
```
## Настраиваем аудит, следящий за изминением конфигов nginx

	Создадим задачу установки auditd :
```
- name: auditd | install auditd
      apt:
        name: auditd
        state: latest
      tags:
        -auditd-package
```

	Настраиваем auditd отредактировав в файле /etc/auditd.conf следующие строки :
````
log_group = root
name_format = HOSTNAME

````
Создадим соответствующую задачу настройки auditd:
````
- name: auditd | configure auditd
      template:
        src: files/web/auditd.conf
        dest: /etc/audit/auditd.conf
      tags:
        - auditd-configurations
````
Для отправки логов аудита на удаленный сервер log необходимы плагины "audit-plugins", создадим задачу 
установки плагинов:
````
- name: audispd-plugins | install audispd-plugins
      apt:
        name: audispd-plugins
        state: latest
      tags:
        -audispd-package
````
Редактируем фаил /etc/audit/audisp-remote.conf добавив следующие строки:
````
remote_server = 192.168.56.13
port = 60
````
Редактируем фаил /etc/audit/plugins.d/au-remote.conf:

````
active = yes
````
Для слежения за конфигурацией nginx в фаил /etc/audit/rules.d/audit.rules добавляем следующие правила:

````
-w /etc/nginx/nginx.conf -p wa -k nginx_conf
-w /etc/nginx/default.d/ -p wa -k nginx_conf
````

Создадим задачи настройки конфигураций auditd:
```
  - name: audispd-plugins | configure transfer logs to server
      template:
        src: files/web/audisp-remote.conf
        dest: /etc/audit/audisp-remote.conf
      tags:
        - auditd-configurations

    - name: audispd-plugins | configure transfer logs to server
      template:
        src: files/web/au-remote.conf
        dest: /etc/audit/plugins.d/au-remote.conf
      tags:
        - auditd-configurations

    - name: audit-rules | configure audit for nginx
      copy:
        dest: /etc/audit/rules.d/audit.rules
        src: files/web/audit.rules
        owner: root
        group: root
        mode: 0640
      notify:
        - restart auditd
      tags:
        - auditd-configurations
````
## Настраиваем сервер log

	
	Разберём фаил playbook.
	Cоздадим задачу настройки тайм-зоны:
```
tasks: 
    - name: set timezone to Asia/Yekaterinburg
      timezone:
        name: Asia/Yekaterinburg
```
	Настраиваем rsyslog на прием логов с удаленного сервера добавив в фаил /etc/rsyslog.conf
следующие строки :
```
module(load="imudp") # needs to be done just once
input(type="imudp" port="514")
module(load="imtcp") # needs to be done just once
input(type="imtcp" port="514")

$template RemoteLogs,"/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log"
*.*                                                           ?RemoteLogs
& ~
```
Создадим соответствующую задачу настройки rsyslog:
````
- name: rsyslog | Configure RSYSLOG to recieve logs
      template:
        src: files/log/rsyslog.conf
        dest: /etc/rsyslog.conf
      notify:
        - restart rsyslog
      tags:
        - rsyslog-configuration
````
	Настраиваем auditd на прием логов с удаленного сервера отредактировав в файле /etc/auditd.conf
следующие строки :
````
log_group = root
name_format = HOSTNAME
tcp_listen_port = 60
tcp_listen_queue = 5
tcp_max_per_addr = 1
tcp_client_max_idle = 0
````
Создадим соответствующую задачу настройки auditd:
````
- name: auditd | Configure Auditd for recieve logs
      template:
        src: files/log/auditd.conf
        dest: /etc/audit/auditd.conf
      tags:
        - auditd-configurations
````
	Для возможности рестарта auditd после применения настроек необходимо отредактировать 
фаил /usr/lib/systemd/system/auditd.service заменив "yes" на "no":
```` 
RefuseManualStop=no
````
И далее произвести daemon-reload.

Создадим соответствующие задачи:
````
- name: auditd | Configure Auditd for restart service
      template: 
        src: files/log/auditd.service
        dest: /usr/lib/systemd/system/auditd.service
      tags:
        - auditd-configurations

    - name: reload systemd
      ansible.builtin.systemd:
        daemon_reload: yes

    - name: auditd | restart auditd
      service:
         name: auditd
         state: restarted
````
## Проверка отправки логов rsyslog

	Попробуем несколько раз зайти на сервер web по адресу http://192.168.56.10
	Далее заходим на log-сервер и смотрим информацию об nginx:
````
[root@log ~]# cat /var/log/rsyslog/web/nginx_access.log
Oct 28 23:56:47 web nginx_access: 192.168.56.1 - - [28/Oct/2024:23:56:47 +0500] "GET / HTTP/1.1" 200 396 "-" "Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0"
Oct 28 23:56:47 web nginx_access: 192.168.56.1 - - [28/Oct/2024:23:56:47 +0500] "GET /favicon.ico HTTP/1.1" 404 134 "http://192.168.56.10/" "Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0"

````
Сервер log получил логи от web.

    Переместим фаил nginx для получения 403-й ошибки:
````
root@web:~# mv /var/www/html/index.nginx-debian.html /var/www/
````
Страница по адресу http://192.168.56.10 ответила 403-й ошибкой. Заходим и сервер log
и смотрим информацию по ошибкам:
````
[root@log ~]# cat /var/log/rsyslog/web/nginx_error.log
Oct 29 00:06:09 web nginx_error: 2024/10/29 00:06:09 [error] 3371#3371: *2 directory index of "/var/www/html/" is forbidden, client: 192.168.56.1, server: _, request: "GET / HTTP/1.1", host: "192.168.56.10"
Oct 29 00:06:13 web nginx_error: 2024/10/29 00:06:13 [error] 3371#3371: *2 directory index of "/var/www/html/" is forbidden, client: 192.168.56.1, server: _, request: "GET / HTTP/1.1", host: "192.168.56.10"
Oct 29 00:06:15 web nginx_error: 2024/10/29 00:06:15 [error] 3371#3371: *2 directory index of "/var/www/html/" is forbidden, client: 192.168.56.1, server: _, request: "GET / HTTP/1.1", host: "192.168.56.10"

````
Видим, что логи принимаются корректно.

## Проверка отправки логов auditd

	Изменим конфигурацию nginx на сервере web отредактировав фаил nginx.conf в редакторе vim.
	Заходим и сервер log и смотрим события доступов к файлам:
````
[root@log ~]# aureport -f

File Report
===============================================
# date time file syscall success exe auid event
===============================================
1. 10/28/24 23:35:02 /etc/nginx/ 44 yes /usr/sbin/auditctl -1 133
2. 10/29/24 00:24:26 /etc/nginx/ 82 yes /usr/bin/vim.basic 1000 338
3. 10/29/24 00:24:26 /etc/nginx/ 257 yes /usr/bin/vim.basic 1000 339
4. 10/29/24 00:24:26 (null) 91 yes /usr/bin/vim.basic 1000 340
5. 10/29/24 00:24:26 /etc/nginx/nginx.conf 188 yes /usr/bin/vim.basic 1000 341

````	
Видим, что логи аудита принимаются корректно.





