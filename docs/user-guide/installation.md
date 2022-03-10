# 安装配置指南

## 安装 Docker

在安装 Autograder 前，需要先安装 Docker。Windows、Linux、macOS 都有相应的平台支持，按照[官方文档](https://docs.docker.com/engine/install/#supported-platforms)进行安装即可。

其中，Windows 平台需要至少 Windows 10 以上的版本，同时需要启用 Hyper-V 技术，有可能与其他虚拟机软件冲突。如果您正在使用其他的虚拟机软件，例如 VMware 或者 VirtualBox，需要查询对应的文档，将虚拟机软件更新到与 Hyper-V 技术兼容的版本。

## 安装 Web 服务

Autograder 以单独的静态可执行二进制文件分发，无需额外的安装步骤，仅需要下载 Autograder 程序，初始化数据库，创建配置文件，并运行即可。

下面以 Linux 为例。假设您已经将二进制文件下载到了 `~/autograder-server` 目录下，首先执行以下命令，初始化数据库并创建配置文件模板：

```bash
./autograder-server --init --email 您的邮箱地址
```

运行后，当前目录会创建 `db` 和 `storage` 文件夹以及文件，`db` 文件夹为 Autograder 存放用户信息等的数据库，`storage` 文件夹用来存放用户上传文件、运行结果等。

在初始化成功后，将输出随机生成的 root 用户密码，请保存到合适的地方，后续需要使用该密码登录 Autograder，登录后可以修改。

之后，可以运行 `./autograder-server --config` 输出配置模板文件。您可以将输出重定向到文件里，文件名必须为 `config.toml`。用文本编辑器打开 `config.toml` 文件并填写以下信息：

### 配置评测机服务

为了保证通信安全，服务器和评测机通信前需要先通过 Token 验证。Token 可以为任意的字符串，建议不少于 32 个字符。

```toml
[hub]
	port=9999 # 评测服务监听端口
	token="" # 评测密钥
    heartbeat-timeout="15s" # 心跳超时，建议设置为略大于评测心跳间隔
```

### 配置 Web 服务

可以配置监听的端口。

```toml
[web]
	port=8080 # Web 服务监听端口
```

### 配置存储

#### 配置数据库路径

```toml
[db.local]
    path="db" # 数据库存储路径，可以为任意路径，需要读写权限
```

#### 配置本地存储

```toml
[fs.local]
	dir="" # 存储目录，可以为任意路径，需要有读写权限
```

#### 配置评测机文件服务

为了保证通信安全，同样需要配置 Token。建议使用随机字符串，不少于 32 个字符。

```toml
[fs.http]
	port=19999 # 监听端口
	token="" # 鉴权密钥
```

### 配置 SMTP 服务器

在 Autograder 中，Email 是用来重置密码以及发送通知的唯一通道，为了保证邮件正常发送，需要配置 SMTP 服务器，对应配置文件中的 `[smtp]` 部分：

```toml
[smtp]
    addr="" # SMTP 服务器的地址，例如 smtp.mailgun.org:
    user="" # SMTP 服务器认证用户
    pass="" # SMTP 服务器认证密码
    from="" # 发信时使用的发件人名称，例如 "Autograder <no-reply@mail.howardlau.me>"
```

具体的配置信息可以在您的邮件服务提供商文档处查询。

### 配置 hCaptcha 验证码服务

为了避免服务器遭受恶意注册用户的攻击，Autograder 支持使用 hCaptcha 验证码服务来防御攻击。您需要在 [hCaptcha 网站](https://dashboard.hcaptcha.com/login) 注册账户，登录后到 [hCaptcha 控制面板](https://dashboard.hcaptcha.com/sites?page=1)获取您的 Site Key，同时需要到 [hCaptcha 账户设置页面](https://dashboard.hcaptcha.com/settings)获取 Secret Key，以便服务器验证。

```toml
[hcaptcha]
    site-key="" # 填入你的 Site Key
    secret-key="" # 填入你的 Secret Key
```

### 配置 GitHub 第三方登录服务（可选）

首先到 [https://github.com/settings/applications/new](https://github.com/settings/applications/new) 创建一个 GitHub OAuth 应用，其中，Name 可以填写任意的名称，Homepage URL 填写您的域名或 IP 地址，例如 `http://autograder.howardlau.me`，Callback URL 填写域名或 IP 后添加 `/github`，例如 `http://autograder.howardlau.me/github`。填写完成后，将 Client ID 填写到配置文件中，同时点击 “Generate a new client secret” 按钮创建一个 Client Secret，并将生成的 Client Secret 填写到配置文件中。需要注意一旦关闭页面后将无法再次查看该 Client Secret，只能重新生成。可以在 [https://github.com/settings/developers](https://github.com/settings/developers) 查看你所有的应用。

```toml
[github]
    client-id="" # 填写应用的 Client ID
    client-secret="" # 填写生成的 Client Secret
```

## 安装评测服务

评测服务同样以单个二进制分发，无需安装，只需要配置好后端服务地址即可运行。

运行 `./autograder-grader --config` 可以输出配置模板，建议重定向到 `config.toml` 文件。为了避免和后端服务的配置文件冲突，可以将文件保存到 `~/.autograder-grader/config.toml`（Windows 为 `C:\.autograder-grader\config.toml`）。

### 配置评测机信息

```toml
[grader]
	concurrency=5 # 评测机并发数，最多可以同时运行几个评测任务
	tags="docker,x64" # 评测机标签，用","分割
	heartbeat-interval="10s" # 评测机心跳间隔，必须小于后端服务配置的心跳超时
```

### 配置后端服务信息

```toml
[hub]
	address="localhost:9999" # 后端服务地址
	token="" # 连接密钥
```

### 配置存储

#### 配置本地存储

在运行评测任务的过程中需要临时使用本地存储提交文件以及输出文件，在评测任务结束后会自动清理。

```toml
[fs.local]
	dir="grader" # 本地存储路径
```

#### 配置远程存储

评测机会通过 HTTP 协议上传和下载文件。

```toml
[fs.http]
	url="http://localhost:19999" # 后端存储服务地址
	token="" # 连接密钥
	timeout="10s" # 请求超时
```

配置完成后，启动 Docker 服务以及后端程序，再启动评测机程序即可。
