> "你，有没有遇到过这种情况：Docker Compose 文件越写越多，容器之间却 ping 不通？"
> ——AptS:1547（盯着 `Connection refused` 的报错发呆）

这是我两年前踩的最大的坑。

那时候我刚学会 Docker Compose，兴冲冲地给每个服务都建了个独立的文件夹：Nginx 一个、Cloudreve 一个、MySQL 一个……看起来很整洁，管理起来也方便。

然后我想让 Nginx 反向代理到 Cloudreve，结果：

```bash
$ docker exec nginx ping cloudreve
ping: cloudreve: Name or service not known
```

**不通。**

我以为是防火墙问题，关了，还是不通。
我以为是 DNS 问题，改了，还是不通。
我甚至怀疑是网卡坏了。

**直到我在 Docker 文档的某个角落看到这句话：**

> "Each Docker Compose project creates its own default network."

**每个 Compose 文件都有自己的独立网络。**

这个坑，我啃了一个星期。

但解决之后，整个架构就通了。从那之后，我的服务器慢慢从几个容器发展到现在的 20+ 个：Nginx 网关、WAF、SSL 自动化、监控链、Minecraft 集群、QQ 机器人……

这篇文章，我就来聊聊这两年的 Docker 折腾史，重点讲讲**跨 Compose 网络联通的优雅解决方案**，以及我是怎么一步步把架构打磨成现在这个样子的。

---

## 一、时间线：从"能跑就行"到"能打能抗"

### v0.x：docker run 一把梭时代（高中）

刚开始接触 Docker 的时候，我只知道一个命令：`docker run`。

想跑个 Nginx？

```bash
docker run -d -p 80:80 nginx
```

想跑个 MySQL？

```bash
docker run -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql
```

**能跑就行。** 配置？数据持久化？网络规划？不存在的。

当时我的服务器状态大概是这样的：
- 10 个容器，每个都是独立的 `docker run` 命令启动的
- 端口映射从 8080 一路排到 8090，我自己都记不清哪个端口对应哪个服务
- 容器重启之后配置全丢，得重新 `docker run`
- 想改点配置？先把容器删了，改完再重新 run

典型的"能跑就行"派。

直到有一天，我手抖执行了一条 `docker system prune -a`，所有容器和镜像全没了。

我盯着空荡荡的 `docker ps` 输出，沉默了五分钟。

**然后开始重新学习 Docker。**

---

### v1.0：单文件维护地狱（高中后期）

重新学习之后，我发现了 Docker Compose 这个好东西。

什么，我可以把容器的配置写在 YAML 文件里？下次直接 `docker-compose up -d` 就能恢复？

**真香诶，真香**

于是我把所有服务都写进了一个 `docker-compose.yml`：

```yaml
# /opt/docker_file/docker-compose.yml
services:
  nginx:
    image: nginx
    ports:
      - 80:80
    # ...

  mysql:
    image: mysql
    # ...

  cloudreve:
    image: cloudreve/cloudreve
    # ...

  minecraft:
    image: itzg/minecraft-server
    # ...

  # 再加 10 个服务...
```

看起来不错？所有服务都在一个文件里，网络也自动联通了。

**但很快问题就来了：**

- 单个文件 800+ 行，上下翻到手酸
- 改一个服务的配置，整个文件都要重新加载
- 重启一个服务？不好意思，`docker-compose restart` 会影响所有服务
- 想给同学看看我的 Minecraft 配置？给他发整个文件？

**维护地狱。**

---

### v1.1：开始分离，手动 network connect

我意识到不能再这样下去了。

**于是开始拆分：每个服务一个独立的文件夹。**

```
/opt/docker_file/
├── nginx/
│   └── docker-compose.yml
├── mysql/
│   └── docker-compose.yml
├── cloudreve/
│   └── docker-compose.yml
└── minecraft/
    └── docker-compose.yml
```

看起来很规整，每个服务独立管理了。

**但是问题来了。**

我想让 Nginx 反向代理到 Cloudreve，结果发现 **两个容器根本 ping 不通**。

经过一番搜索，我发现了问题：每个 Compose 文件都会创建自己的默认网络，所以容器之间是隔离的。

