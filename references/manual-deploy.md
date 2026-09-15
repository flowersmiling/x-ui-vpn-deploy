# 部署执行蓝图

Claude 按此文件的步骤逐条在服务器上执行命令。

## 步骤 0：定义变量

在服务器上执行任何操作前，先定义以下变量。用户提供的值在第一步收集阶段已获取。

```bash
# === 用户提供的值（替换为实际值）===
ROOT_DOMAIN="example.com"
SUBDOMAIN_PREFIX="node1"
EMAIL="you@email.com"
SSH_PORT="22"

# Cloudflare 认证（二选一）
# 方式一：Global API Key
CF_Key="your_global_api_key"
CF_Email="your_cf_email"
# 方式二：API Token
# CF_Token="your_api_token"

# === 自动拼接 ===
DOMAIN="${SUBDOMAIN_PREFIX}.${ROOT_DOMAIN}"   # SUBDOMAIN_PREFIX 为空时直接 DOMAIN="$ROOT_DOMAIN"
XUI_PORT="54321"
CERT_DIR="/root/cert"                         # RHEL 系改成 /etc/nginx/cert（见步骤 1 对照表）
```

后续所有命令直接使用这些变量，无需额外替换。

> **变量持久化（非交互 SSH 环境必做）**：如果不是在一个长期保持的交互式 SSH 会话里操作，而是每步单独 `ssh host "bash -s" < script.sh` / `plink -batch ... "bash -s" < script.sh` 这样喂脚本，shell 变量在两次调用之间不会保留。把上面这段变量定义整体写进 `/root/.secrets/deploy-vars.sh`（`chmod 600`），后续每个脚本开头 `source /root/.secrets/deploy-vars.sh` 再执行，就等价于"同一个会话"。实战中 Windows 本机用 PuTTY 的 `plink -batch -ssh -P 22 -pw '密码' root@IP "bash -s" < step.sh` 这种方式逐步骤喂脚本，配合这个变量文件跑通了全流程。

> **域名核实（先做这步再定变量，实战踩过坑）**：如果 `ROOT_DOMAIN` 疑似免费二级域名分发服务（形如 `cc.cd`、`co.cc` 这类），先核实它本身是否在 Public Suffix List 里：
> ```bash
> curl -s https://publicsuffix.org/list/public_suffix_list.dat | grep -x "$ROOT_DOMAIN"
> ```
> 有输出就说明命中了，直接拿它申请证书会在步骤 4 报 `DNS identifier is invalid`。这种情况下，去查用户实际托管在 Cloudflare 的完整域名（一般就是 `${SUBDOMAIN_PREFIX}.${ROOT_DOMAIN}` 这个完整子域名本身）：
> ```bash
> curl -s "https://dns.google/resolve?name=${SUBDOMAIN_PREFIX}.${ROOT_DOMAIN}&type=NS"
> ```
> 如果返回的 NS 是 `*.ns.cloudflare.com`，说明这个完整子域名才是真正的 Cloudflare 管理区域。把 `ROOT_DOMAIN` 直接改成这个完整子域名，`SUBDOMAIN_PREFIX` 置空（`DOMAIN=ROOT_DOMAIN`，不再额外拼前缀），后续步骤照常执行。

---

## 步骤 1：检查系统环境

```bash
whoami  # 可能不是 root（Azure/AWS 等云镜像常见非 root 账号），后续命令按需加 sudo
cat /etc/os-release | head -3  # Debian/Ubuntu 为主线；AlmaLinux/Rocky/CentOS Stream 走下方"RHEL 系差异对照表"
getenforce 2>/dev/null || echo "no SELinux"  # RHEL 系才有；Enforcing 时步骤 4/7 要额外处理
ss -tlnp | grep -E ':80 |:443 '  # 检查端口占用
```

如果 80/443 已被占用，提醒用户现有服务可能被覆盖，确认后再继续。

> **发行版不是 Debian/Ubuntu 怎么办（实战踩过，AlmaLinux 9.7）**：不要让用户重装系统，本 skill 的架构（Nginx + acme.sh + 3x-ui）在 RHEL 系上完全能跑，只是包管理、防火墙、Nginx 目录布局、fail2ban 后端几处不同。3x-ui 官方安装脚本本身就支持 `almalinux | rocky | rhel | fedora`。检测方法：
> ```bash
> . /etc/os-release; echo "$ID $VERSION_ID"   # almalinux / rocky / centos / rhel → 走 RHEL 分支
> ```
>
> **RHEL 系差异对照表**（每个受影响的步骤里都有对应的"RHEL 系"小节，这里是总览）：
>
> | 项目 | Debian/Ubuntu（主线） | RHEL 系（AlmaLinux/Rocky/CentOS Stream 9） |
> |------|----------------------|---------------------------------------------|
> | 包管理 | `apt install` | `dnf install`，先装 `epel-release`（nginx/fail2ban 在 EPEL 或 AppStream） |
> | sqlite 命令行 | 包名 `sqlite3` | 包名 `sqlite`（命令仍是 `sqlite3`） |
> | cron | 包名 `cron` | 包名 `cronie`，服务名 `crond`，acme.sh 装定时任务前必须已启用 |
> | bcrypt | `python3-bcrypt` | 没有系统包，`pip3 install bcrypt`（RHEL 9 的 pip 没有 externally-managed 限制） |
> | 防火墙 | `ufw` | `firewalld`（`firewall-cmd`），没有 ufw 包 |
> | fail2ban 封禁后端 | `banaction = ufw`，`logpath = /var/log/auth.log` | `banaction = firewallcmd-rich-rules`，`backend = systemd`（没有 auth.log，日志在 journald），额外装 `fail2ban-firewalld` |
> | Nginx 站点目录 | `/etc/nginx/sites-available/` + `sites-enabled/` 软链 | 只有 `/etc/nginx/conf.d/*.conf`，没有 sites-* 目录 |
> | Nginx 默认站点 | `sites-enabled/default` 文件，删掉即可 | 默认 server 块直接写在 `nginx.conf` 里，要整体重写 `nginx.conf` |
> | Nginx 全局加固 | `sed` 改 `ssl_protocols` / `server_tokens` | 原始 `nginx.conf` 里根本没有这两行，`sed` 会静默不生效，重写整份 `nginx.conf` |
> | 证书目录 | `/root/cert/` | 建议 `/etc/nginx/cert/`（SELinux Enforcing 时 nginx 读不了 `/root` 下的 `admin_home_t` 文件） |
> | SELinux | 无 | 可能 Enforcing：`setsebool -P httpd_can_network_connect 1` 才允许 nginx 反代到 127.0.0.1:10000 |
> | 系统更新 | `apt update && apt upgrade` | `dnf upgrade` |
>
> 步骤 3/4/5/10/11/12/13/15 完全一致，不用改。

> **已有服务排查（实战踩过坑）**：这一步顺带把整机当前监听的所有端口和正在跑的容器都摸一遍，为步骤 8 的防火墙配置做准备：
> ```bash
> ss -tlnp
> command -v docker >/dev/null && docker ps -a
> ```
> 如果发现有 Outline（`shadowbox` 容器）、其他面板或代理服务，记下它们用的端口（Outline 典型是一个管理 API 端口 + 一个访问密钥端口，具体值在 `/opt/outline/persisted-state/shadowbox_server_config.json` 里能看到 `apiUrl` 和 `portForNewAccessKeys`），步骤 8 里要把这些端口一并放行，否则部署完 UFW 默认拒绝策略会把这些已有服务全部封死。

