# Drone 指令介紹

## Drone 指令安裝方式

### Drone CLI 能做什麼？

- 設定專案 Secret
- 手動部署專案
- 搭配 Cron Job 執行任務

### 如何安裝

[http://docs.drone.io/cli-installation](http://docs.drone.io/cli-installation)

- 下載 Binary 放到 /usr/local/bin 目錄
- 透過 go get 方式安裝

### 用 Go 語言安裝

```shell
$ go get -u github.com/drone/drone-cli/drone
```

or brew (OSX)

```shell
$ brew tap drone/drone
$ brew install drone
```

### CLI 驗證

drone cli 如何連接 drone server

UI Page > Token > Example CLI Usage

在 .bashrc or .zshrc file 增加下面兩行：

```shell
export DRONE_SERVER=https://c86bfce3.ngrok.io
export DRONE_TOKEN=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0ZXh0IjoiZWRlbjkwMjY3IiwidHlwZSI6InVzZXIifQ.Ul0W1z_LXV524SmesYrLVCgkpZFp6TyZCEJNOuSBP_Y
```

```shell
$ source ~/.zshrc
$ drone info
```

有出現 User 與 Email 就代表成功了

## Drone Secret 指令介紹

### Drone 後台安全性

- 使用後台設定 Secret
  - Secret 會被 Hacker 拿走

```shell
$ drone secret
$ drone secret ls eden90267/drone-nodejs-example
$ drone secret update --name test46 --value drone-secret --image eden90267/drone-nodejs-example --repository eden90267/drone-nodejs-example
$ drone secret ls eden90267/drone-nodejs-example
$ drone secret add --name test47 --value drone-secret --image eden90267/drone-nodejs-example --repository eden90267/drone-nodejs-example
$ drone secret ls eden90267/drone-nodejs-example
$ drone secret add --name readme --value @README.md --image eden90267/drone-nodejs-example --repository eden90267/drone-nodejs-example
$ drone secret ls eden90267/drone-nodejs-example
```

@ 方式是讀取 local 端的檔案內容來寫道 secret value 裡面

## Drone exec 指令介紹

Drone 只能線上測試？ 如何在本機端測試及部署

使用 Drone 測試取代傳統的 docker-compose (限制多)，用 drone yaml
檔案可以清楚直觀的一步一步如何測試如何部署。如果公司還沒導入 drone，可用 drone
cli 來達到本機測試及除錯

### 本機端測試 Yaml 設定

```yaml
# .drone.yml
pipeline:
  backend:
    image: golang
    commands:
      - echo "backend testing"

  frontend:
    image: golang
    commands:
      - echo "frontend testing"
```

```shell
$ docker pull golang # 先把要的 image pull 下來，不然用 drone exec 會 pending 住在那裡下載 (沒給下載進度)
$ drone exec
```

1. 不想每次 commit 才去 server 那裡看結果，就可用 `drone exec`
2. local 端的 secret 也要的話 (因沒跟 drone server 連結)：

  ```yaml
  # .drone.yml
  pipeline:
    backend:
      image: golang
      secrets: [test]
      commands:
        - echo "backend testing"
        - echo $TEST
  
    frontend:
      image: golang
      commands:
        - echo "frontend testing"
  ```

  ```shell
  $ TEST=appleboy drone exec
  ```

3. 有 pipeline 不想在 local 執行 (譬如 deploy)

  ```yaml
  # .drone.yml
  pipeline:
    backend:
      image: golang
      secrets: [test]
      commands:
        - echo "backend testing"
        - echo $TEST
  
    frontend:
      image: golang
      commands:
        - echo "frontend testing"

    deploy:
      image: golang
      commands:
        - echo "frontend testing"
      when:
        local: false
  ```

若尚未導入 drone，可透過 drone cli 工具，幫你把 docker-compose 整合好，不需裝
drone server 可享受到 drone 福利，drone cli 透過 yaml 呼叫 drone api
來幫你做事