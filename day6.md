- cài ingress-nginx
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.hostNetwork=true \
  --set controller.kind=DaemonSet \
  --set controller.service.type=ClusterIP \
  --set controller.dnsPolicy=ClusterFirstWithHostNet
```
- Nếu pod không chạy trên master do taint — cần thêm toleration (DaemonSet bỏ qua taint):
```
helm upgrade ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --reuse-values \
  --set controller.tolerations[0].key=node-role.kubernetes.io/control-plane \
  --set controller.tolerations[0].operator=Exists \
  --set controller.tolerations[0].effect=NoSchedule
```
- tạo ingress
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: devops-api-ingress
  namespace: devops-api
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx       
  rules:
    - host: devops-api.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: devops-api-svc     
                port:
                  number: 80            
```
