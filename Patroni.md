## **Создание отказоустойчивого кластера на основе Patroni**

### **1. Разворачиваем 3 ВМ в Яндекс Облаке и устанавливаем Postgres**</br>
В итоге получаем 3 ВМ, подключаемся к ним по SSH через свою локальную ВМ.
Для этого генирирем SSH-ключ и прописываем публичный ключ в настройках, при создании ВМ в Яндексе.
![Inst](itog/VM.png) 
Далее устанавливаем Postgres 15 на этих 3-х ВМ.</br>
Подключение к ВМ:
```
ssh -i ~/.ssh/id_rsa otus@158.160.70.46
ssh -i ~/.ssh/id_rsa otus2@158.160.64.41
ssh -i ~/.ssh/id_rsa otus3@158.160.21.118
```



Устанавливаем Postgres 15
```
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
```
Проверяем, что все работает.

```
sudo su postgres
psql
```

### **2. Установка Consul**</br>
Consul - распределенное хранилище данных, необходимое для запуска Патрони.

Перекидываем бинарник на машины, где будет стоять Consul

```
scp consul-133531-13cd73 otus@158.160.70.46:\tmp
scp consul-133531-13cd73 otus2@158.160.64.41:\tmp
scp consul-133531-13cd73 otus3@158.160.21.118:\tmp
```
![Inst](itog/consul.png) 
Заходим на каждую машину и устанавливаем и настраиваем консул. Для удобства переименуем бинарник уонсула

```postgres
mv consul-133531-13cd73 consul
---перемещаем бинарник в user/bin
sudo mv consul /usr/bin/
---делаем файл исполняемым
chmod +x /usr/bin/consul
---добавляем пользователя для консула
sudo useradd -r -c 'Consul DCS service' consul
---создаем католог для файлов и конфигурационных данных
sudo mkdir -p /var/lib/consul /etc/consul.d
---устанавливаем владельцем consul 
sudo chown consul:consul /var/lib/consul /etc/consul.d
---даем соответствующие права на папку
sudo chmod 775 /var/lib/consul /etc/consul.d
```
Далее генерируем ключ для консула (на любой из нод кластера). В моем случае на host-01</br>
Запоминаем значениею
```
consul keygen
```
![Inst](itog/key_con.png) </br>
Создаем конфигурационный файл для консула config.json:
```
sudo nano /etc/consul.d/config.json
```
Записываем следующие данные: 
```json
{
    "bind_addr": "0.0.0.0",
    "bootstrap_expect": 3,
    "client_addr": "0.0.0.0",
    "data_dir": "/var/lib/consul",
    "enable_script_checks": true,
    "dns_config": {
        "enable_truncate": true,
        "only_passing": true
    },
    "enable_syslog": true,
    "encrypt": "oLNsP2AU6HUb+aejjqzUXRr1zX0/XkY3I0lH1tHRwB4=",
    "leave_on_terminate": true,
    "log_level": "INFO",
    "rejoin_after_leave": true,
    "retry_join": [
        "host-01",
        "host-02",
        "host-03"
    ],
    "server": true,
    "start_join": [
        "host-01",
        "host-02",
        "host-03"
    ],
   "ui_config": { "enabled": true }
}
```
**Параметры конфига config.json:**