## 步骤 2：安装依赖

```bash
apt update && apt install -y nginx ufw fail2ban curl wget openssl sqlite3 socat cron python3-bcrypt
```

> `python3-bcrypt` 用于后续生成面板密码哈希。如果包不存在，后续会 fallback 到 `pip3 install bcrypt --break-system-packages`（较新的 Debian/Ubuntu 默认锁了系统 Python 环境，直接 `pip3 install` 可能报 `externally-managed-environment` 错误，需要加这个 flag）。3x-ui 3.7+ 可以用 `x-ui setting` 命令行直接改密码，不再需要 bcrypt（见步骤 11）。

**RHEL 系（AlmaLinux/Rocky/CentOS Stream）替代命令**：

```bash
dnf install -y epel-release
dnf install -y nginx firewalld fail2ban fail2ban-firewalld curl wget openssl sqlite socat cronie tar python3 python3-pip
pip3 install bcrypt
systemctl enable --now crond firewalld
```

> `cronie`/`crond` 必须在步骤 4 装 acme.sh 之前就启用，否则 acme.sh 装不上自动续期任务。`fail2ban-firewalld` 提供 firewalld 联动的 banaction。

## 步骤 2.5：加 Swap（内存安全垫，别省略）

**实战踩坑**：不少便宜/入门档 VPS 只有 1GB 甚至更小内存，且默认没有 swap。这类机器在内存压力稍大时（哪怕只是短暂的），容易出现"整机失联"——不仅 SSH 连不上，连云平台自己的 VM Agent / Run Command 通道都可能没反应，只有串行控制台还能进（内核没死，但已经卡到几乎不能处理任何新请求）。加一点 swap 能大幅缓解这种情况，是性价比极高的一步，不要因为"看起来内存够用"就跳过。

```bash
# 判断是否需要加（内存 < 2GB 且没有 swap，建议加）
free -h

# 加 1GB swap（内存特别小的机器可以加到 2GB）
fallocate -l 1G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
grep -q "/swapfile" /etc/fstab || echo "/swapfile none swap sw 0 0" >> /etc/fstab
free -h  # 确认 Swap 那一行不再是 0
```

## 步骤 3：生成安全参数

```bash
mkdir -p /root/.secrets && chmod 700 /root/.secrets

# XHTTP 路径（16 字符 hex，短路径容易被扫描发现）
openssl rand -hex 8 > /root/.secrets/ws_path.txt

# 客户端 UUID
cat /proc/sys/kernel/random/uuid > /root/.secrets/vless_uuid.txt

# X-UI 面板凭据
echo "admin_$(openssl rand -hex 4)" > /root/.secrets/xui_username.txt
openssl rand -base64 18 > /root/.secrets/xui_password.txt

# 伪装站点公司名（随机英文，避免所有部署都叫同一个名字被 GFW 指纹识别）
DECOY_ADJS=(Alpine Summit Ridge Vertex Meridian Cascade Atlas Zenith Stratum Lumina Crestwood Northwind Silverline Boreal Solstice)
DECOY_NOUNS=(Systems Solutions Labs Partners Group Dynamics Ventures Networks Analytics Consulting)
echo "${DECOY_ADJS[RANDOM % ${#DECOY_ADJS[@]}]} ${DECOY_NOUNS[RANDOM % ${#DECOY_NOUNS[@]}]}" > /root/.secrets/decoy_name.txt

chmod 600 /root/.secrets/*.txt
```

读取生成的值，后续步骤需要：

```bash
WS_PATH=$(cat /root/.secrets/ws_path.txt)
UUID=$(cat /root/.secrets/vless_uuid.txt)
XUI_USER=$(cat /root/.secrets/xui_username.txt)
XUI_PASS=$(cat /root/.secrets/xui_password.txt)
DECOY_NAME=$(cat /root/.secrets/decoy_name.txt)

echo "WS_PATH: $WS_PATH"
echo "UUID: $UUID"
echo "XUI_USER: $XUI_USER"
echo "XUI_PASS: $XUI_PASS"
echo "DECOY_NAME: $DECOY_NAME"
```

## 步骤 4：安装 acme.sh 并申请 SSL 证书

```bash
curl https://get.acme.sh | sh -s email="$EMAIL"
```

设置 Cloudflare API（根据用户选择的认证方式，变量在步骤 0 已定义）：

```bash
# 方式一已通过 CF_Key 和 CF_Email 导出
export CF_Key CF_Email
# 方式二取消注释：export CF_Token
```

申请并安装证书：

```bash
~/.acme.sh/acme.sh --set-default-ca --server letsencrypt
~/.acme.sh/acme.sh --issue -d "$ROOT_DOMAIN" -d "*.$ROOT_DOMAIN" --dns dns_cf --keylength ec-256

mkdir -p "$CERT_DIR"
~/.acme.sh/acme.sh --install-cert -d "$ROOT_DOMAIN" --ecc \
  --cert-file "$CERT_DIR/$ROOT_DOMAIN.cer" \
  --key-file "$CERT_DIR/$ROOT_DOMAIN.key" \
  --fullchain-file "$CERT_DIR/fullchain.cer" \
  --ca-file "$CERT_DIR/ca.cer" \
  --reloadcmd "systemctl reload nginx || true"

chmod 600 "$CERT_DIR"/*.key
```

> `--set-default-ca --server letsencrypt`：acme.sh 默认 CA 是 ZeroSSL，偶尔注册/签发慢，Let's Encrypt 更稳。`--reloadcmd` 加 `|| true`，因为这一步 nginx 还没启动，reload 会报 "not active"，不影响证书安装。`CERT_DIR` 在步骤 0 定义，Debian 用 `/root/cert`，RHEL 系用 `/etc/nginx/cert`（SELinux Enforcing 时 nginx 进程读不了 `/root` 下的文件；Disabled 时放哪都行，但统一用 `/etc/nginx/cert` 省事）。
>
> 新格式的 Cloudflare Global API Key（`cfk_` 开头，2026 年起出现）和旧的 37 位 hex key 一样走 `CF_Key`+`CF_Email`，acme.sh 的 `dns_cf` 直接可用，不用换成 Token 方式。拿到凭据先本地验一下比在服务器上跑失败再回头快：
> ```bash
> curl -s "https://api.cloudflare.com/client/v4/zones?name=$ROOT_DOMAIN" -H "X-Auth-Email: $CF_Email" -H "X-Auth-Key: $CF_Key" | head -c 300
> # 期望看到 "status":"active" 和 zone id
> ```

> 如果证书申请失败并报 `DNS identifier is invalid`，回到步骤 0 的"域名核实"部分——大概率是 `ROOT_DOMAIN` 本身是公共后缀（PSL），不是真正可签证书的域名。
>
> 如果报 Cloudflare 认证相关错误（`Unknown X-Auth-Key or X-Auth-Email` 等），是 CF 凭据或邮箱不对，让用户重新检查。

## 步骤 5：创建伪装站点

