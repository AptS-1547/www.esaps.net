> "你，是不是也遇到过'在我电脑上能跑'的问题？"
> ——AptS:1547（手里拿着 Dockerfile 准备糊你脸）

你辛辛苦苦写了个 Node.js 项目，在自己电脑上跑得好好的。

然后你把代码发给同事，他说："跑不起来啊，报错了。"

你检查了半天，发现他的 Node 版本是 14，你的是 18。

于是你让他升级 Node，结果他说："我其他项目还在用 Node 14，不能升。"

你又试着在服务器上部署，结果发现服务器上的 Python 版本不对、系统库缺失、环境变量没配……

**折腾了一整天，你开始怀疑人生。**

这时候，有人告诉你：**"用 Docker 啊，一次打包，到处运行。"**

你半信半疑地装上了 Docker，然后发现——

**真香。**

---

## 一、Docker 到底是什么？用人话讲

Docker 是一个**容器化平台**，但这句话对新手来说等于没说。

我们换个说法：

**Docker 就像是一个"打包机"，它能把你的应用程序、依赖库、配置文件、甚至操作系统环境，全部打包成一个"集装箱"（容器）。**

这个"集装箱"可以在任何装了 Docker 的机器上运行，不管是你的笔记本、同事的电脑、还是云服务器，**都能保证运行结果完全一致**。

### 举个例子

假设你要给朋友送一台组装好的电脑：

* **传统方式**：你把主板、CPU、内存、硬盘分开寄过去，朋友收到后自己组装。结果他发现主板和 CPU 不兼容，内存插槽对不上……
* **Docker 方式**：你把电脑组装好，装进一个标准集装箱里寄过去。朋友收到后，打开集装箱，插上电源，直接开机。

Docker 做的就是这件事：**把应用和环境打包在一起，保证"开箱即用"。**

---

## 二、为什么要用 Docker？它解决了什么问题？

### 问题 1："在我电脑上能跑"综合症

你写的代码在你电脑上跑得好好的，但到了别人那里就报错。

原因可能是：
* Python 版本不一样
* 缺少某个系统库
* 环境变量没配置
* 操作系统不同（你用 macOS，他用 Windows）

**Docker 的解决方案**：把你的代码和运行环境（Python 版本、系统库、环境变量）一起打包成镜像，别人直接运行这个镜像，环境完全一致。

### 问题 2：服务器上装了一堆乱七八糟的东西

你的服务器上可能同时跑着：
* 一个 Node.js 项目（需要 Node 14）
* 一个 Python 项目（需要 Python 3.8）
* 一个 Java 项目（需要 JDK 11）

这些项目的依赖可能会冲突，而且时间长了，你自己都不知道服务器上装了什么。

**Docker 的解决方案**：每个项目跑在独立的容器里，互不干扰。Node 项目用 Node 14 的容器，Python 项目用 Python 3.8 的容器，各玩各的。

### 问题 3：部署太麻烦

传统部署流程：
1. SSH 连上服务器
2. 安装 Node.js / Python / Java
3. 安装依赖（`npm install` / `pip install`）
4. 配置环境变量
5. 启动服务
6. 祈祷不要出错

**Docker 的解决方案**：
1. 在本地构建好镜像
2. 把镜像推到服务器
3. 一条命令启动容器
4. 完事

---

## 三、Docker 的三大核心概念（用人话讲）

Docker 有三个核心概念：**镜像、容器、仓库**。

很多教程会告诉你"镜像是只读模板，容器是运行实例"，但这对新手来说还是太抽象。

我们用更直观的比喻：

### 1. 镜像（Image）= 安装包

镜像就像是一个 **应用程序的安装包**。

* 你从网上下载一个 `.exe` 文件，这就是"安装包"
* 你从 Docker Hub 下载一个 `nginx:latest` 镜像，这也是"安装包"

镜像里包含了：
* 应用程序的代码
* 运行环境（比如 Node.js、Python）
* 依赖库（比如 npm 包、pip 包）
* 配置文件

**镜像是只读的**，你不能直接修改它，就像你不能直接修改一个 `.exe` 文件一样。

### 2. 容器（Container）= 运行中的程序

容器就像是 **双击安装包后，打开的程序**。

* 你双击 `.exe` 文件，程序就运行起来了
* 你运行 `docker run nginx` 命令，容器就启动了

一个镜像可以启动多个容器，就像你可以同时打开多个 Chrome 窗口一样。

**容器是可以修改的**，你可以在容器里创建文件、修改配置，但这些修改只存在于这个容器里，不会影响镜像本身。

### 3. 仓库（Repository）= 应用商店

仓库就像是 **App Store 或者 Google Play**。

* 你从 App Store 下载微信，这就是"仓库"
* 你从 Docker Hub 下载 `nginx` 镜像，这也是"仓库"

