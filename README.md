# OTUS ДЗ 7 Инициализация системы. Systemd

## Домашнее задание

Systemd
Выполнить следующие задания и подготовить развёртывание результата выполнения с использованием Vagrant и Vagrant shell provisioner (или Ansible, на Ваше усмотрение):
1. Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/sysconfig);
2. Из репозитория epel установить spawn-fcgi и переписать init-скрипт на unit-файл (имя service должно называться так же: spawn-fcgi);
3. Дополнить unit-файл httpd (он же apache) возможностью запустить несколько инстансов сервера с разными конфигурационными файлами;

### 1 . Написать сервис, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова. Файл и слово должны задаваться в **/etc/sysconfig**:
Создаем файл конфигурации для сервиса

    touch /etc/sysconfig/watchlog

Зполняем его данными:

    # Configuration file for my watchdog service
    # Place it to /etc/sysconfig
    # File and word in that file that we will be monit
    WORD="ALERT"
    LOG=/var/log/watchlog.log

Создаем **touch /var/log/watchlog.log** и пишем туда строку ***ALERT***

Далее создаем скрипт sh:

```bash
touch /home/vagrant/watchlog.sh
```
И пишем туда:

```bash
#!/bin/bash
WORD=$1
LOG=$2
DATE=`date`
if grep $WORD $LOG &> /dev/null
then
logger "$DATE: I found word, Master!" # logger отправляет лог в системный журнал (/var/log/messages)
else
exit 0
fi
```

После чего необходимо дать нашему скрипту права на исполнение:

```bash     
chmod ugo+x watchlog.sh
```

Далее создаем unit для сервиса (точнее создаем сам сервис), создаем файл nano **/etc/systemd/system/watchlog.service**:

    [Unit]
    Description=My watchlog service
    [Service]
    Type=oneshot
    EnvironmentFile=/etc/sysconfig/watchdog
    ExecStart=/home/vagrant/watchlog.sh $WORD $LOG

Создаем unit для таймера (timer файл), **nano /etc/systemd/system/watchlog.timer**:

    [Unit]
    Description=Run watchlog script every 30 second
    [Timer]
    # Run every 30 second
    OnActiveSec=1sec # Активировать сервис .service при активации таймера (запуск)
    #OnBootSec=1min # Активировать сервис .service файл при загрузке системы
    #OnUnitActiveSec=30sec # 1 раз в 30 секунд
    OnCalendar=*:*:0/30 # Еще одна форма, для того чтобы указать периодичность, 1 раз в 30 секунд
    AccuracySec=1us # Указывается точность
    Unit=watchlog.service
    [Install]
    WantedBy=multi-user.target

Для того, чтобы сервис ***.service*** работал, необходимо в файле **watchlog.timer** указать, чтобы сам сервис активировался при активации таймера. Под активацией таймера, подразумевается старт таймера, что мы и сделали в примере кода выше.

Далее выполняем **systemctl enable watchlog.timer** и **systemctl start watchlog.timer**

Выполняем команду ***tail -f /var/log/messages***:

Jun 18 07:17:01 localhost systemd: Reloading.
Jun 18 07:17:07 localhost systemd: Started Run watchlog script every 30 second.
Jun 18 07:17:07 localhost systemd: Starting Run watchlog script every 30 second.
Jun 18 07:17:08 localhost systemd: Starting My watchlog service...
Jun 18 07:17:08 localhost root: Thu Jun 18 07:17:08 UTC 2020: I found word, Master!
Jun 18 07:17:08 localhost systemd: Started My watchlog service.
Jun 18 07:17:30 localhost systemd: Starting My watchlog service...
Jun 18 07:17:30 localhost root: Thu Jun 18 07:17:30 UTC 2020: I found word, Master!
Jun 18 07:17:30 localhost systemd: Started My watchlog service.

### 2 Из репозитория epel установить spawn-fcgi и переписать init-скрипт на unit-файл (имя service должно называться так же: spawn-fcgi);

Устанавливаем необходимый софт

```bash
yum install epel-release -y && yum install spawn-fcgi php php-cli mod_fcgid httpd -y
```
**etc/rc.d/init.d/spawn-fcg** - скрипт, который будет переписан
Заходим в **/etc/sysconfig/spawn-fcgi** и приводим его к следующему виду:


    SOCKET=/var/run/php-fcgi.sock
    OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -- /usr/bin/php-cgi"

Создаем unit файл **nano /etc/systemd/system/spawn-fcgi.service**:

    [Unit]
    Description=Spawn-fcgi startup service by Otus
    After=network.target
    [Service]
    Type=simple
    PIDFile=/var/run/spawn-fcgi.pid
    EnvironmentFile=/etc/sysconfig/spawn-fcgi
    ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
    KillMode=process
    [Install]
    WantedBy=multi-user.target