**防指纹**：每次部署的伪装站"公司名"都随机生成（见步骤 3 的 `DECOY_NAME`），避免多台 VPS 共享同一特征被批量识别。模板是英文的，避免中文命名在 `.com`/`.uk` 这类域名上看起来突兀。

```bash
mkdir -p "/var/www/$DOMAIN"
cat > "/var/www/$DOMAIN/index.html" << SITEEOF
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>${DECOY_NAME}</title>
    <meta name="description" content="Professional digital solutions and consulting services.">
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif; color: #1a1a2e; line-height: 1.6; }
        .nav { background: #fff; padding: 1rem 2rem; border-bottom: 1px solid #e8e8e8; display: flex; justify-content: space-between; align-items: center; }
        .nav-brand { font-size: 1.3rem; font-weight: 600; color: #2563eb; text-decoration: none; }
        .nav-links { display: flex; gap: 2rem; list-style: none; }
        .nav-links a { text-decoration: none; color: #555; font-size: 0.95rem; }
        .hero { background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); color: white; padding: 5rem 2rem; text-align: center; }
        .hero h1 { font-size: 2.5rem; font-weight: 700; margin-bottom: 1rem; }
        .hero p { font-size: 1.2rem; opacity: 0.9; max-width: 600px; margin: 0 auto 2rem; }
        .btn { display: inline-block; background: #fff; color: #764ba2; padding: 0.8rem 2rem; border-radius: 6px; text-decoration: none; font-weight: 600; }
        .features { display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 2rem; padding: 4rem 2rem; max-width: 1100px; margin: 0 auto; }
        .feature { text-align: center; padding: 2rem; }
        .feature h3 { margin: 1rem 0 0.5rem; font-size: 1.2rem; }
        .feature p { color: #666; font-size: 0.95rem; }
        .icon { font-size: 2.5rem; }
        .footer { background: #f8f9fa; padding: 2rem; text-align: center; color: #888; font-size: 0.85rem; border-top: 1px solid #e8e8e8; }
    </style>
</head>
<body>
    <nav class="nav">
        <a href="#" class="nav-brand">${DECOY_NAME}</a>
        <ul class="nav-links">
            <li><a href="#">Services</a></li>
            <li><a href="#">About</a></li>
            <li><a href="#">Contact</a></li>
        </ul>
    </nav>
    <section class="hero">
        <h1>Digital Solutions for Modern Business</h1>
        <p>We help companies transform their operations with cutting-edge technology and data-driven strategies.</p>
        <a href="#" class="btn">Learn More</a>
    </section>
    <section class="features">
        <div class="feature"><div class="icon">&#9729;</div><h3>Cloud Infrastructure</h3><p>Scalable and secure cloud solutions tailored to your business needs.</p></div>
        <div class="feature"><div class="icon">&#9881;</div><h3>Process Automation</h3><p>Streamline workflows and reduce operational costs with smart automation.</p></div>
        <div class="feature"><div class="icon">&#128200;</div><h3>Data Analytics</h3><p>Turn your data into actionable insights with advanced analytics platforms.</p></div>
    </section>
    <footer class="footer">&copy; 2026 ${DECOY_NAME}. All rights reserved.</footer>
</body>
</html>
SITEEOF
```

## 步骤 6：加固 Nginx 全局配置

```bash
sed -i 's/ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;/ssl_protocols TLSv1.2 TLSv1.3;/' /etc/nginx/nginx.conf
sed -i 's/# server_tokens off;/server_tokens off;/' /etc/nginx/nginx.conf
```

**RHEL 系**：自带的 `nginx.conf` 里既没有 `ssl_protocols` 也没有 `server_tokens` 行（上面两条 `sed` 会静默不生效），而且默认 `server { listen 80 default_server; }` 块直接写在 `nginx.conf` 里，会抢掉我们站点的 80 端口。直接整体重写：

```bash
cp -n /etc/nginx/nginx.conf /etc/nginx/nginx.conf.orig
cat > /etc/nginx/nginx.conf <<'NGX'
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log notice;
pid /run/nginx.pid;
include /usr/share/nginx/modules/*.conf;
events { worker_connections 1024; }
http {
    log_format main '$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent"';
    access_log /var/log/nginx/access.log main;
    sendfile on;
    tcp_nopush on;
    keepalive_timeout 65;
    types_hash_max_size 4096;
    server_tokens off;
    ssl_protocols TLSv1.2 TLSv1.3;
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    include /etc/nginx/conf.d/*.conf;
}
NGX
# SELinux Enforcing 时还要放行 nginx 反代到本机端口
[ "$(getenforce 2>/dev/null)" = "Enforcing" ] && setsebool -P httpd_can_network_connect 1
```

## 步骤 7：配置 Nginx 站点

```bash
# Debian/Ubuntu 写到 sites-available；RHEL 系把路径换成 /etc/nginx/conf.d/$DOMAIN.conf（见下方）
cat > "/etc/nginx/sites-available/$DOMAIN" << NGINXEOF
server {
    listen 80;
    listen [::]:80;
    server_name $DOMAIN;
    return 301 https://\$server_name\$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name $DOMAIN;

    ssl_certificate $CERT_DIR/fullchain.cer;
    ssl_certificate_key $CERT_DIR/$ROOT_DOMAIN.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305';
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;

    server_tokens off;
    root /var/www/$DOMAIN;
    index index.html;

    # XHTTP 代理 → Xray
    location /$WS_PATH {
        proxy_pass http://127.0.0.1:10000;
        proxy_http_version 1.1;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;

        # XHTTP mode=auto 时也需要支持 WebSocket upgrade
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection "upgrade";

        # 禁用缓冲，确保流式传输
        proxy_buffering off;
        proxy_cache off;
    }

    location / { try_files \$uri \$uri/ =404; }
    location ~ /\. { deny all; }
}
NGINXEOF
```

> heredoc 不带引号，`$DOMAIN`、`$ROOT_DOMAIN`、`$WS_PATH`、`$CERT_DIR` 会被 bash 展开为实际值。Nginx 变量（`$server_name`、`$host` 等）用 `\$` 转义以保留。

启用配置：

```bash
ln -sf "/etc/nginx/sites-available/$DOMAIN" /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default
nginx -t && systemctl reload nginx
```

**RHEL 系**：没有 `sites-available`/`sites-enabled`，把上面同样的 server 配置写到 `/etc/nginx/conf.d/$DOMAIN.conf`（heredoc 内容不变，只改 `cat >` 的目标路径），两个 `listen` 行建议加 `default_server`（`listen 80 default_server;` / `listen 443 ssl http2 default_server;`），因为步骤 6 重写后的 `nginx.conf` 里已经没有默认站点了。然后：

```bash
nginx -t && systemctl enable --now nginx && systemctl restart nginx
# 本地验一下伪装站（不经过 CF），期望 200
curl -sk --resolve "$DOMAIN:443:127.0.0.1" -o /dev/null -w '%{http_code}\n' "https://$DOMAIN/"
```

## 步骤 8：配置防火墙

**实战踩坑**：这一步会把 UFW 改成"默认拒绝所有入站"，如果这台机器上有步骤 1 发现的已有服务（比如 Outline），必须在 `ufw enable` 之前一并放行对应端口，否则会直接把已有服务打断。

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow "$SSH_PORT/tcp"
ufw allow 80/tcp
ufw allow 443/tcp
ufw deny "$XUI_PORT/tcp"

