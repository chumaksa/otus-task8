# otus-task8

# Systemd - создание unit-файла

1. Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/sysconfig или в /etc/default);
2. Установить spawn-fcgi и переписать init-скрипт на unit-файл (имя service должно называться так же: spawn-fcgi);
3. Дополнить unit-файл httpd (он же apache2) возможностью запустить несколько инстансов сервера с разными конфигурационными файлами.

Для выполнения задания будем использовать VM с ОС Ubuntu 20.04.6 LTS. \
Развернём тестовый стенд с помощью Vagrant. Для этого создадим Vagranfile следующего содержания:
```

# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "ubuntu/focal64"
  config.vm.define "otus"
  config.vm.hostname = "otus"
  
  config.vm.provider "virtualbox" do |v|
    v.memory = 2048
    v.cpus = 1
  end

end
```

## 1. Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/sysconfig или в /etc/default)

### Решение

Создадим файл watchlog в /etc/default/. В этом файле будем указывать ключевое слово и путь к файлу с логами.
```

vagrant@otus:~$ sudo nano /etc/default/watchlog
vagrant@otus:~$ cat /etc/default/watchlog
# Configuration file for my watchlog service
# Place it to /etc/default
# File and word in that file that we will be monit
WORD="ALERT"
LOG=/var/log/watchlog.log
```

Далее создадим сам файл логов с произвольным содержимым и с ключевым словом "ALERT".
```

vagrant@otus:~$ sudo nano /var/log/watchlog.log
vagrant@otus:~$ cat /var/log/watchlog.log
Feb 23 11:53:05 ubuntu-focal systemd[835]: Reached target Paths.
Feb 23 11:53:05 ubuntu-focal systemd[835]: Reached target Timers.
Feb 23 11:53:05 ubuntu-focal systemd[835]: Starting D-Bus User Message Bus Socket.
Feb 23 11:53:05 ubuntu-focal systemd[835]: Listening on GnuPG network certificate management daemon.
Feb 23 11:53:05 ubuntu-focal systemd[835]: Listening on GnuPG cryptographic agent and passphrase cache (access for web browsers).
Feb 23 11:53:05 ubuntu-focal systemd[835]: Listening on GnuPG cryptographic agent and passphrase cache (restricted).
Feb 23 11:53:05 ubuntu-focal systemd[835]: Listening on GnuPG cryptographic agent (ssh-agent emulation).
Feb 23 11:53:05 ubuntu-focal systemd[835]: Listening on GnuPG cryptographic agent and passphrase cache.
Feb 23 11:53:05 ubuntu-focal systemd[835]: Listening on debconf communication socket.
ALERT

```

Создадим скрипт watchlog.sh в /opt/ и добавим права на запуск.
```

vagrant@otus:~$ sudo nano /opt/watchlog.sh
vagrant@otus:~$ cat /opt/watchlog.sh
#!/bin/bash
WORD=$1
LOG=$2
DATE=`date`
if grep $WORD $LOG &> /dev/null
then
logger "$DATE: I found word, Master!"
else
exit 0
fi
vagrant@otus:~$ ls -l /opt/watchlog.sh 
-rw-r--r-- 1 root root 129 Feb 23 12:18 /opt/watchlog.sh
vagrant@otus:~$ sudo chmod a+x /opt/watchlog.sh
vagrant@otus:~$ ls -l /opt/watchlog.sh 
-rwxr-xr-x 1 root root 129 Feb 23 12:18 /opt/watchlog.sh
```

Далее создаём юнит для нашего сервиса и юнит для таймера.
```

vagrant@otus:~$ sudo nano /etc/systemd/system/watchlog.service
vagrant@otus:~$ cat /etc/systemd/system/watchlog.service
[Unit]
Description=My watchlog service
Wants=watchlog.timer

[Service]
Type=oneshot
EnvironmentFile=/etc/default/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG

[Install]
WantedBy=multi-user.target

vagrant@otus:~$ sudo nano /etc/systemd/system/watchlog.timer
vagrant@otus:~$ cat /etc/systemd/system/watchlog.timer
[Unit]
Description=Run watchlog script every 30 second
Requires=watchlog.service

[Timer]
# Run every 30 second
OnUnitActiveSec=30
Unit=watchlog.service

[Install]
WantedBy=timers.target
```

