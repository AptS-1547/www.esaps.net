<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width=330 height=86 src="//music.163.com/outchain/player?type=2&id=2657523603&auto=0&height=66"></iframe>

## 我的域名邮箱凭什么收费？

那是一个很平常的夜晚，我软趴趴的躺在我的床上上网。
“欸，有个邮件我收一下”
随即，我看到了个我毕生难忘的画面
[caption id="attachment_1644" align="aligncenter" width="1024"]<img src="https://www.esaps.net/wp-content/uploads/2024/07/stupid_mail-1024x308.webp" alt="一款付费邮箱的收费图片" width="1024" height="308" class="size-large wp-image-1644" /> 我不说是谁[/caption]

恶心到我的不是他收钱，而是他这句话
> 比如你拥有 example.com 你可分配 Ava@example.com 给邮箱。

我勒个邮箱啊，在这个人人都有服务器的时代我自己要做域名邮箱我还要给你付钱？
**那不好意思，这篇文章就是针对你的**

~~（实话实说，用它这种架构更好的邮箱服务器肯定比自己搭建邮箱服务器更好。我只是在找写这篇文章的理由）~~
当然，对于我们这种业余级别的~~（也就是连垃圾邮件都看不上的）~~也就没必要去用它更专业的了
太耗钱了啊（；´д｀）ゞ

[caption id="attachment_1925" align="aligncenter" width="150"]<img src="https://www.esaps.net/wp-content/uploads/2024/09/343acea50252bbad919f1e6c87a02791-150x150.webp" alt="" width="150" height="150" class="size-thumbnail wp-image-1925" /> 哎 你这小南娘[/caption]

## 什么是poste.io？为什么选择poste.io？

Poste.io，一款用Docker分发的开箱即用的邮局集合环境，官方宣传可在5分钟内搭建好一个高性能的完整邮局系统
其具有以下特点
- 支持SMTP、IMAP、POP3协议，支持SSL/TLS加密
- 支持多域名，多用户，多邮箱
- 内置了 `ClamAV病毒查杀引擎` 和 `RSPAMD垃圾邮件过滤器`
- 使用 `RoundCube` 作为前端交互，拥有多种管理方式（基于网页的邮局账号管理、RESTful API 和 容器内命令行工具）
- 支持邮件过滤、自动转发、自动回复、邮件签名、邮件黑白名单等功能，可以为每一位邮箱用户限制使用空间或者电子邮件配额
- 支持从Let’s Encrypt获取免费的SSL证书
- 使用Docker容器进行分发，与服务器其他程序进行隔离，便于更新

简单来说，Poste.io 非常适合于我们用来快速搭建可靠的邮箱服务，还是免费的（疯狂暗示）

## 如何搭建poste.io？

**在开始之前，你需要准备以下几个东西：**

- **一个域名**，用来作为你的邮箱域名，比如说 `example.com`
- **一个服务器**，用来部署邮件服务器，这个服务器需要有一个公网 IP 地址
  而且，由于邮件服务器需要使用 25 端口来接收邮件，你需要确保你的服务器提供商允许你使用 25 端口
  目前我在使用的是腾讯云，其 25 端口默认是开启的。如果你发现你的服务器无法使用 25 端口，那么你需要联系你的服务器提供商开启 25 端口。**目前已知 `阿里云` 默认是关闭 25 端口的，而且他们不会帮你解封，你可能需要换一个服务商**
- **一个 DNS 服务商**，用来设置域名的 DNS 记录，这里我使用的是 `Cloudflare`，当然你也可以使用其他的 DNS 服务商
- **一个能够访问互联网的电脑/手机**，用来部署和管理邮件服务器

### 1. 安装Docker

你可以参照这篇文章来安装Docker
[mdx_post]https://www.esaps.net/working-with-docker-how-to-install/[/mdx_post]

### 2. 安装Poste.io

个人习惯，所有的 Docker 容器都使用 Docker Compose 来管理，这里也是使用 Docker Compose 来安装 Poste.io

