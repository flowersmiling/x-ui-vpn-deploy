# 故障排查

## 核心原则：先查客户端，再查服务端

大多数"连不上"的问题出在客户端配置（链接断行、缺参数），不要急着改服务端。

## 整体故障定位

以伪装站点是否能访问作为分水岭：

| 症状 | 大概率原因 | 下一步 |
|---|---|---|
| 客户端连不上，但浏览器能打开 `https://{DOMAIN}` | VPN 链路问题（path / UUID / 客户端配置 / clients 表没生效） | 走下方"客户端"和"服务端"两步排查 |
| 客户端连不上，伪装站也打不开 | VPS 故障 / IP 被封 / 防火墙改动 / 云平台安全组没放行 | 去 VPS 控制台看实例状态、流量上限、最近改动 |
| 所有网站都打不开，服务状态却全是 active | 大概率是 clients 表没生效，见下方"客户端连不上但一切看起来正常" | 查 Xray 详细访问日志确认 |
| 只有个别网站打不开，其他正常 | 客户端代理模式漏了 UDP/DNS，不是服务器问题 | 见下方"个别网站打不开" |
| 之前部署的其他服务（如 Outline）突然连不上 | 本次部署改了 UFW，把已有服务端口封了 | 见下方"部署完后已有服务断了" |
| VPN 突然整体失联，SSH 也进不去 | 服务器资源耗尽（常见是内存 + 无 swap） | 见下方"服务器整体失联" |

> 如果你按 `cf-dns-strategy.md` 配了直连 + CF 兜底双节点，节点切换是 5 秒内生效的客户端单边操作，先切再排查。

---

## 第一步：检查客户端链接

按以下清单逐项排查：

- [ ] 链接是否完整一行？（复制时最容易在 path 参数处断行）
- [ ] 端口是否为 `443`？（不是内部端口 10000）
- [ ] `security=tls` 是否存在？
- [ ] `sni=子域名` 是否存在？
- [ ] `path` 中间有没有换行、空格、乱码？
- [ ] `host` 是否设置为子域名？

---

## 第二步：检查服务端

### 快速诊断表

| 问题 | 检查命令 | 解决方案 |
|------|----------|----------|
| Xray 没监听 | `ss -tlnp \| grep 10000` | `systemctl restart x-ui` |
| outbounds 为 null | 检查 config.json（见下方） | 重设 xrayTemplateConfig |
| **clients 为 null（服务全 active 但连不上，实战踩过）** | 检查 config.json 里 inbound 的 `settings.clients` | 见下方专门章节 |
| 502 Bad Gateway | `systemctl status x-ui` | 重启 X-UI |
| SSL 错误 | `openssl x509 -in /root/cert/fullchain.cer -noout -enddate` | 续期证书 |
| XHTTP 不通 | curl 测试连通性（见下方） | 检查 Nginx location 配置 |
| Nginx 配置错误 | `nginx -t` | 根据错误信息修复 |
| 防火墙问题 | `ufw status` | 确认 443 端口开放，以及云平台安全组/NSG |
| 面板配置不同步 | 对比数据库和 config.json | `systemctl restart x-ui` |
| 证书申请报 DNS identifier invalid | 域名是否为公共后缀（PSL） | 见下方"根域名是公共后缀" |

### 详细诊断命令

```bash
# 1. 服务状态
systemctl is-active nginx x-ui fail2ban

# 2. 端口监听
ss -tlnp | grep -E ':80 |:443 |:10000 |:54321 '

# 3. Xray 配置完整性（outbounds 不能为 null，inbound 的 clients 也不能为 null）
python3 -c "
import json
cfg = json.load(open('/usr/local/x-ui/bin/config.json'))
print('Outbounds:', cfg.get('outbounds'))
print('DNS:', cfg.get('dns'))
for ib in cfg.get('inbounds', []):
    print('Inbound', ib.get('tag'), 'clients:', ib.get('settings', {}).get('clients'))
"

# 4. XHTTP 连通性测试（返回非 502/503 = Xray 在响应；404 也算正常，说明是路径命中但协议校验没过）
WS_PATH=$(cat /root/.secrets/ws_path.txt)
curl -s -o /dev/null -w '%{http_code}' \
  -X POST http://127.0.0.1:10000/$WS_PATH

# 5. 服务器出网测试
curl -s -o /dev/null -w '%{http_code}' --max-time 5 https://www.google.com

# 6. 日志
tail -30 /var/log/nginx/error.log
journalctl -u x-ui --since "30 min ago" --no-pager
```

