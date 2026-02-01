# Разверните инстанс PostgreSQL на виртуальной машине в Яндекс.Облаке или любом другом месте..
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
$names = @("bananaflow")
foreach ($name in $names) {
    yc compute instance create --name $name --hostname $name --cores 8 --memory 16 --create-boot-disk size=15G,type=network-hdd,image-id=fd82kcha04mgqerskh8n --network-interface subnet-name=my-yc-subnet-a,nat-ip-version=ipv4 --ssh-key C:\Users\Admin\Downloads\ssh-key\pub.txt --create-disk type=network-ssd,size=20,auto-delete
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
# Протестируйте производительность с помощью pgbench.
```bash
sudo su postgres
# Инициализация 
pgbench -i postgres
# Запуск бенчмарка для 80 клиентов в 4 потока на 3 минуты с отчетом каждые 30 сек
pgbench -c 80 -j 4 -P 30 -T 180 postgres
```

transaction type: <builtin: TPC-B (sort of)>  
scaling factor: 1  
query mode: simple  
number of clients: 80  
number of threads: 4  
maximum number of tries: 1  
duration: 180 s  
number of transactions actually processed: 93013  
number of failed transactions: 0 (0.000%)  
latency average = 154.884 ms  
latency stddev = 228.033 ms  
initial connection time = 67.965 ms  
tps = 516.339698 (without initial connection time)  
```bash
# Попробуем тоже самое в 8 потоков
pgbench -c 80 -j 8 -P 30 -T 180 postgres
```
transaction type: <builtin: TPC-B (sort of)>  
scaling factor: 1  
query mode: simple  
number of clients: 80  
number of threads: 8  
maximum number of tries: 1  
duration: 180 s  
number of transactions actually processed: 82015  
number of failed transactions: 0 (0.000%)  
latency average = 175.611 ms  
latency stddev = 266.089 ms  
initial connection time = 50.334 ms   
tps = 455.357339 (without initial connection time)  
  
*Хуже первого теста*

# Оптимизируйте настройки PostgreSQL для максимальной производительности.
- Добавим huge_pages
```bash
# проверка возможности использовать
grep HUGETLB /boot/config-$(uname -r) 
# проверка настроенных 
grep Huge /proc/meminfo
# по шагам
sudo head -1 /mnt/vdb/pgdata/postmaster.pid 	--- 19806
grep ^VmPeak /proc/9260/status 		                        --- 232808
grep -i hugepagesize /proc/meminfo		                    --- 2024
echo $((231604 / 2048 + 10)) 			                        --- 123 
sudo sysctl -w vm.nr_hugepages=118

# Применить свой путь к postmaster
sudo sysctl -w vm.nr_hugepages=$(( $(sudo grep ^VmPeak /proc/$(sudo head -1 /mnt/vdb/pgdata/postmaster.pid 2>/dev/null)/status 2>/dev/null | awk '{print $2}') / $(grep -i hugepagesize /proc/meminfo 2>/dev/null | awk '{print $2}') + 10 ))

# sudo systemctl daemon-reload
# sudo systemctl -p --system
```
```bash
# Проверим
pgbench -c 80 -j 4 -P 30 -T 180 postgres
```  
transaction type: <builtin: TPC-B (sort of)>  
scaling factor: 1  
query mode: simple  
number of clients: 80  
number of threads: 4  
maximum number of tries: 1  
duration: 180 s  
number of transactions actually processed: 92650  
number of failed transactions: 0 (0.000%)  
latency average = 155.439 ms  
latency stddev = 224.007 ms  
initial connection time = 62.260 ms  
tps = 514.480831 (without initial connection time)  

*не изменилась*
```conf
# Изменим параметры postgres cогласно калькулятору https://tantorlabs.ru/pgconfigurator

# add 
#   cpu_cores = 8
#   ram_value = 8589934592
#   db_disk_type = SSD
#   workload_db = mixed
#   pg_version = 18
wal_compression = lz4
jit = on
client_connection_check_interval = 5s
default_toast_compression = pglz
enable_async_append = on
autovacuum_vacuum_insert_threshold = 1596
autovacuum_vacuum_insert_scale_factor = 0.01
logical_decoding_work_mem = 64MB
maintenance_io_concurrency = 128
wal_keep_size = 1506MB
hash_mem_multiplier = 2.0
max_parallel_maintenance_workers = 4
max_parallel_workers = 4
max_logical_replication_workers = 4
max_sync_workers_per_subscription = 2
autovacuum = on
autovacuum_max_workers = 4
autovacuum_work_mem = 189MB
autovacuum_naptime = 15s
autovacuum_vacuum_threshold = 1596
autovacuum_analyze_threshold = 798
autovacuum_vacuum_scale_factor = 0.001
autovacuum_analyze_scale_factor = 0.0007
autovacuum_vacuum_cost_limit = 2188
vacuum_cost_limit = 8000
autovacuum_vacuum_cost_delay = 10ms
vacuum_cost_delay = 10ms
autovacuum_freeze_max_age = 500000000
autovacuum_multixact_freeze_max_age = 800000000
shared_buffers = 1779MB
max_connections = 91
max_files_per_process = 1391
superuser_reserved_connections = 4
work_mem = 35MB
temp_buffers = 3990kB
maintenance_work_mem = 196MB
huge_pages = try
fsync = on
wal_level = replica
synchronous_commit = off
full_page_writes = on
wal_buffers = 23MB
wal_writer_delay = 225ms
wal_writer_flush_after = 3050kB
min_wal_size = 1010MB
max_wal_size = 3050MB
max_replication_slots = 10
max_wal_senders = 10
wal_sender_timeout = 300s
wal_log_hints = on
hot_standby = on
wal_receiver_timeout = 300s
max_standby_streaming_delay = 1800s
hot_standby_feedback = on
wal_receiver_status_interval = 10s
checkpoint_timeout = 30min
checkpoint_warning = 30s
checkpoint_completion_target = 0.85
commit_delay = 128
commit_siblings = 12
bgwriter_delay = 54ms
bgwriter_lru_maxpages = 515
bgwriter_lru_multiplier = 7.0
effective_cache_size = 5081MB
cpu_operator_cost = 0.0025
default_statistics_target = 500
random_page_cost = 1.1
seq_page_cost = 1
join_collapse_limit = 9
from_collapse_limit = 9
geqo = on
geqo_threshold = 12
effective_io_concurrency = 128
max_worker_processes = 8
max_parallel_workers_per_gather = 2
max_locks_per_transaction = 190
max_pred_locks_per_transaction = 190
statement_timeout = 86400000
idle_in_transaction_session_timeout = 86400000
# old_snapshot_threshold = 4320 - на неё ругается
```
```bash
sudo sysctl -w vm.nr_hugepages=950 # так как поменялось shared_buffer - требуется увеличить кол-во huge_pages
```
```bash
# Проверим
pgbench -c 80 -j 4 -P 30 -T 180 postgres
```     
transaction type: <builtin: TPC-B (sort of)>  
scaling factor: 1  
query mode: simple  
number of clients: 80  
number of threads: 4  
maximum number of tries: 1  
duration: 180 s  
number of transactions actually processed: 515212  
number of failed transactions: 0 (0.000%)  
latency average = 27.942 ms  
latency stddev = 48.621 ms  
initial connection time = 70.213 ms  
tps = 2862.331983 (without initial connection time)  

*x5 примерно* 
```bash
#сделал инициализация после этого 
pgbench -i postgres -s 100
done in 237.04 s (drop tables 0.02 s, create tables 0.00 s, client-side generate 223.62 s, vacuum 1.28 s, primary keys 12.12 s).
```  
```bash
# Выключил huge_pages 
# Проверим
pgbench -c 80 -j 4 -P 30 -T 180 postgres
```   
transaction type: <builtin: TPC-B (sort of)>  
scaling factor: 100 
query mode: simple  
number of clients: 80  
number of threads: 4  
maximum number of tries: 1  
duration: 180 s  
number of transactions actually processed: 1231825 
number of failed transactions: 0 (0.000%)  
latency average = 11.684 ms  
latency stddev = 87.542 ms  
initial connection time = 71.775 ms  
tps = 6838.356827 (without initial connection time)  

*тут еще выше производительность
```bash
#сделал инициализация после этого 
pgbench -i postgres -s 100
done in 234.86 s (drop tables 0.25 s, create tables 0.00 s, client-side generate 202.98 s, vacuum 1.29 s, primary keys 30.34 s).
```  
```bash
# Включил huge_pages 
# Проверим
pgbench -c 80 -j 4 -P 30 -T 180 postgres
```   
transaction type: <builtin: TPC-B (sort of)>  
scaling factor: 100  
query mode: simple  
number of clients: 80  
number of threads: 4  
maximum number of tries: 1  
duration: 180 s  
number of transactions actually processed: 1553483  
number of failed transactions: 0 (0.000%)  
latency average = 9.262 ms  
latency stddev = 3.227 ms  
initial connection time = 65.981 ms  
tps = 8631.545233 (without initial connection time)  
- Заключение
```
Huge_pages работает хорошо только с большими объёмами данных.
Правильная настройка увеличивает производительность.
от просто поставил postgres до настроенного разница в производительность в 12 раз отличается. 
```
- Удаление ВМ
```powershell
$names = @("bananaflow")
foreach ($name in $names) {
    yc compute instance delete $name
}
yc vpc subnet delete my-yc-subnet-a
yc vpc network delete my-yc-network-main
```
