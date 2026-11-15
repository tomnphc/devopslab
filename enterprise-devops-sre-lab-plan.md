# Enterprise DevOps / SRE Lab trên VMware vSphere và OpenShift SNO

> **Mục tiêu:** Xây dựng một môi trường lab thực hành end-to-end các
> công việc DevOps/SRE trong doanh nghiệp: provisioning hạ tầng, vận
> hành OpenShift, CI/CD, GitOps, observability, security, secrets,
> messaging, backup/restore và incident response.
>
> **Đối tượng sử dụng:** Kỹ sư DevOps/SRE/Platform Engineer; tài liệu
> này cũng là project brief để coding agent triển khai từng phase.
>
> **Nguyên tắc:** Triển khai theo từng phase có tiêu chí nghiệm thu.
> Không cài tất cả cùng lúc. Mọi cấu hình có thể tái tạo phải được lưu
> trong Git; không đưa secret thật hoặc thông tin hạ tầng nội bộ vào
> repository.

------------------------------------------------------------------------

## 0. Executive summary

### 0.1 Kiến trúc mục tiêu

-   **Virtualization:** VMware vSphere do công ty cung cấp.
-   **Kubernetes platform:** OpenShift Container Platform - Single Node
    OpenShift (SNO), cài trên một VM.
-   **Source control / CI:** GitLab và GitLab CI; GitLab Runner chạy
    trên VM riêng hoặc runner dùng chung do công ty quản lý.
-   **Image registry:** GitLab Container Registry trước; Harbor là bài
    tập mở rộng.
-   **CD / GitOps:** Red Hat OpenShift GitOps (Argo CD).
-   **Metrics / alerting:** Prometheus, Grafana, Alertmanager; ưu tiên
    OpenShift built-in monitoring cho cluster và user-workload
    monitoring.
-   **Logging:** Loki + Grafana (hoặc OpenShift Logging nếu có
    entitlement và muốn học hệ sinh thái Red Hat).
-   **Tracing:** OpenTelemetry Collector + Tempo hoặc Jaeger.
-   **Messaging:** Apache Kafka do Strimzi Operator quản lý.
-   **Application:** Một ứng dụng web demo gồm frontend, REST API,
    PostgreSQL, Redis và Kafka.
-   **Security:** Vault, Trivy, SonarQube Community, SBOM, image signing
    là các phase nâng cao.
-   **Automation:** Ansible, Terraform vSphere provider, Helm,
    Kustomize.
-   **Backup / DR:** OADP/Velero khi phù hợp, backup database riêng và
    diễn tập restore.
-   **Operations:** Runbook, SLI/SLO, alert routing, incident
    simulation, capacity và upgrade exercise.

### 0.2 Những quyết định kiến trúc quan trọng

1.  **Không coi SNO là production HA.** SNO có một node nên node failure
    đồng nghĩa cluster workload/control plane bị ảnh hưởng. Đây là lab
    học vận hành, không phải mô hình HA.
2.  **Không đặt toàn bộ hệ sinh thái nặng lên SNO ngay từ đầu.** GitLab,
    registry, monitoring, Kafka, logging và tracing có thể tiêu tốn
    RAM/CPU/storage. Dùng dịch vụ GitLab công ty cấp hoặc VM riêng nếu
    có thể.
3.  **Kafka không phải monitoring.** Kafka là event streaming/message
    platform. Prometheus scrape metrics của Kafka/Strimzi; Grafana hiển
    thị; Alertmanager cảnh báo.
4.  **Tách CI và CD.** GitLab CI build/test/scan/push image. Pipeline
    cập nhật GitOps repository. Argo CD pull repository và reconcile
    trạng thái trên OpenShift. Không để CI chạy `oc apply` trực tiếp
    trong luồng chuẩn.
5.  **Dùng OpenShift-native trước.** OpenShift GitOps, built-in
    monitoring, Operators và CSI phù hợp để học cách vận hành nền tảng
    Red Hat. Dùng Helm/Kustomize cho workload phù hợp.
6.  **Triển khai từng lớp.** Cluster → app → CI → GitOps → monitoring →
    logs/traces → Kafka → secrets/security → backup/DR.
7.  **Không dùng production credentials/data.** Chỉ dùng dữ liệu giả
    lập, account riêng và hostname lab được công ty cho phép.

------------------------------------------------------------------------

## 1. Phạm vi và mục tiêu học tập

### 1.1 Năng lực cần đạt

  -----------------------------------------------------------------------
  Nhóm                                Sau khi hoàn thành cần làm được
  ----------------------------------- -----------------------------------
  Infrastructure                      Tạo VM, sizing, network, DNS, NTP,
                                      storage, snapshot policy; hiểu
                                      dependency của cluster

  Linux                               Quản trị RHEL/Rocky/Ubuntu,
                                      systemd, journal, disk, network,
                                      TLS, firewall

  OpenShift                           Cài SNO, quản lý project, RBAC,
                                      SCC, Route, Service, PVC, Operator,
                                      upgrade và troubleshoot

  CI                                  Pipeline build/test/lint/security
                                      scan, artifact, cache, image
                                      tagging, runner troubleshooting

  GitOps                              Bootstrap Argo CD, App of
                                      Apps/ApplicationSet, promotion,
                                      drift detection, rollback

  Observability                       Metrics, logs, traces, dashboards,
                                      alert rules, alert routing,
                                      correlation

  Messaging                           Kafka topic, partition, consumer
                                      group, lag, retention, replication
                                      concept, schema evolution

  Security                            RBAC least privilege, secret
                                      rotation, image scanning, SBOM,
                                      network policy, TLS

  Reliability                         SLI/SLO, error budget, capacity,
                                      backup/restore, incident management

  Automation                          Ansible, Terraform, Helm,
                                      Kustomize, Git workflows và
                                      idempotency
  -----------------------------------------------------------------------

### 1.2 Tiêu chí hoàn thành cuối project

-   Một developer push code lên GitLab.
-   GitLab CI chạy test, quality/security scan, build container image và
    push registry.
-   Pipeline cập nhật image tag trong GitOps repo.
-   Argo CD tự phát hiện thay đổi và deploy lên OpenShift.
-   Người dùng truy cập ứng dụng qua OpenShift Route.
-   Metrics, logs, traces của ứng dụng và cluster có thể quan sát tập
    trung.
-   Ứng dụng gửi event vào Kafka; consumer xử lý event và lưu kết quả
    vào PostgreSQL.
-   Secret được lấy từ Vault hoặc giải pháp secret management đã chọn,
    không hardcode trong Git.
-   Có dashboard và alert cho lỗi HTTP, latency, pod restart, CPU/RAM,
    PVC, Kafka lag.
-   Có runbook và thực hành ít nhất 5 tình huống incident.
-   Có backup và chứng minh restore thành công.

------------------------------------------------------------------------

## 2. Kiến trúc tổng thể

### 2.1 Logical architecture

