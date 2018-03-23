# 實戰 Node.js 專案

## Node.js 專案測試 (mocha + eslint)

### Node.js 測試及部署流程

- git clone 專案 (drone 預設步驟)
- 專案測試 (選擇 node 環境)
  - mocha test
  - eslint
- 專案打包 (drone-scp plugin)
- 部署到遠端 server (drone-ssh plugin)
- 消息推送 (成功或失敗)

```yaml
# .drone.yml
pipeline:
  install:
    image: node:8.6.0
    commands:
      - node -v
      - npm -v
      - yarn --version
      - yarn config set cache-folder .yarn-cache
      - yarn install --pure-lockfile

  testing:
      image: node:8.6.0
      group: testing
      commands:
        - yarn run test
        - sleep 5

  lint:
      image: node:8.6.0
      group: testing
      commands:
        - yarn run lint
        - sleep 5
```

## Node.js 專案快取檔案 (加速測試)

加速 yarn (npm) install 速度

- remote file cache
- local file cache *

```yaml
# .drone.yml
pipeline:
  restore-cache:
    image: drillster/drone-volume-cache
    restore: true
    mount:
      - ./.yarn-cache
      - ./node_modules
    volumes:
      - /tmp/cache:/cache

  install:
    image: node:8.6.0
    commands:
      - node -v
      - npm -v
      - yarn --version
      - yarn config set cache-folder .yarn-cache
      - yarn install --pure-lockfile

  testing:
      image: node:8.6.0
      group: testing
      commands:
        - yarn run test
        - sleep 5

  lint:
      image: node:8.6.0
      group: testing
      commands:
        - yarn run lint
        - sleep 5

rebuild-cache:
    image: drillster/drone-volume-cache
    rebuild: true
    mount:
      - ./.yarn-cache
      - ./node_modules
    volumes:
      - /tmp/cache:/cache
    when:
      branch: master # 任何 branch or PR 不用 cache rebuild
```

Project Settings 要開 Trusted，因為有操作到 disk，有權限的要求；而 SFTP Cache 就不需要

## Node.js 專案測試及部署流程

```yaml
# .drone.yml
pipeline:
  restore-cache:
    image: drillster/drone-volume-cache
    restore: true
    mount:
      - ./.yarn-cache
      - ./node_modules
    volumes:
      - /tmp/cache:/cache

  install:
    image: node:8.6.0
    commands:
      - node -v
      - npm -v
      - yarn --version
      - yarn config set cache-folder .yarn-cache
      - yarn install --pure-lockfile

  testing:
      image: node:8.6.0
      group: testing
      commands:
        - yarn run test
        - sleep 5

  lint:
      image: node:8.6.0
      group: testing
      commands:
        - yarn run lint
        - sleep 5

  scp:
    image: appleboy/drone-scp
    host:
      - 128.199.163.232
#      - 128.199.163.233
    port: 22
    username: root
    target: /root/drone/${DRONE_REPO}
    secrets: [ssh_key] # 也可用 password 方式
    source:
#      - release.tar.gz # 多檔案的話會用 Makefile 處理壓縮相關
      - "*.js"
      - node_modules
    when:
      branch: master


rebuild-cache:
    image: drillster/drone-volume-cache
    rebuild: true
    mount:
      - ./.yarn-cache
      - ./node_modules
    volumes:
      - /tmp/cache:/cache
    when:
      branch: master # 任何 branch or PR 不用 cache rebuild
```

## Node.js 專案部署 (drone-ssh 外掛)

