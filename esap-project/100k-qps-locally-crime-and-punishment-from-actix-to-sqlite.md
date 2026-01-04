> 在云的尽头，优化才刚刚开始。
> 本文并非一篇完整的 benchmark，也不是什么“业界最佳实践”。它更像是某个开发者凌晨 2 点对着火焰图流泪、摸索 SQLite 调优、怒开 Brotli 和缓存之后，所写下的一份修仙笔记。
> “我只是想跑得快一点，没想到把自己搞到了 CPU 饱和。”

本来只是给朋友部署一个短链服务，想着用 Rust 写个 Actix Web 框架 + SQLite 轻量持久化就完事了。结果压测一跑：**5k QPS**，还有瓶颈。

## 一、Actix Web + SQLite，能撑多少？

起初的 naive 实现大概长这样：

```rust
async fn resolve(path: web::Path<String>) -> impl Responder {
    let db = DB.get().unwrap();
    let target = db.query(&path).await?;

    match target {
        Some(url) => HttpResponse::TemporaryRedirect()
            .insert_header(("Location", url))
            .finish(),
        None => HttpResponse::NotFound().finish(),
    }
}
```

数据库用 SQLite，开启 WAL 模式，并发由 `r2d2` 连接池控制。轻量、小而美，部署一行命令打包 Docker 镜像就能跑。

但一压测，问题就来了。

```
wrk -t4 -c100 -d60s http://localhost:8080/esap
```

死健忘症忘记截图。。。

**结果：1w QPS，瓶颈不在 CPU，也不在数据库。**

## 二、SQLite 不是瓶颈，但优化仍有必要

SQLite 默认配置并不适合高并发访问，即使是只读场景，也需要手动开启 WAL 模式、调低同步等级、调大 mmap size 等优化。

我启用了如下 PRAGMA：

```rust
c.execute_batch(
    "PRAGMA synchronous = NORMAL;
     PRAGMA cache_size = -64000;
     PRAGMA temp_store = memory;
     PRAGMA mmap_size = 536870912;
     PRAGMA journal_mode = WAL;
     PRAGMA wal_autocheckpoint = 1000;
     PRAGMA busy_timeout = 5000;
     PRAGMA optimize;",
)?;
```

* `NORMAL` 同步等级牺牲极小的事务持久性换来更快写入；
* `WAL` 模式避免了锁冲突；
* `mmap_size` 用内存映射替代频繁磁盘 I/O；
* `cache_size` 和 `temp_store` 提高内存利用率。

最终调出来之后，磁盘压力明显降低，查询命中也更稳定了，火焰图上 SQLite 相关函数不再是 CPU 黑洞。
**也是成功从 1w qps 到 2w qps 了**
感谢 `Litestream`，让我发呆三分钟后直接抄出来

## 三、内核火焰图：sendto？

性能还可以更好吗？

我开了 `flamegraph` 抓了一张火焰图，发现大量时间卡在 `__sendto` 系统调用。

[caption id="attachment_2236" align="aligncenter" width="304"]<img src="https://www.esaps.net/wp-content/uploads/2025/06/1748972892543.webp" alt="__sendto 是什么，能吃吗" width="304" height="238" class="size-full wp-image-2236" /> __sendto 系统调用水平占用[/caption]

我一开始以为是返回 JSON 没压缩，网络带宽成了瓶颈。

后来才意识到，**我根本没返回正文！**

* 成功是 `307 Temporary Redirect`，只有 `Location` header；
* 失败是 `404 Not Found` 或 `204 No Content`，一样没有 body。

真正的瓶颈是大量小响应（几十字节）在高并发下触发了过多 `sendto` 系统调用。虽然每次只发几行 header，但 syscall 成本扛不住了。

于是我试了一个骚操作：**启用 response compression（开启 Brotli）**。

虽然响应很小，压缩空间不大，但压完后 header 本身都能省几个字节。**最终结果：QPS 从 2w 干到了 3w+ **

> 有时候，压缩不仅仅是压正文，也是减少 syscall 的“触发面积”。

## 四、缓存是关键：Moka 上场

虽然 SQLite 足够快，但毕竟是文件数据库。

我后来用 [Moka](https://docs.rs/moka/) 加了一层异步缓存，QPS 提升非常明显。原因也很简单：只要缓存命中，连 SQLite 都不用查，直接构造响应 header 返回。

内存缓存 + header-only 响应，**QPS 飙升到 23 万+，性能瓶颈反而转向 CPU。**

```
wrk -t4 -c100 -d60s http://localhost:8080/esap

Requests/sec: 231980.35
Transfer/sec: 39.82MB
```

[caption id="attachment_2239" align="aligncenter" width="1024"]<img src="https://www.esaps.net/wp-content/uploads/2025/06/cc987af0-ac45-48d5-985c-f7a2538bcfe3-1024x591.webp" alt="23w!" width="1024" height="591" class="size-large wp-image-2239" /> 23w![/caption]

## 五、总结：本地 QPS 十万不是梦

这个项目最终形成了以下优化路径：

1. **SQLite 优化：** WAL + mmap + cache 配置，压榨极限吞吐；
2. **异步缓存：** Moka 减少数据库访问，秒杀大部分请求；
3. **压缩优化：** 即便无正文，Brotli 依旧减少 header 大小，降低 syscall 压力；
4. **轻量框架：** Actix Web 本身极快，配合 Rust zero-cost 异步模型，天作之合；

你可能会问：

> 为啥你要这么极限优化一个短链跳转？

因为我想证明一件事：

**不是所有高性能服务都得上 Redis + MySQL + CDN。
一个用 Rust + SQLite 的小项目，在本地，也可以跑到十万 QPS。**

甚至，**性能瓶颈从不在技术，而在你有没有拆解问题的勇气。**

---

> 开发者笔记：
>
> 本项目源码和测试环境已开源，详见 [GitHub: AptS-1547/shortlinker](https://github.com/AptS-1547/shortlinker)
>
> 写文案时使用了火焰图、wrk 测试、flamegraph 等工具，记录每一步真实性能演化过程。欢迎讨论与交流。

喵。
