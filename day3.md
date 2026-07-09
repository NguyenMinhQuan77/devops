- tạo 1 file Dockerfile 
```
# ===== Stage 1: builder =====
# Image chính thức của uv, đã có sẵn uv binary + Python
FROM ghcr.io/astral-sh/uv:python3.12-bookworm-slim AS builder

WORKDIR /app

# Set biến môi trường để uv hoạt động tối ưu trong Docker:
# UV_COMPILE_BYTECODE=1 -> compile sẵn .pyc, app start nhanh hơn 1 chút
# UV_LINK_MODE=copy -> copy file thay vì hardlink (hardlink dễ lỗi khi copy qua layer/stage khác)
ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy

# QUAN TRỌNG: chỉ copy 2 file lock/manifest trước, CHƯA copy code
# Để layer "cài dependency" chỉ bị invalidate khi pyproject.toml/uv.lock đổi,
# không bị invalidate khi mình sửa main.py
COPY pyproject.toml uv.lock ./

# --frozen: bắt buộc dùng đúng version trong uv.lock, không tự ý update -> deterministic
# --no-dev: không cài dependency của mục [dev] (pytest, httpx) -> image production nhẹ hơn
# --no-install-project: chỉ cài dependency, CHƯA cài "project" (vì code chưa copy vào)
RUN uv sync --frozen --no-dev --no-install-project

# Giờ mới copy code thật vào
COPY main.py ./

# Cài lại lần nữa để "install" chính project mình (một số case cần, an toàn để làm)
RUN uv sync --frozen --no-dev


# ===== Stage 2: final image =====
# Base nhẹ, KHÔNG có uv, KHÔNG có build tool -> production image tối giản
FROM python:3.12-slim-bookworm

WORKDIR /app

# Copy toàn bộ virtualenv (.venv) đã sync xong từ stage builder qua
# Đây là điểm chính của multi-stage: "lấy kết quả", bỏ "quá trình"
COPY --from=builder /app/.venv /app/.venv
COPY --from=builder /app/main.py /app/main.py

# Thêm venv vào PATH để chạy trực tiếp bằng "python" hoặc "uvicorn"
# mà không cần gọi qua "uv run"
ENV PATH="/app/.venv/bin:$PATH"

EXPOSE 8000

# Chạy trực tiếp uvicorn từ venv, không cần uv trong image final
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```
 - Note: statge 1 để build, statge 2 để chạy
 - Build code: 
 ```
docker build -t devops-api:v1 .
 ```
 - Kiểm tra
 ```
docker run -d --name api-test -p 8000:8000 \
  -e MONGO_URI="mongodb://admin:admin123@10.24.28.115:27017/?authSource=admin" \
  devops-api:v1

curl http://10.24.28.115:8000/health
 ```