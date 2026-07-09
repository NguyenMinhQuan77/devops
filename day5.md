- tạo secret lưu thông tin quan trọng: 
```
kubectl create secret generic mongo-secret \
  --from-literal=MONGO_URI="mongodb://admin:admin123@10.24.28.115:27017/?authSource=admin"
```
- kiểm tra
```
kubectl get secret mongo-secret -o yaml
```
- kết quả: 
```
apiVersion: v1
data:
  MONGO_URI: bW9uZ29kYjovL2FkbWluOmFkbWluMTIzQDEwLjI0LjI4LjExNToyNzAxNy8/YXV0aFNvdXJjZT1hZG1pbg==
kind: Secret
metadata:
  creationTimestamp: "2026-07-09T04:25:16Z"
  name: mongo-secret
  namespace: default
  resourceVersion: "5881"
  uid: 988d1b30-1684-45e7-997d-8d096a324525
type: Opaque
```
- Tạo file deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devops-api
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
          image: nguyenminhquan77/devops-api:v2
          ports:
            - containerPort: 8000
          env:
            - name: MONGO_URI
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: MONGO_URI
          readinessProbe:         
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:         
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 20
```
- Viết Service (ClusterIP)
```
apiVersion: v1
kind: Service
metadata:
  name: devops-api-svc
spec:
  type: ClusterIP          # chỉ truy cập được từ TRONG cluster, không expose ra ngoài internet
  selector:
    app: devops-api        # chọn đúng pod có label này (khớp với Deployment ở trên)
  ports:
    - port: 80              # port của Service (client trong cluster gọi vào đây)
      targetPort: 8000       # port thật container đang listen (khớp containerPort ở Deployment)
```