请将下方的 `mail.example.com` 替换为你的域名，然后将以下代码保存为 `docker-compose.yml` 文件

```yaml
volumes:
  mail-data:
    name: mail-data
    driver: local
    driver_opts:      # 这部分定义了卷的挂载选项
      o: bind         # 使用bind挂载
      type: none      # 不使用特殊文件系统
      device: ./data  # 将容器内的数据保存到当前目录下的data文件夹

services:
  mailserver:
    hostname: mail.example.com  # 邮件服务器的主机名，必须与你的域名一致
    container_name: mail.example.com
    image: analogic/poste.io    # 使用官方提供的poste.io镜像
    restart: always             # 容器崩溃后自动重启
    ports:                      # 暴露邮件服务所需的所有端口
      - 25:25                   # SMTP - 用于接收其他服务器发来的邮件
      - 80:80                   # HTTP - 用于管理邮件服务器
      - 443:443                 # HTTPS - 用于管理邮件服务器
      - 465:465                 # SMTPS - 加密的SMTP
      - 587:587                 # Submission - 用于客户端提交邮件
      - 110:110                 # POP3 - 收取邮件(明文)
      - 995:995                 # POP3S - 加密的POP3
      - 143:143                 # IMAP - 收取邮件(明文)
      - 993:993                 # IMAPS - 加密的IMAP
      - 4190:4190               # Sieve - 邮件过滤（可选）
    environment:
      - TZ=Asia/Shanghai        # 请将时区设置为你所在的时区（必须）
      - DISABLE_CLAMAV=TRUE     # 禁用病毒扫描以节省资源
      - DISABLE_RSPAMD=TRUE     # 禁用垃圾邮件过滤以节省资源
      - DISABLE_ROUNDCUBE=TRUE  # 禁用 RoundCube Webmail 界面
    volumes:
      - mail-data:/data         # 将数据卷挂载到容器内的/data目录
```

将上面的代码保存为 `docker-compose.yml` 文件并放到你喜欢的目录下，然后执行以下命令

```bash
sudo mkdir ./data
sudo docker-compose up -d
```

在等待的过程中，请你前往你的域名 DNS 管理界面，添加一条 A 记录，将 `mail.example.com` 指向你服务器的 IP 地址
访问 `https://mail.example.com`，你会看到一个初始化界面 owo

（素素，没发现我们这个 compose 直接占用了 80/443 端口吗？）
（如果你的服务器上跑了别的服务，那么你可以修改这个 compose 文件，把这些端口改到别的端口上去，=w=）
（在后文中，我们会用 Docker 部署一个 Nginx 作为反向代理，这样就可以让多个服务共享 80/443 端口了。如果你使用这种方法，那么请你删除 ports 中的 80/443 端口映射）
~~（什么 Docker 仙人）（逃）~~

[caption id="attachment_1987" align="aligncenter" width="1024"]<img src="https://www.esaps.net/wp-content/uploads/2025/03/WechatIMG2-1024x592.webp" alt="Poste.io 的初始化界面" width="1024" height="592" class="size-large wp-image-1987" /> Poste.io 的初始化界面[/caption]

在这里，你需要将：
- `Mailserver hostname` 设置为前面你添加 A 记录的地址
- `Administrator email` 是你的管理员邮箱地址，按照 `名称` @ `你的域名主域名`
  `域名主域名` 指的是填入 `example.com` 酱紫得啦～ 也就是你在域名注册商购买的域名
- `Password` 就是你的密码啦~

当你完成了一切的设置之后，你将会看到这个页面
[caption id="attachment_1986" align="aligncenter" width="1024"]<img src="https://www.esaps.net/wp-content/uploads/2025/03/WechatIMG184-1024x595.webp" alt="Poste.io 管理页面" width="1024" height="595" class="size-large wp-image-1986" /> Poste.io 管理页面[/caption]

虽然现在你已经成功部署了 Web 服务端，但是你还需要设置一些东西，比如说你的域名邮箱的 MX 记录，这样你的邮件才能正常收发

### 3. 设置解析记录