<li>bind_addr — адрес, на котором будет слушать наш сервер консул. Это может быть IP любого из наших сетевых интерфейсов или, как в данном примере, все.
<li>bootstrap_expect — ожидаемое количество серверов в кластере.
<li>client_addr — адрес, к которому будут привязаны клиентские интерфейсы.
<li>datacenter — привязка сервера к конкретному датацентру. Нужен для логического разделения. Серверы с одинаковым датацентром должны находиться в одной локальной сети.
<li>data_dir — каталог для хранения данных.
<li>domain — домен, в котором будет зарегистрирован сервис.
<li>enable_script_checks — разрешает на агенте проверку работоспособности.
<li>dns_config — параметры для настройки DNS.
<li>enable_syslog — разрешение на ведение лога.
<li>encrypt — ключ для шифрования сетевого трафика. В качестве значения используем сгенерированный ранее.
<li>leave_on_terminate — при получении сигнала на остановку процесса консула, корректно отключать ноду от кластера.
<li>log_level — минимальный уровень события для отображения в логе. Возможны варианты "trace", "debug", "info", "warn", and "err".
<li>rejoin_after_leave — по умолчанию, нода покидающая кластер не присоединяется к нему автоматически. Данная опция позволяет управлять данным поведением.
<li>retry_join — перечисляем узлы, к которым можно присоединять кластер. Процесс будет повторяться, пока не завершиться успешно.
<li>server — режим работы сервера.
<li>start_join — список узлов кластера, к которым пробуем присоединиться при загрузке сервера.
<li>ui_config — конфигурация для графического веб-интерфейса.</br>
</br>
Далее проверяем валидность конфига:

```
consul validate /etc/consul.d/config.json
```
![Inst](itog/conf_valid.png)

Должна быть строчка: **Configuration is valid!**
</br>
Далее пишем сервис для запуска консула:
```
sudo nano /etc/systemd/system/consul.service
```
В файл прописываем следующее:
```
[Unit]
Description=Consul Service Discovery Agent
Documentation=https://www.consul.io/
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=consul
Group=consul
ExecStart=/usr/bin/consul agent \
    -node=consul01.dmosk.local \
    -config-dir=/etc/consul.d
ExecReload=/bin/kill -HUP $MAINPID
KillSignal=SIGINT
TimeoutStopSec=5
Restart=on-failure
SyslogIdentifier=consul

[Install]
WantedBy=multi-user.target
```
```postgres
---Перечитываем конфигурацию systemd:
sudo systemctl daemon-reload
---Стартуем сервис:
sudo systemctl start consul
---Разрешаем автоматический старт при запуске сервера:
sudo systemctl enable consul
---Смотрим текущее состояние работы сервиса:
systemctl status consul
```
![Inst](itog/consul_stat.png)
Состояние ноды:
```
consul members
```
![Inst](itog/mem_nod.png)

Создаем конфиг. файл config.json и consul.service на двух остальных машинах, при этом в consul.service меняем наименование ноды на consul02.dmosk.local и consul03.dmosk.local. Также делаем релоад стартуем и смотрим статус. Получаем следующие статусы остальных двух нод:

![Inst](itog/consul_stat3.png)
![Inst](itog/consul_stat2.png)

Также доступен web интерфейс для проверки статуса, обратившись, например по адресу:  
http://158.160.64.41:8500, где ip-адрес - любой из 3-х нод.

![Inst](itog/web_consul.png)
На этом установка consul завершена.

### **3. Установка Patroni**</br>
Перед установкой Patroni останавливаем кластер postgres И удаляем директорию PGDATA
 ```
sudo systemctl disable postgresql --now
sudo su postgres
rm -rf /var/lib/postgresql/15/main/*
```
Далее устанавливаем Patroni, а также все необходимые библиотеки python
```
sudo apt install -y python3 python3-pip python3-psycopg2 
sudo pip3 install patroni[consul] 
sudo mkdir /etc/patroni
```
Создаем конфигурационный файл для Patroni, в котором пропишем след. информацию:

```
sudo nano /etc/patroni/patroni.yml
```

