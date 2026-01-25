*Создание ВМ для ETCD*
# 1 Создайте 3 виртуальные машины для etcd и 3 виртуальные машины для Patroni.
- Создание ВМ:
*так как у меня yandex cli в powershell:*
```powershell
yc vpc network create --name my-yc-network-main --labels my-label=my-main-net --description "my main network in yc"
yc vpc subnet create --name my-yc-subnet-a --zone ru-central1-a --range 192.168.50.0/24 --network-name my-yc-network-main --description "my first subnet via yc"

$names = @("etcd1", "etcd2", "etcd3")
foreach ($name in $names) {
    yc compute instance create --name $name --hostname $name --cores 4 --memory 8 --create-boot-disk size=15G,type=network-hdd,image-id=fd82kcha04mgqerskh8n --network-interface subnet-name=my-yc-subnet-a,nat-ip-version=ipv4 --ssh-key C:\Users\Admin\Downloads\ssh-key\pub.txt --core-fraction 20
}
yc compute instance list

```
- Скачивание ETCD:
```bash
ETCD_VER=v3.6.7
wget https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xvf etcd-${ETCD_VER}-linux-amd64.tar.gz
cd etcd-${ETCD_VER}-linux-amd64
sudo cp etcd etcdctl /usr/local/bin/
# создание папки для данных etcd
sudo mkdir -p /var/lib/etcd && \
sudo chown root:root /var/lib/etcd
```
- Настройка сервиса:  
*Перед тем как использовать нужно указать свои значения*
```bash
NODE_NAME="ETCD3"               # Замените на Node1, Node2 или Node3
NODE_IP="192.168.50.12"        # Замените на IP этого узла
ALL_NODES="ETCD1=http://192.168.50.18:2380,ETCD2=http://192.168.50.25:2380,ETCD3=http://192.168.50.12:2380"

# Теперь скопируйте этот блок:
sudo tee /etc/systemd/system/etcd.service > /dev/null <<EOF
[Unit]
Description=etcd key-value store
Documentation=https://etcd.io
After=network.target

[Service]
User=root
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name ${NODE_NAME} \
  --data-dir=/var/lib/etcd \
  --listen-client-urls http://${NODE_IP}:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://${NODE_IP}:2379 \
  --listen-peer-urls http://${NODE_IP}:2380 \
  --initial-advertise-peer-urls http://${NODE_IP}:2380 \
  --initial-cluster "${ALL_NODES}" \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster-state new
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now etcd
```
*теперь не будем каждый раз при перезагрузке пытаться создать кластер* 
```bash
sudo sed -i 's/--initial-cluster-state new/--initial-cluster-state existing/g' /etc/systemd/system/etcd.service
sudo systemctl daemon-reload
sudo systemctl restart etcd
```


