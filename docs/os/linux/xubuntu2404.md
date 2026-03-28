Xubuntu Linux 24.04
===

## 設定

- swappiness

    swappiness は物理メモリが何 % になったらスワップメモリを使うのかを設定するパラメータで、0 を設定すると Kernel 3.5 からスワップを使わないようになったそうです。  
    デフォルト値は 60 で、極力スワップを使わないようにするためには 1 を設定することが妥当のようです。

    ```bash
    echo "vm.swappiness=1" | sudo tee /etc/sysctl.d/99-swappiness.conf
    sudo reboot
    ```

    設定値が適用されているかどうかは下記のコマンドで確認できます。

    ```bash
    cat /proc/sys/vm/swappiness
    ```

- /etc/fstab

    ディスクのマウントオプションを調整すると、気休め程度ですがディスク I/O の性能を向上させられると言われています。  
    体感できるほどの効果はありません。。。

    ```diff
    --- /tmp/fstab	2024-09-22 08:23:57.570898136 +0900
    +++ /etc/fstab	2024-09-22 08:22:50.595808086 +0900
    @@ -6,6 +6,6 @@
     #
     # <file system> <mount point>   <type>  <options>       <dump>  <pass>
     # / was on /dev/sda2 during installation
    -UUID=2f256458-720f-4bb3-8133-d7a3c31b3390 /               xfs     defaults        0       0
    +UUID=2f256458-720f-4bb3-8133-d7a3c31b3390 /               xfs     defaults,noatime,nodiratime        0       0
     # /boot/efi was on /dev/sda1 during installation
     UUID=CB65-2187  /boot/efi       vfat    umask=0077      0       1
    ```


## インストールアプリ

- CLI およびサーバ

    ```bash
    sudo apt install -y \
        locales man manpages-ja \
        avahi-daemon ssh \
        wget curl jq \
        git vim \
        zip unzip bzip2 p7zip-full
    ```

    | 項目         | 説明                                                                                                                                           |
    | :----------- | :--------------------------------------------------------------------------------------------------------------------------------------------- |
    | avahi-daemon | mDNS とういうマルチキャスト を使用してホスト名で名前を解決する DNS　サーバ。名前の解決範囲はリンクローカルマルチキャストが届く範囲になります。 |

    - 自動起動設定

        ```bash
        sudo systemctl start avahi-daemon ssh
        sudo systemctl enable avahi-daemon ssh
        ```

