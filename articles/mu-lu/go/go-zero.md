# go-zero



## go-zero环境搭建

1. 安装goctl
2. 安装protocbuf\
   [https://github.com/protocolbuffers/protobuf/releases?page=5](https://github.com/protocolbuffers/protobuf/releases?page=5)\
   下载后将压缩包内的protoc移到/usr/local/bin目录
3. protoc-gen-go \
   &#x20;go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
4.  protoc-gen-go-grpc&#x20;

    go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest



### api文件

```go-module
syntax = "v1"

info (
	author:"go-zero"
	date:"2023-08-02"
	desc:"api demo"
)

type (
	UserInfoReq {
		UserId int64 `json:"userId"`
	}
	UserInfoResp {
		UserId   int64  `json:"userId"`
		Nickname string `json:"nickname"`
	}
)

service user-api {
	@doc(
		summary: "获取用户信息"
	)

	@handler userInfo
	post /user/info (UserInfoResp) returns (UserInfoReq)

}
```

执行以下命令生成代码

```
goctl api go --api *.api -dir ../ --style=goZero
```

生成docker file

```
goctl docker -go user.go
```

