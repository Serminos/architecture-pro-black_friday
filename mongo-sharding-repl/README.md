# Задание 3. Репликация
Данная конфигурация Docker Compose разворачивает отказоустойчивый кластер MongoDB:
- **3 конфигурационных сервера** (реплика-сет `config_server`)
- **Шард 1** – реплика-сет из 3 узлов: `shard1-1` (primary), `shard1-2`, `shard1-3`
- **Шард 2** – реплика-сет из 3 узлов: `shard2-1` (primary), `shard2-2`, `shard2-3`
- **Маршрутизатор** `mongos_router`
- **Приложение** `pymongo_api` подключается к маршрутизатору и работает с БД `somedb`, коллекцией `helloDoc

## Требования

- Docker (20.10+) и Docker Compose (2.0+)
- Минимум 2 CPU / 4 ГБ ОЗУ
- Свободные порты: 8080 (API), 27017, 27027, 27037 (configSrv), 27018, 27028, 27038 (шард1), 27019, 27029, 27039 (шард2), 27020 (mongos)

## Запуск кластера

1. Запустите контейнеры
   ```bash
   docker-compose up -d
2. Дождитесь, пока все сервисы перейдут в состояние healthy. Проверьте docker-compose ps.
   ```bash
   docker-compose ps

## Инициализация кластера
1. Инициализация конфигурационного сервера (реплика-сет из 3 узлов)
   ```bash
   docker exec -it configSrv-1 mongosh --port 27017 --eval "rs.initiate({_id: 'config_server',configsvr: true,members: [{ _id: 0, host: 'configSrv-1:27017' },{ _id: 1, host: 'configSrv-2:27027' },{ _id: 2, host: 'configSrv-3:27037' }]})"
   ```
2. Инициализация реплика-сета шарда 1 (3 узла)
   ```bash
   docker exec -it shard1-1 mongosh --port 27018 --eval "rs.initiate({_id: 'shard1',members: [{ _id: 0, host: 'shard1-1:27018' },{ _id: 1, host: 'shard1-2:27028' },{ _id: 2, host: 'shard1-3:27038' }]})"
   ```
3. Инициализация реплика-сета шарда 2 (3 узла)
   ```bash
   docker exec -it shard2-1 mongosh --port 27019 --eval "rs.initiate({_id: 'shard2',members: [{ _id: 0, host: 'shard2-1:27019' },{ _id: 1, host: 'shard2-2:27029' },{ _id: 2, host: 'shard2-3:27039' }]})"
   ```

4. Настройка маршрутизатора (mongos)
   ```bash
   docker exec -it mongos_router mongosh --port 27020 --eval "sh.addShard('shard1/shard1-1:27018');"
   docker exec -it mongos_router mongosh --port 27020 --eval "sh.addShard('shard2/shard2-1:27019');"
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
3. Количество документов в каждом шарде
   ```bash
   docker exec -it shard1-1 mongosh --port 27018 --quiet --eval "db.getSiblingDB('somedb').helloDoc.countDocuments()"
   docker exec -it shard2-1 mongosh --port 27019 --quiet --eval "db.getSiblingDB('somedb').helloDoc.countDocuments()"   
   ```
4. Проверка количества реплик
   1. Для configSrv
      ```bash
      docker exec -it configSrv-1 mongosh --port 27017 --eval "rs.status().members.length"
      ```
   2. Для шарда 1
      ```bash
      docker exec -it shard1-1 mongosh --port 27018 --eval "rs.status().members.length"
      ```
   3. Для шарда 2
      ```bash
      docker exec -it shard2-1 mongosh --port 27019 --eval "rs.status().members.length"
      ```
5. Открыть в браузере http://localhost:8080
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
   "shard2": "shard2/shard2-1:27019,shard2-2:27029,shard2-3:27039",
   "shard1": "shard1/shard1-1:27018,shard1-2:27028,shard1-3:27038"
   },
   "cache_enabled": false,
   "status": "OK"
   }