为了让你的域名邮箱正常收邮件，需要设置以下DNS记录:

- **A 记录**: 将 `mail.example.com` 指向你的服务器IP地址，如果你按照上面的教程一直坐下来的话，你已经设置好了
  A 记录是用来指定域名对应的 IP 地址的，当有人访问你的域名时，DNS 服务器会将域名解析为对应的 IP 地址
- **MX 记录**: 将 `@` 或者域名本身指向 `mail.example.com`，优先级设置为 `10`
  MX 记录是用来指定邮件服务器的，当有人给你发邮件时，对方的邮件服务器会根据 MX 记录找到你的邮件服务器，然后将邮件发送到你的服务器
  其中，优先级越小，优先级越高，当有多个 MX 记录时，邮件服务器会按照优先级从高到低依次尝试发送邮件。这个值一般设置为10，20，30等等
- **CNAME 记录**: 将 `smtp.example.com` 、 `imap.example.com` 和 `pop.example.com` 指向 `mail.example.com`
  这样你的邮件客户端才可以使用这些常用域名连接到你的邮件服务器，当然，你也可以直接使用 `mail.example.com` 连接

当你按照上面的步骤设置好了 DNS 记录之后并且等待 DNS 服务器更新完成之后，你的域名邮箱就可以正常收邮件了。

等等，只能收邮件……？

是的，因为我们还没有设置 DKIM 等记录，你的邮件可能会被当作垃圾邮件被其他邮局抛弃。

### 4. 避免被当作垃圾邮件

在这一节开始之前，我们先简单介绍一下有关于 `邮件安全` 的几个重要概念：

> **DKIM（DomainKeys Identified Mail）**是一种用于验证邮件发送者身份的技术，其通过在邮件头部添加一个签名，接收方可以通过公钥验证签名的合法性，从而判断邮件是否来自合法的发送者。
> **SPF（Sender Policy Framework）**是另一种用于验证邮件发送者身份的技术，它通过在域名的 DNS 记录中添加一个 TXT 记录，指定允许发送邮件的 IP 地址，接收方可以通过查询 DNS 记录来验证邮件的合法性，防止有人冒充你的域名发送垃圾邮件。
> **DMARC（Domain-based Message Authentication, Reporting and Conformance）**是一种用于验证邮件发送者身份的技术，它通过在域名的 DNS 记录中添加一个 TXT 记录，指定邮件的验证策略，是丢弃还是放行，抑或是放到垃圾箱内，以及如何报告验证结果。

这些技术都是用来保护你的域名邮箱不会被攻击者冒充发送垃圾邮件，同时也可以提高邮件的送达率。
在这里，我们将会设置 DKIM、SPF 和 DMARC 记录，以确保你的邮件不会被当作垃圾邮件。

#### 4.1 设置 DKIM 记录

首先，我们需要生成 DKIM 密钥对，然后将公钥添加到 DNS 记录中。

在 Poste.io 管理页面中，点击左边栏的 `Virtual Domains`，然后点击其展示的你的邮件域名，接着点击 `DKIM Keys` 条目中的 `create a new key` 按钮后，你将会看到系统为你生成的 DKIM 密钥对。
[caption id="attachment_2009" align="aligncenter" width="1024"]<img src="https://www.esaps.net/wp-content/uploads/2025/03/31742230233_.pic_-1024x591.webp" alt="Poste.io Virtual Domains 管理页面" width="1024" height="591" class="size-large wp-image-2009" /> Poste.io Virtual Domains 管理页面[/caption]

[caption id="attachment_2010" align="aligncenter" width="1024"]<img src="https://www.esaps.net/wp-content/uploads/2025/03/41742230347_.pic_-1024x593.webp" alt="Poste.io 域名管理页面" width="1024" height="593" class="size-large wp-image-2010" /> Poste.io 域名管理页面[/caption]

[caption id="attachment_2011" align="aligncenter" width="1024"]<img src="https://www.esaps.net/wp-content/uploads/2025/03/51742230433_.pic_-1024x363.webp" alt="我生成出来的一个 DKIM 公钥" width="1024" height="363" class="size-large wp-image-2011" /> 我生成出来的一个 DKIM 公钥[/caption]