**那时候我的解决方案是：手动 `docker network connect`。**

```bash
# 先创建一个网络
docker network create server_for_public

# 启动各个服务
cd /opt/docker_file/nginx && docker-compose up -d
cd /opt/docker_file/cloudreve && docker-compose up -d

# 手动把容器连接到同一个网络
docker network connect server_for_public nginx
docker network connect server_for_public cloudreve
```

这样确实能通了，但是：
- 每次添加新服务都要手动 `network connect`
- 容器重启后网络连接有时候会丢
- 服务器重启后得重新手动连一遍

**能用，但很蠢。**

我知道肯定有更优雅的方案，但当时找不到。

---

### v2.0：顿悟 - external 方案（高三）

某天晚上，我又一次盯着 Docker 文档，突然看到了一个细节：

**可以在一个 Compose 文件里"定义"网络，在其他 Compose 文件里声明这个网络是 `external: true`（已存在）。**

灵光一闪。

**不需要手动创建网络，而是让一个服务（比如 Nginx）负责创建网络，其他服务声明这个网络是 external。**

具体来说：

```yaml
# nginx/docker-compose.yml
networks:
  server_for_public:
    name: server_for_public
    driver: bridge

# cloudreve/docker-compose.yml
networks:
  server_for_public:
    external: true  # 声明这个网络已存在
```

这样，网络的生命周期绑定到 Nginx：
- Nginx 启动 → 网络自动创建
- 其他服务启动 → 连接已存在的网络
- 不需要手动 `docker network create`

**这个方案一出，整个架构就通了。**

高三那段时间，我的服务器终于稳定了下来。所有服务都能正常通信，再也不用手动 `network connect` 了。

**v2.0 标志着架构的质变：从"手动维护"到"声明式管理"。**

---

### v2.1：重构与精细化（大一）

上了大一之后，我开始觉得之前的架构虽然能用，但还不够"优雅"。

首先是网络名字：`server_for_public` 这个名字太泛了，看不出来是干什么的。

**于是我把网络改名为 `nginx-proxy-network`。**

改名的理由很简单：一看就知道这是 Nginx 反向代理用的网络，职责明确。以后新加服务的时候，看到这个名字就知道该连哪个网络，不用再翻文档猜测。`server_for_public` 虽然也能用，但太泛了，像是"公共厕所"一样，什么都能往里塞。

其次，我发现有些服务不应该暴露到 `nginx-proxy-network`。比如 Redis 和 MySQL 这种数据库，它们只需要被特定的应用访问，没必要让 Nginx 也能访问到。还有一些内部组件，比如 Draw.io 的 image-export 服务，也完全不需要对外暴露。

**于是我引入了多网络隔离策略：**

```yaml
# cloudreve/docker-compose.yml
services:
  cloudreve:
    networks:
      - nginx-proxy-network  # 对外：被 Nginx 访问
      - cloudreve-internal   # 对内：访问 Redis

  redis:
    networks:
      - cloudreve-internal  # 只连接内部网络

networks:
  cloudreve-internal:
    driver: bridge
  nginx-proxy-network:
    external: true
```

这样，即使攻击者突破了 Nginx，也无法直接访问 Redis。

**还有数据库的隔离网络 `infra-net`（Driver: none）**，用于更底层的基础设施通信。

第三个重构点是**目录结构的规范化**。

之前的目录比较随意：

```
/opt/docker_file/
├── nginx/
│   └── docker-compose.yml
├── cloudreve/
│   └── docker-compose.yml
└── ...
```

现在变成了这样：

```
/opt/docker_file/
├── gateway/                    # 网关层
│   ├── docker-compose.yml
│   ├── nginx_website/
│   │   ├── config/            # Nginx 配置
│   │   └── website_file/      # 静态文件
│   └── ...
├── acme.sh/                    # SSL 证书管理
│   ├── docker-compose.yml
│   └── ssl/
├── cloudreve/                  # 应用层
│   ├── docker-compose.yml
│   └── cloudreve_data/        # 数据持久化
├── uptimekuma/
│   ├── docker-compose.yml
│   └── uptime-kuma-data/
├── podman/                     # 编排层
│   ├── minecraft/
│   ├── maim-bot/
│   └── ...
└── ...
```

