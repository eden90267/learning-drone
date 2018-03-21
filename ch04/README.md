# Drone 外掛介紹

## 打包檔案上傳 (SCP Plugin)

Drone 沒有內建打包部署功能，請善用 plugin
Marketplace：[http://plugins.drone.io/](http://plugins.drone.io/)

### SCP Plugin

[https://github.com/appleboy/drone-scp](https://github.com/appleboy/drone-scp)

```yaml
scp_Dev:
  image: appleboy/drone-scp
  pull: true # 可以省略
  host: 192.168.1.131 # 可以寫多台
  port: 22
  username: spock
  secrets:
    - source: ssh_key
      target: scp_key
  target: /home/spock/drone/${DRONE_REPO} # 遠端目錄
  source:
    - app.tag.gz # 打包列表，可以多行
  when:
    branch: develop
    status: [ success ]
```

### 實例

```yaml
workspace:
  base: /node
  path: drone/node-example

clone:
  git:
    image: plugins/git
    depath: 50
    tags: true

pipeline:
  frontend:
    image: node:8.3.0
    group: test
    commands:
      - echo "test frontend"

  backend:
    image: node:8.3.0
    group: test
    commands:
      - echo "test backend"

  scp:
    image: appleboy/drone-scp
    host:
      - 128.199.179.216
#      - 128.199.179.216
    username: root
#    password: 1234 # 1.
#    secrets: [ ssh_password ] # 2.
#    secrets: [ ssh_key ] # 3.
    secrets:
      - source: ssh_key_drone_Demo # 4. key 不同把，但目標得要是 ssh_key 命名
        target: ssh_key
    target: /home/drone/demo # 沒的話 plugin 會自動幫你建好
    source:
      - env.js
      - redis.js
```

password 明碼在上面不好，建議用 secrets (UI 建立)

## 執行伺服器指令 (SSH Plugin)

[https://github.com/appleboy/drone-ssh](https://github.com/appleboy/drone-ssh)

```yaml
ssh_dev:
  image: appleboy/drone-ssh
  pull: true
  host: 192.168.1.131
  port: 22
  username: spock
  secrets: [ssh_key]
  script:
    - rm -rf ${DRONE_REPO} && mkdir -p ${DRONE_REPO}
    - tar -zxmf /home/spock/drone/${DRONE_REPO}/app.tar.gz -C ${DRONE_REPO}
    - rm -rf /var/www/html/app
    - cp -r ${DRONE_REPO}/app /var/www/html/
  when:
    branch: develop
    status: [success]
```

### 實例

```yaml
workspace:
  base: /node
  path: drone/node-example

clone:
  git:
    image: plugins/git
    depath: 50
    tags: true

pipeline:
  scp:
    image: appleboy/drone-scp
    host:
      - 128.199.179.216
    username: root
    password: 1234
    target: /home/appleboy/demo
    source:
      - env.js
      - redis.js
      - yarn.lock
 
  ssh:
    image: appleboy/drone-ssh
    host:
      - 128.199.179.216
    username: root
#    password: 1234
    secrets: [ ssh_key ]
    script:
      - echo "Hi I am appleboy"
      - ls -al /home/appleboy/demo/
```

## 上傳映象檔到 Public Registry (像是 Docker Hub)

```yaml
publish_latest:
  image: plugins/docker
  pull: true
  repo: ${DRONE_REPO}
  tags: ['latest']
  secrets: [docker_username, docker_password]
  when:
    event: [push]
    branch: [master]
    local: false
```

### 實例

```yaml
workspace:
  base: /node
  path: drone/node-example

clone:
  git:
    image: plugins/git
    depath: 50
    tags: true

pipeline:
  publish:
    image: plugins/docker # 官方在維護的
    group: publish
    repo: eden90267/drone-nodejs-example
    tags: [latest]
    dockerfile: Dockerfile
    secrets: [docker_username, docker_password]

  publish:
    image: plugins/docker
    group: publish
    repo: eden90267/drone-nodejs-example-2
    tags: [latest]
    dockerfile: Dockerfile.arm
    secrets: [docker_username, docker_password]
```

## 上傳映像檔到 Private Registry (像是 Harbor)

```yaml
publish:
  image: plugins/docker
  insecure: true # 走 http protocol，所以要設為 true，如果走 https 可直接拿掉
  registry: harbor.wu-boy.com
  repo: harbor.wu-boy.com/test/demo
  dockerfile: Dockerfile #  多個的話要指定要多個 pipeline
  secrets: [docker_username, docker_password]
  tags: [latest, 1.1]
```

harbor 由 vmware 出的 private registry，底層用 docker registry 2.0 配上 UI
介面，有專案管理、使用者權限等

如何使用你的私有的 docker image：

```yaml
golang:
  image: harbor.wu-boy.com/test/golang:1.9.0-alpine3.6
  group: testing
  commands:
    - go version
```

drone 後台新增 registry：host name、username 與 password，所以不需要
secrets 的部分

## 搭配 kubernetes 自動化部署

- Drone-Kubernetes*：用 shell script 下 kubernetes 的指令更新 pod
  做升級，可直接更新 pod 的 image tag
- Drone-Kube：用 golang 下去寫的，搭配 k8s API 直接吃 deploy 的 yaml
  檔案更新 kubernetes 的設定

### 範例

```yaml
publish:
  image: plugins/docker
  repo: appleboy/k8s-node-demo
  dockerfile: Dockerfile
  secrets:
    - source: demo_username
      target: docker_username
    - source: demo_password
      target: docker_password
  tags: [latest, '${DRONE_TAG}']
  when:
    event: tag

deploy:
  image: quay.io/honestbee/drone-kubernetes
  namespace: demo # 在 k8s yaml 事先要建立好的
  deployment: k8s-node-demo # 在 k8s yaml 事先要建立好的
  repo: appleboy/k8s-node-demo # 在 k8s yaml 事先要建立好的
  container: k8s-node-demo # 在 k8s yaml 事先要建立好的
  tag: ${DRONE_TAG} # 根據 push 上去的 tag
  secrets:
    - source: k8s_server
      target: plugin_kubernetes_server
    - source: k8s_cret
      target: plugin_kubernetes_cert
    - source: k8s_token
      target: plugin_kubernetes_token
  when:
    event: tag
```

在 k8s yaml 事先要建立好的 (deployment.yml)：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: k8s-node-demo
  namespace: demo
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: k8s-node-demo
        tier: frontend
    spec:
      containers:
      - image: appleboy/k8s-node-demo
        name: k8s-node-demo
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 3

---
apiVersion: v1
kind: Service
metadata:
  name: k8s-node-demo
  namespace: demo
  labels:
    app: k8s-node-demo
spec:
  selector:
    app: k8s-node-demo
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

```shell
$ kubectl create -f k8s # k8s 是資料夾，裡面有 deployment.yaml 檔案
```

## 消息通知 (Discord 範例)

Discord 類似 Slack 通訊軟體

建立 channel > Edit Channel > Webhooks > 建立一個 Webhooks，並 copy 其 URL

再來修改 .drone.yaml

```yaml
install:
  image: node:8.6.0
  commands:
    - exit 2
    - node -v
    - npm -v
    - yarn --version
    - yarn config set cache-folder .yarn-cache
    - yarn install --pure-lockfile

discord:
  image: appleboy/drone-discord
  secrets: [discord_webhook_ikd, discord_webhook_token]
  when:
    status: [success, failure]
```