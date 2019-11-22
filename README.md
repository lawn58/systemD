# systemD

Задание.
1. Написать сервис, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова. Файл и слово должны задаваться в /etc/sysconfig.
2. Из epel установить spawn-fcgi и переписать init-скрипт на unit-файл. Имя сервиса должно также называться.
3. Дополнить юнит-файл apache httpd возможностью запустить несколько инстансов сервера с разными конфигами

№1.

Для начала создаём файл с конфигурацией для сервиса в директории /etc/sysconfig - из неё сервис будет брать необходимые переменные.

cat /etc/sysconfig/watchlog
# Configuration file for my watchdog service
# Place it to /etc/sysconfig
# File and word in that file that we will be monit
WORD="ALERT"
LOG=/var/log/watchlog.log

Затем создаем /var/log/watchlog.log и пишем туда строки на своё усмотрение, плюс ключевое слово ‘ALERT’

Создадим скрипт:
cat /opt/watchlog.sh 
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

Команда logger отправляет лог в системный журнал

Создадим юнит для сервиса:
[Unit]
Description=My watchlog service
[Service]
Type=oneshot
EnvironmentFile=/etc/sysconfig/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG

Создадим юнит для таймера:
[Unit]
Description=Run watchlog script every 30 second
[Timer]
# Run every 30 second
OnUnitActiveSec=30
Unit=watchlog.service
[Install]
WantedBy=multi-user.target

Затем достаточно только стартануть timer:
systemctl start watchlog.timer
И убедиться в результате:
tail -f /var/log/messages



№2

Устанавливаем spawn-fcgi и необходимые для него пакеты:
yum install epel-release -y && yum install spawn-fcgi php php-cli mod_fcgid httpd -y

etc/rc.d/init.d/spawn-fcg - cам Init скрипт, который будем переписывать
Но перед этим необходимо раскомментировать строки с переменными в /etc/sysconfig/spawn-fcgi


Он должен получится следующего вида:
cat /etc/sysconfig/spawn-fcgi 
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.#
# See spawn-fcgi(1) for all possible options.#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -- /usr/bin/php-cgi"

А сам юнит файл будет примерно следующего вида:
cat /etc/systemd/system/spawn-fcgi.service
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


Убеждаемся что все успешно работает:
systemctl start spawn-fcgi
systemctl status spawn-fcgi



№3


Для запуска нескольких экземпляров сервиса будем использовать шаблон в конфигурации файла окружения: /usr/lib/systemd/system/httpd.service

[Unit]
Description=The Apache HTTP Server
After=network.target remote-fs.target nss-lookup.target
Documentation=man:httpd(8)
Documentation=man:apachectl(8)
[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/httpd-%I #!!!!!!добавим параметр %I сюда
ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND
ExecReload=/usr/sbin/httpd $OPTIONS -k graceful
ExecStop=/bin/kill -WINCH ${MAINPID}
KillSignal=SIGCONT
PrivateTmp=true
[Install]
WantedBy=multi-user.target

cp /usr/lib/systemd/system/httpd.service /etc/systemd/system
mv httpd.service httpd@first.service
cp /usr/lib/systemd/system/httpd.service /etc/systemd/system
mv httpd.service httpd@second.service


В самом файле окружения (которых будет два) задается опция для запуска веб-сервера с необходимым конфигурационным файлом:
# /etc/sysconfig/httpd-first
OPTIONS=-f conf/first.conf
# /etc/sysconfig/httpd-second
OPTIONS=-f conf/second.conf

Соответственно в директории с конфигами httpd (/etc/httpd/config) должны лежать два
конфига, в нашем случае это будут first.conf и second.conf

Для удачного запуска, в конфигурационных файлах должны быть указаныуникальные для каждого экземпляра опции Listen и PidFile. Конфиги можноскопировать и поправить только второй, в нем должны быть след опции:

PidFile     /var/run/httpd-second.pid
Listen 8080

То есть в /etc/httpd/conf должно лежить два файла first.conf и second.conf с разными листен портами и пид файлами

Этого достаточно для успешного запуска.


Запустим
systemctl start httpd@first
systemctl start httpd@second

Проверить можно несколькими способами, например посмотреть какиепорты слушаются:

ss -tnulp | grep httpd


