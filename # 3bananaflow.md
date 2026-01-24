# 1 Создайте виртуальную машину с Ubuntu и установите PostgreSQL.
После настройки cli на компьютере https://yandex.cloud/ru/docs/cli/operations/install-cli (у меня почему-то из консоли не отображались стандартные образы)
- Нужно создать инфраструктуру:
```bash
yc vpc network create --name my-yc-network-main --labels my-label=my-main-net --description "my main network in yc"
```
- Нужно созджать подсеть для ВМ:
```bash
yc vpc subnet create --name my-yc-subnet-a --zone ru-central1-a --range 192.168.50.0/24 --network-name my-yc-network-main --description "my first subnet via yc"
```
- Создание самой ВМ:
```bash
yc compute instance create --name bananaflow --hostname bananaflow --cores 2 --memory 4 --create-boot-disk size=15G,type=network-hdd,image-id=fd82kcha04mgqerskh8n --network-interface subnet-name=my-yc-subnet-a,nat-ip-version=ipv4 --ssh-key C:\Users\Admin\Downloads\ssh-key\pub.txt --core-fraction 20
```
- Подключение:
```bash
ssh -i C:\Users\Admin\Downloads\ssh-key\priv-openssh yc-user@158.160.58.162
```

- Установка PostgreSQL
```bash
sudo apt update && sudo apt upgrade -y && sudo mkdir -p /etc/apt/keyrings && curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/keyrings/postgresql.gpg && sudo sh -c 'echo "deb [signed-by=/etc/apt/keyrings/postgresql.gpg] http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && sudo apt update && sudo apt install -y postgresql
```

# 2 Создайте таблицу с данными о перевозках.
```sql
-- 1. Создаем базу данных:
CREATE DATABASE bananaflow;

-- 2. Переключаемся на созданную БД:
\c bananaflow;

-- 3. Создаем таблицу:
CREATE TABLE shipments (
    id SERIAL PRIMARY KEY,
    cargo VARCHAR(100) NOT NULL,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    destination VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 4. Создаем функцию для генерации случайных данных:
-- для груза:
CREATE OR REPLACE FUNCTION random_cargo() RETURNS VARCHAR AS $$
DECLARE
    cargos VARCHAR[] := ARRAY['Бананы', 'Кофе', 'Какао-бобы', 'Сахар', 'Рис', 
                              'Пшеница', 'Кукуруза', 'Чай', 'Хлопок', 'Дерево',
                              'Сталь', 'Нефть', 'Газ', 'Уголь', 'Автомобили',
                              'Электроника', 'Одежда', 'Медикаменты', 'Мебель', 'Строительные материалы'];
BEGIN
    RETURN cargos[floor(random() * array_length(cargos, 1) + 1)];
END;
$$ LANGUAGE plpgsql;

-- для адреса назначения:
CREATE OR REPLACE FUNCTION random_destination() RETURNS VARCHAR AS $$
DECLARE
    destinations VARCHAR[] := ARRAY['Москва', 'Санкт-Петербург', 'Новосибирск', 'Екатеринбург', 'Казань',
                                   'Нижний Новгород', 'Челябинск', 'Омск', 'Самара', 'Ростов-на-Дону',
                                   'Уфа', 'Красноярск', 'Воронеж', 'Пермь', 'Волгоград',
                                   'Хабаровск', 'Владивосток', 'Калининград', 'Сочи', 'Краснодар'];
BEGIN
    RETURN destinations[floor(random() * array_length(destinations, 1) + 1)];
END;
$$ LANGUAGE plpgsql;

-- 5. Заполняем таблицу 1,000,000 строк:
INSERT INTO shipments (cargo, quantity, destination)
SELECT 
    random_cargo(),
    (random() * 1000 + 1)::INTEGER, -- количество от 1 до 1000
    random_destination()
FROM generate_series(1, 1000000);

-- 6. Создаем индексы для оптимизации запросов:
CREATE INDEX idx_shipments_cargo ON shipments(cargo);
CREATE INDEX idx_shipments_destination ON shipments(destination);
CREATE INDEX idx_shipments_quantity ON shipments(quantity);

-- 7. Проверяем количество строк:
SELECT COUNT(*) FROM shipments;

count  
---------
 1000000
(1 row)

```

# 3 Добавьте внешний диск к виртуальной машине и перенесите туда базу данных.

- Создание диска:
```bash
yc compute disk create --name bananaflow_disk_2 --type network-ssd --size 10 --description "second disk for bananaflow"
```
- Подключение диска:
```bash
yc compute instance attach-disk bananaflow --disk-name bananaflow_disk_2 --mode rw --auto-delete
```
- Создадим разделы с помощью fdisk
```bash
sudo fdisk /dev/vdb
```
 *тут я сначала указал создание gpt раздела g, n , остальные значения оставил по-умолчанию Enter*