接着，我们需要将 DKIM 公钥添加到 DNS 记录中。

```bash
以上图为例子，我们需要添加一个 TXT 记录，其中：

主机名为 "s20250318762._domainkey"
注意，主机名应该填写你在 Poste.io 中生成的 DKIM 密钥对的 主机名，不要用我的主机名口牙

记录值为 "v=DKIM1; k=rsa; p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAq2iuiZ2ZnnW9YUU4kqJfoq/CD0i22tp30sidYsT8mvNig6EYjBSFDZ++IA0ILYdHBMtHuWO3sKDDo/6H9MEKpZN4iVDkvm4IIxqiMkISQk3kqUVCVyTLnudJQrTn+sAAnf5ZQKu6JJG636W2rGYOZdOnKWKaHqc1Eu7iuGECrQ35NHNwPE/yZRs06/ANgSplUcoUQZr0HBd9+NmwO1ePm24NnuIiE61yYS8NSonJPuxw3BDhK89cSKvY9UwIaVscSeWqmR0iDS1SjJ0dxj3e9GLkj/h7LJ6YnUZfwt1PM2XFOkEU2gJjgxKZGTLSon2wPI7W6R4KpSTVzc5AthI9IwIDAQAB"

简单解释一下各个字段的含义：
- v=DKIM1：表示使用 DKIM1 版本
- k=rsa：表示使用 RSA 算法
- p="" ：表示 DKIM 公钥，这里是我生成的一个 DKIM 公钥，你需要将你生成的 DKIM 公钥填入这里

注意，记录值需要在开头和结尾添加双引号，否则会被解析为多个记录，并且需要添加 "v=DKIM1; k=rsa;" 前缀
```

在 Cloudflare 中，你可以就像这样添加一个 TXT 记录：
[caption id="attachment_2013" align="aligncenter" width="1024"]<img src="https://www.esaps.net/wp-content/uploads/2025/03/61742230962_.pic_-1024x538.webp" alt="在 Cloudflare 添加 DKIM 记录的示例" width="1024" height="538" class="size-large wp-image-2013" /> 在 Cloudflare 添加 DKIM 记录的示例[/caption]

OK，现在你的 DKIM 记录已经设置好了，接下来我们需要设置 SPF 记录。

#### 4.2 设置 SPF 记录

SPF 记录是用来指定允许发送邮件的 IP 地址的，这样接收方的邮局系统就可以根据 SPF 记录来验证邮件的合法性。

SPF 总共分为三块：版本信息、机制 和 限定符。

我们以一个 SPF 记录来解释各个字段的含义：

```bash
v=spf1 mx ip4:192.168.1.1 include:example.com ~all
```

首先，`v=spf1` 表示使用 SPF1 版本，而后的 `mx` 表示允许 MX 记录中指定的 IP 地址发送邮件，`ip4:192.168.1.1` 表示允许指定的 IP 地址发送邮件，`include:example.com` 表示允许 `example.com` 的 IP 地址发送邮件，`~all` 表示如果邮件不符合 SPF 记录的规则，那么邮件会被标记为垃圾邮件。

主要讲一下限定符：

- `+` 表示要求对方的邮局系统允许通过 SPF 记录中列出的 IP 地址发送邮件
  一般来说，这个操作符是可以省略的，因为默认就是 `+`
- `-` 表示要求对方的邮局系统直接拒绝不符合条件的发件人
- `~` 表示如果邮件不符合 SPF 记录的规则，那么邮件会被标记为垃圾邮件
- `?` 表示如果邮件不符合 SPF 记录的规则，那么邮件会被标记为中性

结尾的 `all` 表示对于没有匹配到的 IP 地址的邮件的处理方式，一般来说，我们可以使用 `~all` 来表示允许 MX 记录中指定的 IP 地址发送邮件，如果邮件不符合 SPF 记录的规则，那么邮件会被标记为垃圾邮件。

