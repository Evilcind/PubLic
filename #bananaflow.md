# 1 Создание проекта
После настройки cli на компьютере https://yandex.cloud/ru/docs/cli/operations/install-cli (у меня почему-то из консоли не отображались стандартные образы)
- Нужно создать сеть:
```bash
yc vpc network create --name my-yc-network-main --labels my-label=my-main-net --description "my main network in yc"
```
- Нужно созджать подсеть для ВМ:
```bash
yc vpc subnet create --name my-yc-subnet-a --zone ru-central1-a --range 192.168.50.0/24 --network-name my-yc-network-main --description "my first subnet via yc"
```
- Создание самой ВМ:
```bash
yc compute instance create --name bananaflow-19920101 --hostname bananaflow --cores 2 --memory 4 --create-boot-disk size=15G,type=network-hdd,image-id=fd82kcha04mgqerskh8n --network-interface subnet-name=my-yc-subnet-a,nat-ip-version=ipv4 --ssh-key C:\Users\Admin\Downloads\ssh-key\pub.txt --core-fraction 20
```
> *--core-fraction 20* - стоит этот параметр для удешвеления ВМ

Предположим, что получили следующую ВМ
Ubuntu 24.04 LTS с 
ip 192.168.50.12

# 2 Настройка SSH-доступа

- Добавьте свой SSH-ключ к виртуальной машине:
> Часть этого задания выполнена при создании ВМ *--ssh-key C:\Users\Admin\Downloads\ssh-key\pub.txt* добавления публичного ключа в создаваемою машину. Осталось только в агента записать приватный ключ. Пара ключей для этого предварительно сгенерирована.

```bash
ssh -i C:\Users\Admin\Download\ssh-key\priv yc-user@192.168.50.10
``` 

# 3 Установка PostgreSQL

- Установите PostgreSQL из пакетов с помощью команды apt install:
```bash
sudo apt update
sudo apt upgrade
sudo apt install -y postgresql
```

# 4 Подключение к PostgreSQL
- Откройте вторую SSH-сессию и подключитесь к PostgreSQL через psql от имени (а где по заданию открытие 1 ссесии?):

> В разных окнах терминала выполняем
```bash
sudo su postgres -c psql
или
sudo -u postgres psql
``` 

# 5 Работа с транзакциями
- Выключите автоматический коммит (auto commit) в обеих сессиях:
>Для начал создам новую БД:  
>```sql
>CREATE DATABASE bananaflow;  
>```
>И переключимся в неё:
>```sql
>\c bananaflow
>```

```sql
\set AUTOCOMMIT off
```
- В первой сессии создайте таблицу shipments (перевозки) и добавьте в неё данные:
```sql
create table shipments(id serial, product_name text, quantity int, destination text);
insert into shipments(product_name, quantity, destination) values('bananas', 1000, 'Europe');
insert into shipments(product_name, quantity, destination) values('coffee', 500, 'USA');
commit;
```
# 6 Изучение уровней изоляции

- Проверьте текущий уровень изоляции с помощью команды:
```sql
show transaction isolation level;
```
*read committed*

- Начните новую транзакцию в обеих сессиях с уровнем изоляции по умолчанию:
- - В первой сессии добавьте новую запись:
```sql 
insert into shipments(product_name, quantity, destination) values('sugar', 300, 'Asia');
```
- - Во второй сессии выполните:
```sql
select * from shipments;
```

- Видите ли вы новую запись? Объясните, почему да или нет.  
*Нет*  
*На уровне `READ COMMITTED` другие транзакции **не видят незакоммиченные изменения** (dirty reads запрещены). Поэтому вставка из сессии 1 не видна, пока не будет `COMMIT`*

- Завершите первую транзакцию:
```sql
commit;
```
- и во второй ссесии снова выполните:
```sql 
select * from shipments;
```
- Видите ли вы новую запись теперь?  
*Да*  
*Так как транзакция в первой ссесии завершилась commit - повторное чтение таблицы включило её в вывод*

# 7 Эксперименты с уровнем изоляции Repeatable Read
- Начните новые транзакции в обеих сессиях с уровнем изоляции repeatable read:
```sql
set transaction isolation level repeatable read;
```
- - В первой сессии добавьте новую запись:
```sql
insert into shipments(product_name, quantity, destination) values('bananas', 2000, 'Africa');
```
- - Во второй сессии выполните:
```sql
select * from shipments;
```
- Видите ли вы новую запись?  
*Нет*  
*На уровне REPEATABLE READ (как и на READ COMMITTED) действует правило: dirty reads запрещены.*

- Завершите первую транзакцию:
```sql
commit;
```
- и во второй ссесии снова выполните:
```sql 
select * from shipments;
```
- Видите ли вы новую запись?  
*Нет*  
*REPEATABLE READ Транзакция будет видеть одни и те же данные на протяжении всей транзакции* 

- Завершите вторую транзакцию
```sql
commit;
```
- И снова выполните во второй ссесии:
```sql 
select * from shipments;
```
- Видите ли вы новую запись?
*Да*  
*Так как во второй ссесии это уже новая транзакция, то данные берутся обновлённые и выводятся все закоммиченные транзакции*