- Thêm endpoint liveness riêng, nhẹ, không đụng Mongo
```
@app.get("/livez")
def liveness_check():
    """Chỉ xác nhận process FastAPI còn phản hồi được, KHÔNG check Mongo"""
    return {"status": "alive"}
```
- sửa deployemnt
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devops-api
  namespace: devops-api
  labels:
    app: devops-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: devops-api
  template:
    metadata:
      labels:
        app: devops-api
    spec:
      containers:
        - name: devops-api
          image: <docker_hub_username>/devops-api:v3
          ports:
            - containerPort: 8000
          envFrom:
            - configMapRef:
                name: app-config
          env:
            - name: MONGO_URI
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: MONGO_URI
          readinessProbe:
            httpGet:
              path: /health          # check Mongo - fail thì rút khỏi Service, KHÔNG restart pod
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 2       # fail 2 lần liên tiếp (10s) mới coi là NotReady - tránh flap do lag mạng tạm thời
          livenessProbe:
            httpGet:
              path: /livez            # chỉ check process - fail thì mới RESTART pod
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 15
            failureThreshold: 3       # fail 3 lần liên tiếp (45s) mới restart - tránh restart quá nhạy
```
- docker stop mongo-external
kết quả
```
devops-api-xxxxx   1/1     Running   0    ...
devops-api-xxxxx   0/1     Running   0    ...   <-- READY chuyển 1/1 -> 0/1, nhưng STATUS vẫn "Running", RESTARTS vẫn 0
```
- readline check thấy mongo lỗi nên dừng nhận requet từ mongo nữa
```
kubectl get endpoints devops-api-svc
```
- kết quả
```
PC1430:code quan.nm2$ kubectl get endpoints devops-api-svc
NAME             ENDPOINTS   AGE
devops-api-svc               133m
PC1430:code quan.nm2$ kubectl get endpoints devops-api-svc
NAME             ENDPOINTS                               AGE
devops-api-svc   192.168.59.15:8000,192.168.59.16:8000   135m
```