---

## 常见踩坑经验

### 1. xrayTemplateConfig 不完整 → outbounds=null

**症状**：客户端显示已连接，但打不开任何网页。

**原因**：3x-ui 用数据库中 `xrayTemplateConfig` 生成 Xray 运行配置，不完整的模板导致缺失字段被设为 null。

**修复**：重新设置完整的模板（参考 `manual-deploy.md` 第 12 步），模板必须至少包含：`dns`、`routing`、`outbounds`（direct + blocked）、`policy`、`api`、`stats`。

### 2. clients 为 null → 所有连接被拒绝（实战真实翻车过，最隐蔽的一类坑）

**症状**：`systemctl is-active nginx x-ui` 全部 active，端口监听正常，Nginx 访问日志显示请求一直在到达（大量 200），伪装站能正常打开——但客户端（VLESS）就是完全连不上任何网站，无论 CF 节点还是直连节点都一样。**因为常规检查全部通过，这个状态很容易被误判为"部署成功、问题在别处"**（比如误诊断成 GFW 封锁、地理位置限制、CDN 延迟等），排查方向完全走偏。

**根因**：3x-ui 较新版本把客户端数据从 `inbounds.settings` 内嵌 JSON 挪到了独立的 `clients` + `client_inbounds` 两张表，运行时生成 config.json 只认新表。如果部署时是照着旧版文档直接把 client 塞进 `inbounds.settings` 的 JSON 里，新版面板根本不读这个字段，生成的运行配置里 `clients` 是 `null`。

**确诊方法**：

```bash
# 看运行配置，clients 是不是 null
python3 -c "
import json
cfg = json.load(open('/usr/local/x-ui/bin/config.json'))
for ib in cfg.get('inbounds', []):
    print(ib.get('tag'), json.dumps(ib.get('settings')))
"

# 临时开详细访问日志确诊（正常排查完记得关掉，见下方"临时开访问日志"）
# 如果看到大量这样的记录，就是这个问题：
# rejected proxy/vless/encoding: invalid request user id: <你的UUID>
tail -50 /var/log/x-ui/xray-access.log
```

**修复**：

```bash
sqlite3 /etc/x-ui/x-ui.db ".schema clients"  # 确认这张表存在，说明是新版 schema

INBOUND_ID=$(sqlite3 /etc/x-ui/x-ui.db "SELECT id FROM inbounds WHERE tag='inbound-10000';")
UUID=$(cat /root/.secrets/vless_uuid.txt)
NOW=$(date +%s%3N)

sqlite3 /etc/x-ui/x-ui.db "INSERT INTO clients (email, uuid, flow, limit_ip, total_gb, expiry_time, enable, reset, created_at, updated_at) VALUES ('default-user', '$UUID', '', 0, 0, 0, 1, 0, $NOW, $NOW);"
CLIENT_ID=$(sqlite3 /etc/x-ui/x-ui.db "SELECT id FROM clients WHERE uuid='$UUID';")
sqlite3 /etc/x-ui/x-ui.db "INSERT INTO client_inbounds (client_id, inbound_id, created_at) VALUES ($CLIENT_ID, $INBOUND_ID, $NOW);"

systemctl restart x-ui
sleep 3
# 重新确认 clients 不再是 null
```