- Отформатирует в нужную файловую систему:
```bash
sudo mkfs.ext4 /dev/vdb1
```
- Смотриуем файловую систему:
```bash 
sudo mkdir /mnt/vdb1
sudo mount /dev/vdb1 /mnt/vdb1

или

sudo mkdir -p /mnt/vdb
UUID=$(sudo blkid -s UUID -o value /dev/vdb1); [ -z "$UUID" ] && echo "UUID не найден" && exit 1; echo "UUID=$UUID /mnt/vdb $(sudo blkid -s TYPE -o value /dev/vdb1) defaults 0 2" | sudo tee -a /etc/fstab >/dev/null
sudo mount -a
sudo systemctl daemon-reload
```
*```df -h ``` - Должен показать новый раздел*
- Создадим там папку для данных и дадим права пользователю postgres:
```bash
sudo mkdir -p /mnt/vdb/pgdata
sudo chown postgres:postgres /mnt/vdb/pgdata
```

- ## Перенос данных вариант с созданием табличного пространства:
- Создание табличного пространства:
```sql
CREATE TABLESPACE tablespace_pgdata LOCATION '/mnt/vdb/pgdata';
```
- - Можно скопировать туда базу данных с новым именем:
```sql
CREATE DATABASE bananaflow_pgdata WITH TEMPLATE bananaflow TABLESPACE tablespace_pgdata;
```
*Нужно убрать все подключения к исхъодной БД*  
``` SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = 'bananaflow' AND pid <> pg_backend_pid(); ```  
   
  
- - Так же тут можно использовать pg_dump/pg_restore: 
```sql
CREATE DATABASE bananaflow_pgdata WITH TABLESPACE tablespace_pgdata;
```
```bash
pg_dump -d bananaflow -Fc | pg_restore -d banaflow_pgdata
```
 
- Через pg_basebackup:
```bash
# Создаем копию данных (так как у меня было создано табличное пространство, требуется ему переопределить место)
sudo -u postgres pg_basebackup -D /mnt/vdb/pgdata/main -X stream -c fast --tablespace-mapping=/mnt/vdb/pgdata=/mnt/vdb/pgdata/pgdata
# Останавливаем postgres
sudo systemctl stop postgresql
# Редактируем файл postgres.conf
sudo sed -i "s|^data_directory =.*|data_directory = '/mnt/vdb/pgdata'|" /etc/postgresql/15/main/postgresql.conf
# Запускаем postgres
sudo systemctl start postgresql
```
*проверка через ```pg_lscluster, systemctl status postgresql@18-main``` показывает работоспосбность кластера
*зашёл в бд и увидел базу там*

# 4 Настройте PostgreSQL для работы с новым диском.  
*так как уже настраивали табличное пространство, тут я буду инициализировать postgres на новый диск*
*пересоздам вм*
```bash
yc compute instance create --name bananaflow --hostname bananaflow --cores 2 --memory 4 --create-boot-disk size=15G,type=network-hdd,image-id=fd82kcha04mgqerskh8n --network-interface subnet-name=my-yc-subnet-a,nat-ip-version=ipv4 --ssh-key C:\Users\Admin\Downloads\ssh-key\pub.txt --core-fraction 20 --create-disk type=network-ssd,size=20,auto-delete
```
```bash
sudo apt update && sudo apt upgrade -y && sudo mkdir -p /etc/apt/keyrings && curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/keyrings/postgresql.gpg && sudo sh -c 'echo "deb [signed-by=/etc/apt/keyrings/postgresql.gpg] http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && sudo apt update && sudo apt install -y postgresql
```
*создание 2 диска и настройка его для пользователя postgres - выше ↑*  
*проверяем текущие кластера ```sudo pg_lsclusters```*  
*удаляем```sudo systemctl stop postgresql  && sudo pg_dropcluster 18 main```*
- Создание postgres в нужном месте
```bash
sudo -u postgres pg_createcluster 18 main --datadir=/mnt/vdb/pgdata --port=5432 --locale=ru_RU.UTF-8 --encoding=UTF8
sudo pg_ctlcluster 18 main start
sudo systemctl enable postgresql@18-main
```

# 5 Проверьте, что данные сохранились и доступны.

После 3 шага базы и в табличном пространстве pg_default и tablespace_pgdata проверены. всё работает. 



- различиные
```bash
yc compute disk-type list
yc compute disk list
```
```bash
yc compute instance delete bananaflow
yc vpc subnet delete my-yc-subnet-a
yc vpc network delete my-yc-network-main
```