***systemctl start spawn-fcgi*** и ***systemctl status spawn-fcgi***:

    [root@lvm system]# systemctl status spawn-fcgi
    ● spawn-fcgi.service - Spawn-fcgi startup service by Otus
    Loaded: loaded (/etc/systemd/system/spawn- fcgi.service; enabled; vendor preset: disabled)
    Active: active (running) since Mon 2019-09-09 03:49:12 UTC; 23min ago
    Main PID: 924 (php-cgi)
    Tasks: 33
    Memory: 5.3M
    CGroup: /system.slice/spawn-fcgi.service
           ├─ 924 /usr/bin/php-cgi
           ├─ 981 /usr/bin/php-cgi
           ├─ 982 /usr/bin/php-cgi
           ├─ 983 /usr/bin/php-cgi
           ├─ 984 /usr/bin/php-cgi
           ├─ 985 /usr/bin/php-cgi
           ├─ 986 /usr/bin/php-cgi
           ├─ 987 /usr/bin/php-cgi
           ├─ 988 /usr/bin/php-cgi
           ├─ 989 /usr/bin/php-cgi
           ├─ 990 /usr/bin/php-cgi
           ├─ 991 /usr/bin/php-cgi
           ├─ 992 /usr/bin/php-cgi
           ├─ 993 /usr/bin/php-cgi
           ├─ 994 /usr/bin/php-cgi
           ├─ 995 /usr/bin/php-cgi
           ├─ 996 /usr/bin/php-cgi
           ├─ 997 /usr/bin/php-cgi
           ├─ 998 /usr/bin/php-cgi
           ├─ 999 /usr/bin/php-cgi
           ├─1000 /usr/bin/php-cgi
           ├─1001 /usr/bin/php-cgi
           ├─1002 /usr/bin/php-cgi
           ├─1003 /usr/bin/php-cgi
           ├─1004 /usr/bin/php-cgi
           ├─1005 /usr/bin/php-cgi
           ├─1006 /usr/bin/php-cgi
           ├─1007 /usr/bin/php-cgi
           ├─1008 /usr/bin/php-cgi
           ├─1009 /usr/bin/php-cgi
           ├─1010 /usr/bin/php-cgi
           ├─1011 /usr/bin/php-cgi
           └─1012 /usr/bin/php-cgi

           Sep 09 03:49:12 lvm systemd[1]: Started Spawn-fcgi startup service by Otus.
           Sep 09 03:49:12 lvm systemd[1]: Starting Spawn-fcgi startup service by Otus...

### 3. Дополнить unit-файл httpd (он же apache) возможностью запустить несколько инстансов сервера с разными конфигурационными файлами;

Для запуска нескольких экземпляров сервиса будем использовать шаблон в конфигурации файла окружения, т.е. шаблонизацию
Копируем файл из **/usr/lib/systemd/system/**,
```bash
cp /usr/lib/systemd/system/httpd.service /etc/systemd/system
```
далее переименовываем
```bash
mv /etc/systemd/system/httpd.service /etc/systemd/system/httpd@.service
```

и приводим к виду:

    [Unit]
    Description=The Apache HTTP Server
    After=network.target remote-fs.target nss-lookup.target
    Documentation=man:httpd(8)
    Documentation=man:apachectl(8)
    [Service]

    Type=notify
    EnvironmentFile=/etc/sysconfig/httpd-%I #(%I - это и есть подстановка)
    ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND
    ExecReload=/usr/sbin/httpd $OPTIONS -k graceful
    ExecStop=/bin/kill -WINCH ${MAINPID}
    KillSignal=SIGCONT
    PrivateTmp=true

    [Install]
    WantedBy=multi-user.target           

Скопируем и изменили файлы настроек сервиса **httpd.service**
```bash
cp /etc/sysconfig/httpd /etc/sysconfig/httpd-first
cp /etc/sysconfig/httpd /etc/sysconfig/httpd-second
```
В каждом из этих файлов задаем опцию, с конфигурационным файлом:

    OPTIONS=-f conf/first.conf
    OPTIONS=-f conf/second.conf

Сами конфигурационные файлы:
```bash
cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd1.conf
cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd2.conf
```
В них вносим правки: разные PidFile **/var/run/httpd-second.pid** - т.е. должен быть указан файл пида и **Listen 8080** - указан порт, который будет отличаться от другого инстанса.
```bash
systemctl start httpd@first.service
systemctl start httpd@second.service
```
        [root@otus /]# systemctl status httpd@first.service
        ● httpd@first.service - The Apache HTTP Server
        Loaded: loaded (/etc/systemd/system/httpd@first.service; disabled; vendor preset: disabled)
        Active: active (running) since Fri 2020-06-19 06:57:16 UTC; 58s ago
          Docs: man:httpd(8)
            man:apachectl(8)
            Main PID: 2989 (httpd)
            Status: "Total requests: 0; Current requests/sec: 0; Current traffic:   0 B/sec"
            CGroup: /system.slice/system-httpd.slice/httpd@first.service
            ├─2989 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
            ├─2990 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
            ├─2991 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
            ├─2992 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
            ├─2993 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
            ├─2994 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
            └─2995 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND

            Jun 19 06:57:16 otus systemd[1]: Starting The Apache HTTP Server...
            Jun 19 06:57:16 otus httpd[2989]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, usi...message
            Jun 19 06:57:16 otus systemd[1]: Started The Apache HTTP Server.
            Hint: Some lines were ellipsized, use -l to show in full.

 и второй:

         [root@otus /]# systemctl status httpd@second.service
         ● httpd@second.service - The Apache HTTP Server
         Loaded: loaded (/etc/systemd/system/httpd@second.service; disabled; vendor preset: disabled)
         Active: active (running) since Fri 2020-06-19 06:57:24 UTC; 7s ago
          Docs: man:httpd(8)
            man:apachectl(8)
            Process: 2947 ExecStop=/bin/kill -WINCH ${MAINPID} (code=exited, status=1/FAILURE)
            Main PID: 3002 (httpd)
            Status: "Processing requests..."
            CGroup: /system.slice/system-httpd.slice/httpd@second.service
            ├─3002 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
            ├─3003 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
            ├─3004 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
            ├─3005 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
            ├─3006 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
            ├─3007 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
            └─3008 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND

            Jun 19 06:57:24 otus systemd[1]: Starting The Apache HTTP Server...
            Jun 19 06:57:24 otus httpd[3002]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, usi...message
            Jun 19 06:57:24 otus systemd[1]: Started The Apache HTTP Server.
            Hint: Some lines were ellipsized, use -l to show in full.
