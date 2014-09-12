Title: Мой "велосипед" для бэкапа веб-сервера
Date: 2014-04-29 13:45
Author: svfat
Category: Сервер
Slug: moj-velosiped-dlya-be-kapa-veb-servera

Мне было лениво было разбираться с специальным софтом для бэкапа типа
backula, amanda и т.д., и я написал этот небольшой bash-скрипт,
незатейливо названный мною "packub", для инкрементального бэкапа файлов
и баз данных сайтов, созданных в ISPConfig 3, в дополнение к имеющейся
системе бэкапа этой панели управления.  Конечно, данный скрипт может
быть настроен для бэкапа сайтов созданных с помощью других панелей, или
вовсе без них. Единственным условием является однообразие структуры
директорий каждого сайта. То есть, все сайты находятся, например, в
/var/www, для каждого сайта создана поддиректория вида example.com,
файлы сайта в поддиректории example.com/web. Но, даже если это и не так,
то этого можно легко добиться с помощью симлинков)  
<!--more-->

В своей работе скрипт использует утилиту rdiff-backup для формирования
собственно инкрементальных бэкапов. И, по-сути, данный скрипт является
оболочкой на ним, заставляя обрабатывать директории, указанные нами для
бэкапа, а также делая дампы баз данных.

Итак, вот такой вот код, его актуальная версия также лежит здесь
<https://github.com/svfat/packub>:

~~~~ {lang="sh"}
#!/bin/bash
#
# packub - bash script for backing up websites and databases
# Author: Stanislav Fateev, 2014
# Email: fateevstas@yandex.ru
# WWW: svfat.ru
#
# requires bash version>=4 and rdiff-backup to run
# strongly recommended to run with superuser rights

#mysql user credentials
DBLOGIN="root"
DBPASSWORD="******************"

#directories (don't forget about slashes)
BACKUP_TO="/home/user/backup/"
DIR_PREFIX="/var/www/"
DIR_POSTFIX="/web/"

#declaring bash associative array as key:value pairs 'site':'database'
declare -A aa

#edit this
aa['yandex.ru']='yandex_database'
aa['google.com']='db_google'
# end of declaration

# test enviroment
if [ ! ${BASH_VERSION%%[^0-9]*} -ge 4 ];
then
echo >&2 "I require bash version>=4 but it's not installed. Aborting."
exit 1
fi

command -v rdiff-backup >/dev/null 2>&1 || { echo >&2 "I require rdiff-backup but it's not installed. Aborting."; exit 1; }

#check if backup directory is exists
if [ ! -d "$BACKUP_TO" ]; then
mkdir $BACKUP_TO
echo "Created directory $BACKUP_TO"
fi

for site in "${!aa[@]}"
do
OUTPUT_DIR=$BACKUP_TO$site
INPUT_DIR=$DIR_PREFIX$site$DIR_POSTFIX
LOGFILE=$OUTPUT_DIR/$site.log
# check if directory we want to backup is exists
if [ -d "$INPUT_DIR" ]; then
# check if output directory is exists - if not, create it
if [ ! -d "$OUTPUT_DIR" ]; then
mkdir $OUTPUT_DIR
echo "Created directory $OUTPUT_DIR" | tee -a $LOGFILE
fi

#make incremental backup and log
echo $'\n\n' >> $LOGFILE
echo "Starting backup for $site. Wait please." | tee -a $LOGFILE
database=${aa[$site]}
databasefile=$INPUT_DIR$database.sql
echo "Dumping database in $databasefile" | tee -a $LOGFILE
mysqldump -u$DBLOGIN -p$DBPASSWORD $database -v --single-transaction 2>&1 1>$databasefile | tee -a $LOGFILE
rdiff-backup --print-statistics $INPUT_DIR $OUTPUT_DIR$DIR_POSTFIX 2>&1 | tee -a $LOGFILE
else
# no input directory
echo "Directory $INPUT_DIR not found. Skipping."
fi
done
~~~~

Для правильной работы, замените пароль к руту mysql в переменной
DBPASSWORD, имена директорий в переменных BACKUP\_TO, DIR\_PREFIX,
DIR\_POSTFIX, и настройте массив aa добавив строки вида
aa['example.com']='db\_example', где example.com имя директории где
находятся файлы сайта, то есть, то что между DIR\_PREFIX и DIR\_POSTFIX,
а db\_example - имя используемой базы данных. В моей системе BACKUP\_TO
настроен на папку дропбокса - таким образом бэкапы автоматически
заливаются в облако.

Устанавливаем [rdiff-backup](http://www.nongnu.org/rdiff-backup/) любым
способом, подходящим для вашей системы.  
Кидаем этот скрипт например в \~/bin/packub.sh (Помните, что в нем в
открытом виде храниться пароль от рута mysql, и следует владельцем файла
сделать пользователя root, поставив затем права доступа типа 700)

После запуска, если все пройдет нормально, должны создаться необходимые
директории в которые забэкапятся файлы с базами. Также в каждой
директории должен быть создан log-файл с кратким отчетом.

Я засунул этот скрипт в cron, и как я уже указал выше, настроил его на
папку дропбокса. Теперь в мой дропбокс еженощно валятся инкрементальные
бэкапы, успокаивая мою нервную систему.

Возможо кому-нибудь мои наработки пригодятся, буду рад комментариям,
форкам и pull-реквестам.

 