# 2 Разверните HA-кластер PostgreSQL с использованием Patroni.
- Подготовка ВМ:  
*я буду использовать ВМ с доп диском, куда будет развернут postgres*
```powershell
$names = @("patroni1", "patroni2", "patroni3")
foreach ($name in $names) {
    yc compute instance create --name $name --hostname $name --cores 4 --memory 8 --create-boot-disk size=15G,type=network-hdd,image-id=fd82kcha04mgqerskh8n --network-interface subnet-name=my-yc-subnet-a,nat-ip-version=ipv4 --ssh-key C:\Users\Admin\Downloads\ssh-key\pub.txt --core-fraction 20 --create-disk type=network-ssd,size=20,auto-delete
}
yc compute instance list

```
```bash
sudo apt update && sudo apt upgrade -y && sudo mkdir -p /etc/apt/keyrings && curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/keyrings/postgresql.gpg && sudo sh -c 'echo "deb [signed-by=/etc/apt/keyrings/postgresql.gpg] http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && sudo apt update && sudo apt install -y postgresql && sudo systemctl stop postgresql && sudo locale-gen ru_RU ru_RU.UTF-8

sudo systemctl stop postgresql
sudo systemctl disable postgresql
sudo pg_dropcluster 18 main
```
```bash
sudo fdisk /dev/vdb
sudo mkfs.ext4 /dev/vdb1
sudo mkdir -p /mnt/vdb

UUID=$(sudo blkid -s UUID -o value /dev/vdb1); [ -z "$UUID" ] && echo "UUID не найден" && exit 1; echo "UUID=$UUID /mnt/vdb $(sudo blkid -s TYPE -o value /dev/vdb1) defaults 0 2" | sudo tee -a /etc/fstab >/dev/null
sudo mount -a
sudo systemctl daemon-reload

sudo mkdir -p /mnt/vdb/pgdata
sudo chmod 0750 /mnt/vdb/pgdata
sudo chown -R postgres:postgres /mnt/vdb/pgdata
```
- Установка Patroni
- - Установка pip и прочие зависимости
```bash
sudo apt install -y python3-pip python3-venv python3-dev build-essential libpq-dev
```
- - Установка виртуального окружения и переход в него
```bash
sudo python3 -m venv /opt/patroni-venv && \
sudo chown -R yc-user:yc-user /opt/patroni-venv && \
source /opt/patroni-venv/bin/activate && \
pip install patroni[etcd3,psycopg3]
```
- - Подготовка файла конфигурации
```bash
sudo nano /etc/patroni.yml
```
```yml
scope: patroni-cluster
namespace: /patroni/
name: "patroni1"     # Замените на имя ноды

restapi:
  listen: 0.0.0.0:8008 
  connect_address: 192.168.50.8:8008 # Замените на IP текущей ноды

#  cafile: /etc/ssl/certs/ssl-cacert-snakeoil.pem
#  certfile: /etc/ssl/certs/ssl-cert-snakeoil.pem
#  keyfile: /etc/ssl/private/ssl-cert-snakeoil.key
#  authentication:
#    username: username
#    password: password

#ctl:
#  insecure: false # Allow connections to Patroni REST API without verifying certificates
#  certfile: /etc/ssl/certs/ssl-cert-snakeoil.pem
#  keyfile: /etc/ssl/private/ssl-cert-snakeoil.key
#  cacert: /etc/ssl/certs/ssl-cacert-snakeoil.pem

#citus:
#  database: citus
#  group: 0  # coordinator

etcd3:
  #Provide host to do the initial discovery of the cluster topology:
  #host: 127.0.0.1:2379
  hosts: 192.168.50.18:2379,192.168.50.25:2379,192.168.50.12:2379 # Заменить на узлы ETCD
  #Or use "hosts" to provide multiple endpoints
  #Could be a comma separated string:
  #hosts: host1:port1,host2:port2
  #or an actual yaml list:
  #hosts:
  #- host1:port1
  #- host2:port2
  #Once discovery is complete Patroni will use the list of advertised clientURLs
  #It is possible to change this behavior through by setting:
  #use_proxies: true

#raft:
#  data_dir: .
#  self_addr: 127.0.0.1:2222
#  partner_addrs:
#  - 127.0.0.1:2223
#  - 127.0.0.1:2224

# The bootstrap configuration. Works only when the cluster is not yet initialized.
# If the cluster is already initialized, all changes in the `bootstrap` section are ignored!
bootstrap:
  # This section will be written into Etcd:/<namespace>/<scope>/config after initializing new cluster
  # and all other cluster members will use it as a `global configuration`.
  # WARNING! If you want to change any of the parameters that were set up
  # via `bootstrap.dcs` section, please use `patronictl edit-config`!
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
#    primary_start_timeout: 300
#    synchronous_mode: false
    #standby_cluster:
      #host: 127.0.0.1
      #port: 1111
      #primary_slot_name: patroni
    postgresql:
      use_pg_rewind: true
      pg_hba:
      # For kerberos gss based connectivity (discard @.*$)
      #- host replication replicator 127.0.0.1/32 gss include_realm=0
      #- host all all 0.0.0.0/0 gss include_realm=0
      #- host replication replicator 127.0.0.1/32 md5
      - local all postgres trust
      - host all all 127.0.0.1/32 md5
      - host all all 192.168.50.0/24 md5
      - host replication replicator 192.168.50.0/24 md5
      - host replication replicator 127.0.0.1/32 md5
      #  - hostssl all all 0.0.0.0/0 md5
#      use_slots: true
      parameters:
#        wal_level: hot_standby
#        hot_standby: "on"
#        max_connections: 100
#        max_worker_processes: 8
#        wal_keep_segments: 8
#        max_wal_senders: 10
#        max_replication_slots: 10
#        max_prepared_transactions: 0
#        max_locks_per_transaction: 64
#        wal_log_hints: "on"
#        track_commit_timestamp: "off"
#        archive_mode: "on"
#        archive_timeout: 1800s
#        archive_command: mkdir -p ../wal_archive && test ! -f ../wal_archive/%f && cp %p ../wal_archive/%f
#      recovery_conf:
#        restore_command: cp ../wal_archive/%f %p

  # some desired options for 'initdb'
  initdb:  # Note: It needs to be a list (some options need values, others are switches)
  - encoding: UTF8
  - locale: ru_RU.UTF-8
  # - data-checksums

  users:
    admin:
      password: admin
      options:
        - createrole
        - createdb
    replicator:
      password: replicator
      options:
        - superuser
        - login
    rewind:
      password: rewind
      options:
        - superuser
        - login
    

  # Additional script to be launched after initial cluster creation (will be passed the connection URL as parameter)
# post_init: /usr/local/bin/setup_cluster.sh

postgresql:
  listen: 0.0.0.0:5432 
  connect_address: 192.168.50.8:5432 # Замените на IP текущей ноды

#  proxy_address: 127.0.0.1:5433  # The address of connection pool (e.g., pgbouncer) running next to Patroni/Postgres. Only for service discovery.
  data_dir: /mnt/vdb/pgdata
  bin_dir: /usr/lib/postgresql/18/bin
#  config_dir:
  pgpass: /tmp/pgpass
  authentication:
    replication:
      username: replicator
      password: replicator
    superuser:
      username: postgres
      password: postgres
    rewind:  # Has no effect on postgres 10 and lower
      username: rewind
      password: rewind
  # Server side kerberos spn
#  krbsrvname: postgres
  parameters:
    # Fully qualified kerberos ticket file for the running user
    # same as KRB5CCNAME used by the GSS
#   krb_server_keyfile: /var/spool/keytabs/postgres
  # unix_socket_directories: '..'  # parent directory of data_dir
  # Additional fencing script executed after acquiring the leader lock but before promoting the replica
  #pre_promote: /path/to/pre_promote.sh

#watchdog:
#  mode: automatic # Allowed values: off, automatic, required
#  device: /dev/watchdog
#  safety_margin: 5

tags:
    # failover_priority: 1
    # sync_priority: 1
    noloadbalance: false
    clonefrom: false
    nostream: false
```