最常用的仓库是 [Docker Hub](https://hub.docker.com/)，上面有几百万个镜像，包括：
* 官方镜像：`nginx`、`mysql`、`redis`、`python`、`node` 等
* 社区镜像：各种开源项目的镜像

你也可以把自己的镜像推到 Docker Hub，分享给别人。

---

## 四、安装 Docker

### Linux 系统（推荐）

一条命令搞定：

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

安装完成后，检查一下：

```bash
docker version
```

如果看到类似这样的输出，说明安装成功了：

```
Client: Docker Engine - Community
 Version:           24.0.7
 ...
```

### Windows / macOS 用户

* **Windows**：下载 [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop)
* **macOS**：下载 [Docker Desktop for Mac](https://www.docker.com/products/docker-desktop) 或者更轻量的 [OrbStack](https://orbstack.dev/)

安装完成后，打开 Docker Desktop，等它启动完成（右下角/右上角的图标变成绿色），就可以用了。

---

## 五、配置镜像加速（国内用户必看）

Docker 默认从 Docker Hub 下载镜像，但在国内速度很慢（你懂的）。

我们需要配置国内镜像源来加速下载。

### Linux 用户

编辑 Docker 配置文件：

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": [
    "https://registry.cn-hangzhou.aliyuncs.com",
    "https://mirror.ccs.tencentyun.com",
    "https://docker.mirrors.ustc.edu.cn"
  ]
}
EOF

# 重启 Docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### Windows / macOS 用户

1. 打开 Docker Desktop
2. 点击右上角的设置图标（齿轮）
3. 选择 "Docker Engine"
4. 在 JSON 配置中添加：

```json
{
  "registry-mirrors": [
    "https://registry.cn-hangzhou.aliyuncs.com",
    "https://mirror.ccs.tencentyun.com",
    "https://docker.mirrors.ustc.edu.cn"
  ]
}
```

5. 点击 "Apply & Restart"

**注意这里绝大多数镜像因为相关政策都被移除了，如果有需要的话可以自行在搜索引擎搜索 `Docker 镜像网址` 并替换上面显示的网址链接**

### 验证是否生效

```bash
docker info | grep "Registry Mirrors"
```

如果看到输出，说明配置成功了。

---

## 六、第一次运行 Docker 容器

现在我们来运行第一个容器，体验一下 Docker 的魔力。

### 运行一个 Nginx 服务器

```bash
docker run -d -p 8080:80 --name my-nginx nginx:latest
```

然后打开浏览器，访问 `http://localhost:8080`，你会看到 Nginx 的欢迎页面。

**恭喜你，你已经成功运行了第一个 Docker 容器！**

但是，这条命令到底做了什么？我们一个参数一个参数地解释：

### 命令解析

```bash
docker run -d -p 8080:80 --name my-nginx nginx:latest
```

* `docker run`：运行一个容器
* `-d`：后台运行（detached mode）
  * 如果不加 `-d`，容器会占用你的终端，你按 `Ctrl+C` 就会停止容器
  * 加了 `-d`，容器在后台运行，你可以继续在终端里输入其他命令
* `-p 8080:80`：端口映射
  * `8080` 是你电脑上的端口（宿主机端口）
  * `80` 是容器内部的端口
  * 意思是：访问你电脑的 8080 端口，会被转发到容器的 80 端口
  * 为什么要映射？因为容器内部的端口外面访问不到，必须映射到宿主机端口
* `--name my-nginx`：给容器起个名字
  * 如果不指定名字，Docker 会随机生成一个名字（比如 `silly_einstein`）
  * 指定名字后，你可以用 `docker stop my-nginx` 来停止容器，而不用记住容器 ID
* `nginx:latest`：镜像名称
  * `nginx` 是镜像名
  * `latest` 是标签（tag），表示最新版本
  * 如果不指定标签，默认就是 `latest`

### 如果镜像不存在会怎样？

当你运行 `docker run nginx:latest` 时，Docker 会先检查本地有没有这个镜像。

* 如果有，直接用本地镜像启动容器
* 如果没有，自动从 Docker Hub 下载镜像，然后启动容器

所以你不需要手动 `docker pull nginx`，直接 `docker run` 就行。

---

## 七、常用命令详解

### 查看正在运行的容器

```bash
docker ps
```

输出类似这样：

```
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                  NAMES
a1b2c3d4e5f6   nginx:latest   "/docker-entrypoint.…"   2 minutes ago   Up 2 minutes   0.0.0.0:8080->80/tcp   my-nginx
```

* `CONTAINER ID`：容器的唯一 ID（前 12 位）
* `IMAGE`：使用的镜像
* `COMMAND`：容器启动时执行的命令
* `CREATED`：容器创建时间
* `STATUS`：容器状态（`Up` 表示运行中）
* `PORTS`：端口映射
* `NAMES`：容器名称