**预防**：任何时候照抄本文档或旧的部署记录之前，先跑一遍 `.schema inbounds` / `.schema clients` 核实当前版本的实际表结构，不要假设面板版本和文档写的时候一样。

### 3. 根域名是 Public Suffix List 上的公共后缀 → 证书申请失败

**症状**：`acme.sh --issue` 报 `Error creating new order... DNS identifier is invalid [域名]`。

**原因**：用户提供的"根域名"实际是类似 `cc.cd`、`co.cc` 这种免费二级域名分发服务，本身在 Public Suffix List 里，CA 拒绝为公共后缀签发证书。

**确诊**：

```bash
curl -s https://publicsuffix.org/list/public_suffix_list.dat | grep -x "$ROOT_DOMAIN"
```

有输出就是命中了。

**修复**：查用户实际拥有并托管在 Cloudflare 的完整子域名（一般是 `子域名.根域名` 整体）：

```bash
curl -s "https://dns.google/resolve?name=子域名.根域名&type=NS"
```

如果 NS 指向 `*.ns.cloudflare.com`，把这个完整子域名当成新的 `ROOT_DOMAIN` 重新走一遍证书申请（不再额外加前缀），后续 Nginx/Xray 配置同步用这个域名。

### 4. 云平台安全组/NSG 没放行端口 → UFW 配置正确但外网连不上

**症状**：服务器本地 `curl localhost` 正常，`ufw status` 显示 80/443 都 ALLOW，但从外网（或者本机之外的任何地方）连接服务器 IP 的 80/443 全部超时。

**原因**：Azure/AWS/GCP 等云平台在 UFW 之外还有一层独立的网络级防火墙（Azure「网络安全组 NSG」、AWS「Security Group」、GCP「防火墙规则」），默认往往只放行 22，80/443 需要额外手动添加入站规则。

**修复**：去对应云平台的网页控制台，找到这台虚拟机关联的安全组/NSG，添加 TCP 80、TCP 443（以及任何其他要用的端口）的入站放行规则。这一步没法通过 SSH 命令完成。

### 5. 个别网站打不开（尤其 Google/YouTube 一类），其他网站正常

**症状**：VPN 整体是通的，能访问大部分网站，唯独 Google、YouTube，或者某些走复杂 CNAME/Traffic Manager 解析链的网站（比如某些机构的 SSO 登录页）连不上，报"响应超时"。

**原因（两类，都是客户端"系统代理"模式的固有缺陷，不是服务器问题）**：

1. **QUIC/HTTP3 走 UDP，绕过只接管 TCP 的系统代理**：Chrome/Edge 对 Google/YouTube 默认走 HTTP/3。排查：`chrome://flags/#enable-quic` 设为 Disabled 测试。
2. **DNS 解析在本地完成，没走代理隧道**：对于走多层 CNAME/Traffic Manager 解析的域名，本地 DNS 容易失败/超时/被污染，浏览器在还没发起代理连接前就已经报错——**服务器端 Xray 的访问日志里完全看不到这个域名的任何记录**（不是被拒绝，是请求根本没到）。即使客户端切到"全局"路由模式问题依旧，基本可以排除路由规则，确认是 DNS 泄漏。

**确诊**：开服务端详细访问日志（见下方），让用户实时重现问题，同时观察日志里有没有出现对应域名的记录。完全没有记录 → 客户端 DNS/UDP 绕过代理；有记录但连接失败 → 再看是不是目标网站本身的问题。

**修复**：客户端切换成 **TUN 模式**（而不是系统代理模式），在系统网络层接管全部流量（含 UDP、DNS），一次性解决以上两类问题。TUN 模式需要管理员权限，可以用 Windows 任务计划程序做成"以最高权限运行"的任务，避免每次手动右键。

### 6. 部署完后，服务器上已有的其他服务（如 Outline）突然连不上

**症状**：这台服务器之前已经在用别的代理服务（Outline/Shadowsocks 等），部署完这个 VLESS 节点之后，原来的服务反而连不上了。