``` text
Developer workstation
       |
       | git push / merge request
       v
+--------------------------+
| GitLab                   |
| - Git repositories       |
| - GitLab CI pipelines    |
| - Container registry     |
+--------------------------+
       |
       | build / test / scan / push image
       | update GitOps repository
       v
+--------------------------+
| GitOps repository        |
| - app manifests          |
| - Helm values/Kustomize  |
| - environment overlays   |
+--------------------------+
       |
       | Argo CD pulls desired state
       v
+--------------------------------------------------+
| VMware vSphere                                   |
|                                                  |
|  +--------------------------------------------+  |
|  | OpenShift SNO VM                           |  |
|  |                                            |  |
|  |  OpenShift platform                        |  |
|  |  - API / Scheduler / Controllers            |  |
|  |  - OVN-Kubernetes / Ingress / Routes        |  |
|  |  - Storage / Operators                      |  |
|  |                                            |  |
|  |  Platform services                         |  |
|  |  - OpenShift GitOps (Argo CD)               |  |
|  |  - Prometheus / Grafana / Alertmanager      |  |
|  |  - Loki / OpenTelemetry / Tempo             |  |
|  |  - Strimzi / Kafka                          |  |
|  |  - Vault integration                       |  |
|  |                                            |  |
|  |  Application namespace                     |  |
|  |  - Frontend                                 |  |
|  |  - REST API                                 |  |
|  |  - Worker / Kafka consumer                  |  |
|  |  - PostgreSQL / Redis                       |  |
|  +--------------------------------------------+  |
+--------------------------------------------------+
       ^
       |
       +------ Users / Admin workstation
       |
       +------ GitLab Runner (prefer external VM)
       |
       +------ DNS / NTP / vCenter / Registry
```

### 2.2 Physical VM đề xuất

  ----------------------------------------------------------------------------------------------------
  VM / Service         Vai trò                                  Sizing khởi điểm Ghi chú
  -------------------- ----------------------------------- --------------------- ---------------------
  `ocp-sno-01`         OpenShift SNO                        12-16 vCPU, 48-64 GB Khuyến nghị cho lab
                                                            RAM, 300-500 GB disk nhiều operator; không
                                                                                 phải minimum

  `gitlab-runner-01`   CI runner (nếu không dùng shared        4-8 vCPU, 8-16 GB Có thể dùng runner
                       runner)                                  RAM, 100 GB disk của công ty

  `tools-01`           Ansible/Terraform/bastion/utility   2-4 vCPU, 4-8 GB RAM, Có thể dùng
  (optional)                                                          60 GB disk workstation/bastion
                                                                                 được cấp

  GitLab               Git + CI + registry                        Ưu tiên GitLab GitLab self-managed
                                                                 instance sẵn có full stack cần sizing
                                                                                 riêng, không nhồi vào
                                                                                 SNO

  Harbor (optional)    Image registry                          4 vCPU, 8 GB RAM, Chỉ triển khai sau
                                                                 disk theo image khi pipeline cơ bản
                                                                       retention chạy ổn
  ----------------------------------------------------------------------------------------------------

**Lưu ý sizing:** Tài liệu Red Hat 2026 ghi nhận SNO basic minimum thay
đổi theo phiên bản: OpenShift 4.21 trở về trước là 8 CPU cores, 16 GB
RAM, 100 GB storage; OpenShift 4.22 trở lên là 4 CPU cores, 16 GB RAM,
100 GB storage. Đây là mức tối thiểu, không phải sizing thực tế cho lab
có nhiều operator. Với stack đề xuất, nên cấp 12-16 vCPU và 48-64 GB RAM
nếu vSphere capacity cho phép. Kiểm tra lại yêu cầu đúng với phiên bản
OCP và phương thức cài đặt trước khi tạo VM.

### 2.3 Chia namespace / project

  Namespace                              Nội dung
  -------------------------------------- -------------------------------------------------
  `openshift-gitops`                     Argo CD Operator và instance
  `openshift-monitoring`                 Platform monitoring do OpenShift quản lý
  `openshift-user-workload-monitoring`   Monitoring cho user workloads
  `observability`                        Grafana/Loki/Tempo/OTel nếu tự triển khai
  `kafka`                                Strimzi Operator và Kafka CR
  `demo-dev`                             Ứng dụng môi trường dev
  `demo-stage`                           Ứng dụng môi trường staging (nếu tài nguyên đủ)
  `vault`                                Vault lab
  `security`                             Trivy Operator / SonarQube (nếu tự host)
  `database`                             PostgreSQL/Redis nếu tách khỏi app namespace
  `backup`                               Backup tooling nếu cần namespace riêng

Không cần tạo tất cả namespace ngay từ đầu. Chỉ tạo khi phase tương ứng
bắt đầu.

------------------------------------------------------------------------

## 3. Chuẩn bị trước khi cài OpenShift

### 3.1 Việc cần xác nhận với đội hạ tầng

-   [ ] Có quyền tạo VM trên vCenter/cluster/resource pool/datastore.
-   [ ] Có đủ CPU/RAM/storage capacity và được phép chạy
    nested/container workloads.
-   [ ] VM network/port group nào được phép sử dụng.
-   [ ] IP tĩnh dành cho SNO.
-   [ ] DNS record cho `api.<cluster>.<base-domain>`.
-   [ ] DNS wildcard `*.apps.<cluster>.<base-domain>`.
-   [ ] DNS forward/reverse phù hợp chính sách nội bộ.
-   [ ] NTP server nội bộ.
-   [ ] Gateway, subnet mask, VLAN, DNS server.
-   [ ] Đường đi tới Red Hat registry hoặc mirror registry.
-   [ ] Proxy, firewall egress, trusted CA nếu môi trường yêu cầu.
-   [ ] Có pull secret hợp lệ và entitlement/subscription phù hợp.
-   [ ] Được phép expose API 6443 và Ingress 80/443 từ các subnet cần
    thiết.
-   [ ] Có phương án cấp phát LoadBalancer/VIP hoặc cơ chế tương ứng cho
    SNO theo installer.
-   [ ] Có StorageClass phù hợp cho PVC; xác nhận CSI/vSphere
    integration và quyền vCenter nếu sử dụng.

Không tự ý mở firewall. Lập bảng source/destination/port/protocol và nhờ
network team phê duyệt theo policy công ty.

### 3.2 Network plan mẫu

Thay toàn bộ giá trị mẫu bằng IP do đội hạ tầng cấp.

  Thành phần      Ví dụ                                   Mô tả
  --------------- --------------------------------------- --------------------------
  Cluster name    `sre-lab`                               Tên cluster
  Base domain     `lab.example.internal`                  Domain lab được cấp
  Node FQDN       `sno-01.lab.example.internal`           Hostname node
  Node IP         `10.10.20.50`                           IP tĩnh
  Gateway         `10.10.20.1`                            Default gateway
  DNS             `10.10.1.10`                            DNS resolver
  API VIP         `10.10.20.51`                           Theo phương thức cài đặt
  Ingress VIP     `10.10.20.52`                           Theo phương thức cài đặt
  API endpoint    `api.sre-lab.lab.example.internal`      API
  Apps wildcard   `*.apps.sre-lab.lab.example.internal`   Route

