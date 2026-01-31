# Настройте бэкапы PostgreSQL с использованием WAL-G, pg_probackup или любого другого аналогичного ПО для базы данных "Лояльность оптовиков".
## Подготовка стэнда
Для начала создадим вм:
*конечно же делаем их с допонительным диском* 
```powershell
yc vpc network create --name my-yc-network-main --labels my-label=my-main-net --description "my main network in yc"
```
```powershell
yc vpc subnet create --name my-yc-subnet-a --zone ru-central1-a --range 192.168.50.0/24 --network-name my-yc-network-main --description "my first subnet via yc"
```
```powershell
$names = @("bananaflow", "bananaflow-remote")
foreach ($name in $names) {
    yc compute instance create --name $name --hostname $name --cores 4 --memory 8 --create-boot-disk size=15G,type=network-hdd,image-id=fd82kcha04mgqerskh8n --network-interface subnet-name=my-yc-subnet-a,nat-ip-version=ipv4 --ssh-key C:\Users\Admin\Downloads\ssh-key\pub.txt --core-fraction 20 --create-disk type=network-ssd,size=20,auto-delete
}
yc compute instance list

```
```bash
sudo apt update && sudo apt upgrade -y && sudo mkdir -p /etc/apt/keyrings && curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/keyrings/postgresql.gpg && sudo sh -c 'echo "deb [signed-by=/etc/apt/keyrings/postgresql.gpg] http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && sudo apt update && sudo apt install -y postgresql-18 &&  sudo locale-gen ru_RU ru_RU.UTF-8

sudo systemctl stop postgresql
sudo systemctl disable postgresql
sudo pg_dropcluster 18 main

```
```bash
# sudo fdisk /dev/vdb # g-n-enter-enter-enter-w
# echo -e "g\nn\n\n\n\nw\n" | sudo fdisk /dev/vdb
sudo parted /dev/vdb --script mklabel gpt mkpart primary 0% 100%
sudo mkfs.ext4 -q /dev/vdb1
sudo mkdir -p /mnt/vdb

UUID=$(sudo blkid -s UUID -o value /dev/vdb1); [ -z "$UUID" ] && echo "UUID не найден" && exit 1; echo "UUID=$UUID /mnt/vdb $(sudo blkid -s TYPE -o value /dev/vdb1) defaults 0 2" | sudo tee -a /etc/fstab >/dev/null
sudo mount -a
sudo systemctl daemon-reload

sudo mkdir -p /mnt/vdb/pgdata
sudo chmod 0750 /mnt/vdb/pgdata
sudo chown -R postgres:postgres /mnt/vdb/pgdata
```
```bash
sudo pg_createcluster -d /mnt/vdb/pgdata/ -l /mnt/vdb/pgdata/logs 18 main --locale=ru_RU.UTF-8 --encoding=UTF8
sudo systemctl start postgresql
sudo systemctl enable postgresql
sudo pg_lsclusters

```
## PG_PROBACKUP
*https://postgrespro.github.io/pg_probackup*  
*https://github.com/postgrespro/pg_probackup*  
*https://postgrespro.ru/docs/postgrespro/18/app-pgprobackup*  
- Установка
```bash
# 1. Установка необходимых инструментов
sudo apt install gpg wget
# 2. Добавление GPG-ключа репозитория pg_probackup
wget -qO - https://repo.postgrespro.ru/pg_probackup/keys/GPG-KEY-PG-PROBACKUP | sudo tee /etc/apt/trusted.gpg.d/pg_probackup.asc
# 3. Настройка переменных окружения для определения версии ОС
. /etc/os-release
# 4. Добавление репозитория с бинарными пакетами
echo "deb [arch=amd64] https://repo.postgrespro.ru/pg_probackup/deb $VERSION_CODENAME main-$VERSION_CODENAME" | sudo tee /etc/apt/sources.list.d/pg_probackup.list
# 5. ДОПОЛНИТЕЛЬНО: Добавление репозитория с исходным кодом
echo "deb-src [arch=amd64] https://repo.postgrespro.ru/pg_probackup/deb $VERSION_CODENAME main-$VERSION_CODENAME" | \
sudo tee -a /etc/apt/sources.list.d/pg_probackup.list
sudo apt update 
sudo apt install -y pg-probackup-18
```
- Предварительная настройка
```bash
# Создаем каталог для хранения резервных копий
sudo mkdir -p /mnt/vdb/backup
sudo chown postgres:postgres /mnt/vdb/backup
# Переключимся на postgres
sudo su postgres
# Инициализируем каталог для бакапов:
pg_probackup-18 init -B /mnt/vdb/backup
# Инициализируем инстанс кластера по его пути и назовем его 'main-backup' и определим что он будет хранить бакапы по выбранному пути
pg_probackup-18 add-instance --instance 'main-backup' -D /mnt/vdb/pgdata/ -B /mnt/vdb/backup
```
- backup на текущий сервер
- - Полный
```bash
# Делаем бэкап - первый раз всегда FULL
pg_probackup-18 backup --instance 'main-backup' -b FULL --stream --temp-slot -B /mnt/vdb/backup
# Или с указанием хоста. бд, но не вижу смысла добавлять их, так как это физическая копия кластера.
pg_probackup-18 backup --instance 'main-backup' -b FULL --stream --temp-slot -B /mnt/vdb/backup -U postgres -h localhost -d bananaflow -p 5432
```
- - Диффиринциальный  
**Не подерживается**
- - Инкрементальный
```bash
# Создадим изменёную копию (сложно мне назвать её diff или inc )
pg_probackup-18 backup --instance 'main-backup' -b DELTA --stream --temp-slot -B /mnt/vdb/backup
```
- backup на удаленный сервер  
*тут навреное помогут команды типа rsync*
- backup с удаленного сервера  
*После долгой возни с ключами ssh, для подключения с удалённого сервера к postgres-серверу*
```bash
pg_probackup-18 init -B /mnt/vdb/backups/
pg_probackup-18 add-instance --instance 'main-remote' -D /mnt/vdb/pgdata -B /mnt/vdb/backups --remote-user=postgres --remote-host=192.168.50.21 --remote-port=22
# не подключается к postgres
pg_probackup-18 backup --instance 'main-remote' -B /mnt/vdb/backups -b FULL --remote-user=postgres --remote-host=192.168.50.21 --remote-port=22
# Оба раза завершилось с ошибкой...

# подключается к postgres (так как стал ругаться на hba файл)
pg_probackup-18 backup --instance 'main-remote' -B /mnt/vdb/backups -b FULL --remote-user=postgres --remote-host=192.168.50.21 --remote-port=22 --stream --temp-slot
# завершился с первого раза успешно
```
- restotre локальный
```bash
## Останавливаем сервис БД  
sudo systemctl stop postgresql
## Удаляем данные 
sudo rm -rf /mnt/vdb/pgdata/ # почему-то со * он не удалил данные, потом может разберусь
sudo mkdir -p /mnt/vdb/pgdata
sudo chmod 0750 /mnt/vdb/pgdata
sudo chown -R postgres:postgres /mnt/vdb/pgdata

sudo su postgres
pg_probackup-18 restore --instance 'main-backup' -i 'T9P25W' -D /mnt/vdb/pgdata/ -B /mnt/vdb/backup
sudo systemctl start postgresql
```
- restore удалённый
```bash
# так как ssh доступ настроен на пользователя postgres менять владельца файлов не требовалось
pg_probackup-18 restore --instance 'main-remote' -D /mnt/vdb/pgdata/ -B /mnt/vdb/backups -i T9P8VK --remote-user=postgres --remote-host=192.168.50.21 --remote-port=22
# Заврешился успешно, но службу там придется запускать всё равно. 
```
- restore на определенную дату
- - Подготовка:
```bash
# Задайте для параметра wal_level значение выше minimal.
# archive_mode должен иметь значение on или always. для реплик требуется значение always. -> в таком случае лучше выставлять значение always
# Установите параметр archive_command:
archive_command = '"/usr/bin/pg_probackup-18" archive-push -B "/mnt/vdb/backup/" --instance main-backup --wal-file-name=%f --compress'
# для локального
archive_command = '"/usr/bin/pg_probackup-18" archive-push -B "/mnt/vdb/backup/" --instance main-remote --wal-file-name=%f --compress --remote-user=postgres --remote-host=192.168.50.27 --remote-port=22 --remote-path=/usr/bin/pg_probackup-18'
# это восстановление не протестировано ( может доберусь позже)
```
```bash
pg_probackup-18 restore --instance 'main-backup' -D /mnt/vdb/pgdata/ -B /mnt/vdb/backup --recovery-target-time="2026-01-31 08:32:09" --recovery-target-action=promote
# Востановление идет видимо в 2 этапа - сначала из бэкапа, а потом еще recovering и надо ждать применения wal файлов.
# --recovery-target-action=promote - для автоматического промота, а то придётся делать команду ниже...
SELECT pg_wal_replay_resume(); -> # 2026-01-31 08:37:09.978 UTC [20555] СООБЩЕНИЕ:  восстановление останавливается перед фиксированием транзакции 774, время 2026-01-31 08:32:27.665391+00

```
- Проверки
```bash
# Смотрим текущие настройки конкретного инстанса и каталога
pg_probackup-18 show-config --instance main-backup -B /mnt/vdb/backup
# Посмотрим на перечень бэкапов
pg_probackup-18 show -B /mnt/vdb/backup
# При неправильном пароле - он тоже создает бэкап со статусом ошибка
# Проверка архивации
SELECT * FROM pg_stat_archiver;
#Удаление архивных wal
pg_probackup-18 delete -B /mnt/vdb/backup/ --instance "main-backup" --delete-wal
# Проверка восстановления
SELECT pg_is_in_recovery();
#для удаленного бэкапа и восстановления изменял следующее
nano  /etc/postgresql/18/main/pg_hba.conf

host    all             postgres        192.168.50.0/24        trust
host    replication     all             192.168.50.0/24         trust

nano  /etc/postgresql/18/main/postgresql.conf

listen_addresses = '*'
sudo systemctl restart postgresql

sudo passwd postgres
sudo useradd -r -m -d /var/lib/postgresql -s /bin/bash postgres

на одном
cat /var/lib/postgresql/.ssh/id_ed25519.pub
на постргресе
echo "ВСТАВЬ_СЮДА_ПУБЛИЧНЫЙ_КЛЮЧ" | sudo tee -a /var/lib/postgresql/.ssh/authorized_keys

# Проверить целостность восстановленной базы данных.
pg_probackup checkdb -B каталог_копий --instance имя_экземпляра -D каталог_данных

# Проверить целостность резервных копий без восстановления.
pg_probackup validate -B каталог_копий

# Объединить цепочку инкрементальных копий в одну полную.
pg_probackup merge -B каталог_копий --instance имя_экземпляра -i ид_резервной_копии

```
## WAL-G
*https://github.com/wal-g/wal-g*
*Так как готовые покеты я вижу только для ubuntu, а хочется какой-то универсальности хотя бы на debian, буду собирать из исходников*
- Подготовка
```bash
# Установить последний компилятор Go
sudo add-apt-repository ppa:longsleep/golang-backports
sudo apt update
sudo apt install -y golang-go
# Установить зависимости библиотек
sudo apt install libbrotli-dev liblzo2-dev libsodium-dev curl cmake
# Получить проект и собрать
# Для Go 1.15 и ниже
# go get github.com/wal-g/wal-g
# Для Go 1.16+ - просто клонировать репозиторий в $GOPATH
# если нужно сэкономить место, добавьте --depth=1 или --single-branch
git clone https://github.com/wal-g/wal-g $(go env GOPATH)/src/github.com/wal-g/wal-g
cd $(go env GOPATH)/src/github.com/wal-g/wal-g
# Опциональные экспорты (см. выше)
export USE_BROTLI=1 # Чтобы собрать с компрессором и декомпрессором brotli
export USE_LIBSODIUM=1 # Чтобы собрать с поддержкой libsodium
export USE_LZO=1 # Чтобы собрать с декомпрессором lzo
make deps
make pg_build
main/pg/wal-g --version 
# ldd ./wal-g - проверка зависимостей, если есть not found нужно доустановить.
sudo cp $(go env GOPATH)/src/github.com/wal-g/wal-g/main/pg/wal-g /usr/local/bin/wal-g
#  sudo cp ./go/src/github.com/wal-g/wal-g/main/pg/wal-g /usr/local/bin/wal-g
sudo chmod ugo+x /usr/local/bin/wal-g
# Создаем каталог для хранения резервных копий и сделаем владельцем пользователя postgres
sudo mkdir -p /mnt/vdb/backup && sudo chown -R postgres:postgres /mnt/vdb/backup
# Создать настроечный файл для wal-g под текущим пользователем = в /var/lib/postgresql/.walg.json
sudo su postgres
nano ~/.walg.json
# Заполняем настройками
{
    "WALG_FILE_PREFIX": "/mnt/vdb/backup",
    "WALG_COMPRESSION_METHOD": "brotli",
    "WALG_DELTA_MAX_STEPS": "5",
    "WALG_UPLOAD_DISK_CONCURRENCY": "4",
    "PGDATA": "/mnt/vdb/pgdata",
    "PGHOST": "localhost"
}
PGHOST - предпочтительно указывать сокет
# Подготовка сервера postgres:
# wal_level=replica
# archive_mode=on
# archive_timeout=60
archive_command='/usr/local/bin/wal-g wal-push \"%p\" >> $PGDATA/log/archive_command.log 2>&1' 
restore_command='/usr/local/bin/wal-g wal-fetch \"%f\" \"%p\" >> $PGDATA/log/restore_command.log 2>&1'
nano  /etc/postgresql/18/main/pg_hba.conf

host all all 127.0.0.1/32 trust

sudo systemctl restart postgresql
```
- Резервная копия локальная
```bash
# Использует переменные postgres для подключения
wal-g --config=/var/lib/postgresql/.walg.json backup-push /mnt/vdb/pgdata
wal-g backup-push /mnt/vdb/pgdata
```
*Тут инкрементальные копии делаются в зависимости от json файла*
- Резервная копия удалённая  
*не нашёл возможности использовать*
- Восстановление локально
```bash
# пришлось править postgres.conf
recovery_target_timeline = 'current' # '1'
recovery_target_action = 'pause' # это если вручную промоутить
# Почему-то при восстановлении postgres видит много новых timeline, примрно равных кол-ву архивов wal-g. 
sudo systemctl stop postgresql
sudo su postgres
rm -rf /mnt/vdb/pgdata/*
wal-g backup-fetch /mnt/vdb/pgdata/ LATEST # либо backup_name из wal-g backup list
touch /mnt/vdb/pgdata/recovery.signal
exit
sudo systemctl start postgresql

```
- Проверки:
```bash
# Список бэкапов
wal-g backup-list 
wal-g backup-list --detail --json | jq .
wal-g wal-show
wal-g wal-show --detailed-json | jq .
```
## Pgbackrest
*https://pgbackrest.org/user-guide.html#installation*  
*https://github.com/pgbackrest/pgbackrest*  
*чем-то напоминает probackup*
- Подготовка
```bash
# Устанавливаем необходимые для сборки зависимости
sudo apt update
sudo apt install -y meson gcc libpq-dev libssl-dev libxml2-dev pkg-config liblz4-dev libzstd-dev libbz2-dev libz-dev libyaml-dev libssh2-1-dev
# Скачиваем архив с pgbackrest
export PGBACKREST_VERSION="2.58.0"
sudo wget -q -O - "https://github.com/pgbackrest/pgbackrest/archive/refs/tags/release/${PGBACKREST_VERSION}.tar.gz" | sudo tar zx -C ./
# Собираем pgbackrest
sudo meson setup ./pgbackrest-$PGBACKREST_VERSION ./pgbackrest-release-$PGBACKREST_VERSION
sudo ninja -C ./pgbackrest-$PGBACKREST_VERSION
sudo cp ./pgbackrest-$PGBACKREST_VERSION/src/pgbackrest /usr/bin
# sudo apt install perl
# Подготовка папок
sudo mkdir -p -m 770 /var/log/pgbackrest
# sudo chown pgbackrest:pgbackrest /var/log/pgbackrest
sudo mkdir -p /etc/pgbackrest
sudo mkdir -p /etc/pgbackrest/conf.d
sudo touch /etc/pgbackrest/pgbackrest.conf
sudo chmod 640 /etc/pgbackrest/pgbackrest.conf
# sudo chown pgbackrest:pgbackrest /etc/pgbackrest/pgbackrest.conf
```
- - на другом сервере можно заного его создать, но я не очень люблю ставить кучу зависимостей, особенно если это будет прод сервер.
- - поэтому я туда просто скачал этот файл
```bash
# на постгресе нужно добавить к установке
# проверяю зависимости ldd pgbackrest
sudo apt install -y libssh2-1
sudo mkdir -p -m 770 /var/log/pgbackrest
sudo mkdir -p /etc/pgbackrest
sudo mkdir -p /etc/pgbackrest/conf.d
sudo touch /etc/pgbackrest/pgbackrest.conf
sudo chmod 640 /etc/pgbackrest/pgbackrest.conf
sudo chown postgres:postgres /var/log/pgbackrest
sudo chown postgres:postgres /etc/pgbackrest/pgbackrest.conf
```
- Настройка SSH доступа
```bash
# на удаленном сервере
ssh-keygen -f ~/.ssh/id_rsa -t rsa -b 4096 -N ""
# на сервере postgres
sudo -u postgres mkdir -m 750 -p /var/lib/postgresql/.ssh
sudo -u postgres ssh-keygen -f /var/lib/postgresql/.ssh/id_rsa -t rsa -b 4096 -N ""
# Копируем публичный ключ postgres сервера на сервер-репозиторий:
(echo -n 'no-agent-forwarding,no-X11-forwarding,no-port-forwarding,' && \
       echo -n 'command="/usr/bin/pgbackrest ${SSH_ORIGINAL_COMMAND#* }" ' && \
       sudo ssh root@192.168.50.16 cat /var/lib/postgresql/.ssh/id_rsa.pub) | \
       tee -a .ssh/authorized_keys
# Копируем публичный ключ репозитория на сервер с postgres:
(echo -n 'no-agent-forwarding,no-X11-forwarding,no-port-forwarding,' && \
       echo -n 'command="/usr/bin/pgbackrest ${SSH_ORIGINAL_COMMAND#* }" ' && \
       sudo ssh root@192.168.50.24 cat ~/.ssh/id_rsa.pub) | \
       sudo -u postgres tee -a /var/lib/postgresql/.ssh/authorized_keys
```
```bash
# открываем доступ для подключения
sudo nano  /etc/postgresql/18/main/postgresql.conf
listen_addresses = '*'
sudo nano  /etc/postgresql/18/main/pg_hba.conf

host    all             all             192.168.50.0/24         trust

ALTER SYSTEM SET archive_command = 'pgbackrest --stanza=main archive-push %p';
ALTER SYSTEM SET archive_mode = 'on';
ALTER SYSTEM SET max_wal_senders = '6';
ALTER SYSTEM SET wal_level = 'replica';

sudo systemctl restart postgresql
```
```bash
# Внесем необходимые настройки в файл конфигурации pgbackrest (/etc/pgbackrest/pgbackrest.conf) на сервере postgres
[main]
pg1-path=/mnt/vdb/pgdata

[global]
log-level-file=detail
repo1-host=192.168.50.24
repo1-host-user=yc-user
```
```bash
# Внесем необходимые настройки в файл конфигурации pgbackrest (/etc/pgbackrest/pgbackrest.conf) на бэкапном сервере
[main]
pg1-host=192.168.50.16
pg1-path=/mnt/vdb/pgdata

[global]
repo1-path=/mnt/vdb/backup
repo1-retention-full=2 # Параметр, указывающий сколько хранить полных бэкапов. Т.е. если у вас есть два полных бэкапа и вы создаете третий, то самый старый бэкап будет удален. Можно произносить как "хранить не более двух бэкапов" - по аналогии с ротациями логов.
start-fast=y # Начинает резервное копирование немедленно, прочитать про этот параметр можно тут https://postgrespro.ru/docs/postgrespro/9.5/continuous-archiving
```
```bash
# Создаем новое хранилище для кластера main
sudo mkdir -m 770 /mnt/vdb/backup
sudo chown -R yc-user /mnt/vdb/backup
sudo chown -R yc-user /etc/pgbackrest/
sudo chown -R yc-user /var/log/pgbackrest
sudo -u pgbackrest pgbackrest --stanza=main stanza-create
```
- Создаем бэкап ( тут сразу настраивался удалённый)
```bash
pgbackrest --stanza=main backup
pgbackrest --stanza=main --type=full backup
pgbackrest --stanza=main --type=full --log-level-console=info backup # с выводом в консоль
```
- Восстановление
```bash
# Останавливаем работающий кластер:
# sudo pg_ctlcluster 18 main stop
sudo systemctl stop postgresql
# Восстанавливаемся из бэкапа:
sudo -u postgres pgbackrest --stanza=main --log-level-console=info --delta --recovery-option=recovery_target=immediate --target-action=promote restores
# Чтобы восстановить базу в состояние последнего ПОЛНОГО бэкапа используйте команду без указания recovery_target:
# sudo -u postgres pgbackrest --stanza=main --log-level-console=info --delta restore
sudo systemctl start postgresql
sudo -u postgres psql -c "select pg_wal_replay_resume()"
```
- Проверки
```bash
# Проверка установленого pgbackrest
pgbackrest version
# Проверка насроенного соединения по ssh
sudo -u postgres ssh yc-user@192.168.50.24 pgbackrest version
ssh postgres@192.168.50.16 pgbackrest version
# Проверка готовности 
sudo -u postgres pgbackrest --stanza=main --log-level-console=info check
pgbackrest --stanza=main --log-level-console=info check
# ПРоверка бэкапов
ls /mnt/vdb/backup/backup/main/ 
SELECT pg_is_in_recovery();
```

## pg_basebackup
```
Один из самый простых инструментов бэкапов. 
Поставляется вместе с сервером postgres, как и pg_dump. 
с 17 версии поддерживается инкрементальный бэкап
думаю дополнить эту часть в будущем
```
# Дополнительно: Снимите бэкап под нагрузкой с реплики.
*может позже доберусь до этого*

```powershell
$names = @("bananaflow", "bananaflow-remote")
foreach ($name in $names) {
    yc compute instance delete $name
}
yc vpc subnet delete my-yc-subnet-a
yc vpc network delete my-yc-network-main

```