规范化之后，整个目录结构变得清晰多了。
网关层、应用层、编排层按照职责分开，一眼就能看出来每个服务属于哪一层。
所有数据目录都统一命名成 `service_name_data/`，不用再猜"这个数据到底放哪了"。
而且备份也方便了，直接 `tar czf backup.tar.gz /opt/docker_file/` 打包整个目录就行，不用到处找文件。

从那之后，架构逐渐成型：
- **网关层**：Nginx + ModSecurity WAF + ACME.sh
- **应用层**：各种 Web 服务，连接 `nginx-proxy-network`
- **编排层**：Podman 管理的复杂应用集群
- **网络隔离**：外部网络 + 多个内部网络，最小权限原则

两年时间，从 1 个容器到 20+ 个容器，从"能跑就行"到"能打能抗"。

---

## 二、史诗级大坑：跨 Compose 文件之间的网络联通

这个坑是我在整个 Docker 学习过程中踩过的**最大的坑**，也是让我真正理解 Docker 网络机制的契机。

### 问题：为什么不同 Compose 文件的容器互相不通？

当你运行 `docker-compose up -d` 时，Docker Compose 会自动为你创建一个默认网络，命名规则是 `{项目名}_default`。

比如：
- `/opt/docker_file/nginx/docker-compose.yml` 会创建 `nginx_default` 网络
- `/opt/docker_file/cloudreve/docker-compose.yml` 会创建 `cloudreve_default` 网络

**这两个网络是完全隔离的。**

所以，即使两个容器在同一台物理服务器上，它们也无法互相访问，就像你家的 Wi-Fi 和邻居家的 Wi-Fi 一样。

### 为什么会这么难？

总结一下前面踩过的坑：

- **单文件方案（v1.0）**：网络是通了，但维护是地狱
- **手动 connect（v1.1）**：能用，但每次都要手动操作，容易出错
- **`--link` 等远古方案**：早就废弃了，不提也罢

这些方案要么牺牲可维护性，要么需要手动干预，都不够优雅。

**那么，有没有一种方案，既能独立管理每个服务，又能自动管理网络联通？**

有。

---

### 最终方案：让一个 Compose 负责创建网络

经过无数次试错，我找到了一个优雅的解决方案：

**让 Nginx（网关）负责创建网络，其他服务声明这个网络为 external。**

#### Nginx 的 docker-compose.yml

```yaml
# /opt/docker_file/website/docker-compose.yml
services:
  nginx-website:
    container_name: nginx-website
    image: e1saps/nginx-modsecurity:stable
    restart: always
    ports:
      - "443:443/tcp"
      - "443:443/udp"
      - "80:80/tcp"
    networks:
      - nginx-proxy-network
    # ...其他配置

networks:
  nginx-proxy-network:
    name: nginx-proxy-network
    driver: bridge
```

**关键点**：
- `networks` 块里**定义**了 `nginx-proxy-network`
- 这个网络会在 Nginx 容器启动时自动创建

#### 其他服务的 docker-compose.yml

```yaml
# /opt/docker_file/cloudreve/docker-compose.yml
services:
  cloudreve:
    container_name: cloudreve
    image: cloudreve/cloudreve:latest
    restart: always
    networks:
      - nginx-proxy-network
      - cloudreve-internal  # 内部网络，用于隔离
    # ...其他配置

networks:
  cloudreve-internal:  # 内部网络，不需要 external
  nginx-proxy-network:
    external: true  # 声明这个网络已经存在
```

**关键点**：
- `nginx-proxy-network` 声明为 `external: true`
- Docker Compose 不会尝试创建这个网络，而是去连接已存在的网络
- 如果网络不存在，启动会报错

---

### 这个方案的优势

1. **网络生命周期绑定到 Nginx**
   - Nginx 启动 → 网络创建
   - Nginx 停止 → 网络可以安全删除（如果没有其他容器在用）
   - 语义清晰：Nginx 是网关，它"拥有"这个网络

2. **其他服务独立启停**
   - 只要 Nginx 在运行，其他服务随便启停
   - 不需要关心网络是否存在，反正 Nginx 已经创建好了

