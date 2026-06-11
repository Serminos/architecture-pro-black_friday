# Задание 2. Шардирование
Данная конфигурация Docker Compose разворачивает кластер MongoDB с двумя шардами 
(один config‑сервер, два шарда, один маршрутизатор). 
Приложение `pymongo_api` подключается к маршрутизатору и работает с БД `somedb`, коллекцией `helloDoc`.

## Требования

- Docker (20.10+) и Docker Compose (2.0+)
- Минимум 2 CPU / 4 ГБ ОЗУ
- Свободные порты: 8080 (API), 27017-27020 (MongoDB)

## Запуск кластера

1. Запустите контейнеры
   ```bash
   docker-compose up -d
2. Дождитесь, пока все сервисы перейдут в состояние healthy. Проверьте docker-compose ps.
   ```bash
   docker-compose ps

## Инициализация кластера
1. Инициализация конфигурационного сервера (configSrv)
   ```bash
   docker exec -it configSrv mongosh --port 27017 --eval "rs.initiate({_id: 'config_server', configsvr: true, members: [{_id: 0, host: 'configSrv:27017'}]})"
   ```
2. Инициализация шарда 1
   ```bash
   docker exec -it shard1 mongosh --port 27018 --eval "rs.initiate({_id: 'shard1', members: [{_id: 0, host: 'shard1:27018'}]})"
   ```
3. Инициализация шарда 2
   ```bash
   docker exec -it shard2 mongosh --port 27019 --eval "rs.initiate({_id: 'shard2', members: [{_id: 0, host: 'shard2:27019'}]})"
   ```

4. Настройка маршрутизатора (mongos)
   ```bash
   docker exec -it mongos_router mongosh --port 27020 --eval "sh.addShard('shard1/shard1:27018');"
   docker exec -it mongos_router mongosh --port 27020 --eval "sh.addShard('shard2/shard2:27019');"
   ```
   ```bash
   docker exec -it mongos_router mongosh --port 27020 --eval "sh.enableSharding('somedb');"
   docker exec -it mongos_router mongosh --port 27020 --eval "sh.shardCollection('somedb.helloDoc', { name: 'hashed' });"   
   ```
## Проверка
1. Наполнение тестовыми данными
   ```bash
   docker exec -it mongos_router mongosh --port 27020 --eval "for (var i = 0; i < 1000; i++) { db.getSiblingDB('somedb').helloDoc.insert({ age: i, name: 'ly' + i }); }"
   ```
2. Проверка всего 1000
   ```bash
   docker exec -it mongos_router mongosh --port 27020 --eval "db.getSiblingDB('somedb').helloDoc.countDocuments()"   
   ```
3. Проверка по шардам(распределено)
   ```bash
   docker exec -it shard1 mongosh --port 27018 --quiet --eval "db.getSiblingDB('somedb').helloDoc.countDocuments()"
   docker exec -it shard2 mongosh --port 27019 --quiet --eval "db.getSiblingDB('somedb').helloDoc.countDocuments()"   
   ```
4. Открыть в браузере http://localhost:8080
   >  {
      "mongo_topology_type": "Sharded",
      "mongo_replicaset_name": null,
      "mongo_db": "somedb",
      "read_preference": "Primary()",
      "mongo_nodes": [
      [
      "mongos_router",
      27020]
      ],
      "mongo_primary_host": null,
      "mongo_secondary_hosts": [],
      "mongo_is_primary": true,
      "mongo_is_mongos": true,
      "collections": {
      "helloDoc": {
      "documents_count": 1000
      }
      },
      "shards": {
      "shard2": "shard2/shard2:27019",
      "shard1": "shard1/shard1:27018"
      },
      "cache_enabled": false,
      "status": "OK"
      }