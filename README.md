# webqqircd

webqqircd类似于bitlbee，在WebQQ(SmartQQ)和IRC间建起桥梁，可以使用IRC客户端收发消息。大部分代码来自[wechatircd](https://github.com/MaskRay/wechatircd)，为适配QQ做了一些修改，去除了wechatircd中的token，因此只支持单客户端。

## 原理

修改WebQQ用的JS，通过WebSocket把信息发送到服务端，服务端兼做IRC服务端，把IRC客户端的命令通过WebSocket传送到网页版JS执行。未实现IRC客户端，因此无法把QQ群的消息转发到另一个IRC服务器(打通两个群的bot)。

## 安装

需要Python 3.5或以上，支持`async/await`语法
`pip install -r requirements.txt`安装依赖

Arch Linux可以安装<https://aur.archlinux.org/packages/webqqircd-git>，会自动在`/etc/webqqircd/`下生成自签名证书(见下文)，导入浏览器即可。

## 运行

### HTTPS、WebSocket over TLS

推荐使用TLS。

- `openssl req -newkey rsa:2048 -nodes -keyout a.key -x509 -out a.crt -subj '/CN=127.0.0.1' -dates 9999`创建密钥与证书。
- Chrome访问`chrome://settings/certificates`，导入a.crt，在Authorities标签页选择该证书，Edit->Trust this certificate for identifying websites.
- Chrome安装Switcheroo Redirector扩展，把<http://pub.idqqimg.com/smartqq/js/mq.js>重定向至<https://127.0.0.1:9002/mq.js>。若js更新，该路径会变化。
- `./webqqircd.py --tls-cert a.crt --tls-key a.key`，会监听127.1:6668的IRC和127.1:9002的HTTPS与WebSocket over TLS

![](https://maskray.me/static/2016-04-11-webqqircd/demo.jpg)

### HTTP、WebSocket

如果嫌X.509太麻烦的话可以不用TLS，但Chrome会在console里给出警告。

- 执行`./webqqircd.py`，会监听127.1:6668的IRC和127.1:9002的HTTP与WebSocket，HTTP用于伺服项目根目录下的`mq.js`。
- 把<http://pub.idqqimg.com/smartqq/js/mq.js>重定向至<http://127.0.0.1:9002/mq.js>。若js更新，该路径会变化。
- 把`mq.js` `var ws = new MyWebSocket('wss://127.0.0.1:9002')`行单引号里面的部分修改成`ws://127.0.0.1:9002`

### IRC客户端

- IRC客户端连接127.1:6668(weechat的话使用`/server add qq 127.1/6668`)，会自动加入`+status` channel
- 登录<http://w.qq.com>
- 回到IRC客户端，可以看到QQ朋友加入了`+status` channel，在这个channel发信并不会群发，只是为了方便查看有哪些朋友。
- QQ朋友的nick优先选取备注名(`RemarkName`)，其次为`DisplayName`(原始JS根据昵称等自动填写的一个名字)

在`+status` channel可以执行一些命令：

- `help`，帮助
- `status`，已获取的QQ朋友、群列表
- `eval $password $expr`: 如果运行时带上了`--password $password`选项，这里可以eval，方便调试，比如`eval $password client.uin2qq_user`

若服务端或客户端重启，刷新WebQQ。

## IRC命令

webqqircd是个简单的IRC服务器，可以执行通常的IRC命令，可以对其他客户端私聊。

以下命令会有特殊作用：

- 程序默认选项为`--join auto`，收到某个QQ群的第一条消息后会自动加入对应的channel，即开始接收该QQ群的消息。
- `/join [channel]`表示开始接收该QQ群的消息
- `/list`，列出所有QQ群
- `/names`，更新当前群成员列表
- `/part [channel]`的IRC原义为离开channel，转换为QQ代表在当前IRC会话中不再接收该QQ群的消息。不用担心，webqqircd并没有主动退出群的功能
- `/query nick`打开与`$nick`的私聊窗口，与之私聊即为在QQ上和他/她/它对话
- `/who channel`，查看群成员列表

## JS改动

原始文件`mq.js`在Chrome DevTools里格式化后得到`orig/mq.pretty.js`，可以用`diff -u orig/mq.pretty.js mq.js`查看改动。

修改的地方都有`//@`标注，结合diff，方便WebQQ更新后重新应用这些修改。增加的代码中大多数地方都用`try catch`保护，出错则`consoleerr(ex.stack)`。

目前的改动如下：

### `mq.js`开头

创建到服务端的WebSocket连接，若`onerror`则自动重连。监听`onmessage`，收到的消息为服务端发来的控制命令：`send_text_message`等。

### 定期把通讯录发送到服务端

获取所有联系人(朋友、订阅号、群)，`deliveredContact`记录投递到服务端的联系人，`deliveredContact`记录同处一群的非直接联系人。

每隔一段时间把未投递过的联系人发送到服务端。

### 收到QQ服务器消息`messageProcess`

原有代码会更新未读标记数及声音提醒，现在改为若成功发送到服务端则不再提醒，以免浏览器的这个标签页造成干扰。

## Python服务端代码

当前只有一个文件`webqqircd.py`，从miniircd抄了很多代码，后来自己又搬了好多RFC上的用不到的东西……

```
.
├── Web                      HTTP(s)/WebSocket server
├── Server                   IRC server
├── Channel
│   ├── StandardChannel      `#`开头的IRC channel
│   ├── StatusChannel        `+status`，查看控制当前QQ会话
│   └── QQRoom               QQ群对应的channel，仅该客户端可见
├── (User)
│   ├── Client               IRC客户端连接
│   ├── QQUser               QQ用户对应的user，仅该客户端可见
├── (IRCCommands)
│   ├── UnregisteredCommands 注册前可用命令：NICK USER QUIT
│   ├── RegisteredCommands   注册后可用命令
```

## 我的配置

<https://wiki.archlinux.org/index.php/Systemd/User>

`~/.config/systemd/user/webqqircd.service`:
```
[Unit]
Description=webqqircd
Documentation=https://github.com/MaskRay/webqqircd
After=network.target

[Service]
WorkingDirectory=%h/projects/webqqircd
ExecStart=/home/ray/projects/webqqircd/webqqircd.py --tls-key a.key --tls-cert a.crt --password a --ignore 不想自动加入的群名0 不想自动加入的群名1

[Install]
WantedBy=multi-user.target
```

WeeChat:
```
/server add qq 127.1/6668 -autoconnect
```