```bash
sudo chown postgres:postgres /etc/patroni.yml 
```
- - Создание файла сервиса для Patroni
```bash
sudo tee /etc/systemd/system/patroni.service > /dev/null <<EOF
[Unit]
Description=Patroni PostgreSQL HA cluster node
After=network.target

[Service]
Type=simple
User=postgres
Group=postgres
Environment=PATH=/opt/patroni-venv/bin:/usr/bin:/bin
ExecStart=/opt/patroni-venv/bin/patroni /etc/patroni.yml
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable patroni
sudo systemctl start patroni
```
# 3 Настройте HAProxy для балансировки нагрузки.
- Создадим ВМ для Haproxy
```powershell
yc compute instance create --name haproxy --hostname haproxy --cores 4 --memory 4 --create-boot-disk size=15G,type=network-hdd,image-id=fd82kcha04mgqerskh8n --network-interface subnet-name=my-yc-subnet-a,nat-ip-version=ipv4 --ssh-key C:\Users\Admin\Downloads\ssh-key\pub.txt --core-fraction 20
yc compute instance list

```
- Установка haproxy
```bash
sudo apt update
sudo apt install -y haproxy
```
```conf
global
    daemon

defaults
    mode tcp
    log global
    option tcplog
    timeout connect 5s
    timeout client 50s
    timeout server 50s

# Простая балансировка для мастера
listen postgres_master
    bind *:5000
    balance roundrobin
    server patroni1 192.168.50.8:5432
    server patroni2 192.168.50.31:5432
    server patroni3 192.168.50.9:5432

# Простая балансировка для реплик
listen postgres_replicas
    bind *:5001
    balance roundrobin
    server patroni1 192.168.50.8:5432 backup
    server patroni2 192.168.50.31:5432
    server patroni3 192.168.50.9:5432

# Статистика
listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /
    stats refresh 5s
```
```bash
sudo systemctl restart haproxy
sudo systemctl enable haproxy
```
*внизу есть команда на проверку roundrobin, она проверена и haproxy работает как надо*  
*haproxy конечно отработал хорошо, но это дополнительный сервер, как альтернатива использовать keepalived с VIP*:
- Установка
```bash
sudo apt update
sudo apt install -y keepalived curl
```
- Проверка установки
```bash
keepalived -v
```
- Настройка Keepalived для Patroni  
На каждой ноде создаём файл ```/etc/keepalived/check_patroni.sh```: 
```bash
#!/bin/bash

# получаем роль ноды из Patroni
ROLE=$(curl -s http://127.0.0.1:8008/role | grep -oP '"role":\s*"\K[^"]+')
STATE=$(curl -s http://127.0.0.1:8008/role | grep -oP '"state":\s*"\K[^"]+' | head -n1)

# проверяем, что нода мастер и в состоянии running
if [[ "$ROLE" == "Leader" && "$STATE" == "running" ]]; then
    exit 0
else
    exit 1
fi
```
- Делаем скрипт исполняемым:
```bash
sudo chmod +x /etc/keepalived/check_patroni.sh
```
*Теперь скрипт вернёт 0 только если нода является мастером и Patroni работает.*
- Разрешим выполнение:
```bash
chmod +x /etc/keepalived/check_patroni.sh
```
- Конфигурация Keepalived:  
Файл: ```/etc/keepalived/keepalived.conf```
Пример для всех узлов (отличается только priority):
```conf
global_defs {
    script_user root
    enable_script_security
}

vrrp_script chk_patroni {
    script "/etc/keepalived/check_patroni.sh"
    interval 3
    fall 2
    rise 1
}

vrrp_instance VI_1 {
    state BACKUP
    interface enp0s3
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 12345
    }
    virtual_ipaddress {
        192.168.50.40/24
    }
    track_script {
        chk_patroni
    }
}

```
- Настройка приоритетов (пример)
```conf
patroni1: priority 100
patroni2: priority 90
patroni3: priority 80
```
*При равных условиях VIP будет у той ноды, у которой выше priority.*  
*Но главное условие — она должна быть master по Patroni.*
- Запуск:
```bash
systemctl enable keepalived --now
```
*При failover Patroni переключает роль → Keepalived через скрипт убирает VIP у старого мастера и вешает его на нового.*  
*Время переключения обычно 2–5 секунд.*

