Debian Linux 12
===

## 設定

* /etc/fstab

    mount オプションに `noatime,nodiratime` を追加すると、ファイルやディレクトリのアクセス時間が、アクセスされる度に更新されなくなるため、若干、ファイルアクセスが速くなるらしい。

    ```text
    UUID=37b5c5fc-4a12-495d-8fed-a0cbfd5c1952    /    xfs    defaults    0    0
    ↓
    UUID=37b5c5fc-4a12-495d-8fed-a0cbfd5c1952    /    xfs    noatime,nodiratime    0    0
    ```

## アプリのインストール

* CLI ツール

    > [!NOTE]
    > root ユーザに昇格してから実行してください。

    ```bash
    apt-get install -y sudo locales man manpages-ja ssh \
        curl wget zip unzip bzip2 p7zip-full csh zsh expect \
        vim emacs \
        git subversion jq \
        sqlite3 \
        cryptsetup
    ```

    * sudo の設定

        ```bash
        /usr/sbin/usermod -aG sudo ユーザID
        ```


## プログラミング言語

* C/C++

    ```bash
    sudo apt-get install -y build-essential gdb cmake exuberant-ctags
    ```

* Go

    ```bash
    sudo apt install -y golang
    ```

* Node.js

    * apt リポジトリのパッケージをインストール場合

        ```bash
        sudo apt-get install -y npm nodejs
        ```

    * nodebrew でインストールする場合

        ```bash
        curl -L git.io/nodebrew | perl - setup
        export PATH=$HOME/.nodebrew/current/bin:$PATH
        nodebrew ls-remote
        nodebrew install-binary v13.12.0
        nodebrew use v13.12.0
        ```

* Python

    * Python 本体

        ```bash
        sudo apt install -y python3 python3-pip
        ```

    * フォーマッター

        ```bash
        pip3 install --break-system-packages black
        ```

    * 静的解析ツール

        ```bash
        pip3 install --break-system-packages flake8
        ```

    * データサイエンス

        ```bash
        pip3 install --upgrade --break-system-packages \
            numpy seaborn \
            matplotlib japanize_matplotlib \
            pandas pandas_profiling pandas-datareader \
            scikit-learn \
            jupyter jupyterlab
        ```

        > [!NOTE]
        > * sklearn だけでは `import sklearn` がエラーになる。sklearn と scikit-learn は別ものらしい。
        > * Debian 12 (というより Python 3.11 だと思う) から、 `pip3 install` コマンドを実行するとき `--break-system-packages` を付けていないとエラーになるようになった。 venv を使うことが推奨されているらしい。

## その他の設定

* SSH 公開鍵認証設定

    * 鍵作成

        ```bash
        ssh-keygen -t rsa -b 4096
        ```

        上記コマンドを実行すると `$HOME/.ssh` 配下に、秘密鍵 (id_rsa) と公開鍵 (id_rsa.pub) が生成されます。

    * 公開鍵 (id_rsa.pub) の扱い

        ```bash
        cat $HOME/.ssh/id_rsa.pub >> $HOME/.ssh/authorized_keys
        ```

    * 秘密鍵 (id_rsa) の扱い

        安全な方法で SSH クライアントにコピーします。秘密鍵の安全なコピー手段を検討するより、SSH クライアントで鍵を生成して公開鍵を SSH サーバにコピーした方が安全性は高いように思えます。

    * SSH クライアントコンフィグ ($HOME/.ssh/config)

        下記のように設定ファイルを用意すると、`ssh ssh_server` のように `Host` で設定した名前指定で SSH ログインできるようになります。

        ```bash
        Host ssh_server
            HostName localhost
            User debian
            Port 22
            IdentityFile /home/debian/.ssh/id_rsa
        ```


* SSH 接続が一定時間で切断される場合

    SSH では一定時間、クライアントから応答がないと自動的に切断する機能があり、デフォルト値 (ClientAliveInterval=0) の場合、応答確認は行わずに切断する、という設定なのですが、たいていはかなりの時間、接続が維持されます。  
    ただ、よくわからないタイミングで切断されることを回避したい場合は、下記のように ClientAliveInterval と ClientAliveCountMax を設定すると `ClientAliveInterval * ClientAliveCountMax` 秒間、無応答でも接続が維持されるようになります。

    ```text title="/etc/ssh/sshd_config"
    #ClientAliveInterval 0
    #ClientAliveCountMax 3
    ↓
    ClientAliveInterval 60
    ClientAliveCountMax 20
    ```


* VMware tools

    * インストール

        ```bash
        sudo apt install -y open-vm-tools
        ```

    * 起動設定

        ```bash
        sudo systemctl start open-vm-tools
        sudo systemctl enable open-vm-tools
        ```


* VirtualBoxの共有フォルダのマウント

    * マウントポイントにするディレクトリを作成 (パスは適当)

        ```bash
        sudo mkdir -p /mnt/vbox_share
        ```

    * /etc/fstab の編集

        ```bash
        sudo bash -c 'echo "vbox_share  /mnt/vbox_share vboxsf  defaults  0  0" >> /etc/fstab'
        ```

    * mount コマンド

        ```bash
        sudo systemctl daemon-reload
        sudo mount -a
        ```


## パフォーマンスモニタリング

* ツールのインストール

    ```bash
    sudo apt-get update
    sudo apt -y install atop sysstat
    ```

* ログ収集間隔の変更およびディスクと i ノードの使用状況をレポート（-S XALL）を追加

    ```bash
    sudo sed -i 's/^LOGINTERVAL=600.*/LOGINTERVAL=60/' /usr/share/atop/atop.daily
    sudo sed -i -e 's|5-55/10|*/1|' -e 's|every 10 minutes|every 1 minute|' -e 's|debian-sa1|debian-sa1 -S XALL|g' /etc/cron.d/sysstat
    sudo bash -c "echo 'SA1_OPTIONS=\"-S XALL\"' >> /etc/default/sysstat"
    ```

* データ収集開始

    ```bash
    sudo sed -i 's|ENABLED="false"|ENABLED="true"|' /etc/default/sysstat
    sudo systemctl enable atop.service cron.service sysstat.service
    sudo systemctl restart atop.service cron.service sysstat.service
    ```

* スクリプトの修正（cron で周期実行する場合必要らしい）

    ```diff
    --- /usr/lib/sysstat/debian-sa1.origin  2023-01-16 21:02:04.764981715 +0900
    +++ /usr/lib/sysstat/debian-sa1 2023-01-16 21:01:02.040984208 +0900
    @@ -6,7 +6,7 @@
    set -e
    
    # Skip in favour of systemd timer
    -[ ! -d /run/systemd/system ] || exit 0
    +#[ ! -d /run/systemd/system ] || exit 0
    
    # Our configuration file
    DEFAULT=/etc/default/sysstat
    ```

* 参考サイト
    * [EC2 Linux インスタンスのモニタリングツールを設定する](https://aws.amazon.com/jp/premiumsupport/knowledge-center/ec2-linux-configure-monitoring-tools/)
