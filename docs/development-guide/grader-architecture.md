# 评测架构介绍

为了提供良好的可扩展性，评测服务被设计为单独的可执行程序，并通过 RPC 接口与服务程序进行通信。

考虑到服务器程序同时负责 Web 服务的运行，我们认为服务器程序运行在拥有可访问的 IP 地址的服务器上（例如云服务器，或者一个可路由的内网地址）。同时，为了节省 IP 地址，评测服务设计为本身不占用可路由的 IP 地址和端口。因此，评测程序本身无需设置为监听端口，也不能被动接收 RPC 请求。也就是说，仅能让评测服务主动连接服务器程序。一般这种情况下，如果评测程序需要获取评测任务，则需要主动轮询服务器程序，或者使用 HTTP Long Polling 技术。如果想要获得更好的实时性和节省反复创建和销毁连接的开销，则可以使用 WebSocket 类似的 TCP 长连接技术。由于项目使用了 gRPC 技术，所以评测获取评测任务，都通过一个 gRPC 双向 Stream RPC 调用来实现，也就是使用了长连接进行通信。

## 注册评测服务

在评测服务开始接收请求前，需要将一些基本信息注册到服务器上，以便调度器根据这些信息进行评测任务的调度。

```protobuf
message GraderInfo {
  string hostname = 1; // 评测服务的主机名，不同评测服务应该使用不同的主机名
  repeated string tags = 2; // 评测服务的标签，用来匹配评测任务
  uint64 concurrency = 3; // 评测任务的并发数，必须大于 0，否则该服务无法调度评测任务
}

message RegisterGraderRequest {
  string token = 1; // 服务器进程配置文件的鉴权 Token
  GraderInfo info = 2;
}

message RegisterGraderResponse {
  uint64 grader_id = 1; // 服务器分配的评测服务 ID
}

service GraderHubService {
  rpc RegisterGrader(RegisterGraderRequest) returns (RegisterGraderResponse);
}
```

当服务进程收到注册请求后，会首先检查连接密钥是否正确，然后会检查该主机名是否已经有在线的评测机，如果有，则返回错误。如果没有，则将评测机提供的信息注册到服务器上，并返回该服务器的 ID。后续的 RPC 调用都将通过这个 ID 进行标识。

## 评测心跳

注册成功后，评测服务将需要建立和服务器的心跳连接，并定时发送心跳包，以便服务器知道评测机是否还在线。同时，服务器程序将通过该心跳连接向评测机主动发送 RPC 请求。

```protobuf
message GraderHeartbeatRequest {
  google.protobuf.Timestamp time = 1;
}

message GradeRequest {
  uint64 submission_id = 1; // 提交 ID
  Submission submission = 2; // 提交的信息
  ProgrammingAssignmentConfig config = 3; // 评测配置
  bool is_cancel = 4; // 是否为取消评测或停止实时日志传输
  bool is_stream_log = 5; // 是否为请求实时日志的请求
  string request_id = 6; // 如果是请求实时日志，则为用来区分不同客户端的 ID，否则为空
}

message GraderHeartbeatResponse {
  repeated GradeRequest requests = 1;
}

service GraderHubService {
  rpc GraderHeartbeat(stream GraderHeartbeatRequest) returns (stream GraderHeartbeatResponse);
}
```

在建立心跳连接时，应当在 gRPC 头部包含 `graderId` 字段，值为注册时返回的 ID。评测机应当定时发送心跳，并接收服务器发来的请求，进行处理。

## 处理评测请求

若 `GradeRequest` 中的 `is_stream_log` 为 `false`，说明这是一个评测请求，按照处理评测请求的流程处理。

### 请求评测

若 `is_cancel` 为 `false`，说明该请求为一个评测请求，评测服务此时应当根据评测配置启动评测任务。启动任务后，应当另外建立一个 gRPC 连接，用来向服务进程发送评测结果。

```protobuf
message GradeReport {
  SubmissionReport report = 1;
  SubmissionBriefReport brief = 2;
  DockerGraderMetadata docker_metadata = 3;
  PendingRank pending_rank = 4;
}

message GradeResponse {
  uint64 submission_id = 1;
  GradeReport report = 2;
}

message GradeCallbackResponse {

}

service GraderHubService {
  rpc GradeCallback(stream GradeResponse) returns (GradeCallbackResponse);
}
```

在评测任务结束或被取消时，应该关闭连接，以便服务程序清理相关资源。

### 取消评测

若 `is_cancel` 为 `true` 说明该请求为取消评测请求，评测服务应当取消当前正在进行的评测任务，并通过上述建立的结果连接返回一个取消成功的消息。

## 处理实时日志请求