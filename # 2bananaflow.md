# 1 Создайте инстанс с Ubuntu 20.04 в Яндекс.Облаке или аналогах.
>В данном случае использовал Oracle VM VirtualBox
>OS Ubuntu 24.04 (192.168.50.232)

# 2 Установите Docker Engine.
```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
- и установим контейнер postgres:18
```bash
sudo docker pull postgres:18
```
# 3 Создайте каталог /var/lib/postgres для хранения данных.
```bash
sudo mkdir -p /var/lib/postgres
```
# 4 Разверните контейнер с PostgreSQL 14, смонтировав в него /var/lib/postgres.
```bash
# Создаем Docker сеть для сетевого подключения
sudo docker network create pg-network
# Запускаем сервер PostgreSQL с монтированием данных
sudo docker run -d \
  --name postgres-server \
  --network pg-network \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_DB=postgres \
  -p 5432:5432 \
  -v /var/lib/postgres:/var/lib/postgresql/ \
  postgres:18
```
# 5 Разверните контейнер с клиентом PostgreSQL.
*Создам 2 контейнера, один с общим network namespace-ом и с общей сетью docker*
```bash
# Создаем клиент, который разделяет network namespace с сервером
sudo docker run -d \
  --name client-namespace \
  --network container:postgres-server \
  postgres:18 \
  sleep infinity
```
```bash
# Создаем клиент, который подключается через Docker сеть
sudo docker run -d \
  --name client-network \
  --network pg-network \
  postgres:18 \
  sleep infinity
```
# 6 Подключитесь из контейнера с клиентом к контейнеру с сервером и создайте таблицу с данными о перевозках.
```bash
sudo docker exec -it client-namespace psql -h localhost -p 5432 -U postgres
```
- Теперь выполняем команды из задания внутри psql:  
```sql
CREATE DATABASE bananaflow;
\c bananaflow
CREATE TABLE shipments(
    id SERIAL PRIMARY KEY,
    product_name TEXT,
    quantity INT,
    destination TEXT
);
INSERT INTO shipments(product_name, quantity, destination) VALUES
('bananas', 1000, 'Europe'),
('bananas', 1500, 'Asia'),
('bananas', 2000, 'Africa'),
('coffee', 500, 'USA'),
('coffee', 700, 'Canada'),
('coffee', 300, 'Japan'),
('sugar', 1000, 'Europe'),
('sugar', 800, 'Asia'),
('sugar', 600, 'Africa'),
('sugar', 400, 'USA');
--Проверяем, что данные созданы
SELECT * FROM shipments;
```
- Вывод:
```sql
id | product_name | quantity | destination
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | bananas      |     1500 | Asia
  3 | bananas      |     2000 | Africa
  4 | coffee       |      500 | USA
  5 | coffee       |      700 | Canada
  6 | coffee       |      300 | Japan
  7 | sugar        |     1000 | Europe
  8 | sugar        |      800 | Asia
  9 | sugar        |      600 | Africa
 10 | sugar        |      400 | USA
(10 строк)
```
- Подключаемся к PostgreSQL через psql внутри контейнера с сетью:
```bash
sudo docker exec -e PGPASSWORD="password" client-network psql -h postgres-server -U postgres -d bananaflow -c "SELECT * FROM shipments;"
```
- Вывод:
```sql
id | product_name | quantity | destination
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | bananas      |     1500 | Asia
  3 | bananas      |     2000 | Africa
  4 | coffee       |      500 | USA
  5 | coffee       |      700 | Canada
  6 | coffee       |      300 | Japan
  7 | sugar        |     1000 | Europe
  8 | sugar        |      800 | Asia
  9 | sugar        |      600 | Africa
 10 | sugar        |      400 | USA
(10 строк)
```

# 7 Подключитесь к контейнеру с сервером с ноутбука или компьютера.
>Для этого была создана рядом ВМ OS Ubuntu 24.04 (192.168.50.128)  
> и установлен клиент:
```bash
sudo apt update && sudo apt upgrade -y && sudo mkdir -p /etc/apt/keyrings && curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/keyrings/postgresql.gpg && sudo sh -c 'echo "deb [signed-by=/etc/apt/keyrings/postgresql.gpg] http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && sudo apt update && sudo apt install postgresql-client-18
```
- Подключение:
```bash
PGPASSWORD="password" psql -h 192.168.50.232 -U postgres -d bananaflow -c "SELECT * FROM shipments;"
```
- Вывод:
```sql
id | product_name | quantity | destination
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | bananas      |     1500 | Asia
  3 | bananas      |     2000 | Africa
  4 | coffee       |      500 | USA
  5 | coffee       |      700 | Canada
  6 | coffee       |      300 | Japan
  7 | sugar        |     1000 | Europe
  8 | sugar        |      800 | Asia
  9 | sugar        |      600 | Africa
 10 | sugar        |      400 | USA
(10 строк)
```

# 8 Удалите контейнер с сервером и создайте его заново.
```bash
# Останавливаем и удаляем все контейнеры
sudo docker stop $(sudo docker ps -aq) 2>/dev/null || true
sudo docker rm $(sudo docker ps -aq) 2>/dev/null || true
```
```bash
# Запускаем сервер PostgreSQL с монтированием данных
sudo docker run -d \
  --name postgres-server \
  --network pg-network \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_DB=postgres \
  -p 5432:5432 \
  -v /var/lib/postgres:/var/lib/postgresql/ \
  postgres:18
  ```
# 9 Проверьте, что данные остались на месте.
>Так как контейнеры с клиентами тоже удалены, воспользуюсь второй ВМ
```bash
PGPASSWORD="password" psql -h 192.168.50.232 -U postgres -d bananaflow -c "SELECT * FROM shipments;"
```
- Вывод:
```sql
id | product_name | quantity | destination
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | bananas      |     1500 | Asia
  3 | bananas      |     2000 | Africa
  4 | coffee       |      500 | USA
  5 | coffee       |      700 | Canada
  6 | coffee       |      300 | Japan
  7 | sugar        |     1000 | Europe
  8 | sugar        |      800 | Asia
  9 | sugar        |      600 | Africa
 10 | sugar        |      400 | USA
(10 строк)
```