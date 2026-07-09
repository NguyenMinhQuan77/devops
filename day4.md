- Viết docker-compose.yml
```
services:
  api:
    build: .                          # build từ Dockerfile ở thư mục hiện tại (Ngày 3)
    image: devops-api:v2
    container_name: devops-api
    ports:
      - "8000:8000"
    environment:
      # Dùng "host.docker.internal" - một hostname đặc biệt Docker cung cấp,
      # tự động resolve về IP của máy host (VM) từ trong container
      MONGO_URI: "mongodb://admin:admin123@10.24.28.115:27017/?authSource=admin"
    extra_hosts:
      # Trên Linux, "host.docker.internal" KHÔNG tự có sẵn (chỉ Mac/Windows Docker Desktop có sẵn)
      # Dòng này bơm thêm entry vào /etc/hosts của container để ánh xạ
      # "host.docker.internal" -> IP thật của host, dùng special value "host-gateway"
      - "10.24.28.115:host-gateway"
    restart: unless-stopped            # tự khởi động lại nếu container bị crash hoặc VM reboot
```
- Chạy code:
```
cd ~/devops-practice/day2-api
docker compose up -d

```

- Rút ra
```
Checklist tự chấm Ngày 4:

 docker compose up -d chạy thành công, không lỗi network
 curl /health trả mongo: connected — xác nhận extra_hosts: host-gateway hoạt động đúng
 Tạo item qua API, thấy xuất hiện trong mongosh (Ngày 1) — xác nhận đúng là Mongo ngoài, không phải Mongo nào khác
 docker compose restart api rồi test lại — app tự connect lại Mongo, data không mất
 Hiểu được lý do vì sao không chạy Mongo trong Compose (tách state ra khỏi app)

Anh chạy thử, nếu /health báo lỗi connect Mongo thì paste log docker compose logs api vào, khả năng cao là do extra_hosts chưa đúng hoặc firewall VM chặn port 27017.
```
