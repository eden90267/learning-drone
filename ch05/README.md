# Drone 外掛撰寫

## 撰寫 Bash Shell Script

為什麼要寫 Plugin ?

1. 擴充 Yaml 參數

  一個 EC2 發布後要寄發 notification
 的案例，透過必要的參數幫你處理，而非每個專案都有一樣的 shell script
 來執行一樣的事情

2. 相同情境可以作 plugin 分享給其他人

### Plugin 四大步驟

- 撰寫程式
- 打包 Docker Image
- 上傳到 Docker Hub
- 測試外掛

### 自訂 Plugin

任何語言都可以寫 Plugin：PHP、Ruby、Bash、Go、Python ...

因為每個步驟都是 docker container 方式，所以沒有限制

### 撰寫 Bash Shell Script Plugin

```yaml
pipeline:
  webhook:
    image: foo/webhook
    url: http://foo.com # 自訂參數
    method: post        # 自訂參數
    body: |             # 自訂參數
      hello world
```

```shell
#!/bin/sh

curl \
  -X ${PLUGIN_METHOD} \
  -d ${PLUGIN_BODY} \
  ${PLUGIN_URL}
```

## 打包 Docker Image 並上傳到 Docker Hub

```shell
#!/bin/sh

curl \
  -X ${PLUGIN_METHOD} \
  -d ${PLUGIN_BODY} \
  ${PLUGIN_URL}
```

```dockerfile
FROM alpine
ADD post.sh /bin/
RUN chmod +x /bin/post.sh
RUN apk -Uuv add curl ca-certificates
ENTRYPOINT /bin/post.sh
````

```shell
$ docker build -t eden90267/webhook .
$ docker image | grep foo
$ docker login
$ docker push eden90267/webhook
```

## 測試 Drone 外掛

```shell
$ docker run --rm \
    -e PLUGIN_METHOD=post \
    -e PLUGIN_URL=https://postman-echo.com/post \
    -e PLUGIN_BODY="hello=world" \
    foo/webhook
```