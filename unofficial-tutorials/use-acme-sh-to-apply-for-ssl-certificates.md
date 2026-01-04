## 什么是 acme.sh？为什么用 acme.sh？
[mdx_github author="acmesh-official" project="acme.sh"][/mdx_github]
> acme.sh 实现了 acme 协议, 可以从 letsencrypt 生成免费的证书
——摘自 acme.sh Github 项目介绍

简单来说，acme.sh 可以帮助你无痛申请并部署免费的 ssl 证书
而相较于 [Let's Encrypt](https://letsencrypt.org/) 在她官网推荐的 [Certbot](https://certbot.eff.org/) 客户端，acme.sh 支持更多的DNS厂商，比方说 DNSPod（使用腾讯云注册域名时默认的 DNS 厂商），在使用 DNS 验证方式验证时更省时省力
并且由于 acme.sh 100%使用 shell 编写，因此你也并不需要做什么软件适配即可使用，也基本不会产生软件冲突（目前我使用经验）
并且，不管你是谁，你无需 root 权限即可安装 acme.sh
*（如果你使用 Webroot mode 申请证书，由于申请过程中需要将文件写入网站根目录，最好还是使用 root 来安装，当然，这是后话）*
而且她有 Docker 部署方式 ~~（什么 Docker Fans ）~~
（相信你一定会爱上她的）

目前 acme.sh 支持[以下方式](https://github.com/acmesh-official/acme.sh?tab=readme-ov-file#supported-modes)申请证书：
- 网站根目录文件验证 *（需要用户拥有网站根目录写权限）*
- 独立验证 *（需要安装 socat 服务）*
- tls alpn 独立验证 *（需要安装 socat 服务）*
- Apache 服务验证 *（可替代网站根目录文件验证）*
- Nginx 服务验证 *（可替代网站根目录文件验证）*
- DNS 验证 *（需要申请 DNS API 或手动添加 DNS 解析）*
- [DNS CNAME 验证](https://github.com/acmesh-official/acme.sh/wiki/DNS-alias-mode)*（请查阅官方文档）*
- [服务配置模式验证](https://github.com/acmesh-official/acme.sh/wiki/Stateless-Mode)*（请查阅官方文档）*

本文参考 acme.sh 官方文档和笔者个人经验书写
如果你对本文有任何问题，欢迎在下方评论区友好讨论

-------------------

## 少说多做开始动手
具体的操作文档你可以点击[此处](https://github.com/acmesh-official/acme.sh/wiki/%E8%AF%B4%E6%98%8E)查看
如果你需要有关Docker部署方面的教程，请在左边菜单选择 `给 Docker Fans 的一点小参考`章节
⚠️ **注意！本文仅介绍基本的用法，详细用法请前往 [acme.sh](http://acme.sh/) 查看**

### 安装acme.sh

仅需要一条命令即可安装 acme.sh
**但请注意，你需要将`my@example.com`这一段替换为你自己的邮箱**
```bash
curl https://get.acme.sh | sh -s email=my@example.com
```

你也可以先克隆 GIT 仓库来安装
```bash
git clone --depth 1 https://github.com/acmesh-official/acme.sh.git
cd acme.sh
./acme.sh --install -m my@example.com
```  

在国内 ~~（较差）~~ 网络环境下，你可以使用此条命令安装
```bash
git clone --depth 1 https://gitee.com/neilpang/acme.sh.git
cd acme.sh
./acme.sh --install -m my@example.com
```

由于自动更新证书需要使用 crond 服务来运行，因此你需要先安装 crond 服务再安装 acme.sh  
当然你也可以添加 `--nocron` 参数来在无 crond 服务下安装，只不过证书自动更新将无法使用  
```bash
curl https://get.acme.sh | sh -s email=example@example.com --nocron
```


### 安装后的tips
在安装过程中，acme.sh 自动写入了你的`.bashrc`文件，这让你可以像使用命令一样调用她  
也就是说，你不需要在 acme.sh 安装目录下就可以调用她

由于 acme.sh 默认 CA 为 [ZeroSSL](https://zerossl.com/)，而在我本人实际使用中发现 ZeroSSL 似乎不太喜欢国内的网络环境，而 [Let's Encrypt](https://letsencrypt.org/) 却完全能够承受  
那，我们先换个 CA 再说

```bash
acme.sh --set-default-ca --server letsencrypt
```
*letsenctypt 字段可以更换为 acme.sh 默认支持的 CA 或支持 acme 协议的链接*
[点击查看 acme.sh 默认支持的CA](https://github.com/acmesh-official/acme.sh?tab=readme-ov-file#supported-ca)

由于 acme.sh 目前经常更新，因此建议在使用之前将其自动更新打开并检查一遍更新  
```bash
acme.sh --upgrade --auto-upgrade
```


## 申请证书吧！

但是，在申请之前，我还是希望你能够看看这些内容：
- 接下来的内容中，我将用 `example.com`作为申请域名，你需要将此域名更换为你自己的域名
- 在申请过程中如果出现任何错误，你可以添加`—-debug`参数来 debug（废话）
本文不会研究如何 debug
- 你可以申请多种类型的证书，仅需在申请命令后方添加参数`—-keylength (2048|3072|4096|8192|ec-256|ec-384|ec-521)`
- 你可以同时为多个域名申请证书。当你同时为多个域名申请证书时，一般而言，你的申请命令中第一个出现的域名即为证书的主域名。如果你不确定证书的主域名是哪一个，可以使用 `acme.sh —-list`来查看
- 当你同时为多个域名申请证书时，如果你仅指定一种验证方式，那么所有的网址都将使用这一种的方式进行验证。当然，你也可以为不同的域名指定不同的验证方式，其类似于这样的格式：

```bash
acme.sh  --issue \
-d aa.com  -w /home/wwwroot/aa.com \
-d bb.com  --dns dns_cf \
-d cc.com  --apache \
-d dd.com  -w /home/wwwroot/dd.com
```

那么现在，我们开干吧

-------------------

### DNS 验证
DNS 验证分为自动 DNS 验证（需要 DNS 厂商提供 API ）和手动 DNS 验证，其中手动 DNS 验证方式不支持自动续签
**仅 DNS 验证支持申请泛域名证书**
*（有可能 DNS CNAME 验证也可以？我到时候试试）*

#### 自动DNS验证
在此之前，请你先打开这个网页：[acme.sh支持的自动DNS验证厂商列表](https://github.com/acmesh-official/acme.sh/wiki/dnsapi)  
现在，你需要找到你的域名当前使用的DNS厂商，这里选择常用的`Cloudflare`和`DNSPod`举例子
*（部分厂商有特殊的自动验证的要求，这就是为什么看文档总比看别人写的文章好的原因）*

##### Cloudflare
Cloudflare 有两种API密钥，一种是区域密钥，一种是全局密钥
点击[此处](https://dash.cloudflare.com/profile/api-tokens)前往 API 获取页面
使用全局密钥的话，当然会更加简便，但也会增加安全风险
全局密钥需要你的`Cloudflare 全局密钥`和`Cloudflare 登录邮箱`，而区域密钥需要你的`用户 Token`和`区域 ID`

[caption id="attachment_1485" align="aligncenter" width="1920"]<img src="https://esaps.net/wp-content/uploads/2024/02/CloudFlare-API页面.webp" alt="CloudFlare API页面示例" width="1920" height="998" class="size-full wp-image-1485" />CloudFlare API 页面示例[/caption]

要查看你的全局密钥，请在你的 API 页面的`Global API Key`这一行点击查看按钮获取你的全局密钥
要使用区域密钥，请点击`创建令牌` -> `编辑区域 DNS` -> `区域资源下选择你的域名` -> `创建令牌`来获取你的用户 Token ，你可以在你的`网站概述`下找到你的域名`区域 ID`

使用全局密钥
```bash
export CF_Key="Your Cloudflare Global API Key"
export CF_Email="Your Cloudflare Login Email"
```

使用区域密钥
```bash
export CF_Zone_ID="Your Cloudflare Website Zone ID"
export CF_Token="Your Cloudflare User API Token"
```

最后我们来申请证书吧！
```bash
acme.sh --issue --dns dns_cf -d example.com -d *.example.com
```

##### DNSPod
甚至DNSPod官方都有 [acme.sh on DNSPod 的教程](https://docs.dnspod.cn/dns/acme-sh/)

如果一切正常，那么你的证书将会被自动下载

#### 手动 DNS 验证
```bash
acme.sh --issue --dns -d example.com \
--yes-I-know-dns-manual-mode-enough-go-ahead-please
```
你会得到类似这样的回应：
```bash
[Sun Feb 11 18:52:50 CST 2024] Add the following TXT record:
[Sun Feb 11 18:52:50 CST 2024] Domain: "_acme-challenge.example.com"
[Sun Feb 11 18:52:50 CST 2024] TXT value: "xxxxxxxxxxxxx-xxxxxxxxxxxxxxxxxxxxxxxxxxxxx";
```
这时请你添加一条 DNS 解析：
- 解析类型为`TXT`
- 名称为上方输出中`Domain`的内容，在本例中为`_acme-challenge.example.com`
⚠ **注意，一般情况下在添加 TXT 解析时 dns 厂商会自动帮你添加`example.com`字段**
- 内容为上方输出中 `TXT value` 的内容，在本例中为`xxxxxxxxxxxxx-xxxxxxxxxxxxxxxxxxxxxxxxxxxxx`

当添加完 DNS 解析之后，请你使用此条命令继续申请证书
```bash
acme.sh --issue --renew -d example.com \
--yes-I-know-dns-manual-mode-enough-go-ahead-please
```
如果一切正常，那么你的证书将会被自动下载

-------------------

### 网站根目录文件验证
作为最朴素（也是最简单）的验证方式，你只需要指定你的域名绑定的网站根目录即可完成验证
本例中以`/home/wwwroot/example.com`作为网站根目录
```bash
acme.sh --issue -d example.com -w /home/wwwroot/example.com
```

如果一切正常，那么你的证书将会被自动下载

------------------

### 独立验证
独立验证也有两种方式，但其实都差不多
如果你的服务器并没有运行网站服务，此时用此种验证方式会更为简单
适用于没有任何服务绑定了 80 端口或者 443 端口的服务器类型
由于此种方式依赖于 socat 服务，我们需要先安装 socat 服务
```bash
#For RHEL Series
sudo dnf install -y socat

#For Debian Series
sudo apt install -y socat
```

**此种验证方式需要监听 80/443 端口，因此你需要使用 root 权限来进行验证**
#### 使用 80 端口
```bash
acme.sh --issue --standalone -d example.com -d www.example.com -d cp.example.com
```

#### 使用 443 端口
```bash
acme.sh --issue --alpn -d example.com -d www.example.com -d cp.example.com
```

如果一切正常，那么你的证书将会被自动下载

-------------------

### 网站服务验证
网站服务验证，顾名思义，你需要先安装 nginx 或者 Apache httpd 这二者其中一个服务方可使用。
她会更改你的网站服务配置，但请放心，证书签发完成之后都会给你改回来
> 其实在实际上，根据我的测试，`网站服务验证`只是`服务配置模式验证`的自动化而已
其实也是`网站根目录文件验证`的另外一种方式

#### Nginx 服务验证
```bash
acme.sh --issue --nginx -d example.com -d www.example.com -d cp.example.com
```

#### Apache 服务验证
```bash
acme.sh --issue --apache -d example.com -d www.example.com -d cp.example.com
```

如果一切正常，那么你的证书将会被自动下载

-------------------

## 安装证书吧！

由于 acme.sh 并不会自动帮我们更改网站服务（如 nginx 和 Apache httpd ）的配置文件，因此我们需要手动告诉 acme.sh 要将签发下来的证书放置在哪里，并执行什么命令使网站服务重载或重启   
在此用以 nginx 为网站服务的服务器为例：

- 我们安装的证书是以 `example.com` 为主域名的证书
- 将公钥文件写入 `/etc/nginx/conf.d/ssl/server.crt` 文件
- 将私钥文件写入 `/etc/nginx/conf.d/ssl/server.key` 文件
- 将CA证书写入 `/etc/nginx/conf.d/ssl/ca.crt` 文件
- 将FullChain文件写入 `/etc/nginx/conf.d/ssl/fullchain.pem` 文件
- 命令nginx服务器重新加载配置文件的命令为 `nginx -s reload`

那么，我们的命令应该是这个样子
```bash
acme.sh --install-cert -d example.com \
--cert-file      /etc/nginx/conf.d/ssl/server.crt \
--key-file       /etc/nginx/conf.d/ssl/server.key \
--ca-file        /etc/nginx/conf.d/ssl/ca.crt \
--fullchain-file /etc/nginx/conf.d/ssl/fullchain.pem \
--reloadcmd      "nginx -s reload"
```

在你执行此命令过后，如果一切正常，你将在你指定的位置发现你的证书（他们很乖的）
如果你同时指定了 `—-reloadcmd`，acme.sh会尝试运行指定的重载命令并将命令运行时的输出打印出来
如果在 acme.sh 运行重载命令时报错也不要担心，你只需要调整好你的 `—-reloadcmd`并且将上面的安装证书命令在运行一次，此时 `—-reloadcmd`将会被覆盖为最新的指定命令
**⚠️注意：光申请证书而不修改任何配置文件是不会有任何效果的，你需要自己修改网站服务的配置文件来将 ssl 证书部署到线上**
请点击[此处](https://cloud.tencent.com/document/product/400/47413)查看ssl证书的部署教程

这些就是全部内容了！感谢你阅读这篇文章

-------------------

## 给 Docker Fans 的一点小参考
好好好，现在我知道你要用Docker了
你可以参考 [acme.sh官方的Docker文档](https://github.com/acmesh-official/acme.sh/wiki/Run-acme.sh-in-docker)
我不太喜欢他的文档，因为要挂载`docker.sock`文件，这可能对安全性造成影响
*（Docker = Root）*
并且如果你要同时给多个容器部署（好像可以？）会有点麻烦

因此，我写了个开箱即用的解决方案：**acme-docker-reloader**
[mdx_github author="AptS-1547" project="acme-docker-reloader"][/mdx_github]

支持在`install-cert`之后在 Docker 宿主机上执行你希望的命令，无需挂载 `docker.sock`，三步搞定部署 owo

### 三步部署 acme-docker-reloader

#### 1. 克隆项目
```bash
git clone https://github.com/AptS-1547/acme-docker-reloader.git
cd acme-docker-reloader
```

#### 2. 运行安装脚本
```bash
sudo ./install.sh
```
安装脚本会提示你输入证书更新后需要执行的重载命令，比如：
- 如果你的 Nginx 在宿主机上运行：`systemctl reload nginx`
- 如果你的 Nginx 在 Docker 容器中：`docker exec nginx nginx -s reload`
- 如果你有多个服务：可以稍后编辑 `config/config.yml` 添加

#### 3. 启动 acme.sh 容器
```bash
docker-compose up -d
```

就这么简单！🎉

### 申请和安装证书

#### 进入容器
由于我们是在 Docker 容器里面运行，所以要用这种方式来操作 acme.sh
```bash
docker exec -it acme.sh bash
```

如果你怕麻烦，你可以执行这条命令来简便一点
```bash
alias acme.sh="docker exec -it acme.sh acme.sh"
```

#### 首次使用：注册账号
因为我们是直接运行了 Docker 容器，所以我们需要先在CA那边注册一个账号
```bash
acme.sh --register-account -m your@email.com
acme.sh --set-default-ca --server letsencrypt
```

#### 申请证书
然后就可以按照上面的内容一样申请证书喽！这里以 Cloudflare DNS 验证为例：

**注意：如果你使用自动 DNS 验证申请的话，最好还是直接进入容器里面先申请一次，让 acme.sh 先保存你的 API 密钥**

```bash
# 进入容器
docker exec -it acme.sh bash

# 设置 Cloudflare API（二选一）
# 方式1: 使用区域密钥（推荐）
export CF_Token="your_cloudflare_token"
export CF_Zone_ID="your_zone_id"

# 方式2: 使用全局密钥
export CF_Key="your_global_api_key"
export CF_Email="your_email"

# 申请证书
acme.sh --issue -d example.com -d *.example.com --dns dns_cf
```

#### 安装证书并设置自动重载
```bash
acme.sh --install-cert -d example.com \
  --cert-file /ssl/example.com/cert.pem \
  --key-file /ssl/example.com/key.pem \
  --fullchain-file /ssl/example.com/fullchain.pem \
  --reloadcmd "bash /acme-reloader.sh"
```

**重点：**
- 证书文件会保存在项目的 `ssl/` 目录下
- `--reloadcmd` 必须设置为 `bash /acme-reloader.sh`，这样才能通知宿主机执行重载命令
- 如果提示 `Host error: Unable to connect to host.`，说明宿主机的守护进程没有启动，执行 `sudo systemctl start acme-reloader-host` 即可

### 配置你的 Web 服务器

证书文件位于项目的 `ssl/` 目录下，配置你的 Nginx（假设你把项目克隆到了 `/opt/acme-docker-reloader`）：

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    # 使用证书的绝对路径
    ssl_certificate /opt/acme-docker-reloader/ssl/example.com/fullchain.pem;
    ssl_certificate_key /opt/acme-docker-reloader/ssl/example.com/key.pem;

    # 其他配置...
}
```

重载 Nginx：
```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 多服务配置（可选）

如果你需要同时重载多个服务（比如 Nginx + Caddy），可以编辑 `config/config.yml`：

```yaml
services:
  nginx:
    command: "systemctl reload nginx"
    enabled: true

  caddy:
    command: "systemctl reload caddy"
    enabled: true
```

### 故障排查

如果遇到问题，可以查看日志：
```bash
# 检查宿主机守护进程状态
sudo systemctl status acme-reloader-host

# 查看详细日志
sudo journalctl -u acme-reloader-host -f
tail -f ./logs/acme-reloader.log
```

完成~！享受你的证书自动化吧！证书会自动续签，续签后自动重载服务，再也不用担心证书过期了 owo
（多嘴一句，你需要自己修改网站服务配置文件来部署，具体看[这里](https://cloud.tencent.com/document/product/400/47413)）

