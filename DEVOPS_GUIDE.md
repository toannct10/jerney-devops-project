# 📘 TÀI LIỆU QUY TRÌNH DEVSECOPS DỰ ÁN JERNEY (PHIÊN BẢN TINH GỌN)

> **Dành cho:** DevOps Fresher / Junior  
> **Mục tiêu:** Cung cấp quy trình DevSecOps chuẩn mực, thực tế, dễ hiểu, dễ debug và tự tin giải thích khi tham gia phỏng vấn tuyển dụng.

---

## 1. TỔNG QUAN KIẾN TRÚC HỆ THỐNG (3-TIER ARCHITECTURE)

Dự án Jerney là một ứng dụng web dạng 3 lớp (3-tier) kinh điển:

```
┌─────────────────────────────────────────────────────────────┐
│                       NGƯỜI DÙNG                            │
└──────────────────────────────┬──────────────────────────────┘
                               │ HTTP Request (Port 80)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. FRONTEND TIER                                            │
│    - Công nghệ: React (Vite) + Nginx Web Server             │
│    - Container: Alpine Nginx, non-root user (port 8080)     │
│    - Vai trò: Serve static assets (HTML/CSS/JS)             │
│               Reverse Proxy các request `/api/*` -> Backend │
└──────────────────────────────┬──────────────────────────────┘
                               │ Proxy Pass HTTP (Port 5000)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. BACKEND TIER                                             │
│    - Công nghệ: Node.js (Express Framework)                 │
│    - Container: Alpine Node 20, non-root user (port 5000)   │
│    - Init Process: dumb-init (xử lý PID 1 & graceful exit)  │
│    - Endpoint kiểm tra sức khỏe: `GET /api/health`          │
└──────────────────────────────┬──────────────────────────────┘
                               │ TCP Connection (Port 5432)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. DATABASE TIER                                            │
│    - Công nghệ: PostgreSQL 16 (Alpine)                      │
│    - Dữ liệu bền vững (Persistence): K8s PVC / Docker Volume│
└─────────────────────────────────────────────────────────────┘
```

---

## 2. CONTAINERIZATION (DOCKER & DOCKER COMPOSE)

### 2.1. Backend Dockerfile (`backend/Dockerfile`)
* **Multi-stage build:**
  * *Stage 1 (`build`):* Chỉ cài đặt production dependencies (`npm ci --only=production`), dọn dẹp npm cache.
  * *Stage 2 (`production`):* Sao chép `node_modules` và mã nguồn (`src/`) từ stage build sang. Tối ưu kích thước image và tách biệt môi trường build.
* **Bảo mật Non-root user:**
  * Tạo group `appgroup` và user `appuser` (UID/GID 1000). Chạy ứng dụng dưới `USER appuser` để giảm thiểu rủi ro bảo mật (nếu container bị chiếm quyền, kẻ tấn công không có quyền root trên host).
* **Quản lý tiến trình với `dumb-init`:**
  * Node.js khi chạy với PID 1 trong container không tự động xử lý các tín hiệu hạt nhân (`SIGTERM`, `SIGINT`) đúng cách.
  * Sử dụng `ENTRYPOINT ["dumb-init", "--"]` giúp chuyển tiếp tín hiệu chuẩn xác, hỗ trợ container tắt an toàn (graceful shutdown) khi có lệnh restart/stop từ Kubernetes hoặc Docker.

### 2.2. Frontend Dockerfile (`frontend/Dockerfile`)
* **Multi-stage build:**
  * *Stage 1 (`build`):* Dùng `node:20-alpine` cài đặt dependencies và build bundle React/Vite thành mã tĩnh (`npm run build` -> thư mục `/app/dist`).
  * *Stage 2 (`production`):* Dùng `nginx:1.27-alpine` để làm web server. Toàn bộ Node.js SDK nặng ~300MB được loại bỏ hoàn toàn, chỉ giữ lại Nginx và các file tĩnh, giúp image siêu nhẹ (~25MB).
* **Nginx Non-root:**
  * Cấu hình phân quyền cho user `nginx` (UID 101) đối với các thư mục `/var/cache/nginx`, `/var/log/nginx`, `/var/run/nginx.pid`.
  * Nginx lắng nghe ở cổng `8080` (vì non-root user trong Linux không được phép bind cổng đặc quyền `< 1024`).

### 2.3. Docker Compose (`docker-compose.yml`)
* **Môi trường phát triển cục bộ (Local Development):**
  * Định nghĩa đầy đủ 3 service: `db`, `backend`, `frontend`.
  * **Cơ chế Healthcheck:** Service `db` có healthcheck `pg_isready -U jerney_user -d jerney_db`.
  * **Kiểm soát thứ tự khởi động (`depends_on`):**
    ```yaml
    backend:
      depends_on:
        db:
          condition: service_healthy
    ```
    Điều này đảm bảo Backend chỉ được bật khi PostgreSQL đã thực sự sẵn sàng kết nối, không bị lỗi crash ngay khi vừa gõ `docker compose up`.
  * **Tính bền vững dữ liệu:** Volume `pgdata` lưu trữ thư mục `/var/lib/postgresql/data` tránh mất dữ liệu khi restart container.

---

## 3. QUY TRÌNH CI/CD (GITHUB ACTIONS)

Tệp workflow: `.github/workflows/ci-cd.yml`  
Quy trình được chuẩn hóa và tinh gọn thành **3 Jobs tuần tự, rõ ràng**:

```
[ Developer Push / Pull Request ]
              │
              ▼
    ┌───────────────────┐
    │  Job 1: ci-test   │  ◄── ESLint Code Quality
    │  (Matrix 2 Comps) │  ◄── npm audit (SCA Dependency Vulnerabilities)
    └─────────┬─────────┘
              │ (Nếu pass)
              ▼
    ┌───────────────────────┐
    │ Job 2: build-scan-push│  ◄── Docker Buildx build image
    │ (Matrix 2 Comps)      │  ◄── Trivy Scan lỗ hổng container
    └─────────┬─────────────┘  ◄── Push GHCR (Chỉ khi push to main)
              │ (Nếu push to main)
              ▼
    ┌───────────────────────┐
    │ Job 3: update-manifest│  ◄── Lấy Git Commit SHA ngắn gọn
    │                       │  ◄── Dùng sed cập nhật tag image vào k8s/jerney.yaml
    └───────────────────────┘  ◄── Git commit & push [skip ci]
```

### Chi tiết các Job:

1. **Phân biệt điều kiện kích hoạt (Trigger):**
   * **Khi mở Pull Request vào `main`:** Chạy Job 1 (`ci-test`) và Job 2 (`build-scan-push` ở chế độ build + scan nội bộ, **KHÔNG push** lên GHCR và **KHÔNG sửa** manifest). Điều này đảm bảo code PR được kiểm tra kỹ trước khi merge.
   * **Khi Push / Merge vào `main`:** Chạy đầy đủ cả 3 job, push image lên GHCR và tự động cập nhật tag trong file manifest K8s.

2. **Job 1: `ci-test` (Kiểm tra chất lượng mã nguồn & thư viện):**
   * Setup môi trường Node.js 20 có bật cache npm.
   * Chạy `npm run lint` để kiểm tra lỗi cú pháp/chuẩn viết code.
   * Chạy `npm audit --audit-level=high` (Software Composition Analysis) để phát hiện các thư viện phụ thuộc của dự án có lỗ hổng bảo mật nghiêm trọng hay không.

3. **Job 2: `build-scan-push` (Đóng gói & Quét bảo mật container):**
   * Sử dụng `docker/build-push-action` build image và nạp vào Docker daemon cục bộ (`load: true`).
   * Sử dụng **Trivy** quét trực tiếp image vừa build để tìm CVE/lỗ hổng cấp hệ điều hành Alpine và các thư viện hệ thống.
   * Nếu pass và đang ở nhánh `main`: Push image lên GitHub Container Registry (GHCR) với 2 tag:
     * Tag Git Commit SHA (ví dụ: `75da1da`)
     * Tag `latest`

4. **Job 3: `update-manifest` (Tự động hóa triển khai GitOps cơ bản):**
   * Lấy Git Commit SHA ngắn hạn (`git rev-parse --short HEAD`).
   * Sử dụng lệnh `sed` với biểu thức chính quy (regex) để cập nhật tag image trong file `k8s/jerney.yaml`:
     ```bash
     sed -i -E "s|(image:.*jerney-backend:).*|\1${IMAGE_TAG}|" k8s/jerney.yaml
     sed -i -E "s|(image:.*jerney-frontend:).*|\1${IMAGE_TAG}|" k8s/jerney.yaml
     ```
   * Tự động tạo commit với thông điệp `[skip ci]` và đẩy lên nhánh `main`.

---

## 4. TRIỂN KHAI TRÊN KUBERNETES (`k8s/jerney.yaml`)

File manifest đơn lẻ được cấu hình tinh gọn, sẵn sàng chạy trên cả cụm cục bộ (Minikube / Kind) lẫn cụm đám mây (AWS EKS):

| Thành phần | Loại Resource K8s | Mục đích & Chi tiết cấu hình |
| :--- | :--- | :--- |
| **Namespace** | `Namespace` (`jerney`) | Cô lập môi trường làm việc của dự án, tránh ảnh hưởng đến các ứng dụng khác trong cluster. |
| **Secret** | `Secret` (`jerney-db-secret`) | Lưu trữ credentials bằng `stringData` (K8s tự động mã hóa base64), cho phép inject trực tiếp qua `envFrom: secretRef` cực kỳ gọn gàng. |
| **Storage (PVC)** | `PersistentVolumeClaim` (`jerney-db-pvc`) | Yêu cầu 10Gi lưu trữ với `ReadWriteOnce`. Không gán cứng StorageClass để K8s tự động dùng **Default StorageClass** của cụm (Minikube/Kind/EKS). |
| **Database** | `Deployment` + `ClusterIP Service` | Replicas = 1; `strategy: Recreate` (để tránh lỗi xung đột gắn RWO volume); có cấu hình Readiness probe bằng lệnh `pg_isready`. |
| **Backend** | `Deployment` + `ClusterIP Service` | Replicas = 2; Service chỉ truy cập nội bộ qua port 5000.<br>**InitContainers:** Pod chạy `busybox` kiểm tra cổng 5432 để đảm bảo DB sẵn sàng mới khởi động backend.<br>**Security:** `runAsNonRoot: true`.<br>**Probes:** HTTP probe tại `/api/health`. |
| **Frontend** | `Deployment` + `NodePort Service` | Replicas = 2; Service loại `NodePort` mở cổng ngoài (port 80 -> targetPort 8080) để người dùng truy cập web.<br>**Security:** `runAsNonRoot: true`.<br>**Probes:** HTTP probe tại `/`. |

### Các cải tiến giúp K8s tinh gọn tối đa nhưng giữ vững chất lượng DevOps:
1. **Tinh giản Labels (`app: jerney-*`):** Thay vì khai báo lặp lại 3 nhãn dài dòng (`app.kubernetes.io/...`), chuẩn hóa về nhãn `app` kinh điển của Kubernetes, giúp giảm hơn 40 dòng cú pháp thừa.
2. **Sử dụng `stringData` trong Secret:** Thay thế việc encode base64 thủ công bằng `stringData`, vừa dễ đọc vừa giúp `backend` và `db` dùng chung `envFrom: secretRef` chỉ với 2 dòng cấu hình.
3. **InitContainer tinh gọn 1 dòng:** Rút ngắn cú pháp wait loop của `busybox` mà vẫn đảm bảo tính năng chống crash pod khi DB chưa sẵn sàng.
4. **Loại bỏ hardcoded UID:** Bỏ `runAsUser: 1000/101` thừa thãi vì Dockerfile đã chỉ định `USER`, chỉ giữ `runAsNonRoot: true` đúng chuẩn nguyên tắc bảo mật.
5. **Đảm bảo 100% chuẩn DevOps:** Vẫn giữ đầy đủ Stateful data persistence (PVC), Self-healing (Probes), QoS (Resource requests/limits), Container security (Non-root), và Microservice dependency handling (InitContainers).

---

## 5. HƯỚNG DẪN THỰC THÀNH KIỂM THỬ

### 5.1. Chạy tại máy cá nhân với Docker Compose
```bash
# Di chuyển vào thư mục dự án
cd Jerney-devops

# Khởi động toàn bộ hệ thống
docker compose up -d --build

# Kiểm tra trạng thái các container
docker compose ps

# Kiểm tra log Backend
docker compose logs -f backend

# Kiểm tra phản hồi API Health Check
curl http://localhost/api/health
# Kết quả mong đợi: {"status":"ok","message":"Jerney API is vibing ✨"}

# Truy cập trình duyệt web
http://localhost

# Dừng và dọn dẹp hệ thống
docker compose down
```

### 5.2. Triển khai lên cụm Kubernetes (Minikube hoặc EKS)
```bash
# 1. Tạo và triển khai toàn bộ manifest
kubectl apply -f k8s/jerney.yaml

# 2. Theo dõi trạng thái triển khai
kubectl get all -n jerney

# 3. Xem pod Backend đã hoàn thành initContainer chưa
kubectl get pods -n jerney -w

# 4. Truy cập giao diện ứng dụng (Port-Forward nếu chạy test local)
kubectl port-forward svc/jerney-frontend 8080:80 -n jerney
# Mở trình duyệt tại: http://localhost:8080
```

---

## 6. BỘ CÂU HỎI PHỎNG VẤN & GỢI Ý TRẢ LỜI CHO DEVOPS FRESHER/JUNIOR

Dưới đây là 7 câu hỏi kinh điển các nhà tuyển dụng thường hỏi đối với một dự án như Jerney:

#### Q1: Tại sao em lại sử dụng Multi-stage build trong Dockerfile?
> **Trả lời:** Em sử dụng Multi-stage build để tách biệt giữa môi trường build và môi trường chạy production. Ví dụ ở Frontend React, em cần Node.js để biên dịch source code sang HTML/CSS/JS, nhưng khi chạy thực tế chỉ cần Nginx để serve static files. Nhờ multi-stage build, container image production loại bỏ được toàn bộ Node.js SDK, giảm kích thước từ ~350MB xuống chỉ còn ~25MB, giúp tiết kiệm dung lượng lưu trữ, kéo image nhanh hơn và giảm thiểu diện tích bề mặt tấn công (attack surface).

#### Q2: Tại sao em lại tạo user riêng thay vì dùng user root mặc định trong container?
> **Trả lời:** Chạy container bằng user root là một lỗ hổng bảo mật nghiêm trọng. Nếu ứng dụng có lỗ hổng (ví dụ Remote Code Execution) hoặc kẻ tấn công thoát được khỏi container (container breakout), họ sẽ có ngay quyền root trên máy host. Do đó, em tạo user non-root (`appuser` trong Backend, `nginx` trong Frontend) và gán `runAsNonRoot: true` trên Kubernetes để tuân thủ nguyên tắc đặc quyền tối thiểu (Least Privilege).

#### Q3: `dumb-init` trong Dockerfile của Backend có nhiệm vụ gì?
> **Trả lời:** Khi một ứng dụng chạy trong container dưới dạng tiến trình chính, nó nhận PID 1. Trong Linux, tiến trình PID 1 có trách nhiệm đặc biệt là dọn dẹp các tiến trình con mồ côi (zombie processes) và bắt các tín hiệu hạt nhân (`SIGTERM`, `SIGINT`). Node.js không được thiết kế để xử lý tốt các tác vụ của PID 1, dẫn đến việc khi ta gõ `docker stop` hoặc Kubernetes xoá Pod, container thường bị treo 10-30 giây rồi mới bị `SIGKILL` cưỡng bức. `dumb-init` đóng vai trò là một init process siêu nhẹ giúp tiếp nhận và chuyển tiếp tín hiệu đúng cách đến tiến trình Node.js, giúp ứng dụng tắt an toàn (graceful shutdown).

#### Q4: Trong Kubernetes, tại sao Backend Pod lại cần `initContainers` để đợi Database?
> **Trả lời:** Backend Node.js của ứng dụng kết nối trực tiếp đến PostgreSQL ngay khi khởi động (`await db.initDB()`). Nếu Database chưa khởi tạo xong hoặc chưa mở port 5432, Backend sẽ gặp lỗi kết nối và sập ngay lập tức (`CrashLoopBackOff`). Mặc dù Kubernetes có cơ chế restart, nhưng việc pod bị crash liên tục không phải là giải pháp tốt. Do đó, em dùng một `initContainer` nhẹ (`busybox`) chạy vòng lặp kiểm tra cổng 5432 của service `jerney-db`, khi nào DB thực sự mở cổng thì container Backend mới bắt đầu chạy.

#### Q5: Phân biệt sự khác nhau giữa Liveness Probe và Readiness Probe?
> **Trả lời:** 
> - **Liveness Probe:** Kiểm tra xem container còn sống hay không. Nếu Liveness probe thất bại liên tục, Kubernetes sẽ kill container và restart lại một container mới.
> - **Readiness Probe:** Kiểm tra xem container đã sẵn sàng nhận traffic người dùng hay chưa. Nếu Readiness probe thất bại, Pod không bị restart, nhưng Kubernetes Service sẽ tạm thời loại Pod đó ra khỏi danh sách Endpoints để không điều hướng request của người dùng vào pod đó.

#### Q6: Tại sao Deployment của Database lại dùng chiến lược `strategy: Recreate` thay vì `RollingUpdate`?
> **Trả lời:** PersistentVolumeClaim của cơ sở dữ liệu sử dụng chế độ truy cập `ReadWriteOnce` (RWO), tức là volume chỉ có thể được mount bởi duy nhất một Node tại một thời điểm. Nếu ta dùng `RollingUpdate`, Kubernetes sẽ tạo Pod mới trước khi xóa Pod cũ, dẫn đến việc cả 2 Pod cùng cố gắng gắn vào một Volume RWO duy nhất gây ra lỗi `Multi-Attach error`. Với `Recreate`, Kubernetes sẽ tắt hẳn Pod cũ rồi mới khởi tạo Pod mới, đảm bảo an toàn cho dữ liệu.

#### Q7: Luồng CI/CD của em xử lý khác nhau như thế nào giữa Pull Request và Push lên nhánh main?
> **Trả lời:** 
> - Với **Pull Request:** Pipeline chỉ chạy bước kiểm tra chất lượng (Linting), quét lỗ hổng thư viện (npm audit), và build thử image để Trivy quét an ninh. Hoàn toàn **không push** image lên GHCR và **không cập nhật manifest** nhằm bảo vệ môi trường release.
> - Với **Push lên main:** Sau khi toàn bộ các bước kiểm tra thành công, pipeline sẽ push image có gắn tag Git Commit SHA lên GHCR, sau đó tự động cập nhật tag mới đó vào manifest Kubernetes để chuẩn bị cho việc triển khai.