这里就用 `v=spf1 mx ~all` 为例子，添加一个 SPF 记录：

```bash
主机名为 "@"
记录值为 "v=spf1 mx ~all"
```

在 Cloudflare 中，你可以就像这样添加一个 TXT 记录：
[caption id="attachment_2016" align="aligncenter" width="1024"]<img src="https://www.esaps.net/wp-content/uploads/2025/03/71742231969_.pic_-1024x527.webp" alt="Cloudflare SPF 设置示例" width="1024" height="527" class="size-large wp-image-2016" /> Cloudflare SPF 设置示例[/caption]

豪德，我们还差一个 DMARC 记录。

#### 4.3 设置 DMARC 记录

DMARC 记录是用来指定邮件的验证策略，是丢弃还是放行，抑或是放到垃圾箱内，以及如何报告验证结果。

DMARC 记录分为三个字段：版本信息、验证策略 和 报告地址。

我们以一个 DMARC 记录来解释各个字段的含义：

```bash
v=DMARC1; p=none; rua=mailto:dmarc@example.com; ruf=mailto:dmarc@example.com
```

- v=DMARC1：DMARC版本号，当前只有一个版本，所以固定为1。
- p=none：邮件验证策略，可以是以下几种值：
  - none：不采取任何行动，只生成报告
  - quarantine：将邮件放入垃圾箱
  - reject：直接拒绝邮件
- rua=mailto:dmarc@example.com
  - 报告地址，用于聚合报告接收地址，如发送数量、验证通过率等
- ruf=mailto:dmarc@example.com
  - 报告地址，用于失败报告接收地址，接收单次验证失败的详细信息

这里就用 `v=DMARC1; p=none; rua=mailto:dmarc@example.com; ruf=mailto:dmarc@example.com` 为例子，添加一个 DMARC 记录：

```bash
主机名为 "_dmarc"
记录值为 "v=DMARC1; p=none; rua=mailto:dmarc@example.com; ruf=mailto:dmarc@example.com"
```

在 Cloudflare 中，你可以就像这样添加一个 TXT 记录：

[caption id="attachment_2019" align="aligncenter" width="1024"]<img src="https://www.esaps.net/wp-content/uploads/2025/03/81742233217_.pic_-1024x528.webp" alt="Cloudflare DMARC 设置示例" width="1024" height="528" class="size-large wp-image-2019" /> Cloudflare DMARC 设置示例[/caption]

其实，如果你注意力惊人的话，你应该能发现 Cloudflare 有一个 DMARC 的设置页面，你可以直接在这个页面设置 DMARC 记录，而不用手动添加 TXT 记录。而且，Cloudflare 还会为你提供一个 DMARC 报告的页面，你可以在这个页面查看 DMARC 报告。
[caption id="attachment_2022" align="aligncenter" width="1024"]<img src="https://www.esaps.net/wp-content/uploads/2025/03/91742233304_.pic_-1024x481.webp" alt="Cloudflare DMARC 看板" width="1024" height="481" class="size-large wp-image-2022" /> Cloudflare DMARC 看板[/caption]