Запускаем наш таймер и через некоторое время проверяем логи:
```

vagrant@otus:~$ sudo systemctl start watchlog.timer 
vagrant@otus:~$ sudo systemctl status watchlog.timer 
● watchlog.timer - Run watchlog script every 30 second
     Loaded: loaded (/etc/systemd/system/watchlog.timer; disabled; vendor preset: enabled)
     Active: active (waiting) since Fri 2024-02-23 13:01:44 UTC; 1min 47s ago
    Trigger: Fri 2024-02-23 13:03:55 UTC; 23s left
   Triggers: ● watchlog.service

Feb 23 13:01:44 otus systemd[1]: Started Run watchlog script every 30 second.

vagrant@otus:~$ sudo cat /var/log/syslog | grep "Master"
Feb 23 12:45:04 ubuntu-focal root: Fri Feb 23 12:45:04 UTC 2024: I found word, Master!
Feb 23 13:01:44 ubuntu-focal root: Fri Feb 23 13:01:44 UTC 2024: I found word, Master!
Feb 23 13:02:23 ubuntu-focal root: Fri Feb 23 13:02:23 UTC 2024: I found word, Master!
Feb 23 13:03:05 ubuntu-focal root: Fri Feb 23 13:03:05 UTC 2024: I found word, Master!
Feb 23 13:03:25 ubuntu-focal root: Fri Feb 23 13:03:25 UTC 2024: I found word, Master!
Feb 23 13:04:05 ubuntu-focal root: Fri Feb 23 13:04:05 UTC 2024: I found word, Master!
Feb 23 13:05:05 ubuntu-focal root: Fri Feb 23 13:05:05 UTC 2024: I found word, Master!
Feb 23 13:05:35 ubuntu-focal root: Fri Feb 23 13:05:35 UTC 2024: I found word, Master!
Feb 23 13:07:00 ubuntu-focal root: Fri Feb 23 13:07:00 UTC 2024: I found word, Master!
Feb 23 13:08:03 ubuntu-focal root: Fri Feb 23 13:08:03 UTC 2024: I found word, Master!
Feb 23 13:09:05 ubuntu-focal root: Fri Feb 23 13:09:05 UTC 2024: I found word, Master!
```

Наш сервис работает.

## Подготовка к 2 и 3 заданию.

В методичке к заданию используется Centos 7. \
Из-за больших различий между Ubuntu 20 и Centos 7 сделаем вторую VM с этой ОС. \
Отредактируем Vagrantfile.
```

# -*- mode: ruby -*-
# vi: set ft=ruby :

MACHINES = {
  :otusubuntu=> {
        :box_name => "ubuntu/focal64",
        :vm_name => "otus",
  },

  :otuscentos => {
        :box_name => "centos/7",
        :vm_name => "otuscentos",
  }

}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|
    
    config.vm.define boxname do |box|
   
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxconfig[:vm_name]

     end
  end
end
```

Всю дальнейшую часть задания будем выполнять в VM otuscentos с ОС Centos 7. 

## 2. Установить spawn-fcgi и переписать init-скрипт на unit-файл (имя service должно называться так же: spawn-fcgi)

### Решение

Устанавливаем spawn-fcgi и необходимые для него пакеты:
```

[vagrant@otuscentos ~]$ su -l
Password: 
[root@otuscentos ~]#  yum install epel-release -y && yum install spawn-fcgi php php-cli mod_fcgid httpd -y
```

Далее необходимо раскоментировать строки с переменными в /etc/sysconfig/spawn-fcgi.
```

[root@otuscentos ~]# vi /etc/sysconfig/spawn-fcgi
[root@otuscentos ~]# cat /etc/sysconfig/spawn-fcgi
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -P /var/run/spawn-fcgi.pid -- /usr/bin/php-cgi"
```

Далее создаём юнит для Systemd.
```

[root@otuscentos ~]# vi /etc/systemd/system/spawn-fcgi.service
[root@otuscentos ~]# cat /etc/systemd/system/spawn-fcgi.service
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
```

Запускаем и проверяем работоспособность сервиса.
```

[root@otuscentos ~]# systemctl daemon-reload 
[root@otuscentos ~]# systemctl start spawn-fcgi.service
[root@otuscentos ~]# systemctl status spawn-fcgi.service 
● spawn-fcgi.service - Spawn-fcgi startup service by Otus
   Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2024-02-23 16:19:39 UTC; 7s ago
 Main PID: 3154 (php-cgi)
   CGroup: /system.slice/spawn-fcgi.service
           ├─3154 /usr/bin/php-cgi
           ├─3155 /usr/bin/php-cgi
           ├─3156 /usr/bin/php-cgi
           ├─3157 /usr/bin/php-cgi
           ├─3158 /usr/bin/php-cgi
           ├─3159 /usr/bin/php-cgi
           ├─3160 /usr/bin/php-cgi
           ├─3161 /usr/bin/php-cgi
           ├─3162 /usr/bin/php-cgi
           ├─3163 /usr/bin/php-cgi
           ├─3164 /usr/bin/php-cgi
           ├─3165 /usr/bin/php-cgi
           ├─3166 /usr/bin/php-cgi
           ├─3167 /usr/bin/php-cgi
           ├─3168 /usr/bin/php-cgi
           ├─3169 /usr/bin/php-cgi
           ├─3170 /usr/bin/php-cgi
           ├─3171 /usr/bin/php-cgi
           ├─3172 /usr/bin/php-cgi
           ├─3173 /usr/bin/php-cgi
           ├─3174 /usr/bin/php-cgi
           ├─3175 /usr/bin/php-cgi
           ├─3176 /usr/bin/php-cgi
           ├─3177 /usr/bin/php-cgi
           ├─3178 /usr/bin/php-cgi
           ├─3179 /usr/bin/php-cgi
           ├─3180 /usr/bin/php-cgi
           ├─3181 /usr/bin/php-cgi
           ├─3182 /usr/bin/php-cgi
           ├─3183 /usr/bin/php-cgi
           ├─3184 /usr/bin/php-cgi
           ├─3185 /usr/bin/php-cgi
           └─3186 /usr/bin/php-cgi

Feb 23 16:19:39 otuscentos systemd[1]: Started Spawn-fcgi startup service by Otus.
```

