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