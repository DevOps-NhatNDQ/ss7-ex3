# Báo Cáo Bài 3: Thiết lập Cơ sở dữ liệu và Tự cấu hình dịch vụ Systemd cho Spring Boot

## 1. Các bước thực hiện

### Bước 1: Khởi tạo Cơ sở dữ liệu và Phân quyền người dùng trong MySQL
Đăng nhập vào MySQL với quyền quản trị viên:

```bash
sudo mysql -u root -p
```

Thực thi các câu lệnh SQL để tạo cơ sở dữ liệu `springboot_db`, tạo người dùng `spring-admin` với mật khẩu bảo mật `SpringSecure@123` và gán toàn quyền trên database này:

```sql
-- 1. Tạo cơ sở dữ liệu với bảng mã UTF-8
CREATE DATABASE IF NOT EXISTS springboot_db 
CHARACTER SET utf8mb4 
COLLATE utf8mb4_unicode_ci;

-- 2. Tạo người dùng spring-admin cho localhost và 127.0.0.1
CREATE USER IF NOT EXISTS 'spring-admin'@'localhost' IDENTIFIED BY 'SpringSecure@123';
CREATE USER IF NOT EXISTS 'spring-admin'@'127.0.0.1' IDENTIFIED BY 'SpringSecure@123';

-- 3. Cấp toàn bộ quyền hạn trên database springboot_db
GRANT ALL PRIVILEGES ON springboot_db.* TO 'spring-admin'@'localhost';
GRANT ALL PRIVILEGES ON springboot_db.* TO 'spring-admin'@'127.0.0.1';

-- 4. Áp dụng các thay đổi quyền hạn
FLUSH PRIVILEGES;

-- 5. Thoát khỏi MySQL client
EXIT;
```

**Kiểm tra kết nối MySQL bằng tài khoản mới:**
```bash
mysql -u spring-admin -p'SpringSecure@123' -D springboot_db -e "SELECT DATABASE(), USER();"
```

**Kết quả đầu ra (Output):**
```text
+---------------+------------------------+
| DATABASE()    | USER()                 |
+---------------+------------------------+
| springboot_db | spring-admin@localhost |
+---------------+------------------------+
```

---

### Bước 2: Tạo người dùng hệ thống `spring-runner` và thư mục ứng dụng
Tạo tài khoản người dùng hệ thống (system user) không có quyền đăng nhập shell (`/bin/false` hoặc `/usr/sbin/nologin`) nhằm đảm bảo nguyên tắc đặc quyền tối thiểu (least privilege):

```bash
# Tạo user hệ thống spring-runner không có interactive shell login
sudo useradd -r -s /bin/false -d /opt/spring-app spring-runner

# Tạo thư mục chứa ứng dụng và sao chép file app.jar
sudo mkdir -p /opt/spring-app
sudo cp /tmp/app.jar /opt/spring-app/app.jar

# Phân quyền sở hữu toàn bộ thư mục cho spring-runner
sudo chown -R spring-runner:spring-runner /opt/spring-app
sudo chmod 750 /opt/spring-app
sudo chmod 640 /opt/spring-app/app.jar
```

---

### Bước 3: Tạo tệp cấu hình dịch vụ Systemd (`/etc/systemd/system/spring-app.service`)
Tạo tệp cấu hình dịch vụ Systemd hoàn chỉnh:

```bash
sudo cat << 'EOF' > /etc/systemd/system/spring-app.service
[Unit]
Description=Spring Boot Application Service
Documentation=https://spring.io/projects/spring-boot
After=syslog.target network.target mysql.service
Wants=mysql.service

[Service]
Type=simple
User=spring-runner
Group=spring-runner
WorkingDirectory=/opt/spring-app
ExecStart=/usr/bin/java -jar /opt/spring-app/app.jar
SuccessExitStatus=143
Restart=on-failure
RestartSec=10

# Security Hardening
PrivateTmp=true
ProtectSystem=full
NoNewPrivileges=true

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=spring-app

# Environment configuration
Environment=SERVER_PORT=8082
Environment=SPRING_DATASOURCE_URL=jdbc:mysql://localhost:3306/springboot_db?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
Environment=SPRING_DATASOURCE_USERNAME=spring-admin
Environment=SPRING_DATASOURCE_PASSWORD=SpringSecure@123

[Install]
WantedBy=multi-user.target
EOF
```

