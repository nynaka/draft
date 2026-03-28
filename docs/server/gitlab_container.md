GitLab Container
===

## 環境構築

下記のようなファイル構成を想定しています。

!!! important
    gitlab_container  
    |-- certs/  
    |-- docker-compose.yaml

## サーバ証明書の用意

GitLab コンテナ構築時にサーバ証明書も用意してくれるのですが、有効期限が 1 カ月と短く、gitlab-runner を使うときに支障が出るので、自前のサーバ証明書を用意します。  
秘密鍵はパスワードを設定するか、SoftHSM のようなアプリで管理すべきだと思いますが、自分しか使用者がいないので、その辺は考えないことにしています。  
以下の作業は `gitlab_container/certs` で実行することを前提にしています。

- 自己認証局秘密鍵・公開鍵の作成

    ```bash
    openssl req \
        -new \
        -out privateca.crt \
        -keyout privateca.key \
        -x509 \
        -days 3650 \
        -newkey rsa:4096 \
        -nodes \
        -subj "/CN=PrivateCA"
    ```

- サーバ証明書作成

    - CSR

        ```bash
        openssl req \
            -new \
            -out dev.gitlab.local.csr \
            -keyout dev.gitlab.local.key \
            -newkey rsa:4096 \
            -nodes \
            -subj "/CN=dev.gitlab.local"
        ```
     
    - Subject Alternative Name の準備

        ここでは `san.txt` とします。

        ```text
        echo "subjectAltName = DNS:dev.gitlab.local,DNS:localhost,IP:127.0.0.1,IP:192.168.1.100" \
            > san.txt
        ```
     
    - CSR へ署名

        ```bash
        openssl x509 \
            -req \
            -days 3650 \
            -in dev.gitlab.local.csr \
            -out dev.gitlab.local.crt \
            -CA privateca.crt \
            -CAkey privateca.key \
            -CAcreateserial \
            -extfile san.txt
        ```

### 自己 CA を信頼する CA リストに追加する

!!!note
    これをしないと、gitlab-runner を使おうとしたときに自己 CA 証明書が信頼されないので、自動テストが必ずエラーになります。

- Debian / Ubuntu

    ```bash
    sudo cp privateca.crt /usr/local/share/ca-certificates/privateca.crt
    sudo update-ca-certificates
    この後、/etc/ssl/certs/privateca.pem が存在するはずです。
    ```

- Python (Requests ライブラリだけかもですが)

    Python では OS の信頼する CA リストは参照せず、/usr/lib/python3/dist-packages/certifi/cacert.pem または venv/lib/python3.10/site-packages/certifi/cacert.pem のような、Python環境の信頼する CA リストを参照しているようです。  
    このファイルに PrivateCA の証明書を追加すると requests が PrivateCA を信頼するようになります。


## コンテナの用意

- docker-compose.yaml

```yaml
services:
  gitlab:
    image: 'gitlab/gitlab-ce:17.5.1-ce.0'
    container_name: gitlab
    restart: always
    hostname: gitlab
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        # Add any other gitlab.rb configuration here, each on its own line
        external_url 'https://dev.gitlab.local'
    ports:
      - '80:80'
      - '443:443'
      - '22:22'
    volumes:
      - './volume/config:/etc/gitlab'
      - './volume/logs:/var/log/gitlab'
      - './volume/data:/var/opt/gitlab'
      - './certs:/etc/gitlab/ssl'
    shm_size: '256m'
```

- [Install GitLab using Docker](https://docs.gitlab.com/ee/install/docker.html)
- [docker-composeでGitLab](https://qiita.com/kujiraza/items/f87d2a9fb42ff227d3e6)


## 初回起動

1. STATUS が `health: starting` ⇒ `healthy` になるまで待つ (5 分くらい？)

    ```text
    CONTAINER ID   IMAGE                     COMMAND             CREATED         STATUS                         
    dc18d732b698   gitlab/gitlab-ce:latest   "/assets/wrapper"   4 minutes ago   Up 9 seconds (health: starting)
    ```
    ↓
    ```text
    CONTAINER ID   IMAGE                     COMMAND             CREATED         STATUS                         
    dc18d732b698   gitlab/gitlab-ce:latest   "/assets/wrapper"   19 minutes ago   Up 15 minutes (healthy)
    ```

2. 初回ログイン

    デフォルトユーザ名は `root` で、デフォルトパスワードは下記コマンドで取得できます。  
    initial_root_password は暫くすると自動で削除されるので、早めに root パスワードを変更するか、このファイルのコピーを保存しておきましょう。

    ```bash
    docker exec gitlab cat /etc/gitlab/initial_root_password
    ```

    ![](./gitlab/00_gitlab_login.png)


## Gitlab Runner

Runner のインストール方法は Girlab リポジトリの

> Settings
>   -> Ci/CD
>   -> Runners
>   -> Available Runners 右端の3点メニュー
>   -> Show runner installation and registration instructions

から参照できます。

- バイナリの手動インストール

    ```bash
    # Download the binary for your system
    sudo curl -L \
        --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64

    # Give it permission to execute
    sudo chmod +x /usr/local/bin/gitlab-runner

    # Create a GitLab Runner user
    sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash

    # Install and run as a service
    sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
    sudo gitlab-runner start
    ```

- Debian / Ubuntu

    ```bash
    # If using a `deb` package based distribution
    curl -s https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
    sudo apt install -y gitlab-runner
    ```

- RHEL / Fedora

    ```bash
    # If using an `rpm` package based distribution
    curl -s https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh | sudo bash
    sudo dnf install -y gitlab-runner
    ```
