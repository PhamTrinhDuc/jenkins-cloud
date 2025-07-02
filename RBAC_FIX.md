# Giải quyết lỗi RBAC Permission

## Vấn đề
Lỗi: `secrets is forbidden: User "system:serviceaccount:model-serving:default" cannot list resource "secrets" in API group "" in the namespace "model-serving"`

## Giải pháp đã áp dụng

### 1. Tạo ServiceAccount riêng
- File: `templates/serviceaccount.yaml`
- Tạo ServiceAccount riêng cho ứng dụng thay vì sử dụng `default`

### 2. Định nghĩa Role với quyền cần thiết
- File: `templates/role.yaml`
- Cấp quyền truy cập `secrets`, `configmaps`, và `deployments`
- Chỉ trong namespace `model-serving`

### 3. Gán quyền thông qua RoleBinding
- File: `templates/rolebinding.yaml`
- Liên kết ServiceAccount với Role

### 4. Cập nhật Deployment
- File: `templates/deployment.yaml`
- Sử dụng ServiceAccount mới: `serviceAccountName: "{{ .Release.Name }}-sa"`

### 5. Tạo Namespace tự động
- File: `templates/namespace.yaml`
- Đảm bảo namespace `model-serving` tồn tại

### 6. Cập nhật Jenkinsfile
- Thêm flag `--create-namespace` để tự động tạo namespace nếu chưa có

## Cách sử dụng

Sau khi áp dụng các thay đổi này, chạy lại pipeline Jenkins. Các resource RBAC sẽ được tạo tự động và ứng dụng sẽ có đầy đủ quyền truy cập cần thiết.

## Kiểm tra

Để kiểm tra quyền đã được cấp đúng, bạn có thể chạy:

```bash
# Kiểm tra ServiceAccount
kubectl get sa -n model-serving

# Kiểm tra Role
kubectl get role -n model-serving

# Kiểm tra RoleBinding
kubectl get rolebinding -n model-serving

# Test quyền truy cập
kubectl auth can-i list secrets --as=system:serviceaccount:model-serving:hpp-sa -n model-serving
```
