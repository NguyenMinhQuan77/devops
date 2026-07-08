- uv là một tool quản lý project Python (thay cho pip + venv + poetry). Nó nhanh hơn pip rất nhiều vì viết bằng Rust, và tự động tạo môi trường ảo (virtual environment) giúp anh — nghĩa là dependency của project này không bị lẫn với Python hệ thống hoặc project khác.
- Download uv

```
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env
uv --version
```
-  Tạo project
```
mkdir -p ~/devops-practice/day2-api
cd ~/devops-practice/day2-api
uv init
```
uv init tạo ra vài file:

pyproject.toml — giống package.json của Node.js, khai báo tên project, version, và dependency (thư viện project cần).
.python-version — chốt version Python dùng.
main.py — file code mẫu rỗng.
- Thêm dependency
```
uv add fastapi pymongo uvicorn
```
- Mở file main.py, xóa hết code mẫu, thay bằng:
```
import os
from fastapi import FastAPI, HTTPException
from pymongo import MongoClient
from pymongo.errors import ConnectionFailure
from bson import ObjectId

app = FastAPI()

# Đọc MONGO_URI từ biến môi trường (KHÔNG hardcode connection string trong code)
# os.getenv trả về None nếu biến chưa được set -> nên cho giá trị default để dev tiện test local
MONGO_URI = os.getenv("MONGO_URI", "mongodb://admin:admin123@localhost:27017/?authSource=admin")

client = MongoClient(MONGO_URI)
db = client["devops_practice"]
collection = db["servers"]


@app.get("/health")
def health_check():
    """Check Mongo còn sống hay không"""
    try:
        # lệnh "ping" là cách chuẩn để test connection Mongo còn sống,
        # không tốn resource như query thật
        client.admin.command("ping")
        return {"status": "ok", "mongo": "connected"}
    except ConnectionFailure:
        # Nếu Mongo chết, trả HTTP 503 (Service Unavailable) thay vì 200 giả vờ ổn
        raise HTTPException(status_code=503, detail="Mongo connection failed")


@app.post("/api/items")
def create_item(item: dict):
    """Tạo document mới trong collection servers"""
    result = collection.insert_one(item)
    # insert_one trả về object có .inserted_id (là ObjectId, kiểu dữ liệu riêng của Mongo)
    # phải str() nó ra vì ObjectId không tự convert sang JSON được
    return {"inserted_id": str(result.inserted_id)}


@app.get("/api/items")
def list_items():
    """List toàn bộ document trong collection servers"""
    items = []
    for doc in collection.find():
        doc["_id"] = str(doc["_id"])  # tương tự, convert ObjectId -> string để trả JSON được
        items.append(doc)
    return items
```
Giái thích: 
@app.get("/health"): đây là "decorator" — nghĩa là "gắn function health_check này vào route GET /health". Khi có ai gọi GET /health, FastAPI tự chạy function này.
item: dict trong create_item(item: dict): FastAPI tự đọc JSON body của request POST và convert thành dict Python, mình không cần tự parse JSON tay.
Tại sao phải đọc MONGO_URI từ env var chứ không hardcode? Vì connection string thường có username/password — hardcode vào code thì lỡ push code lên Git là leak credential ngay. Đây cũng là pattern chuẩn 12-factor app, sau này qua K8s sẽ đưa MONGO_URI vào ConfigMap/Secret.
- Chạy app
```
uv run uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```
- Test

```
# test 1: health check
curl http://localhost:8000/health

# test 2: tạo item mới
curl -X POST http://localhost:8000/api/items \
  -H "Content-Type: application/json" \
  -d '{"hostname":"web-03","ip":"192.168.100.12","role":"webserver"}'

# test 3: list toàn bộ item
curl http://localhost:8000/api/items
```