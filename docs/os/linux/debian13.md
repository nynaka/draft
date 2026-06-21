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
