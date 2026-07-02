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
DOMAIN="${SUBDOMAIN_PREFIX}.${ROOT_DOMAIN}"
XUI_PORT="54321"
```

后续所有命令直接使用这些变量，无需额外替换。

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
cat /etc/os-release | head -3  # 必须是 Debian/Ubuntu
ss -tlnp | grep -E ':80 |:443 '  # 检查端口占用
```

如果 80/443 已被占用，提醒用户现有服务可能被覆盖，确认后再继续。

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

> `python3-bcrypt` 用于后续生成面板密码哈希。如果包不存在，后续会 fallback 到 `pip3 install bcrypt --break-system-packages`（较新的 Debian/Ubuntu 默认锁了系统 Python 环境，直接 `pip3 install` 可能报 `externally-managed-environment` 错误，需要加这个 flag）。

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
~/.acme.sh/acme.sh --issue -d "$ROOT_DOMAIN" -d "*.$ROOT_DOMAIN" --dns dns_cf --keylength ec-256

mkdir -p /root/cert
~/.acme.sh/acme.sh --install-cert -d "$ROOT_DOMAIN" \
  --cert-file "/root/cert/$ROOT_DOMAIN.cer" \
  --key-file "/root/cert/$ROOT_DOMAIN.key" \
  --fullchain-file /root/cert/fullchain.cer \
  --ca-file /root/cert/ca.cer \
  --reloadcmd "systemctl reload nginx"

chmod 600 /root/cert/*.key
```

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

## 步骤 7：配置 Nginx 站点

```bash
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

    ssl_certificate /root/cert/fullchain.cer;
    ssl_certificate_key /root/cert/$ROOT_DOMAIN.key;
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

> heredoc 不带引号，`$DOMAIN`、`$ROOT_DOMAIN`、`$WS_PATH` 会被 bash 展开为实际值。Nginx 变量（`$server_name`、`$host` 等）用 `\$` 转义以保留。

启用配置：

```bash
ln -sf "/etc/nginx/sites-available/$DOMAIN" /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default
nginx -t && systemctl reload nginx
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

## 步骤 10：安装 3X-UI

```bash
bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh)
```

> **交互处理（不同版本流程可能不同，实战踩过坑）**：早期版本的安装器依次要求输入"是否继续安装 → 用户名 → 密码 → 端口"四项，可以用 `echo -e "y\ntemp\ntemp\n54321" | bash <(curl ...)` 一次性喂完。**较新版本的安装器多了"数据库选择（SQLite/PostgreSQL）"、"Base URI Path"、"SSL 证书设置（Let's Encrypt / 自定义 / 跳过）"等额外交互项，顺序和数量都可能变化**；而且检测到脚本是非交互方式运行（没有 controlling tty）时，有的版本会直接跳过所有输入项、自动生成随机用户名/密码/端口，忽略你喂给它的 stdin。
>
> **不管装的时候到底问了什么、给了什么答案都不用纠结**——反正步骤 11 会通过数据库把用户名、密码、端口全部强制覆盖成我们自己生成的值。这一步的唯一目标就是让安装器跑完、把 `/etc/x-ui/x-ui.db` 建出来。
>
> 使用 **MHSanaei/3x-ui**，不要用已停维的 vaxilu/x-ui。

等待安装完成，确认数据库文件存在，并顺手记录一下装的是什么版本（后面步骤 12/13 要根据版本判断 schema）：

```bash
ls -la /etc/x-ui/x-ui.db
x-ui --version 2>&1 || (command -v x-ui >/dev/null && x-ui | head -5)
```

## 步骤 11：配置 3X-UI 面板

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
  \"api\": {\"tag\": \"api\", \"services\": [\"HandlerService\", \"LoggerService\", \"StatsService\"]},
  \"stats\": {}
}');"
```

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

### 先插入 inbound（两种 schema 通用，字段按实际 schema 增减）

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

### 启动并验证 clients 真的生效了（不要跳过这步）

```bash
systemctl start x-ui
sleep 3

# 关键验证：不管走的哪种 schema，最终都要在运行配置里看到真实的 clients，而不是 null
python3 -c "
import json
cfg = json.load(open('/usr/local/x-ui/bin/config.json'))
for ib in cfg.get('inbounds', []):
    print(ib.get('tag'), json.dumps(ib.get('settings')))
"
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
ufw status

# SSL 证书有效期
openssl x509 -in /root/cert/fullchain.cer -noout -enddate
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

配完等 10-20 秒 DNS 生效，然后验证：

```bash
curl -s -o /dev/null -w "HTTPS via CF: %{http_code}\n" --max-time 15 "https://$DOMAIN/"
```

如果这里超时或连不上，且服务器本地 `curl localhost` 正常，优先怀疑云平台安全组/NSG 没放行 80/443（见步骤 8 的说明）。

## 步骤 16：保存配置并输出结果

```bash
SERVER_IP=$(curl -s --max-time 10 ifconfig.me)

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
  Username:   $XUI_USER
  Password:   $XUI_PASS

## Adding New Clients
  Do NOT create new inbounds! Add clients to existing inbound:
  Panel → Inbound list → VLESS-XHTTP-TLS-CF → "+" → Add Client
  Then generate link using this template:
  vless://<UUID>@$DOMAIN:443?encryption=none&security=tls&sni=$DOMAIN&type=xhttp&host=$DOMAIN&path=%2F$WS_PATH&fp=chrome#<NAME>
  WARNING: Panel-exported links have WRONG port, TLS, and transport settings!

## Security
  - UFW: only $SSH_PORT/80/443 (+ any pre-existing service ports) open
  - X-UI: localhost only (127.0.0.1:$XUI_PORT)
  - Xray: localhost only (127.0.0.1:10000)
  - Fail2Ban: SSH $SSH_PORT, 3 attempts → 2hr ban
  - Subscription port: disabled
  - Swap: enabled as a memory-pressure safety net

## Key Files
  - Secrets:     /root/.secrets/
  - Nginx Site:  /etc/nginx/sites-available/$DOMAIN
  - Fail2Ban:    /etc/fail2ban/jail.local
  - X-UI DB:     /etc/x-ui/x-ui.db
  - Xray Config: /usr/local/x-ui/bin/config.json (auto-generated, don't edit)
  - Certs:       /root/cert/

## Optional: 直连主力 + CF 兜底（进阶，跑通基础后再看）
  references/cf-dns-strategy.md
CFGEOF

chmod 600 /root/vpn-config.txt
cat /root/vpn-config.txt
```

将输出中的 VLESS 链接、面板凭据、SSH 隧道命令整理后展示给用户，并按 `SKILL.md` 第三步提醒用户核对 Cloudflare 配置（如果步骤 15 已经自动配好了，这里改成"确认一下"而不是"手动操作"）。