**Quan trọng:** Không mặc định API VIP và Ingress VIP có thể dùng chung
hoặc khác nhau; làm theo đúng hướng dẫn SNO của phiên bản/installer. Với
SNO, các địa chỉ này có thể được cấu hình theo cơ chế đặc thù của
installer.

### 3.3 Chuẩn bị VM trên vSphere

1.  Tạo VM theo sizing đã thống nhất.
2.  Chọn guest OS profile phù hợp với RHCOS/OpenShift installer.
3.  Dùng datastore đủ dung lượng và latency phù hợp.
4.  Kết nối đúng port group/VLAN.
5.  Đảm bảo NIC và MAC ổn định nếu dùng DHCP reservation.
6.  Với VM trên VMware, kiểm tra yêu cầu `disk.EnableUUID=TRUE` theo
    hướng dẫn cài đặt tương ứng.
7.  Không dùng snapshot VM như phương án backup/DR của OpenShift.
8.  Ghi nhận VM ID, datastore, port group, IP, MAC, vCPU, RAM, disk và
    owner.
9.  Kiểm tra thời gian/NTP và DNS trước khi boot discovery ISO.

### 3.4 Chọn phương thức cài đặt

Ưu tiên **Assisted Installer** nếu mục tiêu là dựng lab nhanh và học quy
trình cài đặt SNO. Chọn Agent-based Installer nếu muốn luyện sâu hơn
cách tạo `install-config.yaml`, agent config, ISO và tự động hóa cài
đặt.

Tài liệu tham khảo: - Assisted Installer:
https://docs.redhat.com/en/documentation/assisted_installer_for_openshift_container_platform/2026/html/installing_openshift_container_platform_with_the_assisted_installer/ -
SNO installation:
https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/installing_on_a_single_node/installing_on_a_single_node

### 3.5 Cài SNO bằng Assisted Installer - checklist

1.  Truy cập Assisted Installer bằng tài khoản Red Hat phù hợp.
2.  Tạo cluster mới, chọn OpenShift version còn được hỗ trợ.
3.  Chọn Single Node OpenShift.
4.  Khai báo cluster name, base domain, pull secret và SSH public key.
5.  Cấu hình networking: static IP hoặc DHCP reservation; hostname,
    gateway, DNS, interface.
6.  Khai báo API/Ingress addressing theo yêu cầu wizard và DNS plan.
7.  Chọn disk cài đặt đúng virtual disk.
8.  Tải discovery ISO.
9.  Mount ISO vào VM qua vSphere console.
10. Boot VM và chờ host xuất hiện trong Assisted Installer.
11. Kiểm tra validation; xử lý lỗi DNS, NTP, disk, CPU/RAM, network.
12. Bắt đầu installation và theo dõi từng bước.
13. Sau khi hoàn tất, tải/cấu hình `oc` CLI và đăng nhập cluster.

Ví dụ kiểm tra sau cài đặt:

``` bash
oc login https://api.sre-lab.lab.example.internal:6443
oc get nodes -o wide
oc get clusterversion
oc get clusteroperators
oc get pods -A
oc get storageclass
oc get co
```

Điều kiện nghiệm thu: - Node `Ready`. - ClusterVersion không degraded. -
ClusterOperators ổn định, không có operator degraded kéo dài. -
Console/API truy cập được. - DNS API và wildcard apps hoạt động. - Có
StorageClass/PVC test được nếu ứng dụng cần persistent storage. - Đã ghi
lại cách đăng nhập và quy trình recovery an toàn.

------------------------------------------------------------------------

## 4. Chuẩn bị Git repository

Tạo tối thiểu hai repository riêng biệt.

### 4.1 Repository 1: `sre-lab-app`

Chứa source code ứng dụng và pipeline:

``` text
sre-lab-app/
├── frontend/
├── api/
├── worker/
├── database/
│   └── migrations/
├── tests/
├── Dockerfile
├── .gitlab-ci.yml
├── Makefile
└── README.md
```

### 4.2 Repository 2: `sre-lab-gitops`

Chứa desired state:

``` text
sre-lab-gitops/
├── bootstrap/
│   └── root-app.yaml
├── applications/
│   ├── demo-app/
│   │   ├── base/
│   │   └── overlays/
│   │       ├── dev/
│   │       └── stage/
│   ├── kafka/
│   ├── observability/
│   └── vault/
├── platform/
│   ├── namespaces/
│   ├── quotas/
│   ├── network-policies/
│   └── operators/
└── README.md
```

**Quy tắc:** Secret không mã hóa hoặc secret plaintext không được
commit. Dùng Vault/External Secrets/VSO hoặc SOPS + age nếu có lý do phù
hợp.

------------------------------------------------------------------------

## 5. Xây dựng ứng dụng demo

Ứng dụng nên đủ nhỏ để dễ debug nhưng có đủ thành phần để tạo sự cố thực
tế.

### 5.1 Business flow gợi ý

Mô phỏng ứng dụng giao dịch ngân hàng đơn giản:

1.  Frontend gửi yêu cầu tạo giao dịch.
2.  REST API xác thực payload.
3.  API ghi transaction vào PostgreSQL.
4.  API publish event `transaction.created` lên Kafka.
5.  Worker consume event và cập nhật trạng thái xử lý.
6.  Redis cache một số thông tin đọc thường xuyên.
7.  API cung cấp health endpoint, metrics và distributed trace.

### 5.2 Thành phần ứng dụng

  Component       Chức năng             Tài nguyên lab
  --------------- --------------------- ----------------------------------
  Frontend        Web UI                Deployment + Service + Route
  API             REST API              Deployment + Service + Route
  Worker          Kafka consumer        Deployment
  PostgreSQL      Transaction store     Stateful workload + PVC
  Redis           Cache                 Stateful workload hoặc ephemeral
  Kafka           Event stream          Strimzi Kafka CR
  Migration job   DB schema migration   Job chạy trong deploy flow

Có thể dùng Java Spring Boot, .NET hoặc Node.js. Nếu mục tiêu là học vận
hành, chọn ngôn ngữ team quen thuộc; không để việc học framework làm
chậm project.

### 5.3 Endpoint bắt buộc

-   `/health/live` - liveness.
-   `/health/ready` - readiness; kiểm tra dependency cần thiết.
-   `/metrics` - Prometheus metrics.
-   `/api/transactions` - tạo/list giao dịch demo.
-   `/api/transactions/{id}` - xem trạng thái.
-   Trace ID / correlation ID được log xuyên suốt request.

Không đưa password, token, PAN, PII hoặc dữ liệu nhạy cảm vào log.

