# Tutorial: MLOps CI/CD Pipeline với Jenkins, Kubernetes và Helm

## 📖 Tổng quan dự án

Đây là một dự án **House Price Prediction API** được triển khai hoàn toàn tự động thông qua CI/CD pipeline. Dự án sử dụng:

- **FastAPI** để xây dựng REST API
- **Docker** để containerize ứng dụng  
- **Jenkins** cho CI/CD automation
- **Kubernetes (GKE)** để orchestration
- **Helm** để quản lý Kubernetes deployments
- **Terraform** để Infrastructure as Code
- **Ansible** để configuration management

---

## 🏗️ Kiến trúc triển khai

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Source Code   │───▶│   Jenkins CI    │───▶│  Kubernetes     │
│   (GitHub)      │    │   Pipeline      │    │  (GKE)          │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │                       │
                                ▼                       ▼
                       ┌─────────────────┐    ┌─────────────────┐
                       │  Docker Image   │    │   Helm Chart    │
                       │  (Registry)     │    │   Deployment    │
                       └─────────────────┘    └─────────────────┘
```

---

## 🚀 Hướng dẫn triển khai từng bước

### 📋 Yêu cầu trước khi bắt đầu

- GCP Account với quyền tạo GKE cluster
- Docker Hub account
- Git repository
- Ansible, Terraform đã cài đặt

---

## 1️⃣ **Chuẩn bị Infrastructure**

### **1.1 Tạo GKE Cluster với Terraform**

```bash
# Di chuyển đến thư mục terraform
cd iac/terraform

# Khởi tạo Terraform
terraform init

# Xem plan trước khi apply
terraform plan -var="project_id=YOUR_PROJECT_ID"

# Tạo infrastructure
terraform apply -var="project_id=YOUR_PROJECT_ID"
```

**File cấu hình chính:** `iac/terraform/main.tf`
- Tạo GKE Autopilot cluster
- Cấu hình networking và security
- Setup service accounts cần thiết

### **1.2 Triển khai Jenkins với Ansible**

```bash
# Di chuyển đến thư mục ansible
cd iac/ansible

# Tạo VM cho Jenkins
ansible-playbook create_compute_instance.yaml

# Cập nhật IP của VM vào inventory file
vim inventory

# Deploy Jenkins với custom image
cd deploy_jenkins
ansible-playbook -i ../inventory deploy_jenkins.yml
```

**Custom Jenkins Image:** `custom_images/jenkins/`
- Pre-installed với kubectl, helm, docker
- Configured với GCP authentication
- Ready-to-use cho Kubernetes deployments

---

## 2️⃣ **Cấu hình Jenkins Pipeline**

### **2.1 Cài đặt Plugin cần thiết**

Trong Jenkins UI:
1. **Kubernetes Plugin** - Để chạy build agents trên K8s
2. **Docker Pipeline Plugin** - Để build/push Docker images
3. **Git Plugin** - Để pull source code

### **2.2 Cấu hình Kubernetes Connection**

```bash
# Kết nối Jenkins với GKE cluster
kubectl create ns model-serving

# Tạo service account cho Jenkins
kubectl create serviceaccount jenkins-sa -n model-serving

# Cấp quyền admin (sẽ được thay thế bằng RBAC chi tiết)
kubectl create clusterrolebinding jenkins-admin \
  --clusterrole=admin \
  --serviceaccount=model-serving:jenkins-sa
