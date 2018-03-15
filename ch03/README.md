# Drone 基本用法

## Drone 後台管理介面

- Builds
- Badges
- Secrets
- Registry
- Settings

travisCI 對 yaml 限制較多，沒 Drone 強大

## 使用 Git Clone

Drone 的第一個預設動作就是 git clone。但可改變行為：

```yaml
# .drone.yml
clone:
  git:
    image: plugins/git # Drone 自維護的 git image
    depath: 50 # default 也是 50，只抓 commit 前 50 的
    tags: true # 預設 tags 不會下載下來

pipeline:
  node:
    image: node:8.3.0
    commands:
      - yarn install
      - yarn test
```

## Workspace 介紹

```yaml
workspace:
  base: /node # 裡面產生的資料可以在每個 pipeline 全部共享到 (開一個 volume mount 所有 pipeline 的 containers)
  path: drone/node-example # git clone 後要把所有的 source code 到這裡

clone:
  git:
    image: plugins/git
    depath: 50
    tags: true

pipeline:
  node:
    image: node:8.3.0
    commands:
      - yarn install
      - ./node_modules/.bin/mocha
      - touch /node/a.txt

  ls:
    image: node:8.3.0
    commands:
      - ls -al /node/
```