------------------------------------------------------------------------

## 6. GitLab CI - Continuous Integration

### 6.1 Pipeline mục tiêu

``` text
Push / Merge Request
        |
        v
  Validate & Lint
        |
        v
  Unit / Integration Tests
        |
        v
  SonarQube (optional)
        |
        v
  Dependency & Secret Scan
        |
        v
  Build Container Image
        |
        v
  Trivy Image Scan + SBOM
        |
        v
  Push Image to Registry
        |
        v
  Update image tag in GitOps repo
```

### 6.2 Runner placement

Ưu tiên theo thứ tự:

1.  GitLab Runner do công ty cung cấp.
2.  Runner trên VM riêng ngoài SNO.
3.  Runner trong OpenShift khi đã hiểu SCC, service account, RBAC và
    cách build image an toàn.

Không cấp cluster-admin cho CI runner. Runner chỉ cần quyền cần thiết
với registry và GitOps repository. Nếu runner chạy trong OpenShift, tuân
thủ restricted SCC; tránh privileged Docker-in-Docker nếu không có lý do
và phê duyệt.

### 6.3 Pipeline stages đề xuất

-   `validate`: YAML lint, Dockerfile lint, formatting.
-   `test`: unit test và integration test.
-   `security`: secret scan, dependency scan, Trivy filesystem/image.
-   `build`: build OCI image.
-   `publish`: push image tag bất biến.
-   `gitops-update`: tạo commit/MR cập nhật image tag trong GitOps repo.

### 6.4 Image tagging

Không chỉ dùng `latest`. Nên tag bằng: - commit SHA ngắn; - release
tag; - build number nếu cần.

Ví dụ: `registry.example.internal/sre-lab/api:sha-a1b2c3d`.

Trong môi trường nghiêm túc, ưu tiên deploy bằng digest sau khi image đã
publish và scan.

### 6.5 CI acceptance criteria

-   Merge Request chạy test và scan.
-   Pipeline fail khi unit test fail.
-   Image được push thành công.
-   Image tag truy vết được về commit.
-   Pipeline tạo GitOps MR hoặc commit có kiểm soát.
-   Không có cluster-admin token trong CI variables.
-   Secret variables được masked/protected và không echo ra log.

------------------------------------------------------------------------

## 7. Argo CD / OpenShift GitOps - Continuous Delivery

### 7.1 Cách triển khai

Ưu tiên cài **Red Hat OpenShift GitOps Operator** từ OperatorHub, thay
vì tự cài Argo CD community operator song song. Kiểm tra compatibility
giữa phiên bản Operator và OCP trước khi chọn channel.

Tài liệu: -
https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.21/html/installing_gitops/installing-openshift-gitops -
https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.21/html/understanding_openshift_gitops/about-red-hat-openshift-gitops

### 7.2 Bootstrap

1.  Cài OpenShift GitOps Operator.
2.  Xác nhận Argo CD instance mặc định.
3.  Tạo credential/repository connection tới GitOps repo bằng deploy key
    hoặc token read-only.
4.  Tạo root Application (App of Apps) hoặc ApplicationSet.
5.  Root app quản lý các app con: namespaces, demo app, observability,
    Kafka và security.
6.  Thiết lập project/RBAC và destination namespace có giới hạn.
7.  Bật automated sync/prune/self-heal cho workload phù hợp; không bật
    prune bừa bãi với resource stateful hoặc cluster-scoped.
8.  Thiết lập sync wave/health check cho dependency.

### 7.3 Luồng promotion

-   Dev: tự động deploy sau khi CI cập nhật GitOps.
-   Stage: promotion bằng Merge Request, review và approval.
-   Rollback: revert GitOps commit; Argo CD reconcile về desired state
    cũ.
-   Không sửa resource trực tiếp bằng `oc edit` như cách vận hành lâu
    dài. Nếu emergency change, ghi nhận và đưa thay đổi ngược vào Git để
    tránh drift.

### 7.4 Nghiệm thu

-   Argo CD hiển thị app `Synced` và `Healthy`.
-   Sửa manifest trên Git làm cluster thay đổi theo.
-   Sửa tay resource tạo `OutOfSync`; self-heal hoạt động đúng với app
    được bật.
-   Revert commit đưa app về version trước.
-   GitOps repo không chứa secret plaintext.

------------------------------------------------------------------------

## 8. Monitoring và Alerting

### 8.1 Kiến trúc

Dùng hai lớp:

1.  **OpenShift built-in monitoring:** giám sát control plane và
    platform components; bật user-workload monitoring để scrape app
    metrics.
2.  **User observability stack:** Grafana dashboards, Alertmanager
    integration và các exporter bổ sung. Nếu dùng Prometheus Operator
    riêng, tránh cài chồng operator/CRD không kiểm soát.

OpenShift có Prometheus/Alertmanager tích hợp cho monitoring nội bộ.
Không thay thế hoặc chỉnh sửa tùy tiện các component do cluster quản lý.

Tài liệu bắt đầu: -
https://docs.openshift.com/container-platform/latest/monitoring/monitoring-overview.html -
https://prometheus.io/docs/introduction/overview/ -
https://grafana.com/docs/

### 8.2 Metrics cần thu thập

**Cluster/platform** - Node CPU, memory, filesystem, network. -
Pod/container CPU và memory. - Pod restarts, OOMKilled, pending pods. -
PVC capacity/usage và mount errors. - API server latency/error rate. -
ClusterOperator degraded/progressing. - Deployment unavailable replicas.

**Application** - Request rate. - Error rate (4xx/5xx). - Latency
p50/p95/p99. - JVM/.NET runtime metrics nếu phù hợp. - DB connection
pool, query latency. - Kafka producer errors, consumer lag. - Cache
hit/miss.

**Kafka** - Broker availability. - Under-replicated partitions (nếu có
replication). - Offline partitions. - Consumer group lag. - Request
latency, throughput, disk usage. - JVM heap/GC.

### 8.3 Dashboard cần có

1.  OpenShift Cluster Overview.
2.  Node / Pod / Namespace Capacity.
3.  Application Golden Signals.
4.  PostgreSQL Health.
5.  Kafka Cluster & Consumer Lag.
6.  CI/CD Deployment Status.
7.  SLO / Error Budget.

### 8.4 Alert rules tối thiểu

-   API 5xx rate vượt ngưỡng trong 5 phút.
-   p95 latency vượt SLO trong 10 phút.
-   Pod restart tăng bất thường.
-   Deployment unavailable.
-   PVC usage \> 80% / 90%.
-   Node memory pressure / disk pressure.
-   Kafka consumer lag vượt ngưỡng.
-   Kafka broker hoặc partition không healthy.
-   ClusterOperator degraded.
-   Certificate sắp hết hạn.

Alert phải có severity, owner, runbook URL và mô tả hành động đầu tiên.
Tránh alert chỉ có nội dung "CPU high" mà không có ngữ cảnh.

