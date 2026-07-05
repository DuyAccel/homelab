# 🌌 The Ultimate GitOps Kubernetes Homelab

[![FluxCD](https://img.shields.io/badge/GitOps-FluxCD-blueviolet?style=for-the-badge&logo=flux)](https://fluxcd.io/)
[![Cilium](https://img.shields.io/badge/CNI-Cilium%20(eBPF)-blue?style=for-the-badge&logo=cilium)](https://cilium.io/)
[![Istio](https://img.shields.io/badge/Service%20Mesh-Istio%20Ambient-blue?style=for-the-badge&logo=istio)](https://istio.io/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-Bare--Metal-blue?style=for-the-badge&logo=kubernetes)](https://kubernetes.io/)
[![SOPS](https://img.shields.io/badge/Security-SOPS%20%2B%20Age-green?style=for-the-badge)](https://github.com/getsops/sops)

Chào mừng bạn đến với **k8s-homelab** — dự án xây dựng và quản trị hạ tầng cụm Kubernetes trên thiết bị vật lý (Bare-Metal) theo chuẩn doanh nghiệp. Dự án được vận hành hoàn toàn tự động bằng triết lý **GitOps** (thông qua **FluxCD**) và bảo mật đa tầng với **Istio Ambient Service Mesh**.

Đây không chỉ là nơi lưu trữ cấu hình, mà còn là một **cẩm nang thực tế (guideline)** dành cho những ai muốn nghiên cứu sâu về thiết kế hệ thống Cloud-Native hiện đại, tối ưu hóa mạng với eBPF, và quản lý lưu trữ phân tán.

---

## 🛠️ Trụ Cột Công Nghệ (Tech Stack)

Hệ thống được thiết kế dựa trên các công nghệ tiên tiến nhất trong hệ sinh thái CNCF:

| Phân nhóm | Công nghệ lựa chọn | Mục đích sử dụng |
| :--- | :--- | :--- |
| **GitOps Engine** | **FluxCD v2** | Tự động đồng bộ trạng thái từ Git vào Cluster, tự động cập nhật image. |
| **Networking & Security** | **Cilium (eBPF)** | CNI hiệu năng cao, định tuyến mạng không qua iptables, tích hợp L2 Announcements làm LoadBalancer. |
| **Service Mesh** | **Istio (Ambient Mode)** | Bảo mật mTLS sidecar-less (ztunnel), quản lý phân quyền (AuthorizationPolicy) & L7 routing qua Waypoint Proxy. |
| **Storage Layer** | **Longhorn / OpenEBS** | Phân cấp lưu trữ: Block storage phân tán có tính năng replication. |
| **Object Storage** | **Garage S3** | Lưu trữ S3-compatible dung lượng nhẹ, phân tán đa node, tối ưu cho môi trường Homelab. |
| **Database Server** | **CloudNative-PG** | DBaaS quản lý PostgreSQL Cluster (High Availability, auto-failover, backup/restore). |
| **Secret Management** | **Mozilla SOPS & AGE** | Mã hóa an toàn thông tin nhạy cảm (secrets) trực tiếp trên Git. |
| **Observability** | **VictoriaMetrics & Grafana** | Thu thập metric hiệu năng siêu cao, lưu trữ tối ưu hơn Prometheus truyền thống. |

---

## 🏗️ Kiến Trúc Luồng Dữ Liệu (Data & Network Flow)

Dưới đây là mô hình định tuyến mạng và luồng xử lý yêu cầu đi từ Internet qua các tầng bảo mật của hệ thống để tới ứng dụng:

```mermaid
graph TD
    Client[Client / Internet] -->|Gửi Request| L2[Cilium L2 LoadBalancer]
    L2 -->|Phân bổ IP tĩnh| GW[Gateway API / Istio Ingress]
    
    subgraph ambient ["Istio Ambient Mesh (Sidecar-less)"]
        GW -->|mTLS Encrypted Tunnel| Z[ZTunnel - L4 Secure Transport]
        Z -->|Chuyển tiếp L7 Traffic| WP[Waypoint Proxy - L7 Policies]
        WP -->|Authorized & Validated| App[Application Pod e.g. duynch.com]
    end
    
    subgraph data ["Data & Storage Layers"]
        App -->|Đọc/Ghi Block| LH[Longhorn Distributed Storage]
        App -->|Lưu trữ tệp tin S3| GR[Garage S3 Object Storage]
        App -->|Truy vấn dữ liệu| PG[CloudNative-PG High-Availability]
    end

    style ambient fill:#1d2a4a,stroke:#3b5998,stroke-width:2px;
    style data fill:#1a3325,stroke:#2e7d32,stroke-width:2px;
```

---

## 📁 Tổ Chức Thư Mục Dự Án (Directory Layout)

Cấu trúc Repository được phân chia khoa học theo các lớp trừu tượng (Layers) giúp dễ dàng quản lý đa cluster:

```bash
├── clusters/                   # Tầng cấu hình đặc thù cho từng Cluster
│   ├── homelab/                # Cấu hình cụm Homelab chính
│   │   ├── flux-system/        # Thành phần cốt lõi của FluxCD
│   │   ├── infrastructure.yaml # Kích hoạt hạ tầng dùng chung cho cụm này
│   │   └── stacks/             # Các stack ứng dụng được kích hoạt (Staging/Prod)
│   └── edge-vps/               # Cấu hình cụm VPS biên (Edge Cluster)
│
├── infrastructure/             # Định nghĩa tài nguyên hạ tầng chung (Kustomize)
│   ├── cilium/                 # Cấu hình mạng L2 Announcement & IPAM
│   ├── istio/                  # Bộ cài đặt Istio Ambient (CNI, ztunnel, istiod)
│   ├── longhorn/               # Bộ lưu trữ khối phân tán
│   ├── garage/                 # Cấu hình cụm S3 Storage
│   └── cloudnative-pg/         # PostgreSQL Operator
│
├── sources/                    # Định nghĩa các nguồn Helm / OCI repositories bên thứ 3
│   └── helm/                   # File YAML đăng ký Helm Repository cho FluxCD
│
├── modules/                    # Thư mục chứa các module tái sử dụng (Platform Templates)
│   └── infrastructure/         # Các module giám sát (Monitor, Logging, Authen)
│
└── application/                # Chứa manifests thực tế của các ứng dụng (Workloads)
    └── website/
        └── duynch-com/         # Cấu hình deploy website cá nhân duynch.com
```

---

## 🚀 Hướng Dẫn Khởi Tạo Cụm (Bootstrap Guide)

Để khởi tạo lại hoặc cài đặt mới cụm Kubernetes dựa trên repo này, thực hiện theo các bước sau:

### 1. Cài đặt các công cụ CLI cần thiết
Đảm bảo bạn đã cài đặt các công cụ sau trên máy cá nhân:
* `kubectl`
* `flux` (FluxCD CLI)
* `age-keygen` (để giải mã SOPS)

### 2. Cấu hình Khóa Giải Mã SOPS
Tạo hoặc import khóa bí mật AGE vào cụm Kubernetes để FluxCD có quyền tự động giải mã các Secret:
```bash
cat key.txt | kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=keys.txt=/dev/stdin
```

### 3. Thực hiện Bootstrap FluxCD
Kích hoạt FluxCD đồng bộ với repo này (ví dụ cho cụm `homelab`):
```bash
flux bootstrap github \
  --owner=duyaccel \
  --repository=k8s-homelab \
  --branch=main \
  --path=./clusters/homelab \
  --personal
```

Sau khi chạy lệnh trên, FluxCD sẽ tự động tải các tài nguyên cần thiết, tạo các Custom Resource Definitions (CRDs) và tự động kéo toàn bộ các hạ tầng như Cilium, Istio, Longhorn, CloudNative-PG về cài đặt mà không cần bất kỳ sự can thiệp thủ công nào.

---

## 💡 Điểm Sáng Thiết Kế Kỹ Thuật (Engineering Highlights)

* **Ambient Mesh (No Sidecar)**: Không giống như thiết kế Istio truyền thống tiêm một Envoy proxy vào mọi pod, cụm lab này sử dụng **Istio Ambient mode**. Việc này giúp giảm hơn 70% mức tiêu thụ tài nguyên RAM của cụm, đồng thời loại bỏ việc phải restart lại Pod ứng dụng khi cập nhật Service Mesh.
* **Bare-metal LoadBalancer với Cilium**: Thay vì phụ thuộc vào cloud providers hoặc dùng Metallb, hệ thống sử dụng **Cilium L2 Announcement** để tự động gán các IP tĩnh từ dải mạng cục bộ (ví dụ: `192.168.10.x`) trực tiếp vào Gateway API của K8s.
* **Declarative DBaaS**: Cấu hình PostgreSQL hoàn toàn dưới dạng mã nguồn (Declarative) thông qua CloudNative-PG, hỗ trợ tự phục hồi khi có Node chết vật lý và backup tự động sang cụm S3 của Garage.

---
*Chúc bạn có những trải nghiệm tự học và khám phá vui vẻ trên cụm K8s GitOps này! Mọi đóng góp hoặc câu hỏi vui lòng mở một Issue.*
