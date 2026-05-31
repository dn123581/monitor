# Monitor Stack

Docker Compose stack giám sát toàn diện cho một Docker host, gồm 3 service:
uptime monitoring, log viewer, và system metrics.

## Service

| Service | Image | Host Port | Mô tả |
|---|---|---|---|
| **uptime-kuma** | `louislam/uptime-kuma:1` | `9001` | Giám sát uptime website/service, cảnh báo qua Telegram/Email/Discord |
| **dozzle** | `amir20/dozzle:v8` | `9080` | Real-time log viewer cho tất cả Docker container |
| **netdata** | `netdata/netdata:stable` | `19999` (host mode) | Giám sát hệ thống: CPU, RAM, disk, network, process |

## Yêu cầu

- Docker Engine 20.10+
- Docker Compose v2

## Cài đặt & chạy

```bash
# Clone repo
git clone https://github.com/dn123581/monitor.git
cd monitor

# Khởi động toàn bộ stack
docker compose up -d

# Kiểm tra trạng thái
docker compose ps
```

## Truy cập

| Service | URL |
|---|---|
| uptime-kuma | http://localhost:9001 |
| dozzle | http://localhost:9080 |
| netdata | http://localhost:19999 |

## Cấu trúc file

```
monitor/
├── docker-compose.yml    # Định nghĩa stack
├── README.md             # File này
└── .gitignore
```

## Network

- **monitor-net** (bridge): uptime-kuma và dozzle dùng chung network riêng, tách biệt với default bridge.
- **host**: netdata dùng `network_mode: host` để truy cập trực tiếp host network interfaces và thu thập metrics chính xác.

## Persistent data

| Volume | Service | Mục đích |
|---|---|---|
| `uptime-kuma-data` | uptime-kuma | Lưu cấu hình monitor, lịch sử uptime |
| `netdata-config` | netdata | Cấu hình netdata |
| `netdata-lib` | netdata | Registry, claim token |
| `netdata-cache` | netdata | Metric cache |

## Healthcheck

Cả 3 service đều có healthcheck:

- **uptime-kuma**: `curl http://localhost:3001` (mỗi 30s)
- **dozzle**: `/dozzle healthcheck` (mỗi 30s)
- **netdata**: `curl http://localhost:19999/api/v1/info` (mỗi 30s)

Kiểm tra: `docker compose ps` — cột STATUS sẽ hiển thị `(healthy)`.

## Port mapping rationale

Các port host được chọn để tránh xung đột với các dịch vụ phổ biến:

- `9001` thay vì `3001` (Grafana dùng 3000)
- `9080` thay vì `8080` (Jenkins, nhiều web app dùng 8080)
- `19999` giữ nguyên (Netdata default, ít bị trùng)

## Bảo mật

- Docker socket được mount **read-only** (`:ro`) trên tất cả service.
- uptime-kuma và dozzle chạy trong bridge network riêng, không dùng host network.
- netdata cần elevated capability (`SYS_PTRACE`, `SYS_ADMIN`) và `pid: host` để thu thập system metrics. Nếu chạy trên môi trường production, cân nhắc giới hạn truy cập vào netdata dashboard qua reverse proxy + auth.

## Cập nhật

```bash
# Pull image mới nhất trong cùng major version
docker compose pull

# Restart container với image mới
docker compose up -d
```

Netdata dùng tag `stable`, uptime-kuma dùng tag `1`, dozzle dùng tag `v8` — tất cả đều là pin theo major version, đảm bảo không có breaking change khi pull update.
