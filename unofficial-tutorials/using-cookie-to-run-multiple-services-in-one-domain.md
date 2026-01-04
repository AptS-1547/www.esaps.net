> 前几天有个小登的网站需要我帮忙部署，使用的是前后端分离的架构，`VUE + Quart Flask`。同时他还有别的应用（比如 `Nextcloud`、`Jenkins` 等）需要部署到同一个三级域名上。在尝试子目录部署无果后，1547 决定采用 `Nginx Cookie` 判断分流的方法

## 1. Cookie 是什么？

`Cookie` 是 HTTP 协议中用于在客户端和服务器之间存储小块数据的机制。它们通常用于存储用户的会话信息、偏好设置和其他状态信息。`Cookie` 由服务器发送到客户端，并在后续请求中由客户端发送回服务器。
`Cookie` 的基本结构如下：

```txt
Set-Cookie: <cookie-name>=<cookie-value>; Expires=<date>; Path=<path>; Domain=<domain>; Secure; HttpOnly
```

简单理解，`Cookie` 就是一个变量，通过 `HTTP` 请求头部传递给浏览器，浏览器会在后续的请求中将这个变量带上。

那么，既然是个变量，我们当然可以通过读取变量的值来知道用户需要什么服务了。

## 2. Nginx 又是什么？

Nginx 是一个高性能的 HTTP 和反向代理服务器，支持多种协议。它被广泛用于负载均衡、缓存和静态文件服务等场景。Nginx 的配置文件使用一种类似于 C 语言的语法，允许用户定义服务器的行为和处理请求的方式。

## 3. Nginx Cookie 分流

Nginx Cookie 分流是指根据请求中的 Cookie 值来决定将请求转发到哪个后端服务。通过设置不同的 Cookie 值，可以实现对不同服务的请求进行分流处理。

> 提问：为什么会想到用 Cookie 分流而不是根据 URL 路径来分流？
> 答：因为 Nextcloud 前端获取文件的路径都 TM 写死了。同时，前端 API 请求路径写死了，只能发送到他指定的 API 路径。
> 这就导致了我们无法通过 URL 路径来分流请求。

**不支持路径部署的都不是好应用.jpg**

在这里我们需要用到 `Nginx` 的 `map` 指令来实现 Cookie 分流。`map` 指令可以根据请求中的 Cookie 值设置一个变量，然后在 `location` 块中使用这个变量来决定请求的转发目标。
下面是一个简单的示例：

```nginx
map $cookie_X_Backend_Id $backend {
    default        http://wordpress:80;
    "openwebui"    http://openwebui-ollama:8080;
    "nextcloud"    http://nextcloud:80;
}

server {
    listen 80;
    server_name a-lot-of-services.1547.me;

    resolver 127.0.0.11 valid=30s;

    location / {
        proxy_pass $backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

在上面的示例中，我们先使用 `map` 指令定义了一个变量 `$backend`，他会根据 Cookie 中 `X_Backend_Id` 的值来设置不同的后端服务地址并赋值给 `$backend` 变量。

然后在 `location` 块中，我们使用 `$backend` 变量来决定请求的转发目标。

- 如果 Cookie 中没有 `X_Backend_Id`，则默认转发到 `http://wordpress:80`。
- 如果 Cookie 中包含 `X_Backend_Id=openwebui`，则转发到 `http://openwebui-ollama:8080`。
- 如果 Cookie 中包含 `X_Backend_Id=nextcloud`，则转发到 `http://nextcloud:80`。

这样，我们就实现了根据 Cookie 值来分流请求的功能。

需要说明的是，`resolver 127.0.0.11 valid=30s;` 这一行是为了让 `Nginx` 能够动态解析 Docker 内部的 DNS 名称，防止出现 `502 Bad Gateway` 错误。
在生产环境中，你应该按需配置。

## 4. 测试

在完成配置后，我们需要测试一下是否能够正常访问不同的服务。我们可以使用 `curl` 命令来模拟请求，并查看返回的结果。

```bash
# 测试访问 WordPress，没有 Cookie 或者 Cookie 没有被匹配上
curl -H http://a-lot-of-services.1547.me
# 测试访问 OpenWebUI
curl -H "Cookie: X_Backend_Id=openwebui" http://a-lot-of-services.1547.me
# 测试访问 Nextcloud
curl -H "Cookie: X_Backend_Id=nextcloud" http://a-lot-of-services.1547.me
```

如果一切正常，我们应该能够看到不同服务的响应结果。

## 5. 一些小扩展

### 5.1 允许用户选择服务

目前我们的配置文件仅支持在 Nginx 服务器上直接设置 Cookie 值来分流请求，这对于普通用户来说并不友好。
我们可以写一个 `HTML` 页面，提供一个下拉框让用户选择需要访问的服务，然后通过 JavaScript 设置 Cookie 值。

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>服务选择</title>
    <script>
        function setCookie(name, value, days) {
            var expires = "";
            if (days) {
                var date = new Date();
                date.setTime(date.getTime() + (days * 24 * 60 * 60 * 1000));
                expires = "; expires=" + date.toUTCString();
            }
            document.cookie = name + "=" + (value || "") + expires + "; path=/";
        }

        function selectService() {
            var service = document.getElementById("service").value;
            setCookie("X_Backend_Id", service, 7);
            window.location.href = "http://a-lot-of-services.1547.me";
        }
    </script>
</head>
<body>
    <h1>选择要访问的服务</h1>
    <select id="service" onchange="selectService()">
        <option value="">请选择服务</option>
        <option value="openwebui">OpenWebUI</option>
        <option value="nextcloud">Nextcloud</option>
    </select>
</body>
</html>
```

在上面的代码中，我们使用 JavaScript 设置了一个 Cookie 值 `X_Backend_Id`，然后跳转到 Nginx 服务器的地址。用户选择不同的服务后，Cookie 值会被设置为对应的服务名称并跳转到域名根目录，这样子用户就可以通过下拉框选择需要访问的服务了。
在我实际的生产环节中，我是通过修改一个 `txt` 文件来实现动态添加修改服务的功能的，你可以自己实现一下。

### 5.2 假如，我是说假如，你要用 default 来指引到 VUE 项目

直接贴配置文件

```nginx
    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        if ($backend != "") {
            proxy_pass $backend;
            break;
        }
        
        root /usr/share/nginx/html/bot-frontend;
        try_files $uri $uri/ /index.html;
    }
```

在上面的代码中，我们使用了 `if` 指令来判断 `$backend` 变量是否为空，如果不为空，则转发到对应的后端服务，否则就直接访问 VUE 项目。
这样我们就实现用 default 来指引到 VUE 项目了。
~~(因为小登的网站是他们学校的正规军，其实是不能开其他非业务服务的，得要用这种方法来打个掩护)~~

## 6. 总结

通过使用 Nginx 的 Cookie 分流功能，我们可以在同一个三级域名上部署多个服务，并根据用户的选择来决定请求的转发目标。这种方法不仅灵活，而且易于扩展，可以根据实际需求进行调整。
当然，使用 Cookie 分流也有一些限制，比如需要用户手动设置 Cookie 值，或者需要在前端页面中提供选择服务的功能。
在实际应用中，我们可以根据具体的需求和场景来选择合适的分流方式。
这种是属于比较极端的情况，应该在没有域名解析权限的情况下才会使用这种方法。
希望这篇文章能对你有所帮助，如果你有任何问题或者建议，欢迎在评论区留言讨论。