### 查看所有容器（包括停止的）

```bash
docker ps -a
```

为什么要加 `-a`？因为 `docker ps` 默认只显示运行中的容器，停止的容器不会显示。

### 停止容器

```bash
docker stop my-nginx
```

停止后，容器还在，只是不运行了。你可以用 `docker ps -a` 看到它。

### 启动已停止的容器

```bash
docker start my-nginx
```

### 重启容器

```bash
docker restart my-nginx
```

### 删除容器

```bash
docker rm my-nginx
```

**注意**：只能删除已停止的容器。如果容器还在运行，需要先停止，或者用 `-f` 强制删除：

```bash
docker rm -f my-nginx
```

### 查看容器日志

```bash
docker logs my-nginx
```

如果你想实时查看日志（类似 `tail -f`），加上 `-f` 参数：

```bash
docker logs -f my-nginx
```

### 进入容器内部

```bash
docker exec -it my-nginx bash
```

* `exec`：在运行中的容器里执行命令
* `-it`：交互式终端（interactive + tty）
* `bash`：要执行的命令（进入 bash shell）

进入容器后，你可以像操作普通 Linux 系统一样操作容器内部。

退出容器：输入 `exit` 或按 `Ctrl+D`。

### 查看本地镜像

```bash
docker images
```

输出类似这样：

```
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
nginx        latest    a1b2c3d4e5f6   2 weeks ago    142MB
```

### 删除镜像

```bash
docker rmi nginx:latest
```

**注意**：如果有容器正在使用这个镜像，需要先删除容器，才能删除镜像。

---

## 八、实战：5 分钟部署一个网站

现在我们来做一个完整的实战案例：用 Docker 部署一个简单的静态网站。

### 第一步：创建项目目录

```bash
mkdir docker-website && cd docker-website
```

### 第二步：创建网站内容

```bash
cat > index.html << 'EOF'
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>Hello from Docker!</title>
    <style>
        body {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            font-family: Arial, sans-serif;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
        }
        .container {
            background: white;
            padding: 40px;
            border-radius: 10px;
            box-shadow: 0 10px 30px rgba(0,0,0,0.3);
            text-align: center;
        }
        h1 {
            color: #667eea;
            margin-bottom: 20px;
        }
        p {
            color: #666;
            line-height: 1.6;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>🎉 Hello from Docker!</h1>
        <p>如果你看到这个页面，说明你已经成功使用 Docker 部署了一个网站！</p>
        <p>这个网站运行在一个 Nginx 容器里，是不是很简单？</p>
    </div>
</body>
</html>
EOF
```

### 第三步：创建 Dockerfile

Dockerfile 是用来定义镜像的"配方"，告诉 Docker 如何构建镜像。

```bash
cat > Dockerfile << 'EOF'
# 使用官方 Nginx 镜像作为基础镜像
FROM nginx:alpine

# 把我们的网站文件复制到 Nginx 的默认目录
COPY index.html /usr/share/nginx/html/index.html

# 暴露 80 端口（这只是声明，实际映射在 docker run 时指定）
EXPOSE 80
EOF
```

**Dockerfile 解析**：

* `FROM nginx:alpine`：基于 Nginx 的 Alpine 版本（Alpine 是一个超小的 Linux 发行版，只有 5MB）
* `COPY index.html /usr/share/nginx/html/index.html`：把我们的 `index.html` 复制到容器里
* `EXPOSE 80`：声明容器会使用 80 端口（这只是文档作用，实际端口映射在 `docker run` 时指定）

### 第四步：构建镜像

```bash
docker build -t my-first-website .
```

* `build`：构建镜像
* `-t my-first-website`：给镜像起个名字（tag）
* `.`：Dockerfile 所在的目录（当前目录）

构建过程中，你会看到类似这样的输出：

```
[+] Building 2.3s (7/7) FINISHED
 => [internal] load build definition from Dockerfile
 => => transferring dockerfile: 200B
 => [internal] load .dockerignore
 => => transferring context: 2B
 => [internal] load metadata for docker.io/library/nginx:alpine
 => [1/2] FROM docker.io/library/nginx:alpine
 => [internal] load build context
 => => transferring context: 500B
 => [2/2] COPY index.html /usr/share/nginx/html/index.html
 => exporting to image
 => => exporting layers
 => => writing image sha256:a1b2c3d4e5f6...
 => => naming to docker.io/library/my-first-website
```

### 第五步：运行容器

```bash
docker run -d -p 8080:80 --name website-demo my-first-website
```

### 第六步：访问网站

打开浏览器，访问 `http://localhost:8080`，你会看到你的网站！

### 清理

如果你想删除容器和镜像：

```bash
# 停止并删除容器
docker rm -f website-demo

# 删除镜像
docker rmi my-first-website
```

---

