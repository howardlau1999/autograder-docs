# 后端架构介绍

本系统使用 Go 语言开发，前后端使用 gRPC-Web 进行通信，API 协议定义在 `proto/api.proto` 文件中。后端可以嵌入前端 Web 网站资源，这样只使用一个二进制就可以启动服务。后端服务除了提供 Web 访问界面以及接口以外，还负责评测任务的调度和评测机管理。

目前，考虑到系统一般用于 100 人左右的课堂教学，并发访问量以及总的数据量都不大，为了方便部署，数据存储使用了 `Pebble` KV 数据库，数据均存储在本地。如果有需要，可以修改 `repository` 下面的文件，实现不同 `Repository` 接口，对接其他的如 SQL 数据库存储。

## API 鉴权

除了登录注册等少数接口外，所有接口都需要进行用户鉴权以及访问鉴权。为了复用代码，将鉴权逻辑抽象了出来，使用 Interceptor 功能统一处理，相关代码位于 `pkg/grpc/interceptor.go` 中。

鉴权首先会从请求头中获取 `authorization` 头，并检查 JWT Token 是否有效，然后，后续会针对不同的功能进行鉴权。例如检查用户是否有权限修改课程、创建作业等。

在鉴权过程中获取的一些信息，例如用户在某个课程内的身份，会保存在 `Context` 中，后续的处理函数可以使用对应的 Key 从 `Context` 中获取。例如，用户登录信息会使用 `userInfoCtxKey` 保存在 `Context` 中，可以使用 `ctx.Value(userInfoCtxKey{}).(*model_pb.UserTokenPayload)` 获取。

## 文件上传与下载

由于 gRPC 并不适合用于传输文件，因此所有的文件都是用普通的 HTTP 协议传输的。在上传和下载文件前，需要先发送一个 gRPC 请求获取 Token，然后使用该 Token 向服务器发起文件传输的 HTTP 请求。

```protobuf
message InitUploadRequest {
  uint64 manifest_id = 1;
  string filename = 2;
  string token = 3;
  uint64 filesize = 4;
}

message InitUploadResponse {
  string token = 1;
}

message InitDownloadRequest {
  uint64 submission_id = 1;
  string filename = 2;
  bool is_directory = 3;
  bool is_output = 4;
}

enum DownloadFileType {
  Binary = 0;
  Text = 1;
  Image = 2;
  PDF = 3;
}

message InitDownloadResponse {
  string token = 1;
  DownloadFileType file_type = 2;
  string filename = 3;
  int64 filesize = 4;
}

service AutograderService {
    rpc InitUpload(InitUploadRequest) returns (InitUploadResponse);
    rpc InitDownload(InitDownloadRequest) returns (InitDownloadResponse);
}
```

其中上传请求中的 `manifest_id` 需要先使用 `CreateManifest` 请求创建临时的上传目录，并获取 ID 以及将目录绑定给用户。

```protobuf
message CreateManifestRequest {
  uint64 assignment_id = 1;
}

message CreateManifestResponse {
  uint64 manifest_id = 1;
}
```

其中 `assignment_id` 用来获取该作业的上传限制，后续添加文件的时候，总的大小不能超过该限制。

用户上传的文件都将存储在本地。对于上传了的文件，系统会定时检查，如果超过一定时间没有提交，则会自动删除，以节省空间。

## 评测任务管理

由于评测机和后端服务可能不是部署在同一台机器上，他们之间使用 gRPC 进行通信，并使用 HTTP 协议传输文件。用于评测机通信的端口和用于 Web 服务的端口不同，且 gRPC 和 HTTP 端口是分开的。也就是说后端服务需要占用三个端口。

### 创建提交

用户在文件上传对话框上传完文件并点击提交后，将根据 `manifest_id` 以及 `assignment_id` 将文件归档、创建提交记录，并生成一个评测任务发送给评测机进行评测。

```protobuf
message CreateSubmissionRequest {
  uint64 assignment_id = 1;
  uint64 manifest_id = 2;
}

message CreateSubmissionResponse {
  uint64 submission_id = 1;
}

service AutograderService {
    rpc CreateSubmission(CreateSubmissionRequest) returns (CreateSubmissionResponse);
}
```

在创建提交的时候，后端服务会先获取本次上传的所有文件列表并保存到提交信息中，然后获取 `assignment_id` 对应的评测配置，将该提交标记为待评测状态后，调用评测服务将评测任务添加到调度队列中。然后，启动单独的线程，监听评测服务发送的评测报告。监听线程收到评测报告后，会先将报告写入数据库，然后查找订阅了该提交的用户，将评测报告实时发送给用户。

### 订阅提交

为了给用户及时的反馈，评测报告将通过一个长连接实时发送给用户展示。对于未完成的提交，可以通过 `SubscribeSubmission` 这个接口订阅信息。

```protobuf
message SubscribeSubmissionRequest {
  uint64 submission_id = 1;
}

message SubscribeSubmissionResponse {
  uint64 score = 1;
  uint64 maxScore = 2;
  SubmissionStatus status = 3;
  PendingRank pending_rank = 4;
}

service AutograderService {
    rpc SubscribeSubmission(SubscribeSubmissionRequest) returns (streams SubscribeSubmissionResponse);
}
```

无论评测是否运行中，在订阅请求发送后，后端服务会先发送一次提交的当前状态。如果评测任务已经结束（评测完成、被取消、发生错误），则会立即关闭连接。否则，后端服务会将连接添加到订阅列表，接收评测报告。评测结束后同样会关闭连接并清理资源。

### 评测任务调度

在后端服务器中，会在内存中保存所有待评测的提交任务，并在后台调度评测任务。评测任务调度是事件驱动的，以下事件发生的时候将触发任务调度：

- 提交任务被添加到待评测队列
- 有提交任务完成了评测或被取消
- 有新的评测机上线

在每一轮调度中，后端服务会遍历待调度的提交任务，筛选出符合提交标签的评测机，并根据评测机上报的并发度将其分配给空闲的评测机。如果没有空闲的评测机，则会等待直到有空闲的评测机上线或者有其他任务结束。

当一个任务分配到了评测机后，后端服务会先将该任务以及对应的评测机 ID 写入数据库，并将该任务移除出调度队列，加入运行队列。在收到评测结束的报告后，后端服务会将评测结果写入数据库，并将该任务从运行队列中移除，清理所有相关资源。

### 评测机下线处理

如果一台评测机在评测过程中丢失心跳或关闭了心跳连接，后端服务将认为该评测机下线并标记其为“离线”状态。同时，会将已经调度给该评测机所有提交重新添加到调度队列中。

### 评测机上线处理

在评测机上线后，后端会将接收其注册请求，并将对应信息写入数据库。然后标记其为“在线”状态，并触发调度器尝试进行调度。

### 服务重启处理

如果在有评测任务还没结束运行的时候重启了服务进程，服务进程会先在启动后扫描数据库中未完成的提交，并将其重新添加到调度队列。当评测任务结束运行之后，会清理未完成标记。

## 评测文件传输

评测机使用单独的 HTTP 端口上传和下载文件。HTTP 请求的路径就是文件的路径，`GET` 请求为下载，`POST` 请求为上传。文件内容都是直接放在 HTTP Body 里，无需 Multipart Form 编码。为了保证接口安全，需要在每个请求头中携带评测 Token，和评测机注册时的 Token 保持一致。

## 服务指标监控

后端服务还集成了 Prometheus 客户端，并再使用一个单独的 HTTP 接口提供监控数据。