**原因**：本次部署的第 8 步把 UFW 改成了"默认拒绝所有入站，只放行 22/80/443"，已有服务用的端口没有额外放行，被一并封死。

**确诊**：

```bash
ufw status numbered
# 对照已有服务实际监听的端口，看是不是没在放行列表里
ss -tlnp
docker ps -a   # 如果是容器化部署（比如 Outline 的 shadowbox）
```

Outline 的实际端口可以从它的配置文件里读：

```bash
cat /opt/outline/persisted-state/shadowbox_server_config.json
# apiUrl 里的端口号是管理 API 端口
# portForNewAccessKeys 是新建密钥默认使用的数据端口
```

**修复**：把发现的端口加入 UFW 放行：

```bash
ufw allow <端口>/tcp comment "outline-xxx"
ufw allow <端口>/udp comment "outline-xxx"   # Shadowsocks 数据端口通常 TCP+UDP 都要放
ufw status numbered
```

**预防**：部署前第一步系统检查时，先 `ss -tlnp` + `docker ps -a` 摸一遍已有服务，做完防火墙那一步之前心里有数要放行哪些额外端口，而不是等用户反馈"别的服务坏了"才回头补。

### 7. 服务器整体失联：SSH、云平台管理通道都进不去，但网站还能访问

**症状**：VLESS/Outline 突然全部连不上，SSH 连接卡在"等待服务器返回版本号"（banner exchange）不动，云平台的 Run Command/VM Agent 也没有响应或一直转圈，但 Nginx 的伪装站首页居然还能勉强打开（可能比平时慢）。

**原因**：服务器资源（通常是内存）耗尽或严重不足，导致新进程几乎无法创建/响应——sshd 能完成 TCP 三次握手，但后续需要 fork 新进程处理登录的步骤卡死；云平台 VM Agent 同理。已经在运行、不需要新建进程的服务（比如 nginx 的事件循环处理已建立好的静态内容请求）还能勉强响应一阵子。这种情况在**内存较小又没配置 swap 的 VPS**上尤其容易出现——没有 swap 就没有缓冲，内存压力一上来直接从"正常"跳到"几乎失联"，中间没有一个能让人察觉的过渡阶段。

**排查升级路径**（从轻到重）：

1. 先用 SSH 常规方式连接，等待 15-30 秒看是否只是慢而不是完全卡死
2. 用云平台的 **Run Command**（走管理平面，不走 SSH）执行诊断脚本，看是否也无响应
3. 用云平台的 **串行控制台（Serial Console）**——这个走虚拟串口，完全独立于网络和 VM Agent，只要内核没死就该有反应。能看到 `login:` 提示但登录本身也超时，说明内核层面已经严重卡死（可能是磁盘 I/O 死锁），基本只能重启
4. 以上都不行，走云平台的 **重启（Restart）**；如果重启按钮不可用/无响应，用 **停止（Stop）再启动（Start）**，效果等同，数据不会丢（只有临时磁盘会清空）

**恢复后必做**：

```bash
free -h        # 确认内存和 swap 情况
df -h /        # 排除磁盘写满的可能性
journalctl --list-boots           # 看有几次开机记录
journalctl -b -1 --no-pager | grep -i -E "out of memory|oom.kill"   # 查上一次开机（崩溃前）有没有 OOM 记录
```

**预防**：给内存偏小（尤其 <2GB）又没有 swap 的实例加 1-2GB swap（见 `manual-deploy.md` 步骤 2.5），把"硬失联"变成"降级但还能进去排查"。同时检查有没有排障时开着忘记关的高频写日志（比如临时开启的 Xray 详细访问日志），长期占用额外 I/O 也是潜在的雪上加霜因素。

---

## 临时开启 Xray 详细访问日志（排障用，用完记得关）

平时 `xrayTemplateConfig` 里 `log.access` 是 `"none"`，看不到每条连接的目标地址和结果。怀疑连接被拒绝、或者要确认某个域名的流量到底有没有进隧道时，临时开启：