------------------------------------------------------------------------

## 9. Logging và Distributed Tracing

### 9.1 Logging

Chọn một trong hai hướng, không cài đồng thời cả hai từ đầu:

-   **Loki + Grafana:** gọn hơn cho lab, dễ kết hợp Grafana.
-   **OpenSearch:** phù hợp nếu muốn học search/index lifecycle, shard,
    heap và cluster tuning.

Đề xuất khởi đầu: Loki + Grafana.

Log format nên là JSON, có: - timestamp; - severity; - service; -
environment; - trace_id; - request_id; - namespace/pod; - message.

Không log credential, authorization header, secret, thông tin định danh
cá nhân hoặc payload nhạy cảm.

### 9.2 Tracing

Dùng OpenTelemetry SDK/agent trong ứng dụng và OpenTelemetry Collector
để nhận, xử lý, export traces tới Tempo hoặc Jaeger.

Trace cần nối được: `Frontend -> API -> PostgreSQL/Kafka -> Worker`.

Bài tập: - Tạo một request chậm. - Xem trace để xác định span chậm. - So
sánh log theo trace ID. - Xác định lỗi do DB, Kafka hay ứng dụng.

------------------------------------------------------------------------

## 10. Kafka trên OpenShift bằng Strimzi

### 10.1 Vai trò

Kafka mô phỏng event streaming cho transaction events, audit events hoặc
notification events. Strimzi Operator quản lý Kafka resources thông qua
Kubernetes CRD.

Tài liệu: -
https://strimzi.io/docs/operators/latest/full/deploying.html -
https://strimzi.io/docs/operators/latest/full/overview.html

### 10.2 Triển khai theo cấp độ

**Level 1 - functional lab** - Cài Strimzi Operator tương thích OCP. -
Dựng Kafka cluster nhỏ, 1 broker, storage phù hợp. - Tạo topic
`transaction-events`. - Tạo user producer/consumer. - Deploy API
producer và worker consumer. - Test produce/consume, retry, consumer
lag.

**Level 2 - reliability exercise** - Tăng partition. - Chạy nhiều
consumer replica. - Quan sát rebalance và lag. - Test consumer
restart. - Test retention và message replay. - Test quota,
authentication và TLS.

**Level 3 - HA concepts** - Nghiên cứu cluster nhiều broker, replication
factor và failure domains. - Không tuyên bố HA khi chạy trên SNO: mọi
broker vẫn phụ thuộc cùng một node/failure domain. - Muốn kiểm chứng
broker/node HA thật sự, cần cluster nhiều node hoặc Kafka ngoài cluster.

### 10.3 Nghiệm thu

-   Producer publish được event.
-   Consumer xử lý và commit offset.
-   Consumer lag có metric và dashboard.
-   Restart consumer không mất dữ liệu ngoài semantics đã thiết kế.
-   Có runbook xử lý lag, broker unavailable và disk pressure.

------------------------------------------------------------------------

## 11. Security và Secrets Management

### 11.1 Vault

Vault là lựa chọn phù hợp để học secrets lifecycle và tích hợp workload
identity.

Bài tập theo thứ tự: 1. Cài Vault lab theo kiến trúc phù hợp tài nguyên.
2. Tạo KV v2 secret engine. 3. Tạo policy least privilege. 4. Cấu hình
Kubernetes auth. 5. Cho service account của ứng dụng authenticate. 6.
Đồng bộ secret qua Vault Secrets Operator (VSO) hoặc inject secret theo
mô hình đã chọn. 7. Thử rotate secret và xác minh ứng dụng nhận giá trị
mới. 8. Tích hợp database dynamic credentials ở phase nâng cao. 9. Thực
hành audit log, token TTL, revoke và recovery.

Không dùng Vault dev mode cho dữ liệu cần giữ. Không coi một Vault
instance trên cùng SNO là DR.

### 11.2 Image và source scanning

-   Trivy: filesystem, dependency, image scan.
-   SonarQube Community: code quality và coverage (nếu tài nguyên đủ).
-   Secret scanning: phát hiện credential bị commit nhầm.
-   SBOM: tạo CycloneDX/SPDX cho image.
-   Image signing/verification: Cosign ở phase nâng cao.

### 11.3 Kubernetes/OpenShift security

-   RBAC least privilege.
-   ServiceAccount riêng cho từng workload.
-   Không dùng default service account cho workload nhạy cảm.
-   Tuân thủ restricted SCC; không chạy privileged trừ khi có use case
    được duyệt.
-   NetworkPolicy: chỉ cho phép luồng cần thiết.
-   Resource requests/limits.
-   Pod security context và non-root.
-   TLS cho endpoint và service cần thiết.
-   Quản lý certificate lifecycle.

------------------------------------------------------------------------

## 12. Storage, Backup và Disaster Recovery

### 12.1 Storage

-   Kiểm tra vSphere CSI/StorageClass được hỗ trợ với OCP version và
    quyền vCenter.
-   Tạo PVC test trước khi deploy database/Kafka.
-   Xác minh access mode, expansion, reclaim policy và snapshot support.
-   Theo dõi capacity và latency.
-   Không mặc định local storage của một node có thể chịu được
    node/datastore failure.

### 12.2 Backup

Tách rõ: - **Application data backup:** PostgreSQL logical/physical
backup, Kafka retention/replication tùy use case. - **Kubernetes
resource backup:** OADP/Velero phù hợp version, plugin và storage
backend. - **Vault backup:** snapshot theo hướng dẫn Vault, lưu ở nơi
độc lập. - **Git:** Git repository là nguồn khôi phục desired state,
nhưng không thay thế backup persistent data.

### 12.3 Restore drill

Thực hiện: 1. Backup PostgreSQL. 2. Xóa một bảng hoặc tạo
namespace/database test bị lỗi. 3. Restore vào môi trường test. 4. Kiểm
tra row count và tính nhất quán. 5. Ghi lại RPO/RTO thực tế. 6. Lặp lại
với Vault snapshot nếu Vault được triển khai. 7. Kiểm tra khả năng dựng
lại application từ GitOps repo.

Không dùng vSphere VM snapshot như cách backup duy nhất cho OpenShift
control plane/etcd.

------------------------------------------------------------------------

## 13. Automation và Infrastructure as Code

### 13.1 Terraform

Dùng Terraform vSphere provider để: - Tạo VM. - Gắn network. - Cấu hình
CPU/RAM/disk. - Clone template nếu workflow cho phép. - Xuất output
IP/VM ID.

Không quản lý toàn bộ lifecycle OpenShift bằng Terraform nếu chưa hiểu
rõ resource ownership và installer. Ban đầu chỉ dùng Terraform cho hạ
tầng VM.

### 13.2 Ansible

Dùng Ansible cho: - Chuẩn hóa Linux utility VM. - Cài `oc`, `kubectl`,
`helm`, `ansible`, `podman`. - Kiểm tra DNS/NTP/port. - Cấu hình runner
host. - Tạo user/group và baseline hardening. - Chạy preflight checks.