```
name: host-02
scope: postgres

watchdog:
  mode: off

consul:
  host: "localhost:8500"
  register_service: true
  #token: <consul-acl-token>

restapi:
  listen: 0.0.0.0:8008
  connect_address: "158.160.70.46:8008"
  auth: 'patroni:patroni'

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        archive_mode: "on"
        wal_level: hot_standby
        max_wal_senders: 10
        wal_keep_segments: 8
        archive_timeout: 1800s
        max_replication_slots: 5
        hot_standby: "on"
        wal_log_hints: "on"
      pg_hba:
        - local all all trust
        - host replication replicator 158.160.45.235/32 trust
        - host replication replicator 158.160.70.46/32 trust
        - host replication replicator 158.160.64.41/32 trust
        - host replication replicator 127.0.0.1/32 trust
        - host all all 0.0.0.0/0 scram-sha-256
    

initdb:
  - encoding: UTF8
  - data-checksums

postgresql:
  pgpass: /var/lib/postgresql/15/.pgpass
  listen: 0.0.0.0:5432
  connect_address: "158.160.70.46:5432"
  data_dir: /var/lib/postgresql/15/main/
  bin_dir: /usr/lib/postgresql/15/bin/
  pg_rewind:
    username: postgres
    password: password
  replication:
    username: replicator
    password: replicator
  superuser:
    username: postgres
    password: postgres
```
**Описание параметров для файла patroni.yml:**
<li>name — имя узла, на котором настраивается данный конфиг.
<li>scope — имя кластера. Его мы будем использовать при обращении к ресурсу, а также под этим именем будет зарегистрирован сервис в consul.
<li>consul-token — если наш кластер consul использует ACL, необходимо указать токен.
<li>restapi-connect_address — адрес на настраиваемом сервере, на который будут приходить подключения к patroni.
<li>restapi-auth — логин и пароль для аутентификации на интерфейсе API.
<li>pg_hba — блок конфигурации pg_hba для разрешения подключения к СУБД и ее базам. Необходимо обратить внимание на подсеть для
строки host replication replicator. Она должна соответствовать той, которая используется в вашей инфраструктуре.
<li>postgresql-pgpass — путь до файла, который создаст патрони. В нем будет храниться пароль для подключения к postgresql.
<li>postgresql-connect_address — адрес и порт, которые будут использоваться для подключения к СУДБ.
<li>postgresql - data_dir — путь до файлов с данными базы.
<li>postgresql - bin_dir — путь до бинарников postgresql.
<li>pg_rewind, replication, superuser — логины и пароли, которые будут созданы для базы.</br>
</br>

Далее создаем сервис patroni.service для patroni:
```
sudo nano /etc/systemd/system/patroni.service
```
Прописываем след информацию:
```
[Unit]
Description=Patroni service
After=syslog.target network.target

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni/patroni.yml
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=process
TimeoutSec=30
Restart=no

[Install]
WantedBy=multi-user.target
```
```postgres
---Перечитываем конфигурацию systemd:
sudo systemctl daemon-reload
---стартуем сервис
sudo systemctl start patroni
---Разрешаем автозапуск
sudo systemctl enable patroni
--Проверяем статусы сервиса на обоих серверах:
sudo systemctl status patroni
```
![Inst](itog/patroni_stat.png)
Видим, что сервис активный.

ПРоверяем также командой:
```
patronictl -c /etc/patroni/patroni.yml list
```
![Inst](itog/pat_stat.png)
</br>
Далее устанавливаем для остальных двух.</br>
Итого получаем статус:</br>
![Inst](itog/patroni_itog.png)</br>
Перезапустил host-01, статус реплик изменился от streaming на running:</br>
![Inst](itog/pat_stat_run.png)
</br>Но через некоторое время статус реплик стал опять streaming.
изменим в файле.</br>
Мы можем переопределить ролдь Leader на другую ноду, для этог выполним:
```
patronictl -c /etc/patroni/patroni.yml switchover postgres
```
![Inst](itog/swich.png)</br>

Под 1 система указывает, кто сейчас является лидером, в нашем случае host-01</br>
Далее под 2 указывает какие есть кандидаты, ввожу host-02. Затем видно из скриншота, что лидер поменялся, как мы и хотели.

Также для теста пробуем отключить один из серверов, например host-02, который является лидером. Роли перераспределяются между оставшимися: </br>
![Inst](itog/swich_off.png)</br>
Теперь лидер host-03.

### **4. Установка PgBouncer**
Это программа для создания пула соединений, позволяет уменьшить накладные расходы на базу данных в случае, когда очень большое количество физических соединений ведет к падению производительности PostgreSQL
```
sudo apt install pgbouncer
```
Далее открываем конфиг:
```
sudo nano /etc/pgbouncer/pgbouncer.ini 
```