3. **配置清晰**
   - 一眼就能看出哪个服务是"网络主人"，哪些是"租客"
   - 每个服务的网络依赖关系一目了然

4. **内外网隔离**
   - 可以给每个服务创建额外的内部网络（如 `cloudreve-internal`）
   - 需要对外暴露的服务连接 `nginx-proxy-network`
   - 不需要对外的服务（如 Redis、MySQL）只连接内部网络

---

### 启动顺序

正确的启动顺序是：

```bash
# 1. 先启动 Nginx（创建网络）
cd /opt/docker_file/website
docker-compose up -d

# 2. 启动其他服务（连接已存在的网络）
cd /opt/docker_file/cloudreve
docker-compose up -d

cd /opt/docker_file/uptimekuma
docker-compose up -d

# ...以此类推
```

如果你先启动其他服务，会报错：

```
ERROR: Network nginx-proxy-network declared as external, but could not be found.
```

这时候，启动 Nginx 就可以了。

---

## 三、最终架构：三层设计 + 20+ 容器

经过两年的迭代，我的服务器架构最终形成了这样的结构：

### 架构总览

```
┌─────────────────────────────────────────┐
│         网关层（Gateway Layer）          │
│  Nginx + ModSecurity WAF + ACME.sh      │
│  - 统一入口 (80/443)                     │
│  - SSL 自动化                            │
│  - WAF 防护                              │
└─────────────┬───────────────────────────┘
              │ nginx-proxy-network
              │
┌─────────────┴───────────────────────────┐
│         应用层（Application Layer）      │
│  - Cloudreve (网盘)                      │
│  - Uptime-Kuma (监控)                    │
│  - Code-Server (在线 IDE)                │
│  - Draw.io (画图工具)                    │
│  - Xiu (直播服务器)                      │
│  - Kuma-Mieru (监控仪表板)               │
│  ...                                     │
└─────────────┬───────────────────────────┘
              │
┌─────────────┴───────────────────────────┐
│         编排层（Orchestration Layer）    │
│  - Minecraft 服务器集群 (Podman)         │
│  - QQ Bot (Maim-Bot)                     │
│  - 虚拟网络 (Vnts)                       │
│  - 内网穿透 (FRP)                        │
│  - 远程桌面 (RustDesk)                   │
└─────────────────────────────────────────┘
```

### 网关层：Nginx + ModSecurity + ACME.sh

**Nginx** 是整个架构的核心，负责：
- HTTP/HTTPS 统一入口（80/443 端口）
- 反向代理到各个服务
- SSL/TLS 终止
- 虚拟主机路由（10 个域名）

**ModSecurity**：Web 应用防火墙，拦截 SQL 注入、XSS 等常见攻击。

**ACME.sh**：SSL 证书自动化管理，Let's Encrypt 90 天自动续期。

**关键点**：
- Nginx 的 compose 文件负责**创建** `nginx-proxy-network`
- 挂载 SSL 证书目录，统一管理所有域名的证书
- 通过容器名反向代理：`proxy_pass http://cloudreve:5212`

---

### 应用层：核心服务清单

这一层是各种实际业务服务，都连接到 `nginx-proxy-network`（external: true）。

**主要服务**：
- **Cloudreve**：私有云盘，配合 Redis 做缓存（Redis 在独立内部网络）
- **Uptime-Kuma**：监控工具，定时检查所有服务健康状态
- **Code-Server**：在线 VS Code，远程写代码
- **Draw.io**：在线画图工具，需要配合 image-export 容器
- **Xiu**：RTMP 直播服务器
- **Kuma-Mieru**：监控仪表板，配合 Uptime-Kuma 展示

**网络隔离策略**：
```yaml
# 以 Cloudreve 为例
services:
  cloudreve:
    networks:
      - nginx-proxy-network  # 对外：被 Nginx 访问
      - cloudreve-internal   # 对内：访问 Redis

  redis:
    networks:
      - cloudreve-internal  # 只在内部网络，不暴露
```

这样设计的原因很简单：即使有一天 Nginx 被攻破了，攻击者也无法直接访问 Redis 或 MySQL，因为它们根本不在同一个网络里。
**这就是最小权限原则——每个服务只能访问它需要访问的东西，其他一概不给。**

---

### 编排层：Podman 管理的复杂应用