### 13.3 Helm và Kustomize

-   Helm: cài package/Operator chart khi phù hợp.
-   Kustomize: quản lý overlays dev/stage.
-   Argo CD: reconcile manifests.
-   Không để Helm, Argo CD và người vận hành cùng tranh quyền sở hữu một
    resource.

------------------------------------------------------------------------

## 14. SLO, Incident Response và bài tập vận hành

### 14.1 SLO mẫu

  ------------------------------------------------------------------------
  SLI                               SLO lab mục tiêu Cách đo
  --------------------- ---------------------------- ---------------------
  API availability           99% trong cửa sổ 7 ngày Successful requests /
                                                     total requests

  API latency                  95% request \< 500 ms Histogram p95

  Error rate                                   \< 1% 5xx / total requests

  Event processing        95% event xử lý \< 10 giây Event timestamp vs
                                                     processed timestamp
  ------------------------------------------------------------------------

Đây là mục tiêu học tập, không phải cam kết production. Định nghĩa
measurement window và loại trừ maintenance rõ ràng.

### 14.2 Incident game days

  -----------------------------------------------------------------------
  ID                      Sự cố mô phỏng          Kỹ năng cần thực hành
  ----------------------- ----------------------- -----------------------
  INC-01                  API trả HTTP 500        Logs, traces,
                                                  deployment rollback

  INC-02                  API latency tăng        Golden signals, trace,
                                                  DB query

  INC-03                  PostgreSQL hết          Pool, metrics,
                          connection              connection leak

  INC-04                  Kafka consumer lag tăng Consumer group,
                                                  partition, throughput

  INC-05                  Pod OOMKilled           Requests/limits, heap,
                                                  memory dashboard

  INC-06                  PVC gần đầy             Storage metrics,
                                                  cleanup, expansion

  INC-07                  Secret hết hạn/sai      Vault auth, rotation,
                                                  audit

  INC-08                  GitOps drift            Argo CD diff,
                                                  self-heal, Git
                                                  correction

  INC-09                  CI pipeline fail        Runner, registry,
                                                  dependency, test

  INC-10                  Node SNO unavailable    Hiểu single-node
                                                  limitation, recovery
                                                  procedure

  INC-11                  Certificate hết hạn     TLS chain, renewal,
                                                  route/service

  INC-12                  Restore database        Backup verification,
                                                  RPO/RTO
  -----------------------------------------------------------------------

Với từng incident, viết postmortem ngắn: impact, timeline, detection,
root cause, resolution, prevention, action items.

------------------------------------------------------------------------

## 15. Lộ trình triển khai theo phase

Không triển khai mọi thành phần cùng lúc. Mỗi phase chỉ chuyển tiếp khi
acceptance criteria đạt.

### Phase 0 - Design và preflight

**Công việc** - Chốt OCP version và installer. - Xác nhận
entitlement/pull secret. - Chốt VM sizing, IP, DNS, NTP, network,
storage. - Chốt GitLab/runner/registry sử dụng ở đâu. - Tạo sơ đồ kiến
trúc và inventory.

**Deliverables** - `docs/architecture.md` - `docs/network-plan.md` -
`docs/vm-inventory.md` - `docs/decision-log.md`

**Exit criteria:** Đội hạ tầng xác nhận tài nguyên và network; không còn
blocker về DNS/IP/entitlement.

### Phase 1 - vSphere VM và SNO

**Công việc** - Provision VM. - Cài SNO bằng Assisted Installer. - Cấu
hình `oc` CLI. - Kiểm tra ClusterOperators, DNS, Routes, StorageClass. -
Ghi lại cluster access/recovery notes.

**Deliverables** - `infra/terraform/` (nếu được phép) -
`docs/openshift-install.md` - `docs/cluster-baseline.md`

**Exit criteria:** Cluster healthy và truy cập được từ
workstation/bastion.

### Phase 2 - Deploy ứng dụng thủ công

**Công việc** - Tạo namespace `demo-dev`. - Deploy
frontend/API/PostgreSQL. - Tạo Service, Route, PVC, probes,
requests/limits. - Kiểm tra application logs và health endpoints.

**Deliverables** - `sre-lab-app` -
`sre-lab-gitops/applications/demo-app/base`

**Exit criteria:** Truy cập web được; dữ liệu tồn tại qua pod restart;
resource limits/probes hoạt động.

### Phase 3 - GitLab CI

**Công việc** - Cấu hình runner. - Pipeline lint/test/scan/build/push. -
Thiết lập registry auth. - Tag image theo commit SHA. - Tạo pipeline
artifact/SBOM nếu có thể.

**Exit criteria:** Push code tạo image mới; pipeline fail đúng khi test
fail; không lộ secret.

### Phase 4 - Argo CD GitOps

**Công việc** - Cài OpenShift GitOps Operator. - Bootstrap root
Application. - Kết nối GitOps repo. - Chuyển app sang deploy hoàn toàn
qua GitOps. - Pipeline cập nhật manifest repo; Argo CD sync.

**Exit criteria:** CI không cần quyền deploy trực tiếp vào cluster;
commit GitOps điều khiển deployment; rollback bằng Git revert.

### Phase 5 - Monitoring và Alerting

**Công việc** - Bật user-workload monitoring. - Expose application
metrics. - Tạo ServiceMonitor/PodMonitor khi phù hợp. - Tạo dashboard và
alert rules. - Kiểm thử notification route.

**Exit criteria:** Nhìn thấy golden signals và nhận được alert test.

### Phase 6 - Logging và Tracing

**Công việc** - Chuẩn hóa JSON logs. - Deploy Loki. - Instrument app
bằng OpenTelemetry. - Deploy Collector và Tempo/Jaeger. - Liên kết logs
với trace ID.

**Exit criteria:** Một request có thể truy vết từ metrics → trace →
logs.

### Phase 7 - Kafka / Event-driven application

**Công việc** - Cài Strimzi. - Tạo Kafka cluster phù hợp lab. - Tạo
topic/user. - Tích hợp producer/consumer. - Dashboard consumer lag.

**Exit criteria:** Event được publish/consume và có metric/alert.

### Phase 8 - Security và Vault

**Công việc** - Trivy/SonarQube/secret scanning. - Vault KV, Kubernetes
auth, policy. - Tích hợp VSO hoặc injection. - NetworkPolicy/RBAC/SCC
review. - Secret rotation exercise.

**Exit criteria:** App lấy secret runtime; không có secret plaintext
trong Git; có audit/rotation runbook.

### Phase 9 - Backup, DR và reliability

**Công việc** - Backup database, Vault và persistent data. - OADP/Velero
nếu phù hợp. - Restore drill. - SLO dashboard. - Thực hành incident game
days.

**Exit criteria:** Restore được dữ liệu và app; có RPO/RTO đo được; có
postmortem cho incident.

