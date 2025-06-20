# 🚀 V2Ray + VLESS + WS + TLS + CDN + Nginx 梯子部署指南

## 一、准备工作

- VPS：一台境外 VPS（推荐 Debian 11+）
- 域名：已解析到 VPS IP，使用 Cloudflare 管理
- Cloudflare 账户：用于开启 CDN 和托管 TLS
- 端口：开放 TCP 80 和 443

## 二、创建 Cloudflare API Token（用于自动签发证书）

- 登录 Cloudflare → 个人资料 → API Tokens → Create Token
- 选择“Edit zone DNS”模板
- 指定你自己的域名作为作用域
- 创建后复制 Token，稍后用

## 三、安装环境

```bash
# 更新系统
apt update && apt upgrade -y

# 安装必要软件
apt install -y nginx curl socat
```

## 四、安装 acme.sh（自动获取证书）

```bash
curl https://get.acme.sh | sh -s email=youremail

# 设置 Cloudflare API Token 环境变量
export CF_Token="你的Cloudflare API Token"

# 申请证书（替换域名）
acme.sh --issue --dns dns_cf -d yourdomain.com --keylength ec-256

# 安装证书到指定目录（自动续期时重载 nginx）
acme.sh --install-cert -d yourdomain.com \
--key-file /etc/nginx/ssl/yourdomain.key \
--fullchain-file /etc/nginx/ssl/yourdomain.crt \
--reloadcmd "systemctl reload nginx"
```

## 五、安装 V2Ray

```bash
bash <(curl -Ls https://github.com/v2fly/fhs-install-v2ray/raw/master/install-release.sh)
```

## 六、配置 V2Ray

编辑 /usr/local/etc/v2ray/config.json：

```bash
{
  "inbounds": [{
    "port": 10000,
    "listen": "127.0.0.1",
    "protocol": "vless",
    "settings": {
      "clients": [
        {
          "id": "你的UUID",
          "flow": ""
        }
      ],
      "decryption": "none"
    },
    "streamSettings": {
      "network": "ws",
      "security": "none",
      "wsSettings": {
        "path": "/raypath"
      }
    }
  }],
  "outbounds": [{
    "protocol": "freedom",
    "settings": {}
  }]
}
```

- UUID：执行命令 uuidgen 生成
- 路径 /raypath 可以自定义，建议复杂点更安全

## 七、配置 Nginx

创建目录存放证书：

```bash
mkdir -p /etc/nginx/ssl
```

编辑 /etc/nginx/sites-available/default：

```bash
server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    ssl_certificate /etc/nginx/ssl/yourdomain.crt;
    ssl_certificate_key /etc/nginx/ssl/yourdomain.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location /raypath {
        proxy_redirect off;
        proxy_pass http://127.0.0.1:10000;
        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        root /var/www/html;
        index index.html index.htm;
    }
}

server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://$host$request_uri;
}
```

## 八、启动服务

```bash
systemctl restart v2ray
systemctl restart nginx
```

## 九、Cloudflare 设置

- 确保域名 DNS 的 A 记录指向 VPS，且代理状态为 橙色云朵（启用代理）
- SSL/TLS 模式选择 Full (strict)
- 可开启防护功能增强安全

## 十、客户端配置示例（V2RayN/V2RayNG）

- 地址：yourdomain.com
- 端口：443
- 用户 ID：你的 UUID
- 传输协议：WebSocket
- 路径：/raypath
- TLS：开启
- 伪装域名：填写 yourdomain.com

## 附：生成 VLESS 链接示例

```bash
vless://你的UUID@yourdomain.com:443?encryption=none&security=tls&type=ws&host=yourdomain.com&path=%2Fraypath#VLESS-CDN
```

## 🔍 为什么 WARP 可以解决 DNS 污染？

Cloudflare WARP 是一种 基于 WireGuard 的加密 VPN/隧道服务，它具有以下几个特性：

| 功能                      | 说明                                     |
| ----------------------- | -------------------------------------- |
| ✅ **全程加密传输**            | 所有数据走 WireGuard 隧道，不会被 GFW 看到明文 DNS 请求 |
| ✅ **使用 Cloudflare DNS** | 使用 `1.1.1.1` 和 `DoH/DoT`，不可被污染或篡改      |
| ✅ **全局出站绕墙能力**          | 某些 VPS 支持 WARP 后访问 Google、YouTube 无需代理 |
| ✅ **对 DNS 污染免疫**        | 不论你本地网络怎么劫持，WARP 直接从隧道出口出访问互联网         |

🚫 但注意：WARP 有两个使用方式，效果不同！

| 使用方式                 | 是否能解决 DNS 污染 | 是否能翻墙 |
| -------------------- | ------------ | ----- |
| ✅ VPS 上装 WARP        | ✔️ 是         | ✔️ 是  |
| ✅ 本机装 WARP 客户端       | ✔️ 是         | ✔️ 是  |
| ⚠️ DNS 仅指向 `1.1.1.1` | ❌ 否（仍可能被劫持）  | ❌ 否   |

✅ 如果你是 VPS 环境，可以这样用 WARP：

1️⃣ 安装 WARP（推荐脚本：WARP-GO 或 warp-cli）

```bash
bash <(curl -fsSL https://git.io/warp.sh)
```
这个脚本可以选择：

- 添加 WARP IPv6 出口
- 添加 WARP IPv4 出口
- 全局流量走 WARP（适合 DNS 污染严重时）

2️⃣ 查看是否成功：

```bash
curl https://www.cloudflare.com/cdn-cgi/trace | grep warp
```

输出应该是：

```bash
warp=on
```

✅ 小结：是否值得用 WARP？

| 场景                           | 建议              |
| ---------------------------- | --------------- |
| DNS 被污染严重、acme.sh 无法申请证书     | **强烈建议启用**      |
| VPS 没有 IPv6，但 WARP 可以提供      | **建议启用**        |
| 想让 VPS 能直接访问 Google / GitHub | **推荐启用**        |
| 本地环境使用（比如 Windows）           | **WARP App 即可** |

✅ 推荐：WARP + cloudflared 联合使用

- WARP 解决出口污染 + 整体墙问题
- cloudflared 解决本地 DNS 加密