# 4 Проверьте отказоустойчивость кластера, имитируя сбой на одном из узлов.
```
patronictl -c /etc/patroni.yml list
+ Cluster: patroni-cluster (7599321554204184883) +----+-------------+-----+------------+-----+
| Member   | Host          | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+---------------+---------+-----------+----+-------------+-----+------------+-----+
| patroni1 | 192.168.50.8  | Leader  | running   |  1 |             |     |            |     |
| patroni2 | 192.168.50.31 | Replica | streaming |  1 |  0/1E000000 | 217 | 0/1DFFFFD0 | 217 |
| patroni3 | 192.168.50.9  | Replica | streaming |  1 |  0/1E000000 | 217 | 0/1DA5FFC0 | 223 |
+----------+---------------+---------+-----------+----+-------------+-----+------------+-----+

patronictl -c /etc/patroni.yml switchover

patronictl -c /etc/patroni.yml list
+ Cluster: patroni-cluster (7599321554204184883) +----+-------------+-----+------------+-----+
| Member   | Host          | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+---------------+---------+-----------+----+-------------+-----+------------+-----+
| patroni1 | 192.168.50.8  | Replica | streaming |  2 |  0/2B9A8610 |   0 | 0/2B9A8610 |   0 |
| patroni2 | 192.168.50.31 | Leader  | running   |  2 |             |     |            |     |
| patroni3 | 192.168.50.9  | Replica | running   |  1 |  0/2B9A8610 |   0 | 0/2B9A8610 |   0 |
+----------+---------------+---------+-----------+----+-------------+-----+------------+-----+'

Имитируем выключение patroni2

patronictl -c /etc/patroni.yml list
+ Cluster: patroni-cluster (7599321554204184883) +----+-------------+-----+------------+-----+
| Member   | Host          | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+---------------+---------+-----------+----+-------------+-----+------------+-----+
| patroni1 | 192.168.50.8  | Replica | streaming |  3 |  0/2B9A8890 |   0 | 0/2B9A8890 |   0 |
| patroni2 | 192.168.50.31 | Replica | stopped   |    |     unknown |     |    unknown |     |
| patroni3 | 192.168.50.9  | Leader  | running   |  3 |             |     |            |     |
+----------+---------------+---------+-----------+----+-------------+-----+------------+-----+

Запуск patroni2

patronictl -c /etc/patroni.yml list
+ Cluster: patroni-cluster (7599321554204184883) +----+-------------+-----+------------+-----+
| Member   | Host          | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+---------------+---------+-----------+----+-------------+-----+------------+-----+
| patroni1 | 192.168.50.8  | Replica | streaming |  3 |  0/2B9B0778 |   0 | 0/2B9B0778 |   0 |
| patroni2 | 192.168.50.31 | Replica | streaming |  3 |  0/2B9B0778 |   0 | 0/2B9B0778 |   0 |
| patroni3 | 192.168.50.9  | Leader  | running   |  3 |             |     |            |     |
+----------+---------------+---------+-----------+----+-------------+-----+------------+-----+
```
# 5 Дополнительно: Настройте бэкапы с использованием WAL-G или pg_probackup.
*Не делал, pg_probackup не захотел подключаться, и написал ERROR: Communication error: end of data* 

