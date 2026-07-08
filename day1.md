- Chạy container mongo (nhớ mở port 27017 ra ngoài), connect bằng mongosh, thực hiện insert/find/update/delete tay để quen cấu trúc document JSON.
- Chạy container MongoDB
```
docker run -d \
  --name mongo-external \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=admin123 \
  -v mongo-data:/data/db \
  --restart unless-stopped \
  mongo:7
```
- Connect vào bằng mongosh
```
docker exec -it mongo-external mongosh -u admin -p admin123 --authenticationDatabase admin
```
- Thực hành CRUD tay
```
// 1. Chọn/tạo database (Mongo tạo lazy — chỉ thật sự tạo khi có data)
use devops_practice

// 2. insertOne — tạo document trong collection "servers"
db.servers.insertOne({
  hostname: "web-01",
  ip: "192.168.100.10",
  role: "webserver",
  cpu_cores: 4,
  status: "active",
  createdAt: new Date()
})

// Thêm vài document nữa để có data thực hành find
db.servers.insertMany([
  { hostname: "web-02", ip: "192.168.100.11", role: "webserver", cpu_cores: 2, status: "active" },
  { hostname: "db-01", ip: "192.168.100.20", role: "database", cpu_cores: 8, status: "maintenance" }
])

// 3. find — xem toàn bộ
db.servers.find()

// find có điều kiện + pretty print
db.servers.find({ role: "webserver" }).pretty()

// find với operator
db.servers.find({ cpu_cores: { $gte: 4 } })

// 4. updateOne — sửa 1 field
db.servers.updateOne(
  { hostname: "db-01" },
  { $set: { status: "active" } }
)

// update với $inc (tăng số)
db.servers.updateOne(
  { hostname: "web-01" },
  { $inc: { cpu_cores: 2 } }
)

// 5. deleteOne
db.servers.deleteOne({ hostname: "web-02" })

// Kiểm tra lại collection và database
show collections
db.servers.countDocuments()
```