这一层是一些需要更复杂权限管理或多容器协同的应用。

为什么这些用 Podman 而不是 Docker？主要是因为 Podman 的 Rootless 模式安全性更好，不需要 root 权限就能跑容器。

~~然后？其实就没有优点了（）~~

**主要应用**：
- **Minecraft 集群**：Velocity 代理 + 多个后端服务器
- **Maim-Bot**：QQ 机器人
- **Vnts**：P2P 虚拟组网
- **FRP**：内网穿透
- **RustDesk**：远程桌面服务

---

### 数据持久化策略

所有服务都使用 **bind mount** 而不是 Docker volume：

```yaml
volumes:
  cloudreve-backend-data:
    name: cloudreve-backend-data
    driver: local
    driver_opts:
      type: none
      device: ./cloudreve_data  # 直接挂载到宿主机目录
      o: bind
```

为什么不用 Docker volume 而用 bind mount？

**太方便了**

备份的时候直接 `tar` 打包整个目录就行，迁移的时候复制目录到新服务器改个路径甚至放到原路径下就能用。而且调试的时候可以直接在宿主机上查看和修改文件，不用进容器里各种找。
最重要的是，所有数据都在 `./service_name/` 目录下，清清楚楚，不会藏在 Docker 的 `/var/lib/docker/volumes/` 里面让你找半天。

---

## 四、核心设计原则和踩过的坑

### 设计原则

#### 1. 单一职责原则

每个服务独立管理，有自己的目录、配置文件、数据目录：

```
/opt/docker_file/
├── nginx/
│   ├── docker-compose.yml
│   ├── config/
│   └── website_file/
├── cloudreve/
│   ├── docker-compose.yml
│   └── cloudreve_data/
└── uptimekuma/
    ├── docker-compose.yml
    └── uptime-kuma-data/
```

这样做的好处是显而易见的：修改一个服务不会影响其他服务，想备份哪个服务就备份哪个，想重启哪个就重启哪个。不像以前 v1.0 那样，改一个配置整个文件都要重新加载。

#### 2. 网络隔离原则

- **外部网络**（`nginx-proxy-network`）：需要被 Nginx 访问的服务
- **内部网络**（如 `cloudreve-internal`）：服务内部组件之间的通信

**例子**：
- Cloudreve 需要被 Nginx 访问 → 连接 `nginx-proxy-network`
- Redis 只需要被 Cloudreve 访问 → 只连接 `cloudreve-internal`

这样，即使攻击者突破了 Nginx，也无法直接访问 Redis。

#### 3. 安全防护原则

- **WAF**：ModSecurity 拦截常见 Web 攻击
- **SSL/TLS**：所有外部服务强制 HTTPS
- **自动更新**：证书自动续期，减少人为失误
- **最小权限**：容器只挂载必要的目录，只开放必要的端口

#### 4. 可观测性原则

- **监控**：Uptime-Kuma 收集所有服务的健康状态
- **可视化**：Kuma-Mieru 提供监控大屏
- **日志**：所有服务的日志都通过 Docker 日志系统收集

---

### 踩过的坑

#### 坑 1：Podman 的 SELinux 标签

在使用 Podman 时，如果宿主机开启了 SELinux，需要给 volume 加上 SELinux 标签：

```yaml
volumes:
  - ./data:/app/data:Z  # Z 表示私有标签
  # 或者
  - ./data:/app/data:z  # z 表示共享标签
```

不加标签的话，容器会报 "Permission denied" 错误。

#### 坑 2：证书自动更新后没有重载 Nginx

ACME.sh 续期证书后，需要重载 Nginx 才能生效。