# 如果步骤 1 发现了已有服务，在这里补充放行，例如 Outline 的典型端口：
# ufw allow <管理API端口>/tcp comment "outline-management"
# ufw allow <访问密钥端口>/tcp comment "outline-access-key"
# ufw allow <访问密钥端口>/udp comment "outline-access-key"

echo "y" | ufw enable
```

**RHEL 系（firewalld 替代 ufw）**：firewalld 默认策略就是"未放行的入站全部拒绝"，所以不需要 `deny` 面板端口，也不需要单独 deny 任何东西，只做放行。默认 public 区域通常预放行了 `cockpit` 和 `dhcpv6-client`，顺手去掉：

```bash
firewall-cmd --permanent --add-service=ssh      # SSH 端口不是 22 时改用 --add-port=$SSH_PORT/tcp
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --remove-service=cockpit 2>/dev/null
firewall-cmd --permanent --remove-service=dhcpv6-client 2>/dev/null
# 已有服务的端口在这里补：firewall-cmd --permanent --add-port=<端口>/tcp
firewall-cmd --reload
firewall-cmd --list-all | grep -E 'services|ports'   # 期望 services: http https ssh
```

> 云平台（Azure/AWS/GCP 等）通常还有一层独立于 UFW 之外的网络级防火墙（Azure 叫"网络安全组 NSG"，AWS 叫"Security Group"，GCP 叫"防火墙规则"）。**只放行 UFW 不够**，必须同时确认云平台控制台里也放行了 22/80/443（以及任何已有服务的端口）。这一层无法通过 SSH 命令配置，必须提醒用户去对应云平台的网页控制台操作。判断方法：服务器上 `curl localhost` 一切正常，但从外网 `curl`/`telnet` 到服务器 IP 却连不上，基本可以确定是这一层的问题。

## 步骤 9：配置 Fail2Ban

```bash
cat > /etc/fail2ban/jail.local << JAILEOF
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 5
banaction = ufw

[sshd]
enabled = true
port = $SSH_PORT
logpath = /var/log/auth.log
maxretry = 3
bantime = 7200
JAILEOF

systemctl enable fail2ban && systemctl restart fail2ban
```

**RHEL 系**：没有 `/var/log/auth.log`（sshd 日志在 journald），也没有 ufw 这个 banaction。`jail.local` 改成：

```bash
cat > /etc/fail2ban/jail.local << JAILEOF
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 5
banaction = firewallcmd-rich-rules
backend = systemd

[sshd]
enabled = true
port = $SSH_PORT
maxretry = 3
bantime = 7200
JAILEOF

systemctl enable fail2ban && systemctl restart fail2ban
sleep 2 && fail2ban-client status sshd | head -4   # 能看到 Filter/Actions 就是起来了
```

## 步骤 10：安装 3X-UI

```bash
bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh)
```

> **交互处理（不同版本流程可能不同，实战踩过坑）**：早期版本的安装器依次要求输入"是否继续安装 → 用户名 → 密码 → 端口"四项，可以用 `echo -e "y\ntemp\ntemp\n54321" | bash <(curl ...)` 一次性喂完。**较新版本的安装器多了"数据库选择（SQLite/PostgreSQL）"、"Base URI Path"、"SSL 证书设置（Let's Encrypt / 自定义 / 跳过）"等额外交互项，顺序和数量都可能变化**；而且检测到脚本是非交互方式运行（没有 controlling tty）时，有的版本会直接跳过所有输入项、自动生成随机用户名/密码/端口，忽略你喂给它的 stdin。
>
> **不管装的时候到底问了什么、给了什么答案都不用纠结**——反正步骤 11 会通过数据库把用户名、密码、端口全部强制覆盖成我们自己生成的值。这一步的唯一目标就是让安装器跑完、把 `/etc/x-ui/x-ui.db` 建出来。
>
> 使用 **MHSanaei/3x-ui**，不要用已停维的 vaxilu/x-ui。

**推荐做法（3x-ui 3.7+ 安装器支持正式的非交互模式，实战验证 2026-09）**：安装脚本里有 `prompt_or_default` 函数，`NONINTERACTIVE=1` 时所有 `read -rp` 都改从环境变量取值，不再有"喂 stdin 被忽略"的不确定性。先把脚本下载下来看一眼有哪些 `XUI_*` 变量（`grep -oE 'XUI_[A-Z_]+' install.sh | sort -u`），然后直接把我们自己生成的凭据喂进去，步骤 11 就只剩收尾工作：

```bash
export NONINTERACTIVE=1 XUI_NONINTERACTIVE=1
export XUI_USERNAME="$XUI_USER"
export XUI_PASSWORD="$XUI_PASS"
export XUI_PANEL_PORT="$XUI_PORT"
export XUI_WEB_BASE_PATH="panel"      # 安装器要求 ≥4 字符，步骤 11 再改回 /
export XUI_SSL_MODE=none              # 面板走 SSH 隧道，不要安装器自己去申请证书
export XUI_DB_TYPE=sqlite
export XUI_SERVER_IP="$(curl -s --max-time 10 ifconfig.me)"

curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh -o /tmp/xui-install.sh
bash /tmp/xui-install.sh > /tmp/xui-install.log 2>&1 < /dev/null
echo "rc=$?"
grep -aE 'Installation Complete|Username:|Port:|WebBasePath|error|failed' /tmp/xui-install.log | sed 's/\x1b\[[0-9;]*m//g'
```

> 非交互模式下安装器会把面板绑到 `0.0.0.0`（注释里写明"cloud images must stay reachable"），步骤 11 用 `x-ui setting -listenIP 127.0.0.1` 改回来。`< /dev/null` 是为了万一某个版本还有漏网的 `read`，让它立刻拿到 EOF 走默认值而不是挂住。

等待安装完成，确认数据库文件存在，并顺手记录一下装的是什么版本（后面步骤 12/13 要根据版本判断 schema）：

```bash
ls -la /etc/x-ui/x-ui.db
x-ui --version 2>&1 || (command -v x-ui >/dev/null && x-ui | head -5)
```

## 步骤 11：配置 3X-UI 面板

**推荐做法（3x-ui 3.7+，实战验证）**：面板自带 `x-ui setting` 命令行，能一次改完凭据、端口、base path、监听地址，自己处理 bcrypt 和表结构，比手写 sqlite 稳：

```bash
systemctl stop x-ui
/usr/local/x-ui/x-ui setting -username "$XUI_USER" -password "$XUI_PASS" -port "$XUI_PORT" -webBasePath "/" -listenIP "127.0.0.1"
# 期望三行：Username and password updated successfully / Base URI path set successfully / listen 127.0.0.1 set successfully

# 订阅端口和 secret 没有命令行开关，还是走 sqlite。用"先查有没有再决定 INSERT/UPDATE"的写法，避开下面说的 INSERT OR REPLACE 重复行问题
set_setting() {
  local n=$(sqlite3 /etc/x-ui/x-ui.db "SELECT COUNT(*) FROM settings WHERE key='$1';")
  if [ "$n" = "0" ]; then sqlite3 /etc/x-ui/x-ui.db "INSERT INTO settings (key, value) VALUES ('$1', '$2');"
  else sqlite3 /etc/x-ui/x-ui.db "UPDATE settings SET value='$2' WHERE key='$1';"; fi
}
set_setting subEnable false
set_setting subListen 127.0.0.1
set_setting secret "$(openssl rand -hex 16)"

