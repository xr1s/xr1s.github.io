+++
title = "通过 Matrix-WeChat 将微信消息转发至 Beeper"
date = "2024-11-26"
type = "post"
tags = ["Miscellaneous"]
+++

平时不怎么用微信，原因是微信会导致我给 PowerToys 配置的快捷键映射全部失效，具体参考这个 [PowerToys Issue](https://github.com/microsoft/PowerToys/issues/21877)。

总之，虽然 Issue 里已经解决了这个问题，但是长期的习惯下来，我也不怎么在 Windows 上挂微信了。根本原因还是我微信上朋友很少。

不过最近因为特殊原因被迫 24 小时挂微信，得知有通过 [matrix-wechat](https://github.com/duo/matrix-wechat) 将微信消息转发到 Matrix 的方法，因此尝试了一下。按照项目作者写的一个[简单的教程](https://duo.github.io/posts/matrix-qq-wechat/)，可以使用他的 docker image 快速 docker-compose up 一系列服务。

不过因为本人懒+穷，不原因自建一个 matrix homeserver，于是采用群友建议的 Beeper 服务器方案，成功将微信消息转发至 Beeper.com 上，也可以使用任意 Matrix 客户端接受消息。

## 概念

Matrix 是一个公开的分布式加密通信协议。只要是实现了 Matrix 协议的服务器，都可以使用 Matrix 协议互相通信。通过这个协议，在 A Matrix 服务器上注册的用户同样可以和 B Matrix 服务器上注册的用户互相发送消息、处于同一个群组。

最常见的 Matrix 服务器是 matrix.org，当然本文会用到 Beeper 的服务器 beeper.com。同时由于 matrix 协议是公开的，任何人都可以自建自己的服务器，因此有无数的 Homeserver 的存在。

我们一般通过 Matrix 客户端去 Matrix 服务器获取信息展示出来，一个比较知名的 Matrix 客户端包括 [Element.io](app.element.io)，可以直接用网页端登录。

<!-- 用户的 ID——一般简写作 MXID——由两部分组成，第一部分是用户自己注册的 ID 前缀以 `@`，第二部分是服务器名。这两部分用冒号分割 `:`。比如我的 MXID 是 `@xr1s:matrix.org`。 -->

那么逻辑就比较清晰了，我们是希望有一个自建的 Homeserver，为每一个微信好友创建一个虚拟 Matrix 账号，然后将微信消息转发到这个 Homeserver 上。这样的话，就可以用任意的 Matrix 客户端读取，而不必须在每个手机电脑上登录微信了。只是为了在外地也能读到某个 Homeserver 上的信息，这个 Homeserver 就必须要可以公网访问，这样 Matrix 客户端才能访问到。

这里的 Beeper 就起到了这个 Homeserver 的作用。因为 Beeper 为每个账号提供了一个 Hungry Server，因此不需要用户自己提供公网 IP、不需要用户自己提供域名，本地有一台电脑连上 Beeper 就可以跑 Bridge。

Matrix-WeChat 是 Bridge，它的作用是桥接本地应用 WeChat 和 Matrix 服务器。Bridge 也被称作 AppService。

## 准备

需要在 [beeper.com](https://www.beeper.com) 下载 Beeper 客户端后注册一个账号，注册非常简单，只需要提供邮箱和一个唯一 ID 也就是用户名。不需要记忆密码，免费，不要有心理负担。

后文假设你注册的用户名是 `xr1s`，由于 Beeper 原生支持 Matrix 协议，因此你直接获得了一个 Matrix 账号 `@xr1s:beeper.com`，可以在任何 Matrix 客户端登录。顺便欢迎来 [Coder OT](https://matrix.to/#/#coder_ot:matrix.org) 玩玩。

下载 Beeper 官方的 Bridge Manager [bbctl](https://github.com/beeper/bridge-manager)，它的作用是连接 Matrix Bridge 和 Beeper 服务器。

然后还需要下载 [matrix-wechat](https://github.com/duo/matrix-wechat) 和 [matrix-wechat-agent](https://github.com/duo/matrix-wechat-agent) **源代码**。重点注意 Agent 不要下 Release 了，因为他 Agent Release 版本有 Bug，修复后没有发新版，除此之外，启动方式和文档中所写的不一样，它不会读配置文件，所有参数都需要命令行传递。Matrix-WeChat Release 可能没有问题，但是同一个作者，保险起见还是自己编译吧。对，我们需要编译，所以需要基础的 Go 开发环境，不过 Go 编译很简单。

将 matrix-wechat 编译成 Linux ELF，matrix-wechat-agent 编译成 Windows EXE。

```bash
GOOS=windows GOARCH=amd64 go build .
```

就可以交叉编译了，还是很简单的。

matrix-wechat-agent.exe 还需要几个 dll 来支持运行，去 [ljc545w/ComWeChatRobot](https://github.com/ljc545w/ComWeChatRobot) 下载 Release，目前的最新版本是 3.7.0.30-0.1.1-pre，其中 3.7.0.30 是作者反编译的微信版本，因此使用的协议也是该版本微信的协议。Release 是一个 zip，我们只需要其中的 http/wxDriver.dll、http/wxDriver64.dll 和 http/SWeChatRobot.dll 这三个文件。

最后，我们仍然需要下载 ComWeChatRobot 对应的微信版本并安装到 Windows 或者 Wine 中。可以在[微信官网](https://weixin.qq.com/cgi-bin/readtemplate?lang=zh_CN&t=weixin_faq_list&head=true)找历史版本。或者使用[微信官网的 Web Archive](https://web.archive.org/web/20220618182906/https://pc.weixin.qq.com/)下载 3.7.0.30 版本的微信。

```goat
+-------------------------------+ +-----------------------+
|                               | |                       |
|   +-------------------------+ | |  +---------------+    |
|   | matrix-wechat-agent.exe +-+-+->| matrix-wechat |    |
|   +-------------------------+ | |  +-------+-------+    |
|                               | |          |            |
|         Local Windows or Wine | |       .-'             |
+-------------------------------+ |      |                |
                                  |      v                |
    +--------------------------+  | +-------+             |
    | Beeper.com Matrix Server |<-+-+ bbctl |             |
    +--------------------------+  | +-------+ Local Linux |
                                  +-----------------------+
```

确认一下目前有这几个二进制可执行文件和动态加载库：

- bbctl
- matrix-wechat

- matrix-wechat-agent.exe
- wxDriver.dll
- wxDriver64.dll
- SWeChatRobot.dll


其中 bbctl 和 matrix-wechat 在 Linux 中运行，matrix-wechat-agent.exe 在 Windows 或者 Wine 中运行。

## 配置

### bbctl

首先使用 bbctl 登录刚注册的 Beeper 账号，很简单：

```bash
./bbctl login
```

然后输入 Beeper 发送到你邮箱中的验证码即可。

首先需要在 Beeper 服务器上注册一个 bridge，使用 `register` 子命令，后面跟着 appservice 名称。

前缀 `sh-` 是必要的，用于 Beeper 识别为自建 Self-Hosting Bridge，后面的名字就是 bot 名，自己随便取就行，不会有冲突（每个 Beeper 账号下独立，同账号下不能有同名 bridge，不同账号 bridge 同名无所谓）。

```bash
./bbctl register sh-wx
```

Beeper 会返回给你一个 Registration 文件，是给 bbctl 使用的，先保存下来，随便放到一个文件里都可以，比如 registration.yaml 就行，可以不放到 bbctl 同一个目录下，命令行传参指定配置文件。不过为了方便管理，也不是不行。

```yaml
id: 00000000-0000-0000-0000-000000000000
url: http://localhost:17778
as_token: hua_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX_XXXXXX
hs_token: huh_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX_XXXXXX
sender_localpart: sh-wxbot
rate_limited: false
namespaces:
    users:
        - regex: '@sh-wx_.+:beeper\.local'
          exclusive: true
```

其中 `id`、`as_token`、`hs_token` 都是用来和 Beeper 服务器验证身份的。后续也同样需要填入到 Bridge 配置文件中。

`url` 是本地 Matrix Bridge 监听的地址，需要和 Matrix Bridge 配置文件中的 `appservice.hostname` 与 `appservice.port` 一致。到文章目前还没生成，但是可以先填写默认值 `http://localhost:17778`，后续可以根据自己需要同时修改两处。

`sender_localpart` 是 Bridge 在 Beeper 服务器上的 Bot 名称，在 Bridge 配置文件中要保持一致。

`namespaces.users` 是 Bridge 在 Beeper 上能使用的用户名，Bridge 生成的用户 MXID 需要匹配这个正则，可以在 Bridge 配置文件中配置。

### Matrix WeChat

在 matrix-wechat 所在目录直接执行下面的代码，生成一个 config.yaml 模板，需要注意的是这里配置文件必须要和二进制在同目录了：

```bash
touch config.yaml
./matrix-wechat
```

因为 config.yaml 是空，所以程序会报错退出。但是没关系，我们主要是用到 matrix-wechat 往 config.yaml 写入的默认配置（如果本来不存在这个文件，它是不会创建的，所以需要手动创建一个）。

下面是几个需要注意修改的字段，其它没有提及的字段不用动就好：

```yaml
homeserver:
    address: https://matrix.beeper.com/_hungryserv/xr1s
    domain: beeper.local
    software: hungry
appservice:
    database:
        type: sqlite3-fk-wal
        uri: "file:matrix-wechat.data?_txlock=immediate"
    id: "00000000-0000-0000-0000-000000000000"
    bot:
        username: sh-wxbot
    as_token: "hua_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX_XXXXXX"
    hs_token: "huh_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX_XXXXXX"
bridge:
    hs_proxy: ""
    username_template: sh-wx_{{.}}
    permissions:
        "@xr1s:beeper.com": admin
    encryption:
        allow: true
        default: true
        appservice: true
```

`homeserver.address` 的值和 `bridge.permissions` 的用户名可以通过调用 `./bbctl whoami` 获得。

`appservice.database` 可以先用 SQLite 快速把 Bridge 先跑起来，后续再自行配置 PostgreSQL。SQLite 也能跑，就是性能低，这篇文章就不讲解 PostgreSQL 相关知识了。

`appservice.address` 和 `appservice.hostname`、`appservice.port` 是 Bridge 本地监听地址，这三个值要保持一致，同时修改。如果要修改的话，同样需要同步在 bbctl 的 Registration 文件中修改。

`appservice.id`、`appservice.as_token`、`appservice.hs_token` 都是 Beeper 分配给你的 ID 和密钥，都在 `./bbctl register` 中可以看，也就是上面的 Registration 文件，保存好就行。

`appserivce.bot.username` 要和 bbctl 中 register 的 `sender_localpart` 相匹配，是你的管理 Bot 名称，一切配置完成后在 Matrix 客户端私聊 Bot 就可以发指令控制了。

`bridge.hs_proxy` 不是什么重要的东西，只是为了去掉一句报错才加的这一行，没有也可以跑。

`bridge.username_template` 是 appservice 注册的用户 MXID 模板，需要注意要和 registration.yaml 中 `namespaces` 匹配，`namespaces` 是 Beeper 服务器为 AppService 预留的用户名格式，如果不符合格式的话，会出现为微信好友注册 Matrix 账号失败的情况。

`bridge.permisions` 是配置哪个用户对该 Bridge Bot 拥有什么样的权限，一般来说只要给自己配置 admin 就行了。

`bridge.encryption` 主要是桥到端加密信息的配置，是为了避免主服务器（也就是 Beeper）能读到我们微信聊天的明文。

另外注意 `bridge.listen_address` 和 `bridge.secret` 这两个字段，是自定义配置的 IP 端口和密钥，是留给 matrix-wechat-agent.exe 通信的。目前可以暂时先保留默认值 `0.0.0.0:20002`、`foobar` 不改动。

### Matrix WeChat Agent

配置文件直接用仓库里 example 稍微改改就行，同样需要放到可执行文件同目录下 configuration.yaml。

```yaml
wechat:
  version: 3.7.0.30
  listen_port: 22222
  init_timeout: 10s
  request_timeout: 30s

service:
  addr: ws://localhost:20002
  secret: foobar
  ping_interval: 30s

log:
  level: info
```

建议是改一改 `service.addr` 和 `service.secret`，当然 matrix-wechat 的配置文件中对应字段也需要同步修改。

## 启动

```bash
./bbctl proxy -r registration.yaml
./matrix-wechat
```

```powershell
.\matrix-wechat-agent.exe`
```

Agent 会启动一个 WeChat 实例，可以把它扔到 Wine 里眼不见心不烦。

需要注意 Agent 必须要自己编译了，不然会报奇怪的错误：

```
Error in appservice websocket: json: cannot unmarshal number into Go struct field WebsocketMessage.type of type string
Error in appservice websocket: websocket: close 1006 (abnormal closure): unexpected EOF
```

## Matrix 客户端初始化

服务启动完成后，随便找一个 Matrix 客户端登录 Beeper 账号 `@xr1s:beeper.com`（当然，你可以直接用 Beeper 自己的客户端，但是我不喜欢）。

私聊 `@sh-wxbot:beeper.local`。注意这里的 `sh-wxbot` 来自 bbctl 注册 bridge 时候使用的名称，储存在 Registration 文件的 `sender_localpart` 字段。beeper.local 是固定的，Beeper 服务器会为每个用户隔离出独立的 beeper.local，也就是说每个用户访问 beeper.local 的时候，Beeper 服务端都会使用用户名去找该用户注册的 Bridge。

如果启动成功一切顺利的话，这时候是能直接搜到这个 Bot 的，去发一条消息，比如 `help`，有回应就成功一半了。然后发送 `login`，会发送一张二维码让你登录微信。登录成功后最好分别使用私人（文件传输助手就行）和群聊发送消息测试一下。应该来说是没啥问题的。

## 收尾

需要注意的是如果 matix-wechat 或者 matrix-wechat-agent 退出，那后续重启就需要重新扫码登录了。似乎是微信的限制，所以目前最好是保持常开。

如果需要删除 Beeper 上的服务器，使用 `./bbctl delete sh-wx` 即可。记得把本地的 SQLite 数据文件或者 PostgreSQL 数据表清空，否则下次注册可能会遇到服务器上仍未创建的私聊或者群聊，matrix-bridge 读数据库发现存在，就不为你创建新的账号或房间，直接收发消息导致报错的问题。

然后可以 Docker 化这些配置，我还是懒，反正逻辑就是上面这样，Docker 搞起来也不麻烦。