**解决方案**：在 ACME.sh 的 `--reloadcmd` 里加上重载命令：
（可以参考 [这篇文章](https://www.esaps.net/use-acme-sh-to-apply-for-ssl-certificates/ "这篇文章") 的 1.5 节捏～）

```bash
acme.sh --install-cert -d example.com \
  --key-file /path/to/key \
  --fullchain-file /path/to/cert \
  --reloadcmd "docker exec nginx-website nginx -s reload"
```

#### 坑 3：容器内的 DNS 解析问题

有时候容器内无法解析外部域名，是因为 Docker 的 DNS 服务器配置有问题。

**解决方案**：在 `/etc/docker/daemon.json` 里指定 DNS：

```json
{
  "dns": ["8.8.8.8", "1.1.1.1"]
}
```

#### 坑 4：bind mount 的权限问题

有些容器（如 Nginx）以非 root 用户运行，需要确保宿主机目录的权限正确：

```bash
sudo chown -R 1000:1000 ./nginx_website/
```

或者在 `docker-compose.yml` 里指定 `user`：

```yaml
services:
  nginx:
    user: "1000:1000"
```

#### 坑 5：日志文件占满磁盘

Docker 的日志文件默认不会自动清理，时间长了会占满磁盘。

**解决方案**：在 `/etc/docker/daemon.json` 里配置日志大小限制：

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

---

## 五、总结：两年的成长与思考

回头看这两年的折腾，我学到了很多：

### 技术层面

- **Docker 不只是 `docker run`**：网络、存储、编排都有讲究
- **架构设计的重要性**：一开始就规划好，后期省很多事
- **自动化的价值**：证书自动续期、监控自动报警，能自动化的都自动化
- **安全不是附加项**：WAF、SSL、网络隔离，从一开始就要考虑

### 踩坑经验

- **跨 Compose 网络联通**：让一个服务作为 network master，其他服务 external 连接
- **数据持久化**：bind mount 比 volume 更直观，备份迁移都方便
- **日志管理**：不限制日志大小，迟早会被撑爆磁盘
- **权限问题**：SELinux、文件权限、容器用户，都是坑

### 对其他折腾党的建议

1. **从简单开始，逐步优化**
   - 一开始不需要搞那么复杂，先把服务跑起来
   - 遇到问题再优化，不要过度设计

2. **文档和配置要备份**
   - 定期备份 `/opt/docker_file/` 整个目录
   - 把配置文件推到 Git 仓库（记得删敏感信息）

3. **监控很重要**
   - 早点上监控，不然服务挂了都不知道
   - Uptime-Kuma 这种轻量级工具就够用了

4. **不要怕重构**
   - 架构不合理就推倒重来
   - 配置文件写得乱就整理一遍
   - 别舍不得，长痛不如短痛

5. **记得看日志**
   - 出问题先看日志：`docker logs <container>`
   - 99% 的问题日志里都有答案

---

## 六、番外：电费和硬盘空间

两年下来，还有两个不得不提的"隐藏 Boss"：

### 电费

24 小时不间断运行，电费是笔不小的开支。

我的服务器配置：
- CPU：Intel i5-12400
- 内存：32GB
- 硬盘：1TB SSD + 2TB HDD
- 功耗：大约 60W（空载）~ 120W（满载）

按平均 80W 算，一个月大约：
```
80W × 24h × 30天 = 57.6 kWh
57.6 kWh × 0.6元/kWh ≈ 35 元
```

**一年 400 多块钱的电费**，比很多云服务器便宜，但也不算少了。

### 硬盘空间

最开始我以为 1TB 够用了，结果：
- Docker 镜像：50GB
- Minecraft 世界存档：200GB
- Cloudreve 网盘文件：300GB
- 各种日志和临时文件：100GB
- 数据库备份：50GB

**1TB 根本不够用。**

后来又加了块 2TB 的机械硬盘，专门放 Minecraft 存档和网盘文件。

---

## 结语

两年时间，从高中到大一，从 1 个容器到 20+ 个容器。

这不是什么值得炫耀的成就，但确实是我一点点踩坑、一点点优化出来的。

如果你也在折腾服务器，希望这篇文章能给你一些启发，或者至少让你少踩几个坑。

如果你有更好的架构设计，或者踩过更奇葩的坑，欢迎在评论区交流（虽然我可能回复得很慢）。

最后，感谢陪我踩坑的这台服务器，两年了还没炸。

**希望它能再撑两年。**

---

*本文由 AptS:1547 编写，基于个人两年的实际踩坑经验。*
*如果你觉得这篇文章对你有帮助，可以点个赞或分享给其他折腾党。*
*如果你发现文章里有错误或者有更好的方案,欢迎在评论区指出。*

~~（电费账单就别给我看了，我不想算）~~