## 九、容器的生命周期：数据去哪了？

现在我们来讲一个**新手最容易踩的坑**：容器的数据持久化。

### 问题：容器删了，数据就没了

假设你在容器里创建了一个文件：

```bash
# 进入容器
docker exec -it my-nginx bash

# 创建一个文件
echo "Hello Docker" > /tmp/test.txt

# 退出容器
exit
```

现在删除容器：

```bash
docker rm -f my-nginx
```

然后重新启动一个新容器：

```bash
docker run -d -p 8080:80 --name my-nginx nginx:latest
```

进入容器，你会发现 `/tmp/test.txt` 不见了：

```bash
docker exec -it my-nginx bash
cat /tmp/test.txt  # 文件不存在
```

**为什么？因为容器是"一次性"的。**

容器删除后，容器内部的所有修改都会丢失。就像你卸载一个软件，软件的数据也会被删除一样。

### 解决方案：数据卷（Volume）

如果你想让数据持久化，需要使用**数据卷**。

数据卷就像是一个"外接硬盘"，挂载到容器里。容器删了，数据卷还在。

#### 方式一：匿名数据卷

```bash
docker run -d -p 8080:80 --name my-nginx -v /usr/share/nginx/html nginx:latest
```

* `-v /usr/share/nginx/html`：创建一个匿名数据卷，挂载到容器的 `/usr/share/nginx/html` 目录

#### 方式二：命名数据卷

```bash
docker run -d -p 8080:80 --name my-nginx -v my-data:/usr/share/nginx/html nginx:latest
```

* `-v my-data:/usr/share/nginx/html`：创建一个名为 `my-data` 的数据卷，挂载到容器的 `/usr/share/nginx/html` 目录

#### 方式三：绑定挂载（最常用）

```bash
docker run -d -p 8080:80 --name my-nginx -v /path/on/host:/usr/share/nginx/html nginx:latest
```

* `-v /path/on/host:/usr/share/nginx/html`：把宿主机的 `/path/on/host` 目录挂载到容器的 `/usr/share/nginx/html` 目录

这样，你在宿主机上修改文件，容器里也会同步更新。

**这是最常用的方式**，因为你可以直接在宿主机上编辑文件，不用进容器。

---

## 十、常见问题 & 排坑指南

### ❌ "Cannot connect to the Docker daemon"

**原因**：Docker 服务没启动。

**解决方案**：

```bash
# Linux
sudo systemctl start docker

# Windows/macOS
打开 Docker Desktop，等它启动完成
```

### ❌ "port is already allocated"

**原因**：端口被占用了。

比如你运行 `docker run -p 8080:80 nginx`，但 8080 端口已经被其他程序占用了。

**解决方案**：

* 换个端口：`docker run -p 8081:80 nginx`
* 或者找到占用端口的程序，关掉它

### ❌ "No such container"

**原因**：容器不存在或者名字写错了。

**解决方案**：

用 `docker ps -a` 查看所有容器，确认容器名称。

### ❌ 容器启动后立即退出

**原因**：容器里的主进程退出了，容器就会停止。

**解决方案**：

查看容器日志，看看为什么退出：

```bash
docker logs <container-name>
```

### ❌ 镜像下载太慢

**原因**：没配置镜像加速。

**解决方案**：

参考前面的"配置镜像加速"章节。

---

## 十一、小结：你已经掌握了 Docker 的基础

到这里，你已经学会了：

* ✅ Docker 是什么，解决了什么问题
* ✅ 镜像、容器、仓库的概念
* ✅ 安装和配置 Docker
* ✅ 运行第一个容器
* ✅ 常用命令（`run`、`ps`、`stop`、`rm`、`logs`、`exec`）
* ✅ 构建自己的镜像（Dockerfile）
* ✅ 数据持久化（Volume）

这些知识已经足够你在日常开发中使用 Docker 了。

但是，Docker 还有很多高级功能：

* **Docker Compose**：用一个配置文件管理多个容器
* **Docker 网络**：让容器之间互相通信
* **Docker Swarm / Kubernetes**：容器编排，管理成百上千个容器

这些内容我们会在后续的文章中详细讲解。

---

## 十二、下一步学什么？

如果你想继续深入学习 Docker，推荐按照这个顺序：

1. **Docker Compose**：学会用一个配置文件管理多个容器（下一篇就是这个）
2. **Docker 网络**：学会让容器之间互相通信
3. **Dockerfile 最佳实践**：学会写出高效、安全的 Dockerfile
4. **Docker 镜像优化**：学会减小镜像体积，加快构建速度

敬请期待下一篇：

> **《Docker 入门指南 - 用 Docker Compose 管理你的应用》**

---

*本文由 AptS:1547 编写，基于个人经验与公开资料整理。如果你在使用 Docker 时遇到了问题，欢迎在评论区交流！*
