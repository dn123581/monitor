# Monitor Stack + WireGuard VPN

Docker Compose stack giám sát toàn diện cho một Docker host, gồm 4 service:
WireGuard VPN, uptime monitoring, log viewer, và system metrics.

## Service

| Service | Image | Host Port | Mô tả |
|---|---|---|---|
| **wg-easy** | `ghcr.io/wg-easy/wg-easy:15` | `51820/udp`, `51821/tcp` | WireGuard VPN server + web admin |
| **uptime-kuma** | `louislam/uptime-kuma:1` | `9001` | Giám sát uptime website/service, cảnh báo qua Telegram/Email/Discord |
| **dozzle** | `amir20/dozzle:v8` | `9080` | Real-time log viewer cho tất cả Docker container |
| **netdata** | `netdata/netdata:stable` | `19999` (host mode) | Giám sát hệ thống: CPU, RAM, disk, network, process |

## Yêu cầu

- Docker Engine 20.10+
- Docker Compose v2
- Kernel module `wireguard` (thường có sẵn trên kernel 5.6+)

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

### Qua localhost (trên VPS)

| Service | URL |
|---|---|
| wg-easy admin | `http://localhost:51821` |
| uptime-kuma | `http://localhost:9001` |
| dozzle | `http://localhost:9080` |
| netdata | `http://localhost:19999` |

### Qua WireGuard VPN (từ máy remote)

Kết nối VPN vào VPS, sau đó truy cập qua gateway IP của bridge:

| Service | URL |
|---|---|
| uptime-kuma | `http://10.42.42.1:9001` |
| dozzle | `http://10.42.42.1:9080` |
| netdata | `http://10.42.42.1:19999` |

> **Thêm client VPN**: vào `http://<vps-ip>:51821`, tạo client mới và quét QR hoặc tải file `.conf`.

## Cấu trúc file

```
monitor/
├── docker-compose.yml            # Toàn bộ stack (VPN + monitor)
├── README.md
├── .github/workflows/deploy.yml  # CI/CD auto deploy
└── .gitignore
```

## Network

- **wireguard_wg** (bridge, `10.42.42.0/24`): wg-easy, uptime-kuma, và dozzle dùng chung network này. Các service monitor có thể được truy cập từ VPN client qua gateway `10.42.42.1`.
- **host**: netdata dùng `network_mode: host` để truy cập trực tiếp host network interfaces và thu thập metrics chính xác. Cũng truy cập được từ VPN qua `10.42.42.1:19999`.

## Persistent data

| Volume | Service | Mục đích |
|---|---|---|
| `etc_wireguard` | wg-easy | Cấu hình WireGuard, client peers |
| `uptime-kuma-data` | uptime-kuma | Lưu cấu hình monitor, lịch sử uptime |
| `netdata-config` | netdata | Cấu hình netdata |
| `netdata-lib` | netdata | Registry, claim token |
| `netdata-cache` | netdata | Metric cache |

## Healthcheck

Cả 4 service đều có healthcheck:

- **wg-easy**: web UI port `51821` luôn sẵn sàng
- **uptime-kuma**: `curl http://localhost:3001` (mỗi 30s)
- **dozzle**: `/dozzle healthcheck` (mỗi 30s)
- **netdata**: `curl http://localhost:19999/api/v1/info` (mỗi 30s)

Kiểm tra: `docker compose ps` — cột STATUS sẽ hiển thị `(healthy)`.

## Port mapping rationale

Các port host được chọn để tránh xung đột với các dịch vụ phổ biến:

- `51820/udp` — WireGuard default, giữ nguyên
- `51821/tcp` — wg-easy web admin
- `9001` thay vì `3001` (Grafana dùng 3000)
- `9080` thay vì `8080` (Jenkins, nhiều web app dùng 8080)
- `19999` giữ nguyên (Netdata default, ít bị trùng)

## Bảo mật

- Docker socket được mount **read-only** (`:ro`) trên tất cả service.
- uptime-kuma và dozzle chạy trong bridge network riêng `wireguard_wg`, không expose port ra ngoài internet. Chỉ truy cập được từ localhost hoặc qua VPN.
- **WireGuard**: chỉ client có key hợp lệ mới kết nối được. Port `51820/udp` cần mở trên firewall.
- **wg-easy web admin** (`51821`): cân nhắc không expose port này ra internet. Nếu cần, dùng firewall giới hạn IP hoặc reverse proxy + auth.
- netdata cần elevated capability (`SYS_PTRACE`, `SYS_ADMIN`) và `pid: host` để thu thập system metrics. Nếu chạy trên môi trường production, cân nhắc giới hạn truy cập vào netdata dashboard qua reverse proxy + auth.

## Cập nhật

```bash
# Pull image mới nhất trong cùng major version
docker compose pull

# Restart container với image mới
docker compose up -d
```

Netdata dùng tag `stable`, uptime-kuma dùng tag `1`, dozzle dùng tag `v8`, wg-easy dùng tag `15` — tất cả đều là pin theo major version, đảm bảo không có breaking change khi pull update.