# Проверки состояний

- Провверка ETCD
```bash
export ENDPOINTS="http://192.168.50.18:2379,http://192.168.50.25:2379,http://192.168.50.12:2379"
etcdctl --endpoints=$ENDPOINTS endpoint status --write-out=table
etcdctl --endpoints=$ENDPOINTS endpoint health
```
- дополнения к Patroni и его проверка
```bash 
sudo locale-gen ru_RU ru_RU.UTF-8
locale -a

source /opt/patroni-venv/bin/activate
patronictl -c /etc/patroni.yml list
sudo journalctl -u patroni --no-pager -n 100

etcdctl --endpoints=$ENDPOINTS get / --prefix --keys-only
etcdctl --endpoints=$ENDPOINTS del /patroni/ --prefix

```
- Проверка Haproxy
```bash
curl http://192.168.50.10:8404/monitor

echo "192.168.50.10:5001:postgres:postgres:postgres" > ~/.pgpass
chmod 600 ~/.pgpass

for i in {1..5}; do
  psql -h 192.168.50.10 -p 5001 -U postgres -t -c "SELECT 'Connection $i -> ' || inet_server_addr();"
done
```
- Проверка keepalived:
```bash
ip addr show eth0
```
*должен отобразится еще один ip*
- Удаление всего
```powershell
$names = @("etcd1", "etcd2", "etcd3", "patroni1", "patroni2", "patroni3", "haproxy")
foreach ($name in $names) {
    yc compute instance delete $name
}
yc vpc subnet delete my-yc-subnet-a
yc vpc network delete my-yc-network-main

```
