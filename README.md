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








