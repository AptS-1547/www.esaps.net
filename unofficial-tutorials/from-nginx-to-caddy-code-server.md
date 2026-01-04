> 本文以部署 code-server 为例，带你体验一下 Caddy 这位自动签 HTTPS 的“懒人福音”，同时对比我之前用 Nginx + acme.sh 的传统组合。全程使用 Docker，仅为复刻部署痛点与爽点。

---

## 一、背景交代：曾经的我也写 Nginx 配置写到想跑路

曾经是一个靠 Google + StackOverflow 生存的普通部署民工，我给我的网站使用的是 Nginx + acme.sh：

```nginx
server {
    listen 443 ssl;
    server_name code.example.com;

    ssl_certificate /etc/ssl/code.crt;
    ssl_certificate_key /etc/ssl/code.key;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

然后每次证书更新、服务重启、莫名其妙 403，我都要怀疑人生一遍。

别误会，我并不讨厌 Nginx，它是个强大又稳定的工具。我甚至还用它配过 [ModSecurity 做 WAF](https://github.com/AptS-1547/nginx-modsecurity "ModSecurity 做 WAF")，用起来稳如老狗。但对于一些轻量级服务，比如 code-server、某些只需要转发 + HTTPS 的容器服务——就显得有些杀鸡用牛刀了。

于是，我试了试 Caddy，~~然后我就没回去过。~~

---

## 二、用 Docker 跑个 code-server（不装插件，先跑起来）

我们先用 Docker Compose 跑一个 code-server，先不考虑 HTTPS，加密之事交给后面。

```yaml
services:
  code-server:
    image: codercom/code-server:latest
    container_name: code-server
    networks:
      - caddy-network
    volumes:
      - ./local:/home/coder/.local:Z
      - ./config:/home/coder/.config:Z
      - ./project:/home/coder/project:Z
    environment:
      - PUID=1000
      - PGID=1000
      - DOCKER_USER=1000
    user: "1000:1000"
    restart: unless-stopped
    # 注意：不暴露端口到主机，只在容器网络内通信

  caddy:
    image: caddy:latest
    container_name: caddy
    networks:
      - caddy-network
    ports:
      - "80:80/tcp"
      - "443:443/tcp"
      - "443:443/udp"     # HTTP/3 协议支持
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    restart: unless-stopped

networks:
  caddy-network:
    driver: bridge

volumes:
  caddy_data:
  caddy_config:
```

现在 code-server 完全不暴露端口了，只有 Caddy 能访问它。虽然安全了一点点，但是，我们怎么能够让 Caddy 转发流量出来呢？

---

## 三、Caddy 上场：自动 HTTPS？真的只要一行!

Caddy 的配置文件叫 `Caddyfile`，长这样：

```
code.example.com {
    reverse_proxy code-server:8080
}
```

你没看错，就这样。

关键点：`code-server:8080` 而不是 `localhost:8080`，因为在 Docker 网络里，服务名就是主机名。这比在宿主机上配置还要干净——不用担心端口冲突，不用记住谁占了哪个端口。

## 四、一键启动：Docker Compose 的快感

```bash
docker compose up -d
```

就这样，一条命令搞定：

* code-server 容器启动，开始监听内部 8080 端口
* Caddy 容器启动，自动申请 `code.example.com` 的 HTTPS 证书
* 证书存储在 Docker volume 里，重启不丢失
* 80 端口自动重定向到 443，HTTPS 强制生效

第一次启动会看到 Caddy 疯狂输出日志，那是它在跟 Let's Encrypt 握手申请证书。稍等片刻，就能愉快地通过 `https://code.example.com` 访问你的远程 VSCode 了。

部署门槛之低，适合手还没从 Nginx 伤痛中恢复过来的开发者。(bushi)

---

## 五、HTTPS 验证方式：Caddy 默认 http-01，有坑提前避

虽然 Caddy 看起来像个“HTTPS 魔法棒”，但其实背后干的还是老三样，只不过它帮你封装得很好。

重点是：Caddy 默认使用的是 **HTTP-01** 验证方式。这意味着：

* 你必须已经将 `code.example.com` 正确解析到服务器公网 IP
* 并且服务器的 **80 端口必须畅通**（Caddy 会自动监听）
* 确保防火墙、云服务商安全组都放行了 80/443

否则 Caddy 就会哭着跟你说：

```bash
[ERROR] obtaining certificate: failed to get certificate: ...
```

相比之下，我之前用的 acme.sh 更像一个命令行工具箱：支持 DNS-01 / HTTP-01 / ALPN-01，各种奇技淫巧随你挑，尤其适合那种不方便开放 80/443 的“地下服务”。

如果你的服务器在某些特殊网络环境（比如内网、NAT 后面），可以考虑 DNS-01 验证。虽然 Caddy 也支持，但配置会复杂一些，需要提供 DNS API 凭据：

```
code.example.com {
    tls {
        dns cloudflare {env.CLOUDFLARE_API_TOKEN}
    }
    reverse_proxy code-server:8080
}
```

不过，这也是选择 Caddy 的 trade-off：易用 vs 灵活。适合场景不同罢了。

---

## 六、Nginx VS Caddy：容器化时代的对比

| 维度         | Nginx + acme.sh          | Caddy + Docker           |
| ---------- | ------------------------ | ------------------------ |
| HTTPS 管理   | 手动配置 + cron 更新 + 容器挂载卷  | 自动申请 + 自动续期 + Volume 持久化 |
| 配置复杂度      | proxy + SSL + 容器网络配置     | 几乎傻瓜式，容器服务名即可            |
| 容器编排友好度    | 需要额外处理证书卷、网络配置           | 天生为容器时代而生                |
| 多服务代理      | upstream 配置相对复杂          | 一个域名一个代码块，清晰明了           |
| 重启后证书丢失风险  | 需要正确挂载 acme.sh 生成的证书    | Docker Volume 自动持久化      |

一句话总结：**Caddy 是你懒得手写配置时的部署队友，Nginx 则是在你需要最大控制权时的可靠选手。**

---

## 七、最终成品结构参考

```
.
├── docker-compose.yml  # 容器编排配置
├── Caddyfile          # Caddy 配置文件
├── local/             # code-server 用户配置
├── config/            # code-server 全局配置
└── project/           # 你的代码项目目录
```

启动命令：

```bash
# 启动所有服务
docker compose up -d

# 查看日志（观察证书申请过程）
docker compose logs -f caddy

# 停止服务
docker compose down
```

证书和配置数据会自动存储在 Docker volumes 里，即使 `docker compose down` 也不会丢失。

搞定，访问 `https://code.example.com`，你的远程开发环境已经热乎上桌了。

---

## 尾声：用什么工具，从来都不是非黑即白

我喜欢 Caddy，但我依然保留着 Nginx 配置仓库；我用 acme.sh 自动签某些非标准服务的证书，也用 Caddy 快速暴露临时 Web 服务。

没有最好的工具，只有最适合的场景。Caddy 的存在，并不是为了替代 Nginx，而是给我们提供一个新的选择：
**当你只想把服务跑起来，不想再为了 HTTPS 而熬夜对着 nginx.conf 发呆时，它就很香。**