# 把 API token 存下来，步骤 13 要用（3.7+ 安装时自动生成）
/usr/local/x-ui/x-ui setting -getApiToken 2>&1 | sed 's/\x1b\[[0-9;]*m//g' | grep -Eo 'apiToken: .+' | awk '{print $2}' > /root/.secrets/xui_api_token.txt
chmod 600 /root/.secrets/xui_api_token.txt
```

装的是老版本、没有 `x-ui setting -listenIP` 这类参数时，退回下面的 sqlite 方式：

```bash
systemctl stop x-ui

# 面板仅 localhost 访问
sqlite3 /etc/x-ui/x-ui.db "INSERT OR REPLACE INTO settings (key, value) VALUES ('webListen', '127.0.0.1');"
sqlite3 /etc/x-ui/x-ui.db "INSERT OR REPLACE INTO settings (key, value) VALUES ('webPort', '$XUI_PORT');"

# 更新面板凭据（bcrypt 密码哈希）
# 确保 bcrypt 模块可用
python3 -c "import bcrypt" 2>/dev/null || pip3 install bcrypt -q --break-system-packages
# 注意：openssl rand -base64 生成的密码仅含 A-Za-z0-9+/= 字符，不含单引号，此处安全
HASHED=$(python3 -c "
import bcrypt
password = '$XUI_PASS'
print(bcrypt.hashpw(password.encode(), bcrypt.gensalt(10)).decode())
")
sqlite3 /etc/x-ui/x-ui.db "UPDATE users SET username='$XUI_USER', password='$HASHED' WHERE id=1;"

# 生成新 secret
NEW_SECRET=$(openssl rand -hex 16)
sqlite3 /etc/x-ui/x-ui.db "UPDATE settings SET value='$NEW_SECRET' WHERE key='secret';"

# 禁用订阅端口（默认会在公网暴露！）
sqlite3 /etc/x-ui/x-ui.db "INSERT OR REPLACE INTO settings (key, value) VALUES ('subEnable', 'false');"
sqlite3 /etc/x-ui/x-ui.db "INSERT OR REPLACE INTO settings (key, value) VALUES ('subListen', '127.0.0.1');"
```

> ⚠️ **`INSERT OR REPLACE` 在新版 schema 下可能产生重复行**：新版 3x-ui 的 `settings` 表只有 `key` 上的普通索引，没有 UNIQUE 约束，`INSERT OR REPLACE` 在没有唯一约束冲突的情况下等价于普通 `INSERT`，如果这个 key 之前已经存在一行（比如安装器自己写过默认 `webPort`），执行完会变成两行同 key 不同 value 的记录，面板读取哪一行是不确定的。**执行完这一步，务必检查一下有没有产生重复行**：
> ```bash
> sqlite3 /etc/x-ui/x-ui.db "SELECT key, COUNT(*) c FROM settings GROUP BY key HAVING c > 1;"
> ```
> 如果有重复，保留你想要的那一行，删掉其余的：
> ```bash
> sqlite3 /etc/x-ui/x-ui.db "DELETE FROM settings WHERE key='webPort' AND value != '$XUI_PORT';"
> ```

顺手记一下面板的 `webBasePath`（新版可能会自动生成一个随机路径前缀，访问面板要带上）：

```bash
WEBBASEPATH=$(sqlite3 /etc/x-ui/x-ui.db "SELECT value FROM settings WHERE key='webBasePath';")
echo "webBasePath: $WEBBASEPATH"
```

## 步骤 12：设置 Xray 模板

模板必须完整，否则 outbounds 为 null 导致流量无法出站：

```bash
sqlite3 /etc/x-ui/x-ui.db "INSERT OR REPLACE INTO settings (key, value) VALUES ('xrayTemplateConfig', '{
  \"log\": {\"access\": \"none\", \"dnsLog\": false, \"loglevel\": \"warning\"},
  \"dns\": {\"servers\": [\"8.8.8.8\", \"1.1.1.1\"]},
  \"inbounds\": [
    {\"tag\": \"api\", \"listen\": \"127.0.0.1\", \"port\": 62789, \"protocol\": \"tunnel\", \"settings\": {\"rewriteAddress\": \"127.0.0.1\"}}
  ],
  \"routing\": {
    \"domainStrategy\": \"IPIfNonMatch\",
    \"rules\": [
      {\"type\": \"field\", \"inboundTag\": [\"api\"], \"outboundTag\": \"api\"},
      {\"type\": \"field\", \"outboundTag\": \"blocked\", \"protocol\": [\"bittorrent\"]}
    ]
  },
  \"outbounds\": [
    {\"tag\": \"direct\", \"protocol\": \"freedom\", \"settings\": {}},
    {\"tag\": \"blocked\", \"protocol\": \"blackhole\", \"settings\": {}}
  ],
  \"policy\": {\"levels\": {\"0\": {\"statsUserDownlink\": true, \"statsUserUplink\": true}}, \"system\": {\"statsInboundDownlink\": true, \"statsInboundUplink\": true}},
  \"api\": {\"tag\": \"api\", \"services\": [\"HandlerService\", \"LoggerService\", \"StatsService\", \"RoutingService\"]},
  \"metrics\": {\"tag\": \"metrics_out\", \"listen\": \"127.0.0.1:11111\"},
  \"stats\": {}
}');"
```

> **`inbounds` 里的 api 入站不能省（2026-09-15 实战补的坑）**：3x-ui 自带的默认模板里有这个 `tag: api` 的本地入站，面板靠它找到 Xray gRPC API 端口来做客户端热加载。本 skill 早期模板只写了 `api` 块没写这个入站，后果是：面板日志每几秒一行 `Failed to initialize Xray API: invalid Xray API port: 0`；每次加/删客户端都走不了热加载（日志 `Error in adding client on local : local xray is not running`），退化成由一个约 30 秒一次的巡检任务整体重启 Xray——客户端 30 秒内不可用、已连接用户被断一次，而且期间用程序去查会误判成"加客户端不生效 / 入站损坏"。补上之后加客户端秒级生效、不重启 Xray（日志变成 `Client added on local`）。
> - 协议名：Xray 25.x 起 `dokodemo-door` 改名 `tunnel`，`address` 改 `rewriteAddress`，跟随面板自带模板（`internal/web/service/config.json`）写即可；老 Xray 用 `dokodemo-door` + `settings.address`。
> - 62789 和 11111 只监听 127.0.0.1，不需要开防火墙。
> - **热加载后 `config.json` 不会更新**（它只在 Xray 重启时重新生成），所以判断客户端是否生效不要再看 `config.json`，用 `GET /panel/api/clients/get/<email>` 回读，或直接用该 UUID 连一次。
> - 已经部署过的机器补救：用 python 读出 `xrayTemplateConfig`，加上 `inbounds`/`metrics`、把 `api.services` 补上 `RoutingService`，写回后 `systemctl restart x-ui`，`ss -tlnp | grep -E ':62789|:11111'` 能看到即可。

> `log.access` 平时保持 `"none"`。排障时可以临时改成一个文件路径开详细日志（见 `troubleshooting.md`），但**排障结束后一定要记得改回 `"none"` 并 `systemctl restart x-ui`**——长期开着不会立刻出事（体量不大的话），但没必要一直占用额外的磁盘 I/O，也曾经在一次内存本就紧张的机器上被怀疑是雪上加霜的因素之一。

## 步骤 13：创建 VLESS 入站与客户端

> ⚠️ **这是整个部署过程中最容易踩、也最不容易发现的坑（实战真实翻车过）：3x-ui 不同大版本的客户端数据存储方式不一样。**
>
> - **旧版**（`inbounds` 表有 `all_time` 字段）：客户端列表内嵌在 `inbounds.settings` 这个 JSON 字段里的 `clients` 数组中，插入一条 inbound 记录就够了。
> - **新版**（`inbounds` 表没有 `all_time`，多了 `node_id`/`share_addr_strategy` 等字段）：客户端数据被拆到两张独立的表——`clients`（客户端本体：uuid/email/flow 等）和 `client_inbounds`（客户端与 inbound 的关联关系）。**运行时生成 Xray 配置时，`inbounds.settings.clients` 这个字段会被忽略**，只认 `clients` + `client_inbounds` 两张表。如果只插入 `inbounds` 一条记录、不管新表，效果是：面板能登录、inbound 看着配置正确、`systemctl status x-ui` 显示 active、Nginx 访问日志一切正常——但 Xray 运行配置（`config.json`）里这个 inbound 的 `clients` 字段是 `null`，客户端连接請求会被 Xray 直接拒绝（日志报 `rejected proxy/vless/encoding: invalid request user id: xxx`），所有网站都打不开。**这个状态很容易被误判为"已部署成功"**，因为常规的服务状态检查、端口监听检查全部正常，只有专门去看 Xray 的详细访问日志才能发现。
>
> **执行这一步之前，先查一遍实际 schema，判断走哪条路**：
> ```bash
> sqlite3 /etc/x-ui/x-ui.db ".schema inbounds" | grep -q "all_time" && echo "OLD_SCHEMA" || echo "NEW_SCHEMA"
> ```

### 路线 A（新版 schema 首选，3x-ui 3.7.0 实战验证）：走面板 API，不碰 sqlite

新版把 client 拆表之后，手写 sqlite 要同时对 `inbounds`/`clients`/`client_inbounds` 三张表的字段和默认值负责，版本一变就得重查。面板自己的 HTTP API 会替你把这些都写对，而且 3.7+ 安装时就生成了一个 API token，不用登录拿 cookie：

```bash
TOKEN=$(cat /root/.secrets/xui_api_token.txt)
B="http://127.0.0.1:$XUI_PORT"    # webBasePath 是 / 时；如果步骤 11 保留了 base path，要拼进去
systemctl start x-ui && sleep 3

# 先确认 token 可用
curl -s "$B/panel/api/inbounds/list" -H "Authorization: Bearer $TOKEN"   # 期望 {"success":true,...}

# 三段 JSON 作为字符串字段传给 API（API 要求 settings/streamSettings/sniffing 是 JSON 字符串，不是嵌套对象）
# 注意：client 里不要带 "tgId":""，3.7 把 tgId 改成了 int64，传空字符串会报 cannot unmarshal string ... tgId
SETTINGS="{\"clients\":[{\"id\":\"$UUID\",\"flow\":\"\",\"email\":\"default-user\",\"limitIp\":0,\"totalGB\":0,\"expiryTime\":0,\"enable\":true,\"subId\":\"\",\"reset\":0}],\"decryption\":\"none\",\"fallbacks\":[]}"
STREAM="{\"network\":\"xhttp\",\"security\":\"none\",\"xhttpSettings\":{\"path\":\"/$WS_PATH\",\"host\":\"$DOMAIN\",\"mode\":\"auto\"}}"
SNIFFING="{\"enabled\":true,\"destOverride\":[\"http\",\"tls\",\"quic\"],\"metadataOnly\":false,\"routeOnly\":true}"
python3 - "$SETTINGS" "$STREAM" "$SNIFFING" > /tmp/inbound.json <<'PY'
import json, sys
print(json.dumps({"up":0,"down":0,"total":0,"remark":"VLESS-XHTTP-TLS-CF","enable":True,"expiryTime":0,
  "listen":"127.0.0.1","port":10000,"protocol":"vless",
  "settings":sys.argv[1],"streamSettings":sys.argv[2],"sniffing":sys.argv[3]}))
PY
curl -s -X POST "$B/panel/api/inbounds/add" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" --data @/tmp/inbound.json
# 期望 {"success":true,"msg":"Inbound has been successfully created.", ...}

# API 返回成功后 config.json 不一定马上重新生成，必须重启一次
systemctl restart x-ui && sleep 4
```

> 几个实战里踩到的细节：
> - **不要用 `curl -X POST /login` 拿 cookie**：3.7 的登录接口有 CSRF 保护，非浏览器请求即使带上 `Origin`/`Referer` 也是 403 空响应。Bearer token 走 `/panel/api/*` 路由没有这个限制。
> - **API 创建的入站 tag 是 `in-10000-tcp`**，不是本文档旧步骤里的 `inbound-10000`。后面任何 `WHERE tag='inbound-10000'` 的查询都要改成 `WHERE port=10000`。
> - 直接往 `inbounds` 表 INSERT 一行（哪怕字段全对、`enable=1`）在 3.7.0 上会被 config 生成逻辑**静默忽略**，`config.json` 的 `inbounds` 是空数组、10000 端口不监听、日志没有任何报错——比 CHANGELOG 里记的"clients 为 null"更彻底。所以新版 schema 优先走 API；下面的路线 B 只在 API 也用不了时才用。

### 路线 B（旧版 schema，或 API 不可用时的兜底）：直接写 sqlite

先插入 inbound（两种 schema 通用，字段按实际 schema 增减）：

```bash
STREAM="{\"network\":\"xhttp\",\"security\":\"none\",\"xhttpSettings\":{\"path\":\"/$WS_PATH\",\"host\":\"$DOMAIN\",\"mode\":\"auto\"}}"
SNIFFING="{\"enabled\":true,\"destOverride\":[\"http\",\"tls\",\"quic\"],\"metadataOnly\":false,\"routeOnly\":true}"

# 旧版 schema（有 all_time 字段，clients 内嵌 JSON）：
SETTINGS_OLD="{\"clients\":[{\"id\":\"$UUID\",\"flow\":\"\",\"email\":\"default-user\",\"limitIp\":0,\"totalGB\":0,\"expiryTime\":0,\"enable\":true,\"tgId\":\"\",\"subId\":\"\",\"reset\":0}],\"decryption\":\"none\",\"fallbacks\":[]}"
sqlite3 /etc/x-ui/x-ui.db "INSERT INTO inbounds (user_id, up, down, total, all_time, remark, enable, expiry_time, listen, port, protocol, settings, stream_settings, tag, sniffing) VALUES (1, 0, 0, 0, 0, 'VLESS-XHTTP-TLS-CF', 1, 0, '127.0.0.1', 10000, 'vless', '$SETTINGS_OLD', '$STREAM', 'inbound-10000', '$SNIFFING');"

# 新版 schema（无 all_time，settings.clients 留空，客户端另外写两张表）：
SETTINGS_NEW="{\"clients\":[],\"decryption\":\"none\",\"fallbacks\":[]}"
sqlite3 /etc/x-ui/x-ui.db "INSERT INTO inbounds (user_id, up, down, total, remark, enable, expiry_time, listen, port, protocol, settings, stream_settings, tag, sniffing) VALUES (1, 0, 0, 0, 'VLESS-XHTTP-TLS-CF', 1, 0, '127.0.0.1', 10000, 'vless', '$SETTINGS_NEW', '$STREAM', 'inbound-10000', '$SNIFFING');"
```

### 新版 schema 额外要做：写入 clients + client_inbounds 两张表

```bash
INBOUND_ID=$(sqlite3 /etc/x-ui/x-ui.db "SELECT id FROM inbounds WHERE tag='inbound-10000';")
NOW=$(date +%s%3N)

sqlite3 /etc/x-ui/x-ui.db "INSERT INTO clients (email, uuid, flow, limit_ip, total_gb, expiry_time, enable, reset, created_at, updated_at) VALUES ('default-user', '$UUID', '', 0, 0, 0, 1, 0, $NOW, $NOW);"
CLIENT_ID=$(sqlite3 /etc/x-ui/x-ui.db "SELECT id FROM clients WHERE uuid='$UUID';")
sqlite3 /etc/x-ui/x-ui.db "INSERT INTO client_inbounds (client_id, inbound_id, created_at) VALUES ($CLIENT_ID, $INBOUND_ID, $NOW);"
```

### 启动并验证 clients 真的生效了（不要跳过这步，A/B 两条路线都要做）

```bash
systemctl restart x-ui
sleep 3

# 关键验证：不管走的哪种路线，最终都要在运行配置里看到这个 inbound 且 clients 有真实内容，而不是 null / 空数组
python3 -c "
import json
cfg = json.load(open('/usr/local/x-ui/bin/config.json'))
print('inbound count:', len(cfg.get('inbounds') or []))
for ib in cfg.get('inbounds') or []:
    print(ib.get('tag'), ib.get('listen'), ib.get('port'), json.dumps(ib.get('settings')))
"
ss -tlnp | grep '127.0.0.1:10000'   # 必须有
```

看到类似 `{"clients": [{"email": "default-user", "id": "你的UUID"}], ...}` 才算真正成功。**如果看到 `"clients": null`，说明客户端数据没有正确落地到当前版本实际读取的位置**——回头确认一下：
1. `.schema clients`、`.schema client_inbounds` 这两张表是否存在（新版才有）
2. `SELECT * FROM clients;` 和 `SELECT * FROM client_inbounds;` 是否真的插入成功、`inbound_id` 是否对应上了
3. 改完数据库有没有 `systemctl restart x-ui`（不重启不会重新生成 config.json）

## 步骤 14：健康检查

逐项检查，全部通过才算部署成功：

```bash
# 服务状态
systemctl is-active nginx x-ui fail2ban

# 端口监听
ss -tlnp | grep -E ':80 |:443 |:10000 |:54321 '

# Xray 监听在 localhost
ss -tlnp | grep '127.0.0.1:10000'

# X-UI 面板仅 localhost
ss -tlnp | grep "127.0.0.1:$XUI_PORT"

# Xray outbounds 不为 null，且 inbound 的 clients 不为 null（后者是实战踩过的坑，务必检查）
python3 -c "
import json
cfg = json.load(open('/usr/local/x-ui/bin/config.json'))
print('Outbounds:', cfg.get('outbounds'))
print('DNS:', cfg.get('dns'))
for ib in cfg.get('inbounds', []):
    print('Inbound', ib.get('tag'), 'clients:', ib.get('settings', {}).get('clients'))
"

# 防火墙状态（确认已有服务端口——如果有——也在放行列表里）
ufw status                              # Debian/Ubuntu
firewall-cmd --list-all 2>/dev/null     # RHEL 系

# 公网监听面：只应该有 22/80/443；看到 2096（订阅端口）或 54321 绑在 0.0.0.0 就是步骤 11 没生效
ss -tlnp | awk 'NR>1{print $4}' | sort -u

# Xray API 入站（步骤 12 模板里的 tag: api）必须在 127.0.0.1:62789 监听，否则面板无法热加载客户端
ss -tlnp | grep -E '127.0.0.1:62789 ' || echo "MISSING: xray api inbound (check xrayTemplateConfig inbounds)"

# SSL 证书有效期 + 自动续期任务
openssl x509 -in "$CERT_DIR/fullchain.cer" -noout -enddate
crontab -l | grep -c acme.sh            # 期望 1

# XHTTP 链路（本机 → Nginx → Xray）：期望 404（Xray 在响应但握手不合法），502 说明 Xray 没监听
WS_PATH=$(cat /root/.secrets/ws_path.txt)
curl -sk --resolve "$DOMAIN:443:127.0.0.1" -o /dev/null -w '%{http_code}\n' -X POST "https://$DOMAIN/$WS_PATH"
```

如果任一项不通过，参考 `troubleshooting.md` 对应的诊断和修复方法。

> **光看服务 active、端口 listening 不够，务必确认 `clients` 字段有真实内容**——这是本次实战里唯一一个"所有常规检查都通过、但实际完全不能用"的坑，专门留了这一条检查项来堵它。

## 步骤 15：Cloudflare 自动配置（如果拿到了 API 凭据，优先自动做）

有 Global API Key/Token 的话，不用等用户手动点 Cloudflare 面板，直接调 API 把 DNS 和 TLS 设置一次性配完：

```bash
CF_EMAIL="your_cf_email"
CF_KEY="your_global_api_key"

# 查 zone id
ZONE_ID=$(curl -s -X GET "https://api.cloudflare.com/client/v4/zones?name=$DOMAIN" \
  -H "X-Auth-Email: $CF_EMAIL" -H "X-Auth-Key: $CF_KEY" -H "Content-Type: application/json" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['result'][0]['id'])")

SERVER_IP=$(curl -s --max-time 10 ifconfig.me)

# A 记录，橙色云代理
curl -s -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records" \
  -H "X-Auth-Email: $CF_EMAIL" -H "X-Auth-Key: $CF_KEY" -H "Content-Type: application/json" \
  --data "{\"type\":\"A\",\"name\":\"$DOMAIN\",\"content\":\"$SERVER_IP\",\"ttl\":1,\"proxied\":true}"

# SSL 模式 strict
curl -s -X PATCH "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/settings/ssl" \
  -H "X-Auth-Email: $CF_EMAIL" -H "X-Auth-Key: $CF_KEY" -H "Content-Type: application/json" \
  --data '{"value":"strict"}'

# 最低 TLS 1.2
curl -s -X PATCH "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/settings/min_tls_version" \
  -H "X-Auth-Email: $CF_EMAIL" -H "X-Auth-Key: $CF_KEY" -H "Content-Type: application/json" \
  --data '{"value":"1.2"}'

# WebSocket 开启
curl -s -X PATCH "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/settings/websockets" \
  -H "X-Auth-Email: $CF_EMAIL" -H "X-Auth-Key: $CF_KEY" -H "Content-Type: application/json" \
  --data '{"value":"on"}'
```

> 先 `GET .../dns_records` 看一眼有没有已存在的同名记录再 POST，避免建出重复 A 记录。`ssl`/`min_tls_version`/`websockets` 三个设置也可以先 GET 一下，只 PATCH 不对的项。

配完等 10-20 秒 DNS 生效，然后验证（这两条在本机跑，不是在服务器上）：

```bash
# 伪装站经 CF：期望 200，且响应头有 server: cloudflare / cf-ray
curl -sI --max-time 20 "https://$DOMAIN/" | grep -iE '^(HTTP|server|cf-ray)'
# XHTTP 路径经 CF：期望 404（Xray 在响应）；5xx = Nginx/Xray 那一层断了，521/522 = CF 连不上源站
curl -s -o /dev/null -w "XHTTP via CF: %{http_code}\n" --max-time 20 -X POST "https://$DOMAIN/$WS_PATH"
```

如果这里超时或连不上，且服务器本地 `curl localhost` 正常，优先怀疑云平台安全组/NSG 没放行 80/443（见步骤 8 的说明）。

## 步骤 16：保存配置并输出结果

```bash
SERVER_IP=$(curl -s --max-time 10 ifconfig.me)
INBOUND_INFO=$(sqlite3 /etc/x-ui/x-ui.db "SELECT id || ' (tag=' || tag || ', remark=' || remark || ')' FROM inbounds WHERE port=10000;")
WEBBASEPATH=$(/usr/local/x-ui/x-ui setting -show true 2>&1 | sed 's/\x1b\[[0-9;]*m//g' | grep -Eo 'webBasePath: .+' | awk '{print $2}')

VLESS_LINK="vless://${UUID}@${DOMAIN}:443?encryption=none&security=tls&sni=${DOMAIN}&type=xhttp&host=${DOMAIN}&path=%2F${WS_PATH}&fp=chrome#VLESS-XHTTP-TLS-CF"

cat > /root/vpn-config.txt << CFGEOF
========================================
  VPN Configuration - $DOMAIN
========================================

## Architecture
Client → Cloudflare CDN (443) → Nginx (TLS) → Xray (127.0.0.1:10000) → Internet

## Domain
  Domain:     $DOMAIN
  Server IP:  $SERVER_IP (hidden behind Cloudflare)

## VLESS Client Link (copy as ONE LINE, no line breaks!)
$VLESS_LINK

## Manual Client Config
  Protocol:   VLESS
  Address:    $DOMAIN
  Port:       443 (NOT 10000)
  UUID:       $UUID
  Encryption: none
  Transport:  XHTTP
  Path:       /$WS_PATH
  Host:       $DOMAIN
  TLS:        Enabled (MUST be on)
  SNI:        $DOMAIN (MUST be set)
  Fingerprint: chrome

## Client app setting
  Prefer TUN mode over system-proxy mode to avoid QUIC/DNS leaking around the proxy.

## X-UI Panel Access (SSH tunnel ONLY)
  SSH Tunnel: ssh -p $SSH_PORT -L $XUI_PORT:localhost:$XUI_PORT user@$SERVER_IP
  URL:        http://localhost:$XUI_PORT/${WEBBASEPATH:-}
  basePath:   ${WEBBASEPATH:-/}
  Username:   $XUI_USER
  Password:   $XUI_PASS
  Inbound:    ${INBOUND_INFO:-run: sqlite3 /etc/x-ui/x-ui.db "SELECT id,tag FROM inbounds;"}

## Adding New Clients
  Do NOT create new inbounds! Add clients to existing inbound:
  Panel → Inbound list → VLESS-XHTTP-TLS-CF → "+" → Add Client
  Then generate link using this template:
  vless://<UUID>@$DOMAIN:443?encryption=none&security=tls&sni=$DOMAIN&type=xhttp&host=$DOMAIN&path=%2F$WS_PATH&fp=chrome#<NAME>
  WARNING: Panel-exported links have WRONG port, TLS, and transport settings!

## Security
  - Firewall (ufw / firewalld): only $SSH_PORT/80/443 (+ any pre-existing service ports) open
  - X-UI: localhost only (127.0.0.1:$XUI_PORT)
  - Xray: localhost only (127.0.0.1:10000)
  - Fail2Ban: SSH $SSH_PORT, 3 attempts → 2hr ban
  - Subscription port: disabled
  - Swap: enabled as a memory-pressure safety net

## Key Files
  - Secrets:     /root/.secrets/
  - Nginx Site:  /etc/nginx/sites-available/$DOMAIN  (RHEL: /etc/nginx/conf.d/$DOMAIN.conf)
  - Fail2Ban:    /etc/fail2ban/jail.local
  - X-UI DB:     /etc/x-ui/x-ui.db
  - X-UI API:    /root/.secrets/xui_api_token.txt (3.7+, Authorization: Bearer ...)
  - Xray Config: /usr/local/x-ui/bin/config.json (auto-generated, don't edit)
  - Certs:       $CERT_DIR/

## Optional: 直连主力 + CF 兜底（进阶，跑通基础后再看）
  references/cf-dns-strategy.md
CFGEOF

chmod 600 /root/vpn-config.txt
cat /root/vpn-config.txt
```

将输出中的 VLESS 链接、面板凭据、SSH 隧道命令整理后展示给用户，并按 `SKILL.md` 第三步提醒用户核对 Cloudflare 配置（如果步骤 15 已经自动配好了，这里改成"确认一下"而不是"手动操作"）。

**给用户的总结里必须单独列出以下四个明文值**，不能只藏在 VLESS 链接或面板里，事后再回头问是不必要的往返：

- **XHTTP path，不带前斜杠**（`$WS_PATH` 本身，如 `1f59afd80fa7f7eb`，即 `/root/.secrets/ws_path.txt` 的原始内容）：链接里是 URL 编码的 `path=%2F...`，Nginx location 和 Xray 的 `xhttpSettings.path` 里才是 `/$WS_PATH`；给用户的总结只输出斜杠后的值
- **UUID**：同理
- **Inbound ID + tag**：后续用 API 加客户端（`addClient` 要传 `id`）、加直连入站、排障定位都按 id。如果部署中删过重建过入站（比如 3.7.0 上手插行被忽略后改走 API），id 不是 1，必须现查
- **面板 basePath**：决定隧道后访问 `http://localhost:54321/` 还是 `http://localhost:54321/<basePath>/`，也决定 API 前缀

现查命令（写进步骤 16 之前跑一次，把值填进 vpn-config.txt）：

```bash
INBOUND_INFO=$(sqlite3 /etc/x-ui/x-ui.db "SELECT id || ' (tag=' || tag || ', remark=' || remark || ')' FROM inbounds WHERE port=10000;")
WEBBASEPATH=$(/usr/local/x-ui/x-ui setting -show true 2>&1 | sed 's/\x1b\[[0-9;]*m//g' | grep -Eo 'webBasePath: .+' | awk '{print $2}')
echo "Inbound: $INBOUND_INFO"; echo "basePath: $WEBBASEPATH"
```

推荐总结格式：

```
XHTTP path:   1f59afd80fa7f7eb    （不带前斜杠；服务器 /root/.secrets/ws_path.txt）
UUID:         xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Inbound ID:   2  (tag=in-10000-tcp, remark=VLESS-XHTTP-TLS-CF)
basePath:     /                    （面板 URL http://localhost:54321/，API 前缀 /panel/api/）
```