```yaml
# .drone.yml
pipeline:
  restore-cache:
    image: drillster/drone-volume-cache
    restore: true
    mount:
      - ./.yarn-cache
      - ./node_modules
    volumes:
      - /tmp/cache:/cache

  install:
    image: node:8.6.0
    commands:
      - node -v
      - npm -v
      - yarn --version
      - yarn config set cache-folder .yarn-cache
      - yarn install --pure-lockfile

  testing:
      image: node:8.6.0
      group: testing
      commands:
        - yarn run test
        - sleep 5

  lint:
      image: node:8.6.0
      group: testing
      commands:
        - yarn run lint
        - sleep 5

  scp:
    image: appleboy/drone-scp
    host:
      - 128.199.163.232
#      - 128.199.163.233
    port: 22
    username: root
    target: /root/drone/${DRONE_REPO}
    secrets: [ssh_key] # 也可用 password 方式
    source:
#      - release.tar.gz # 多檔案的話會用 Makefile 處理壓縮相關
      - "*.js"
      - process.json
      - node_modules
    when:
      branch: master

  ssh:
    image: appkeboy/drone-ssh
    host:
      - 128.199.163.232
    port: 22
    username: root
    command_timeout: 120 # 整個 script 檔跑超過 120 秒就送 timeout 給你，回傳失敗，不過一般不會設定
    secrets: [ssh_key]
    script:
      - . /root/.nvm/nvm.sh && nvm use 8.6.0
      - rm -rf ${DRONE_REPO} && mkdir -p ${DRONE_REPO}
      - cp -r drone/${DRONE_REPO}/* ${DROPE_REPO}/
      - cd ${DRONE_REPO} && pm2 restart process.json
    when:
      branch: master


  rebuild-cache:
    image: drillster/drone-volume-cache
    rebuild: true
    mount:
      - ./.yarn-cache
      - ./node_modules
    volumes:
      - /tmp/cache:/cache
    when:
      branch: master # 任何 branch or PR 不用 cache rebuild
```

## Node.js 專案使用 Dockerfile 部署

用 Docker 方式較容易流動與部署

```yaml
pipeline:
  restore-cache:
    image: drillster/drone-volume-cache
    restore: true
    mount:
      - ./.yarn-cache
      - ./node_modules
    volumes:
      - /tmp/cache:/cache

  install:
    image: node:8.6.0
    commands:
      - node -v
      - npm -v
      - yarn --version
      - yarn config set cache-folder .yarn-cache
      - yarn install --pure-lockfile

  testing:
      image: node:8.6.0
      group: testing
      commands:
        - yarn run test
        - sleep 5

  lint:
      image: node:8.6.0
      group: testing
      commands:
        - yarn run lint
        - sleep 5

  scp:
    image: appleboy/drone-scp
    host:
      - 128.199.163.232
#      - 128.199.163.233
    port: 22
    username: root
    target: /root/drone/${DRONE_REPO}
    secrets: [ssh_key] # 也可用 password 方式
    source:
#      - release.tar.gz # 多檔案的話會用 Makefile 處理壓縮相關
      - "*.js"
      - process.json
      - node_modules
      - Dockerfile
    when:
      branch: master

  ssh-1:
    image: appkeboy/drone-ssh
    group: ssh
    host:
      - 128.199.163.232
    port: 22
    username: root
    command_timeout: 120 # 整個 script 檔跑超過 120 秒就送 timeout 給你，回傳失敗，不過一般不會設定
    secrets: [ssh_key]
    script:
      - . /root/.nvm/nvm.sh && nvm use 8.6.0
      - rm -rf ${DRONE_REPO} && mkdir -p ${DRONE_REPO}
      - cp -r drone/${DRONE_REPO}/* ${DROPE_REPO}/
      - cd ${DRONE_REPO} && pm2 restart process.json
    when:
      branch: master

  ssh-2:
    image: appleboy/drone-ssh
    group: ssh
    host:
      - 128.199.163.232
    port: 22
    username: root
    command_timeout: 120
    secrets: [ssh_key]
    script:
      - rm -rf docker/${DRONE_REPO} && mkdir -p docker/${DRONE_REPO}
      - cp -r drone/${DRONE_REPO}/* docker/${DRONE_REPO}/
      - cd docker/${DRONE_REPO} && docker build -t appleboy/node . && docker rm -f app && docker run -d --name app -p 8081:8080 appleboy/node
    when:
      branch: master


  rebuild-cache:
    image: drillster/drone-volume-cache
    rebuild: true
    mount:
      - ./.yarn-cache
      - ./node_modules
    volumes:
      - /tmp/cache:/cache
    when:
      branch: master # 任何 branch or PR 不用 cache rebuild
```