- GUI アプリ

    - Libre Office

        ```bash
        sudo apt install -y \
            libreoffice libreoffice-l10n-ja libreoffice-help-ja
        ```

    - 画像、動画

        ```bash
        sudo apt install -y gimp shotwell vlc mplayer
        ```

    - ターミナル

        ```bash
        sudo apt install -y guake
        ```

    - インターネット関連

        ```bash
        sudo apt install -y \
            chromium-browser chromium-browser-l10n \
            firefox firefox-locale-ja \
            thunderbird thunderbird-locale-ja
        ```

    - [Google Chrome](https://www.google.com/intl/ja/chrome/)

        ```bash
        wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
        sudo dpkg -i google-chrome-stable_current_amd64.deb
        ```

    - [Microsoft Edge](https://www.microsoft.com/ja-jp/edge?form=MA13FJ)

        ```bash
        wget https://packages.microsoft.com/repos/edge/pool/main/m/microsoft-edge-stable/microsoft-edge-stable_144.0.3719.92-1_amd64.deb?brand=M102 \
            -O /tmp/microsoft-edge-stable_144.0.3719.92-1_amd64.deb
        sudo dpkg -i /tmp/microsoft-edge-stable_144.0.3719.92-1_amd64.deb
        ```

    - [Eclipse](https://www.eclipse.org/)

        eclipse-java-2024-09-R-linux-gtk-x86_64.tar.gz の例  
        eclipse-java-2024-09-R-linux-gtk-x86_64.tar.gz は、ブラウザ等で公式サイトからダウンロードしてください。

        ```bash
        sudo tar zxvf /tmp/eclipse-java-2024-09-R-linux-gtk-x86_64.tar.gz -C /usr/local/bin/
        ```

        アプリケーションメニューには下記のように登録します。

        ```ini
        echo "[Desktop Entry]
        Type=Application
        Encoding=UTF-8
        Name=Eclipse
        Comment=Eclipse IDE
        Exec=/usr/local/bin/eclipse/eclipse
        Icon=/usr/local/bin/eclipse/icon.xpm
        Categories=Development;
        Terminal=false
        NoDisplay=false" > $HOME/.local/share/applications/eclipse.desktop
        ```
    
        - 日本語化

            pleiades のインストール
    
            ```bash
            wget https://ftp.jaist.ac.jp/pub/mergedoc/pleiades/build/stable/pleiades.zip \
                -O /tmp/pleiades.zip
            cd /tmp
            unzip pleiades.zip
            sudo cp -R /tmp/plugins/jp.sourceforge.mergedoc.pleiades /usr/local/bin/eclipse/plugins
            sudo cp -R /tmp/features/jp.sourceforge.mergedoc.pleiades /usr/local/bin/eclipse/features
            ```

            /usr/local/bin/eclipse/eclipse.ini に下記 2 行を追加

            ```text
            -Xverify:none
            -javaagent:/usr/local/bin/eclipse/plugins/jp.sourceforge.mergedoc.pleiades/pleiades.jar
            ```

            pleiades.jar は相対パスではなくフルパスじゃないとダメみたいです。

    - [Intellij IDEA](https://www.jetbrains.com/idea/)

        ideaIC-2024.1.4.tar.gz は、ブラウザ等で公式サイトからダウンロードしてください。

        ```bash
        sudo tar zxvf ideaIC-2024.1.4.tar.gz -C /usr/local/bin/
        ```

        アプリケーションメニューには下記のように登録します。

        ```ini
        echo "[Desktop Entry]
        Type=Application
        Encoding=UTF-8
        Name=Intellij IDEA
        Comment=Intellij IDEA Community
        Exec=/usr/local/bin/idea-IC-241.18034.62/bin/idea.sh
        Icon=/usr/local/bin/idea-IC-241.18034.62/bin/idea.svg
        Categories=Development;
        Terminal=false
        NoDisplay=false" > $HOME/.local/share/applications/intellij.desktop
        ```

        アプリの日本語化が必要な場合は、Intellij IDEA 起動後、`Plugins` ⇒ `Japanese Language Pack` をインストールして再起動すると日本語化されます。

    - [PyCharm](https://www.jetbrains.com/ja-jp/pycharm/)

        pycharm-community-2024.2.3.tar.gz は、ブラウザ等で公式サイトからダウンロードしてください。

        ```bash
        sudo tar zxvf pycharm-community-2024.2.3.tar.gz -C /usr/local/bin/
        ```

        アプリケーションメニューには下記のように登録します。

        ```ini
        echo "[Desktop Entry]
        Type=Application
        Encoding=UTF-8
        Name=PyCharm-CE
        Comment=PyCharm Community
        Exec=/usr/local/bin/pycharm-community-2024.2.3/bin/pycharm.sh
        Icon=/usr/local/bin/pycharm-community-2024.2.3/bin/pycharm.svg
        Categories=Development;
        Terminal=false
        NoDisplay=false" > $HOME/.local/share/applications/pycharm-ce.desktop
        ```

        アプリの日本語化が必要な場合は、Intellij IDEA 起動後、`Plugins` ⇒ `Japanese Language Pack` をインストールして再起動すると日本語化されます。

    - [VSCode](https://code.visualstudio.com/)

        ```bash
        wget "https://code.visualstudio.com/sha/download?build=stable&os=linux-deb-x64" -O /tmp/vscode.deb
        sudo dpkg -i /tmp/vscode.deb
        ```

        * VSCode のフォント設定

            ```json
            {
                "editor.fontFamily": "'UDEV Gothic JPDOC', Consolas, 'Courier New', monospace",
                "editor.fontSize": 16,
                "debug.console.fontSize": 16,
                "terminal.integrated.fontSize": 16,
            }
            ```

## セキュリティ

### ファイアウォール

- ufw (Uncomplicated Firewall)

    ```bash
    # インストール
    sudo apt install -y ufw

    # 現在の状態を確認 (おそらく inactive になっている)
    sudo ufw status

    # すべての外部からのアクセスを拒否し、内部からのアクセスのみ許可
    sudo ufw default deny incoming
    sudo ufw default allow outgoing

    # SSH接続を許可 (重要: これをしないとリモートから締め出される可能性があります)
    sudo ufw allow ssh

    # ファイアウォールを有効化
    sudo ufw enable
    ```


### ウイルス対策

- [ClamAV](../../app/clamav.md)


## プログラミング言語

- C/C++

    ```bash
    sudo apt-get install -y \
        build-essential gdb cmake autoconf \
        universal-ctags
    ```

- Go

    ```bash
    sudo apt install -y golang
    ```

- Java

    ```bash
    sudo apt install -y openjdk-21-jdk maven gradle
    ```

- Nodejs

    - apt リポジトリのパッケージをインストール場合

        ```bash
        sudo apt-get install -y npm nodejs
        ```

    - nodebrew でインストールする場合

        ```bash
        curl -L git.io/nodebrew | perl - setup
        export PATH=$HOME/.nodebrew/current/bin:$PATH
        nodebrew ls-remote
        nodebrew install-binary v20.10.0
        nodebrew use v20.10.0
        ```

- Python

    - Python 本体

        ```bash
        sudo apt install -y python3 python3-venv
        ```

    - データサイエンス

        ```bash
        pip3 install --upgrade \
            numpy seaborn \
            matplotlib japanize_matplotlib \
            pandas pandas_profiling pandas-datareader \
            scikit-learn \
            jupyter jupyterlab
        ```

        !!! note
            sklearn だけでは import sklearn がエラーになる。sklearn と scikit-learn は別ものらしい。

- Rust

    ```bash
    sudo apt install -y rustc
    ```


## 日本語フォント

- apt

    ```bash
    sudo apt install -y fonts-ipafont fonts-ipaexfont fonts-ricty-diminished
    ```

- [UDEV Gothic](https://github.com/yuru7/udev-gothic)

    ```bash
    wget https://github.com/yuru7/udev-gothic/releases/download/v2.2.0/UDEVGothic_HS_v2.2.0.zip -O /tmp/UDEVGothic_HS.zip
    wget https://github.com/yuru7/udev-gothic/releases/download/v2.2.0/UDEVGothic_NF_v2.2.0.zip -O /tmp/UDEVGothic_NF.zip
    wget https://github.com/yuru7/udev-gothic/releases/download/v2.2.0/UDEVGothic_v2.2.0.zip -O /tmp/UDEVGothic.zip
    mkdir -p .fonts/udev_gothic
    cd .fonts/udev_gothic
    unzip /tmp/UDEVGothic_HS.zip
    unzip /tmp/UDEVGothic_NF.zip
    unzip /tmp/UDEVGothic.zip
    ```

## [Docker](https://docs.docker.com/engine/install/ubuntu/)

* 関連ツールのインストール

    ```bash
    sudo apt-get update
    sudo apt-get -y install \
        ca-certificates \
        curl
    ```

* GPG key のインストール

    ```bash
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
        -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc
    ```

* リポジトリの追加

    ```bash
    sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
    Types: deb
    URIs: https://download.docker.com/linux/ubuntu
    Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
    Components: stable
    Signed-By: /etc/apt/keyrings/docker.asc
    EOF
    ```

* Docker のインストール

    ```bash
    sudo apt-get update
    sudo apt install -y \
        docker-ce docker-ce-cli \
        containerd.io docker-buildx-plugin \
        docker-compose-plugin
    ```

* ユーザ権限の設定

    ```bash
    sudo usermod -aG docker ubuntu
    ```

* 自動起動設定

    ```bash
    sudo systemctl enable docker
    sudo systemctl start docker
    ```

## リモートデスクトップ

- xrdp のインストール

    ```bash
    sudo apt install -y xrdp
    ```

- xrdp サーバーの起動

    ```bash
    sudo systemctl start xrdp
    sudo systemctl enable xrdp
    ```

この後、リモートデスクトップクライアントアプリで接続すると Linux のデクストップ画面が見えると思います。


## 仮想化環境用ツール

- VMware tools

    - インストール

        ```bash
        sudo apt install -y open-vm-tools open-vm-tools-desktop
        ```

    - 起動設定

        ```bash
        sudo systemctl start open-vm-tools
        sudo systemctl enable open-vm-tools
        ```

    - /etc/fstab 設定例

        ```bash
        sudo bash -c 'echo ".host:/ /mnt/hgfs fuse.vmhgfs-fuse allow_other,auto_unmount,defaults 0 0" >> /etc/fstab'
        ```

        - 事前に sudo mkdir -p /mnt/hgfs 等でマウントポイントとなるディレクトリを作成しておいてください。
        - VMware Workstation で共有フォルダの設定を有効にしておいて下さい。

    - mount コマンド

        ```bash
        sudo systemctl daemon-reload
        sudo mount -a
        ```

- VirtualBoxの共有フォルダのマウント

    - マウントポイントにするディレクトリを作成 (パスは適当)

        ```bash
        sudo mkdir -p /mnt/vbox_share
        ```

    - /etc/fstab の編集

        ```bash
        sudo bash -c 'echo "vbox_share  /mnt/vbox_share vboxsf  defaults  0  0" >> /etc/fstab'
        ```

    - mount コマンド

        ```bash
        sudo systemctl daemon-reload
        sudo mount -a
        ```