### Phase 10 - Nâng cao

Chọn theo mục tiêu: - Harbor registry, Cosign, SBOM attestation. -
OpenSearch thay Loki. - Kyverno hoặc Gatekeeper policy. - External
Secrets Operator. - cert-manager và PKI. - KEDA autoscaling theo Kafka
lag. - Chaos Mesh/Litmus (chỉ khi hiểu rõ rủi ro trên SNO). -
Terraform/Ansible end-to-end. - OpenShift multi-node cluster thứ hai để
học HA và multi-cluster GitOps. - Service mesh nếu có use case rõ ràng.

------------------------------------------------------------------------

## 16. Thứ tự ưu tiên công nghệ

Không cần cài tất cả. Bảng này phân biệt phần bắt buộc và phần nên để
sau.

  -----------------------------------------------------------------------
  Ưu tiên                 Công nghệ               Lý do
  ----------------------- ----------------------- -----------------------
  P0                      OpenShift SNO, `oc`,    Nền tảng lab
                          Linux, DNS, storage     

  P0                      GitLab CI, Runner,      Học build/test/package
                          Container Registry      

  P0                      OpenShift GitOps / Argo Học CD và desired state
                          CD                      

  P0                      Prometheus, Grafana,    Monitoring và alerting
                          Alertmanager            

  P1                      Loki                    Centralized logging

  P1                      OpenTelemetry +         Distributed tracing
                          Tempo/Jaeger            

  P1                      Strimzi + Kafka         Event-driven
                                                  architecture và
                                                  consumer lag

  P1                      Trivy + SonarQube       Shift-left security và
                                                  code quality

  P1                      Vault + VSO             Secrets lifecycle

  P1                      PostgreSQL + Redis      Stateful app và
                                                  dependency
                                                  troubleshooting

  P2                      Ansible + Terraform     Infrastructure
                                                  automation

  P2                      OADP/Velero             Backup/restore
                                                  Kubernetes resources

  P2                      Harbor + Cosign         Supply-chain security

  P2                      KEDA                    Event-driven
                                                  autoscaling

  P2                      Kyverno/Gatekeeper      Policy as code

  P3                      OpenSearch              Alternative
                                                  logging/search stack

  P3                      Service Mesh            Học khi có nhu cầu
                                                  traffic management/mTLS

  P3                      Chaos engineering       Chỉ nên làm sau khi có
                                                  baseline, monitoring và
                                                  backup
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 17. Resource budget và giới hạn SNO

### 17.1 Nguyên tắc phân bổ

-   Dành đủ tài nguyên cho OpenShift system components trước.
-   Bật user workload từng bước, theo dõi memory pressure.
-   Tránh cùng lúc chạy GitLab full stack, SonarQube, OpenSearch, Kafka
    nhiều broker và tracing backend trên một SNO.
-   Đặt requests/limits thực tế; không đặt limit quá thấp khiến operator
    và database OOM.
-   Dùng retention ngắn cho metrics/logs/traces trong lab.
-   Dọn image, PVC, log và old build artifacts định kỳ.

### 17.2 Profile đề xuất

**Profile A - constrained** - SNO: 8 vCPU / 32 GB RAM / 250 GB. - GitLab
external/shared. - Runner external. - Built-in monitoring. - Demo app +
PostgreSQL. - Kafka 1 broker nhỏ. - Chỉ thêm một observability backend ở
mỗi lần thử.

**Profile B - recommended** - SNO: 12-16 vCPU / 48-64 GB RAM / 300-500
GB. - GitLab external/shared. - Runner external. - Demo app +
PostgreSQL + Redis. - Argo CD. - Loki + Grafana + OTel/Tempo theo từng
phase. - Kafka 1 broker cho functional testing. - Vault lab nhỏ.

**Profile C - expanded** - SNO: 16+ vCPU / 64+ GB RAM / 500 GB+. - Một
số backend vẫn nên chuyển ra VM riêng. - Không biến Profile C thành
tuyên bố HA; vẫn chỉ là một node.

------------------------------------------------------------------------

## 18. Repository layout cho toàn bộ project

Có thể dùng một repository quản lý tài liệu và automation:

``` text
enterprise-devops-sre-lab/
├── README.md
├── docs/
│   ├── architecture.md
│   ├── network-plan.md
│   ├── vm-inventory.md
│   ├── decisions.md
│   ├── runbooks/
│   ├── incidents/
│   └── diagrams/
├── infra/
│   ├── terraform/
│   │   └── vsphere/
│   └── ansible/
│       ├── inventory/
│       ├── playbooks/
│       └── roles/
├── app/
│   └── README.md
├── gitops/
│   ├── bootstrap/
│   ├── platform/
│   └── applications/
├── observability/
│   ├── dashboards/
│   ├── alerts/
│   └── otel/
├── security/
│   ├── policies/
│   └── scanning/
└── scripts/
    ├── preflight/
    └── validation/
```

Không bắt buộc gom tất cả source code vào một repo. Có thể giữ `app` và
`gitops` ở hai repository độc lập như phần trên; repository này chỉ đóng
vai trò project documentation/automation.

------------------------------------------------------------------------

## 19. Hướng dẫn cho coding agent

Agent phải làm việc theo các quy tắc sau:

1.  Đọc file này và các tài liệu trong `docs/` trước khi đề xuất thay
    đổi.
2.  Chỉ triển khai **một phase tại một thời điểm**.
3.  Trước mỗi phase, tạo checklist prerequisites và liệt kê thông tin
    còn thiếu.
4.  Không tự bịa IP, hostname, domain, credentials, pull secret,
    registry endpoint, vCenter details hoặc storage class.
5.  Dùng biến/placeholder rõ ràng cho mọi thông tin môi trường.
6.  Ưu tiên tài liệu chính thức của Red Hat, GitLab, Argo CD,
    Prometheus, Grafana, Strimzi, HashiCorp và dự án upstream.
7.  Kiểm tra compatibility matrix giữa OCP version, Operator channel,
    CRD, Helm chart và Kubernetes API trước khi cài.
8.  Không dùng manifest/API version lỗi thời nếu có API được hỗ trợ mới
    hơn.
9.  Không tạo resource cluster-scoped hoặc cấp cluster-admin nếu chưa
    giải thích lý do.
10. Không lưu secret plaintext trong Git, Markdown, shell history hoặc
    CI logs.
11. Với lệnh có tác động phá hủy (delete, prune, wipe, restore, rotate,
    upgrade), phải giải thích impact và yêu cầu xác nhận trước khi chạy.
12. Mọi manifest phải có namespace rõ ràng khi phù hợp, labels, resource
    requests/limits, probes và security context.
13. Mọi deployment qua GitOps phải có nguồn manifest trong Git; tránh
    thay đổi live cluster không được ghi lại.
14. Sau mỗi bước, cung cấp lệnh verify và expected result.
15. Nếu một bước fail, dừng và troubleshoot dựa trên output thực tế;
    không tiếp tục chồng thêm thay đổi.