```bash
systemctl stop x-ui
python3 -c "
import sqlite3, json
conn = sqlite3.connect('/etc/x-ui/x-ui.db')
c = conn.cursor()
c.execute('SELECT value FROM settings WHERE key=?', ('xrayTemplateConfig',))
cfg = json.loads(c.fetchone()[0])
cfg['log']['access'] = '/var/log/x-ui/xray-access.log'
cfg['log']['loglevel'] = 'info'
c.execute('UPDATE settings SET value=? WHERE key=?', (json.dumps(cfg), 'xrayTemplateConfig'))
conn.commit()
print('enabled')
"
systemctl start x-ui

# 实时看
tail -f /var/log/x-ui/xray-access.log
```

看的时候注意：`log.access` 这个字段的值不一定是你写的原始字符串，某些版本会把日志目录规整到自己的默认位置（比如统一放进 `/var/log/x-ui/` 下），如果按你设的路径 `tail` 不到内容，先确认一下面板实际用的路径（可以从运行配置 `config.json` 里的 `log.access` 字段确认，或者直接 `ls -la /var/log/x-ui/` 看看有没有生成新文件）。

**排障结束后一定要关掉**，改回 `"none"` 并 `systemctl restart x-ui`：

```bash
systemctl stop x-ui
python3 -c "
import sqlite3, json
conn = sqlite3.connect('/etc/x-ui/x-ui.db')
c = conn.cursor()
c.execute('SELECT value FROM settings WHERE key=?', ('xrayTemplateConfig',))
cfg = json.loads(c.fetchone()[0])
cfg['log']['access'] = 'none'
cfg['log']['loglevel'] = 'warning'
c.execute('UPDATE settings SET value=? WHERE key=?', (json.dumps(cfg), 'xrayTemplateConfig'))
conn.commit()
"
systemctl start x-ui
```

---

## 其他常见踩坑经验

### 3x-ui 默认开启订阅端口

**症状**：`ss -tlnp` 发现公网上多了一个意外端口（如 2096）。

**修复**：
```bash
sqlite3 /etc/x-ui/x-ui.db "INSERT OR REPLACE INTO settings (key, value) VALUES ('subEnable', 'false');"
sqlite3 /etc/x-ui/x-ui.db "INSERT OR REPLACE INTO settings (key, value) VALUES ('subListen', '127.0.0.1');"
systemctl restart x-ui
```

### 面板导出的 VLESS 链接不能直接用

面板显示的是 Xray 内部配置（端口 10000、TLS 关闭），不适用于 Cloudflare CDN 架构。必须手动拼接链接：端口改 443、加 `security=tls`、加 `sni`。

### 客户端链接复制断行

VLESS 链接很长，复制时经常在 path 参数处断行，导致路径中间出现 `\n` 或空格。这是最常见的"连不上"原因。

### Xray 运行配置与面板不同步

在面板里删除/修改入站后，`/usr/local/x-ui/bin/config.json` 可能没有及时更新。不确定时执行 `systemctl restart x-ui` 强制重新生成。

### 传输协议演进

本 skill 已使用 XHTTP 替代 WebSocket（Xray 官方推荐的迁移方向）。XHTTP 分片成多个短 HTTP 请求，流量特征更像正常网页浏览，抗检测更强，弱网环境下也更稳定。

### settings 表 INSERT OR REPLACE 产生重复行

新版 3x-ui 的 `settings` 表没有 UNIQUE 约束，`INSERT OR REPLACE` 遇到已存在的 key 不会真正"替换"，而是插入新的一行，导致同一个 key 有多行不同的值，面板/Xray 读到哪一行不确定。改完任何 `settings` 表的值之后，习惯性检查一下：

```bash
sqlite3 /etc/x-ui/x-ui.db "SELECT key, COUNT(*) c FROM settings GROUP BY key HAVING c > 1;"
```

有重复就手动删掉多余的行，只保留期望的那一条。
