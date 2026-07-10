- Khái niệm cần hiểu trước
ConfigMap vs Secret khác gì nhau? Cả hai đều dùng để đưa config ra ngoài code (không hardcode), nhưng:

ConfigMap: chứa config không nhạy cảm — ví dụ tên database, log level, feature flag. Lưu plaintext, ai có quyền get configmap là đọc được ngay.
Secret: chứa data nhạy cảm — password, token, connection string có credential. Lưu dạng base64-encoded (chú ý: base64 không phải mã hóa, chỉ là encode — ai biết cách là decode được ngay, Secret an toàn hơn ConfigMap chủ yếu nhờ RBAC giới hạn quyền đọc + K8s có thể mã hóa Secret at-rest trong etcd nếu cluster cấu hình, không phải nhờ base64). Đây là lý do MONGO_URI (có username/password) phải nằm ở Secret, không phải ConfigMap.
- secret qua env và volume
qua env update secret phải restart pods
còn qua valume thì không phải restart