16. Cập nhật `docs/` và `README.md` sau khi hoàn tất phase.
17. Tạo runbook cho từng thành phần có stateful data.
18. Không áp dụng cấu hình production vào lab nếu chưa hiểu security
    implications.

### 19.1 Quy trình làm việc chuẩn của agent

Với mỗi task:

``` text
1. Inspect current state
2. Read relevant official documentation
3. Check version compatibility
4. Identify prerequisites and unknown values
5. Propose implementation plan
6. Generate/update files
7. Validate YAML / Terraform / Ansible syntax
8. Explain commands and expected output
9. Ask approval before destructive or external changes
10. Execute only when authorized
11. Verify actual state
12. Update docs and runbook
```

### 19.2 Prompt khởi động cho agent

``` text
Bạn là Senior Platform Engineer/DevOps/SRE chịu trách nhiệm hỗ trợ triển khai
project Enterprise DevOps/SRE Lab trên VMware vSphere và OpenShift SNO.

Hãy đọc toàn bộ file project plan này trước khi làm việc.

Yêu cầu:
- Không triển khai tất cả stack cùng lúc.
- Bắt đầu ở Phase 0: Design và preflight.
- Đầu tiên hãy tạo danh sách thông tin cần tôi xác nhận: vSphere permissions,
  VM resource quota, VLAN/port group, static IP, DNS, NTP, proxy/firewall,
  OCP version, pull secret/entitlement, storage và GitLab/Runner/Registry.
- Không tự giả định các thông tin hạ tầng.
- Tạo checklist preflight, network plan template và VM inventory template.
- Dùng tài liệu chính thức và kiểm tra compatibility với phiên bản thực tế.
- Mỗi lệnh phải giải thích mục đích, nơi chạy và expected output.
- Không chạy lệnh destructive hoặc thay đổi hệ thống công ty khi chưa có xác nhận.
- Sau mỗi phase phải có acceptance criteria và cập nhật tài liệu.
- Ưu tiên OpenShift-native và GitOps; CI không deploy trực tiếp vào cluster.
- Bảo vệ credentials và dữ liệu nội bộ.

Chỉ thực hiện Phase 0 trước, chưa cài đặt hoặc thay đổi hạ tầng.
```

------------------------------------------------------------------------

## 20. Checklist tổng kết

### Infrastructure

-   [ ] vSphere VM được provision và inventory đầy đủ.
-   [ ] DNS, NTP, gateway, VLAN và firewall được xác nhận.
-   [ ] SNO cài đặt thành công.
-   [ ] StorageClass và PVC hoạt động.

### CI/CD

-   [ ] GitLab project và runner hoạt động.
-   [ ] Pipeline test/build/scan/push hoạt động.
-   [ ] Argo CD sync từ GitOps repo.
-   [ ] Rollback qua Git revert thành công.

### Observability

-   [ ] Prometheus scrape application metrics.
-   [ ] Grafana dashboard có golden signals.
-   [ ] Alertmanager gửi được test alert.
-   [ ] Logs tập trung và tra cứu được.
-   [ ] Trace liên kết được với logs.

### Messaging

-   [ ] Strimzi quản lý Kafka.
-   [ ] Producer/consumer chạy.
-   [ ] Consumer lag có dashboard và alert.
-   [ ] Có runbook Kafka.

### Security

-   [ ] RBAC least privilege.
-   [ ] Trivy scan pipeline.
-   [ ] Không có secret plaintext trong Git.
-   [ ] Vault auth/secret rotation hoạt động.
-   [ ] TLS/certificate lifecycle được kiểm tra.

### Reliability

-   [ ] PostgreSQL backup và restore thành công.
-   [ ] Vault backup và restore đã được kiểm thử nếu có triển khai.
-   [ ] Có SLI/SLO.
-   [ ] Có ít nhất 5 incident game days.
-   [ ] Có postmortem và action items.

------------------------------------------------------------------------

## 21. Tài liệu tham khảo chính thức

### Red Hat / OpenShift

-   Assisted Installer 2026:
    https://docs.redhat.com/en/documentation/assisted_installer_for_openshift_container_platform/2026/html/installing_openshift_container_platform_with_the_assisted_installer/
-   SNO installation:
    https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/installing_on_a_single_node/installing_on_a_single_node
-   OpenShift GitOps 1.21:
    https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.21/html/understanding_openshift_gitops/about-red-hat-openshift-gitops
-   OpenShift monitoring:
    https://docs.openshift.com/container-platform/latest/monitoring/monitoring-overview.html

### CI/CD và GitOps

-   GitLab CI/CD: https://docs.gitlab.com/ci/
-   GitLab Runner: https://docs.gitlab.com/runner/
-   Argo CD: https://argo-cd.readthedocs.io/

### Observability

-   Prometheus: https://prometheus.io/docs/
-   Grafana: https://grafana.com/docs/
-   Loki: https://grafana.com/docs/loki/latest/
-   OpenTelemetry: https://opentelemetry.io/docs/
-   Tempo: https://grafana.com/docs/tempo/latest/

### Kafka

-   Strimzi deployment:
    https://strimzi.io/docs/operators/latest/full/deploying.html
-   Strimzi overview:
    https://strimzi.io/docs/operators/latest/full/overview.html
-   Apache Kafka: https://kafka.apache.org/documentation/

### Security / Automation / Backup

-   HashiCorp Vault: https://developer.hashicorp.com/vault/docs
-   Trivy: https://trivy.dev/latest/
-   Terraform vSphere provider:
    https://registry.terraform.io/providers/vmware/vsphere/latest/docs
-   Ansible: https://docs.ansible.com/
-   Velero: https://velero.io/docs/
-   OADP:
    https://docs.openshift.com/container-platform/latest/backup_and_restore/application_backup_and_restore/oadp-intro.html

------------------------------------------------------------------------

## 22. Kết luận và khuyến nghị triển khai

Thứ tự nên làm thực tế:

1.  Dựng SNO ổn định trên vSphere.
2.  Deploy app web + PostgreSQL.
3.  GitLab CI build và push image.
4.  Argo CD triển khai từ GitOps repository.
5.  Prometheus/Grafana/Alertmanager.
6.  Logging và tracing.
7.  Kafka/Strimzi.
8.  Vault và security scanning.
9.  Backup/restore và incident exercises.
10. Mở rộng Terraform/Ansible, policy-as-code và chaos engineering.

**Không nên bắt đầu bằng việc cài Kafka, Vault, Loki, OpenSearch,
SonarQube và Harbor cùng lúc.** Làm như vậy dễ biến project thành bài
toán thiếu RAM/disk và troubleshooting operator thay vì học đúng quy
trình DevOps/SRE. Một luồng end-to-end chạy tốt, có monitoring và có thể
rollback/restore giá trị hơn nhiều công cụ được cài nhưng chưa vận hành
được.
