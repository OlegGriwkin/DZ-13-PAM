Запретить всем пользователям, кроме группы admin логин в выходные(суббота и воскресенье), без учета праздников
Необходимо решить задачу по ограничению доступа пользователей в систему по ssh.
● ”day” - имеет удаленный доступ каждый день с 8 до 20;
● “night” - с 20 до 8;
● “friday” - в любое время, если сегодня пятница.

Cоздадим 3х пользователей
```sudo useradd day```
```sudo useradd night```
```sudo useradd friday```

Назначим им пароли:
```echo "otus2022"| sudo passwd --stdin day```
```echo "otus2022"| sudo passwd --stdin night```
```echo "otus2022"| sudo passwd --stdin friday```

Чтобы быть уверенными, что на стенде разрешен вход через ssh по
паролю выполним:
```sudo bash -c "sed -i 's/^PasswordAuthentication.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config && systemctl restart sshd.service"```

Внесем изменения в /etc/security/time.conf для настройки доступа пользователя с учетом времени.
```vi /etc/security/time.conf```
*;*;day;Al0800-2000
*;*;night;!Al0800-2000
*;*;friday;Fr


ВАР 1. Приведем файл /etc/pam.d/sshd к виду:
```vi /etc/pam.d/sshd```
account required pam_nologin.so
account required pam_time.so 


После чего в отдельном терминале можно проверить доступ к серверу по ssh для созданных пользователей. 
Пользователь day    - успешно авторизовался
Пользователь night  - выдает ошибку: remote side unexpectedly closed network connection
Пользователь friday - выдает ошибку: access denied (проверял в среду)

ВАР 2. Выполнить при подключении пользователя скрипт.
Приведем файл /etc/pam.d/sshd к виду:
```vi /etc/pam.d/sshd```
account required pam_nologin.so
account required pam_exec.so /usr/local/bin/test_login.sh

Создадим скрипт:
```vi /usr/local/bin/test_login.sh```
#!/bin/bash
if [ $PAM_USER = "friday" ]; then
 if [ $(date +%a) = "Fri" ]; then
 exit 0
 else
 exit 1
 fi
fi
hour=$(date +%H)
is_day_hours=$(($(test $hour -ge 8; echo $?)+$(test $hour -lt 20; echo $?)))
if [ $PAM_USER = "day" ]; then
 if [ $is_day_hours -eq 0 ]; then
 exit 0
 else
 exit 1
 fi
fi
if [ $PAM_USER = "night" ]; then
 if [ $is_day_hours -eq 1 ]; then
 exit 0
 else
 exit 1
 fi
fi
EOF

Даем права на исполнение скрипта:
```chmod 777 /usr/local/bin/test_login.sh```
Проверить права:
```ls -l file /usr/local/bin/test_login.sh```

Подключим репозиторий и установим pam_script:
```for pkg in epel-release pam_script; do yum install -y $pkg; done```

Переименовать pam_exec в pam_script:
```vi /etc/pam.d/sshd```
Account required pam_exec.so /usr/local/bin/test_login.sh заменить на Account required pam_script.so /usr/local/bin/test_login.sh

Установим пакет nmap-ncat(CentOS).
yum install nmap-ncat -y

Войдем на стендовую машину под пользователем day и попробуем выполнить команду nc и получим сообщение об ошибке. 
[day@PAM ~]$ ncat -l -p 80
Ncat: bind to :::80: Permission denied. QUITTING. # непривелигерованный пользователь day не может открыть для прослушивания 80й порт.





