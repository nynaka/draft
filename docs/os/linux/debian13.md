Debian Linux 13
===

## 設定

### mDNS

- インストール

    ```bash
    sudo apt install -y avahi-daemon
    ```

- 起動設定

    ```bash
    sudo systemctl start avahi-daemon
    sudo systemctl enable avahi-daemon
    ```

### SSH

- インストール

    ```bash
    sudo apt install -y ssh
    ```

- 起動設定

    ```bash
    sudo systemctl start ssh
    sudo systemctl enable ssh
    ```

- ed25519 キーペア作成

    ```bash
    ssh-keygen -t ed25519 -C "debian@debian13.local"
    ```

- 秘密鍵から公開鍵を作成する

    ```bash
    ssh-keygen -y -f [秘密鍵のファイルパス] > [作成する公開鍵のファイルパス]
    ```

- 公開鍵の登録

    ```bash
    cat $HOME/.ssh/id_ed25519.pub >> $HOME/.ssh/authorized_keys
    ```

- SSH クライアントの設定

    秘密鍵 ($HOME/.ssh/id_ed25519) を SSH クライアントにコピーします。

    - Windows の場合

        ```text title="C:\Users\ユーザ\.ssh\config"
        Host debian13.local
            HostName debian13.local
            IdentityFile C:\Users\ユーザ\.ssh\id_ed25519_debian13
            User rocky
        ```


### [Docker](https://docs.docker.com/engine/install/debian/)

1. 古いバージョンの削除 (インストールしていた場合)

    ```bash
    sudo apt remove $(dpkg --get-selections \
        docker.io docker-compose docker-doc \
        podman-docker containerd runc | cut -f1)
    ```

2. Docker's official GPG key の登録

    ```bash
    sudo apt update
    sudo apt install -y ca-certificates curl
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/debian/gpg \
        -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc
    ```

3. APT リポジトリの追加

    ```bash
    sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
    Types: deb
    URIs: https://download.docker.com/linux/debian
    Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
    Components: stable
    Signed-By: /etc/apt/keyrings/docker.asc
    EOF
    ```

4. Docker のインストール

    ```bash
    sudo apt update
    sudo apt install -y \
        docker-ce docker-ce-cli containerd.io \
        docker-buildx-plugin docker-compose-plugin
    ```

5. ユーザ権限の設定

    ```bash
    sudo usermod -aG docker debian
    ```

6. 自動起動設定

    ```bash
    sudo systemctl enable docker
    sudo systemctl start docker
    ```