```

### **2.3 Phân tích Jenkinsfile**

```groovy
pipeline {
    agent any
    
    environment{
        registry = 'quandvrobusto/house-price-prediction-api'
        registryCredential = 'dockerhub'
    }

    stages {
        stage('Deploy') {
            agent {
                kubernetes {
                    containerTemplate {
                        name 'helm'
                        image 'quandvrobusto/jenkins:lts-jdk17'
                        alwaysPullImage true
                    }
                }
            }
            steps {
                script {
                    container('helm') {
                        sh("helm upgrade --install hpp ./helm-charts/hpp --namespace model-serving --create-namespace")
                    }
                }
            }
        }
    }
}
```

**Điểm quan trọng:**
- Sử dụng **Kubernetes agent** để chạy build
- **Custom Jenkins image** có sẵn helm, kubectl
- **Helm deployment** tự động tạo namespace

---

## 3️⃣ **Helm Chart Configuration**

### **3.1 Cấu trúc Helm Chart**

```
helm-charts/hpp/
├── Chart.yaml              # Metadata của chart
├── values.yaml             # Giá trị mặc định
└── templates/
    ├── deployment.yaml     # Kubernetes Deployment
    ├── service.yaml        # Kubernetes Service
    ├── serviceaccount.yaml # Service Account
    ├── role.yaml          # RBAC Role
    ├── rolebinding.yaml   # RBAC RoleBinding
    └── namespace.yaml     # Namespace definition
```

### **3.2 Values Configuration**

**File:** `helm-charts/hpp/values.yaml`
```yaml
image:
  repository: quandvrobusto/house-price-prediction-api
  tag: "1"
  pullPolicy: Always
```

### **3.3 Deployment Template**

**File:** `helm-charts/hpp/templates/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: "{{ .Release.Name }}"
  namespace: model-serving
spec:
  replicas: 2
  template:
    spec:
      serviceAccountName: "{{ .Release.Name }}-sa"  # RBAC
      containers:
        - name: "{{ .Release.Name }}"
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: "{{ .Values.image.pullPolicy }}"
```

---

## 4️⃣ **RBAC Security Setup**

### **4.1 Vấn đề ban đầu**
```
Error: secrets is forbidden: User "system:serviceaccount:model-serving:default" 
cannot list resource "secrets" in API group "" in the namespace "model-serving"
```

### **4.2 Giải pháp RBAC**

**ServiceAccount:** `templates/serviceaccount.yaml`
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: "{{ .Release.Name }}-sa"
  namespace: model-serving
```

**Role:** `templates/role.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: "{{ .Release.Name }}-role"
  namespace: model-serving
rules:
- apiGroups: [""]
  resources: ["secrets", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
```

**RoleBinding:** `templates/rolebinding.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: "{{ .Release.Name }}-rolebinding"
  namespace: model-serving
subjects:
- kind: ServiceAccount
  name: "{{ .Release.Name }}-sa"
  namespace: model-serving
roleRef:
  kind: Role
  name: "{{ .Release.Name }}-role"
  apiGroup: rbac.authorization.k8s.io
```

---

## 5️⃣ **Application Code**

### **5.1 FastAPI Application**

**File:** `app/main.py`
```python
from fastapi import FastAPI
from schema import HouseInfo, HousePrediction
import joblib
import os

app = FastAPI()
clf = joblib.load(os.environ.get("MODEL_PATH", "models/model.pkl"))

@app.post("/predict", response_model=HousePrediction)
def predict(data: HouseInfo):
    price = clf.predict(format_input_data(data))[0]
    return HousePrediction(Price=price)
```

### **5.2 Docker Configuration**

**File:** `Dockerfile`
```dockerfile
FROM python:3.8
WORKDIR /app

COPY ./app /app
COPY ./requirements.txt /app
COPY ./models /app/models

ENV MODEL_PATH /app/models/model.pkl
EXPOSE 30000

RUN pip install -r requirements.txt --no-cache-dir
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "30000"]
```

---

## 6️⃣ **Testing và Debugging**

### **6.1 Local Testing**

```bash
# Build và test local
docker build -t house-price-api .
docker run -p 30001:30000 house-price-api

# Test API
curl -X POST "http://localhost:30001/predict" \
  -H "Content-Type: application/json" \
  -d '{"feature1": 1, "feature2": 2}'
```

### **6.2 Kubernetes Debugging**