OK，现在你的域名邮箱已经设置好了 DKIM、SPF 和 DMARC 记录，你的邮件就不会被当作垃圾邮件了。
你可以使用 [Mail Tester](https://www.mail-tester.com/) 来测试你的邮件是否符合标准。

[caption id="attachment_2024" align="aligncenter" width="1024"]<img src="https://www.esaps.net/wp-content/uploads/2025/03/101742233621_.pic_-1024x567.webp" alt="一个合格的邮件示例 owo" width="1024" height="567" class="size-large wp-image-2024" /> 一个合格的邮件示例 owo[/caption]

但是，这并不安全，我们是不是忘记给我们的邮件服务器添加 SSL 证书了？

### 5. 使用 Poste.io 添加 SSL 证书

在 Poste.io 管理页面中，点击左边栏的 `System settings`，然后点击 `TLS Certificate`，点击 `issue free letsencrypt.org certificate` 按钮。

[caption id="attachment_2027" align="aligncenter" width="1024"]<img src="https://www.esaps.net/wp-content/uploads/2025/03/111742234064_.pic_-1024x873.webp" alt="在 Poste.io 申请免费的 SSL 证书" width="1024" height="873" class="size-large wp-image-2027" /> 在 Poste.io 申请免费的 SSL 证书[/caption]

在弹出的页面中，`Common name` 填写你的域名，比如 `mail.example.com`，并且在 `Alternative names` 中填写你的其他域名，比如 `smtp.example.com`、`imap.example.com` 和 `pop.example.com`，然后点击 `Save Changes` 按钮。过一会儿，你的 SSL 证书就会生成好了。

[caption id="attachment_2028" align="aligncenter" width="1024"]<img src="https://www.esaps.net/wp-content/uploads/2025/03/121742234179_.pic_-1024x730.webp" alt="获取 SSL 证书吧！" width="1024" height="730" class="size-large wp-image-2028" /> 获取 SSL 证书吧！[/caption]

如果出现错误，你可以尝试删除 `Alternative names` 中的域名，然后再次尝试申请 SSL 证书。

现在完成！你的域名邮箱已经可以正常收发邮件了，而且还是自有免费的！

如果你需要 Nginx 反向代理的话，请继续往下看 owo

### 6. 使用 Nginx 反向代理

> Docker!
> --AptS:1547

我们这么假设，服务器是一台 ~~All In Boom~~ 全能服务器，我们把很多的网站都开在这个服务器上。
老板给了你一个需求，你要能够让用户访问 `HTTP/HTTPS` 服务的默认端口访问全部网站，并且还要根据主机名称对应到不同的站点。
在这个时候，我们需要一个反向代理程序（如 Nginx）来帮助我们解决这个问题。

#### 6.1 前置内容

首先，让我们创建必要的目录结构：

```bash
sudo mkdir -p nginx_website/website_file
sudo mkdir -p nginx_website/config
```

并且新建一个 Docker Network

```bash
sudo docker network create nginx-proxy-network
```

#### 6.2 创建 docker-compose.yml

以下是一个用于部署 Nginx 反向代理的 docker-compose.yml 文件：

```yaml
volumes:
  nginx-website-file:             # Nginx 网站文件 Volume
    name: nginx-website-file
    driver: local
    driver_opts:
      o: bind
      type: none
      device: ./nginx_website/website_file
  nginx-website-config:           # Nginx 配置文件 Volume
    name: nginx-website-config
    driver: local
    driver_opts:
      o: bind
      type: none
      device: ./nginx_website/config

services:
  nginx-website:
    container_name: nginx-website
    image: nginx:1.24.0
    restart: always
    hostname: example.com         # 修改为你的主域名
    ports:
      - 443:443                   # HTTPS 端口
      - 80:80                     # HTTP 端口
    networks:
      - server_for_public
    environment:
      - NGINX_HOST=example.com  # 修改为你的主域名
      - TZ=Asia/Shanghai        # 时区设置
    volumes:
      - nginx-website-file:/usr/share/nginx/html  # 网站文件
      - nginx-website-config:/etc/nginx:ro        # Nginx 配置文件（只读）
      - /path/to/ssl/:/etc/nginx/conf.d/ssl/:ro   # SSL 证书目录（只读），请根据你的实际路径修改

networks:
  server_for_public:
    driver: bridge
  nginx-proxy-network:
    external: true  # 用于连接其他服务的网络
```

吼吼，又是一个梦幻联动。

[mdx_post]https://www.esaps.net/use-acme-sh-to-apply-for-ssl-certificates/[/mdx_post]

你可以根据这篇文章去设置 Nginx 的 SSL 证书，以及设置自动化续签管理

完成之后，使用 `docker compose up -d` 先启动 Nginx 容器，并让他生成默认的配置文件出来

#### 6.3 创建邮件服务器的反向代理配置

创建 `nginx_website/config/conf.d/mail.conf` 文件，并在下方填入以下内容

```txt
server {
    listen 80;
    # 修改为你的邮件服务器实际域名
    # 这里将 smtp imap pop 添加进去是为了 poste.io 能够去获取这些域名的证书
    server_name mail.example.com imap.example.com pop.example.com smtp.example.com; 

    return 301 https://mail.example.com$request_uri;  # 修改为你的邮件服务器实际域名
}

server {
    listen 443 http2 ssl;

    # 修改为你的邮件服务器实际域名
    server_name imap.example.com pop.example.com smtp.example.com;
    
    # 请填写证书文件的相对路径或绝对路径
    # 这里的 /etc/nginx/conf.d/ssl/ 对应着你在 compose 文件中设置的宿主机文件夹
    ssl_certificate /etc/nginx/conf.d/ssl/fullchain.pem;
    # 请填写私钥文件的相对路径或绝对路径
    ssl_certificate_key /etc/nginx/conf.d/ssl/server.key;

    ssl_session_timeout 5m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_prefer_server_ciphers on;

    add_header Strict-Transport-Security "max-age=31536000"; # 可选安全头
    add_header X-Content-Type-Options nosniff;               # 可选安全头

    return 301 https://mail.example.com$request_uri;         # 访问其他域名直接跳转网页邮箱界面
}

server {
    listen 443 http2 ssl;

    server_name mail.example.com; # 修改为你的邮件服务器实际域名

    # 请填写证书文件的相对路径或绝对路径
    # 这里的 /etc/nginx/conf.d/ssl/ 对应着你在 compose 文件中设置的宿主机文件夹
    ssl_certificate /etc/nginx/conf.d/ssl/fullchain.pem;
    # 请填写私钥文件的相对路径或绝对路径
    ssl_certificate_key /etc/nginx/conf.d/ssl/server.key;

    ssl_session_timeout 5m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_prefer_server_ciphers on;

    add_header Strict-Transport-Security "max-age=31536000"; # 可选安全头
    add_header X-Content-Type-Options nosniff;               # 可选安全头

    # 反向代理配置
    location / {
        proxy_pass http://mail.example.com;  # 指向你的邮件服务器容器，填写容器名称即可
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_hide_header X-Powered-By;
        proxy_read_timeout 90;
    }
}
```

保存后我们还需要更改 Poste.io 容器的配置

#### 6.4 修改 Poste.io 容器的配置

需要修改之前的 Poste.io 容器配置，移除 80 和 443 端口映射，并将其添加到 Nginx 的网络中：

```yaml
services:
  mailserver:
    hostname: mail.example.com
    container_name: mail.example.com
    image: analogic/poste.io
    restart: always
    ports:
      - 25:25     # SMTP
      # 删除 80 和 443 端口映射
      - 465:465   # SMTPS
      - 587:587   # Submission
      - 110:110   # POP3
      - 995:995   # POP3S
      - 143:143   # IMAP
      - 993:993   # IMAPS
      - 4190:4190 # Sieve
    networks:
      - nginx-proxy-network  # 添加到 Nginx 网络
    environment:
      - TZ=Asia/Shanghai
      - DISABLE_CLAMAV=TRUE
      - DISABLE_RSPAMD=TRUE
      - DISABLE_ROUNDCUBE=TRUE
    volumes:
      - mail-data:/data

networks:
  nginx-proxy-network:
    external: true  # 用于连接其他服务的网络
```

并且重启 Poste.io 容器

```bash
sudo docker compose up -d
```

#### 6.5 重载 Nginx 反向代理

最后，重新载入 Nginx 的配置文件

```bash
sudo docker exec nginx-website nginx -s reload
```

现在，通过 Nginx 反向代理，你可以同时运行多个服务，包括你的邮件服务器，所有服务都能通过各自的域名使用 80 和 443 端口。

注意：反向代理后，需要确保 Poste.io 正确处理转发的请求头，特别是 X-Forwarded-For 和 X-Real-IP，以便正确识别客户端 IP 地址。

一切完成！享受专属于你的邮局系统吧！