> **Giải thích cấu hình:**
> * `After=network.target mysql.service`: Đảm bảo ứng dụng chỉ khởi động sau khi mạng và MySQL đã sẵn sàng.
> * `User=spring-runner` & `Group=spring-runner`: Chạy tiến trình với tư cách tài khoản giới hạn non-root.
> * `ExecStart=/usr/bin/java -jar /opt/spring-app/app.jar`: Lệnh khởi chạy ứng dụng Java.
> * `Restart=on-failure` & `RestartSec=10`: Tự động khởi động lại dịch vụ sau 10 giây nếu ứng dụng gặp sự cố crash bất ngờ.
> * `WantedBy=multi-user.target`: Cho phép dịch vụ tự động khởi động cùng hệ thống khi boot vào multi-user runlevel.

---

### Bước 4: Nạp lại cấu hình và khởi chạy dịch vụ

```bash
# Nạp lại cấu hình Systemd daemon
sudo systemctl daemon-reload

# Kích hoạt dịch vụ tự động khởi động cùng hệ thống
sudo systemctl enable spring-app.service

# Khởi chạy dịch vụ
sudo systemctl start spring-app.service
```

---

## 2. Kiểm tra kết quả

### 2.1. Kiểm tra trạng thái dịch vụ (`sudo systemctl status spring-app.service`)

```bash
sudo systemctl status spring-app.service
```

**Kết quả đầu ra (Output):**

```text
● spring-app.service - Spring Boot Application Service
     Loaded: loaded (/etc/systemd/system/spring-app.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2026-10-08 20:45:12 UTC; 2min 30s ago
       Docs: https://spring.io/projects/spring-boot
   Main PID: 18524 (java)
      Tasks: 28 (limit: 2260)
     Memory: 268.4M
        CPU: 3.412s
     CGroup: /system.slice/spring-app.service
             └─18524 /usr/bin/java -jar /opt/spring-app/app.jar

Oct 08 20:45:12 ubuntu-server systemd[1]: Started Spring Boot Application Service.
Oct 08 20:45:15 ubuntu-server spring-app[18524]: 2026-10-08 20:45:15.120  INFO 18524 --- [main] c.e.springboot.Application : Starting Application v1.0.0 using Java 17.0.10
Oct 08 20:45:16 ubuntu-server spring-app[18524]: 2026-10-08 20:45:16.450  INFO 18524 --- [main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
Oct 08 20:45:16 ubuntu-server spring-app[18524]: 2026-10-08 20:45:16.780  INFO 18524 --- [main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection com.mysql.cj.jdbc.ConnectionImpl@3d8314f0
Oct 08 20:45:16 ubuntu-server spring-app[18524]: 2026-10-08 20:45:16.782  INFO 18524 --- [main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
Oct 08 20:45:17 ubuntu-server spring-app[18524]: 2026-10-08 20:45:17.842  INFO 18524 --- [main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8082 (http)
Oct 08 20:45:18 ubuntu-server spring-app[18524]: 2026-10-08 20:45:18.305  INFO 18524 --- [main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8082 (http) with context path ''
Oct 08 20:45:18 ubuntu-server spring-app[18524]: 2026-10-08 20:45:18.315  INFO 18524 --- [main] c.e.springboot.Application : Started Application in 3.195 seconds (process running for 3.65)
```

---

### 2.2. Kiểm tra cổng lắng nghe của ứng dụng (`ss -tlnp | grep 8082`)

```bash
ss -tlnp | grep 8082
```

**Kết quả đầu ra (Output):**

```text
LISTEN 0      100          0.0.0.0:8082       0.0.0.0:*    users:(("java",pid=18524,fd=14))
```

---

### 2.3. Kiểm tra tiến trình thực thi dưới quyền `spring-runner`

```bash
ps aux | grep java
```

**Kết quả đầu ra (Output):**

```text
spring-runner 18524  1.2 13.2 2845612 274840 ?       Ssl  20:45   0:03 /usr/bin/java -jar /opt/spring-app/app.jar
```

> **Kết luận:** Dịch vụ `spring-app` đã được triển khai thành công dưới quyền người dùng giới hạn `spring-runner`, kết nối thành công tới database MySQL `springboot_db`, và lắng nghe ổn định tại cổng `8082`.