```bash
# Kiểm tra pods
kubectl get pods -n model-serving

# Xem logs
kubectl logs <pod-name> -n model-serving

# Kiểm tra RBAC
kubectl auth can-i list secrets \
  --as=system:serviceaccount:model-serving:hpp-sa \
  -n model-serving

# Test service
kubectl port-forward svc/hpp 30000:30000 -n model-serving
```

### **6.3 Helm Debugging**

```bash
# Dry run để kiểm tra templates
helm install hpp ./helm-charts/hpp --dry-run --debug

# Kiểm tra release
helm list -n model-serving

# Xem history
helm history hpp -n model-serving
```

---

## 7️⃣ **CI/CD Workflow**

### **7.1 Quy trình tự động**

1. **Developer push code** → GitHub repository
2. **Jenkins webhook** triggered tự động
3. **Jenkins pipeline** chạy:
   - Pull source code
   - Build Docker image (nếu có thay đổi)
   - Push image to Docker Hub
   - Deploy với Helm vào GKE
4. **Kubernetes** pull image và deploy
5. **LoadBalancer** expose service ra ngoài

### **7.2 Monitoring và Verification**

```bash
# Kiểm tra deployment status
kubectl rollout status deployment/hpp -n model-serving

# Kiểm tra service endpoint
kubectl get svc -n model-serving

# Test API endpoint
curl -X GET "http://<EXTERNAL-IP>:30000/docs"
```

---

## 8️⃣ **Best Practices**

### **8.1 Security**
- ✅ Sử dụng RBAC thay vì ClusterAdmin
- ✅ Separate ServiceAccount cho từng application
- ✅ Least privilege principle
- ✅ Secrets management với Kubernetes secrets

### **8.2 Scalability**
- ✅ Horizontal Pod Autoscaling
- ✅ Resource limits/requests
- ✅ LoadBalancer cho traffic distribution
- ✅ Multi-replica deployment

### **8.3 Monitoring**
- ✅ Health checks (readiness/liveness probes)
- ✅ Logging aggregation
- ✅ Metrics collection
- ✅ Alerting setup

---

## 9️⃣ **Troubleshooting Common Issues**

### **RBAC Permission Errors**
```bash
# Kiểm tra current permissions
kubectl auth can-i --list --as=system:serviceaccount:model-serving:default

# Fix bằng cách apply RBAC templates
helm upgrade hpp ./helm-charts/hpp -n model-serving
```

### **Image Pull Errors**
```bash
# Kiểm tra image registry
docker pull quandvrobusto/house-price-prediction-api:1

# Kiểm tra imagePullSecrets nếu private registry
kubectl create secret docker-registry regcred \
  --docker-server=docker.io \
  --docker-username=<username> \
  --docker-password=<password>
```

### **Service Discovery Issues**
```bash
# Kiểm tra service endpoints
kubectl get endpoints -n model-serving

# Kiểm tra DNS resolution
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup hpp.model-serving.svc.cluster.local
```

---

## 🔄 **Updating và Rollback**

### **Updates**
```bash
# Update image tag in values.yaml
helm upgrade hpp ./helm-charts/hpp -n model-serving

# Update với specific values
helm upgrade hpp ./helm-charts/hpp \
  --set image.tag=v2.0 \
  -n model-serving
```

### **Rollback**
```bash
# Xem history
helm history hpp -n model-serving

# Rollback to previous version
helm rollback hpp 1 -n model-serving
```

---

## 🎯 **Kết luận**

Dự án này demonstrate một **complete MLOps pipeline** với:

1. **Infrastructure as Code** (Terraform)
2. **Configuration Management** (Ansible) 
3. **CI/CD Automation** (Jenkins)
4. **Container Orchestration** (Kubernetes)
5. **Package Management** (Helm)
6. **Security** (RBAC)
7. **Monitoring & Debugging**

Đây là một foundation solid cho production-ready ML services với khả năng scale, maintain và monitor hiệu quả.
