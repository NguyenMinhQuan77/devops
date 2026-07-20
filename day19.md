- Bước 1: Dump toàn bộ data từ Mongo cũ (10.24.28.115)
```
mongodump \
  --uri="mongodb://admin:admin123@10.24.28.115:27017/devops_practice?authSource=admin" \
  --out=/tmp/mongo-migration-$(date +%Y%m%d)
```

- Bước 2: Restore vào Mongo mới (10.24.28.114)
```
mongorestore \
  --uri="mongodb://admin:admin123@10.24.28.114:27017/devops_practice?authSource=devops_practice" \
  --nsInclude="devops_practice.*" \
  /tmp/mongo-migration-*/devops_practice/
```
- Sang vm restore verify lại
```
mongosh "mongodb://admin:admin123@10.24.28.114:27017/devops_practice?authSource=admin" \
  --eval "db.servers.countDocuments()"
mongosh "mongodb://admin:admin123@10.24.28.114:27017/devops_practice?authSource=admin"   --eval "db.servers.find().pretty()"
```