## 3. Дополнить unit-файл httpd (он же apache2) возможностью запустить несколько инстансов сервера с разными конфигурационными файлами.

### Решение

Для запуска нескольких экземпляров сервиса необходимо использовать шаблоны. \
Для этого в конфигурационный файл добавим параметр %I.
```

[root@otuscentos ~]# cp /lib/systemd/system/httpd.service /etc/systemd/system/httpd@.service
[root@otuscentos ~]# vi /etc/systemd/system/httpd@.service
[root@otuscentos ~]# cat /etc/systemd/system/httpd@.service
[Unit]
Description=The Apache HTTP Server
After=network.target remote-fs.target nss-lookup.target
Documentation=man:httpd(8)
Documentation=man:apachectl(8)

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/httpd-%I
ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND
ExecReload=/usr/sbin/httpd $OPTIONS -k graceful
ExecStop=/bin/kill -WINCH ${MAINPID}
# We want systemd to give httpd some time to finish gracefully, but still want
# it to kill httpd after TimeoutStopSec if something went wrong during the
# graceful stop. Normally, Systemd sends SIGTERM signal right after the
# ExecStop, which would kill httpd. We are sending useless SIGCONT here to give
# httpd time to finish.
KillSignal=SIGCONT
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

В самом файле окружения (которых будет два) задается опция для запуска веб-сервера с необходимым конфигурационным файлом:
```

[root@otuscentos ~]# vi /etc/sysconfig/httpd-first
[root@otuscentos ~]# cat /etc/sysconfig/httpd-first
OPTIONS=-f conf/first.conf

[root@otuscentos ~]# vi /etc/sysconfig/httpd-second
[root@otuscentos ~]# cat /etc/sysconfig/httpd-second 
OPTIONS=-f conf/second.conf
```

Для удачного запуска, в конфигурационных файлах должны быть указаны уникальные для каждого экземпляра опции Listen и PidFile.\
Правим конфиги, запускаем сервисы и проверяем.
```

[root@otuscentos ~]# cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/first.conf
[root@otuscentos ~]# cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/second.conf
[root@otuscentos ~]# vi /etc/httpd/conf/second.conf
[root@otuscentos ~]# systemctl start httpd@first
[root@otuscentos ~]# systemctl start httpd@second

root@otuscentos ~]# systemctl status httpd@first
● httpd@first.service - The Apache HTTP Server
   Loaded: loaded (/etc/systemd/system/httpd@.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2024-02-23 17:01:32 UTC; 19s ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 3569 (httpd)
   Status: "Total requests: 0; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/system-httpd.slice/httpd@first.service
           ├─3569 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─3570 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─3571 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─3572 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─3573 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─3574 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           └─3575 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND

Feb 23 17:01:32 otuscentos systemd[1]: Starting The Apache HTTP Server...
Feb 23 17:01:32 otuscentos httpd[3569]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1. Set the 'ServerName' directive globally to suppress this message
Feb 23 17:01:32 otuscentos systemd[1]: Started The Apache HTTP Server.
[root@otuscentos ~]# systemctl status httpd@second
● httpd@second.service - The Apache HTTP Server
   Loaded: loaded (/etc/systemd/system/httpd@.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2024-02-23 17:01:37 UTC; 19s ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 3583 (httpd)
   Status: "Total requests: 0; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/system-httpd.slice/httpd@second.service
           ├─3583 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─3584 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─3585 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─3586 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─3587 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─3588 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           └─3589 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND

Feb 23 17:01:37 otuscentos systemd[1]: Starting The Apache HTTP Server...
Feb 23 17:01:37 otuscentos httpd[3583]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1. Set the 'ServerName' directive globally to suppress this message
Feb 23 17:01:37 otuscentos systemd[1]: Started The Apache HTTP Server.

[root@otuscentos ~]#  ss -tnulp | grep httpd
tcp    LISTEN     0      128    [::]:8080               [::]:*                   users:(("httpd",pid=3589,fd=4),("httpd",pid=3588,fd=4),("httpd",pid=3587,fd=4),("httpd",pid=3586,fd=4),("httpd",pid=3585,fd=4),("httpd",pid=3584,fd=4),("httpd",pid=3583,fd=4))
tcp    LISTEN     0      128    [::]:80                 [::]:*                   users:(("httpd",pid=3575,fd=4),("httpd",pid=3574,fd=4),("httpd",pid=3573,fd=4),("httpd",pid=3572,fd=4),("httpd",pid=3571,fd=4),("httpd",pid=3570,fd=4),("httpd",pid=3569,fd=4))
```







