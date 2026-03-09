[SoftHSM](https://github.com/softhsm/SoftHSMv2)
===

## インストール

- Debian 系 Linux (Debian Linux、Ubuntu Linux 他)

    ```bash
    sudo apt install -y softhsm2 opensc
    ```

- Redhat 系 Linux (Fedora、Alma Linux、Rocky Linux)

    ```bash
    sudo dnf install -y softhsm opensc
    ```

- ソースからビルドする場合

    - ビルドツールのインストール

        - Debian / Ubuntu

            ```bash
            sudo apt install -y \
                build-essential autoconf libtool libssl-dev
            ```

        - RHEL / Fedora

            ```bash
            sudo dnf groupinstall -y "Development Tools"
            sudo dnf install -y \
                openssl-devel opensc
            ```

    - ソースからビルドする

        !!! note
            SoftHSM の 2026年3月1日時点の最新版である 2.7.0 は、署名回りの処理でバグがあるらしいので、当面 2.6.1 を使用しています。

        ```bash
        git clone -b 2.6.1 https://github.com/softhsm/SoftHSMv2 SoftHSMv2-2.6.1
        cd SoftHSMv2-2.6.1
        ./autogen.sh
        ./configure --enable-ecc --enable-eddsa
        make && sudo make install
        ```

### PKCS11 ライブラリの位置

| OS                                                  | ライブラリパス                        |
| :-------------------------------------------------- | :------------------------------------ |
| Debian 系 Linux (Debian Linux、Ubuntu Linux 他)     | /usr/lib/softhsm/libsofthsm2.so       |
| Redhat 系 Linux (Fedora、Alma Linux、RockyLinux 他) | /usr/lib64/softhsm/libsofthsm.so      |
| ソースからビルドした場合                            | /usr/local/lib/softhsm/libsofthsm2.so |


## softhsm2-util

`softhsm2-util` は SoftHSM トークンを初期化および管理するためのコマンドラインツールです。

- トークン情報の表示

    ```bash
    softhsm2-util --show-slots
    ```

- トークンの初期化

    ```bash
    # softhsm2-util --init-token --slot <slot> --label <label>
    softhsm2-util --init-token --slot 0 --label "SoftHSM Token"
    ```

    SO (Security Officer) PIN と user PIN の設定を求められます。

    > Please enter the new SO PIN:
    > Please re-enter the new SO PIN:
    > Please enter the new user PIN:
    > Please re-enter the new user PIN:

- 鍵のインポート

    ```bash
    # softhsm2-util --import <path> --slot <slot> --label <label> --id <hex-id>
    softhsm2-util --import mykey.pem --slot 0 --label "mykey" --id 1234
    ```

## PKCS11 での操作

- PKCS11 ライブラリパスの設定

    ```bash
    export PKCS11_MODULE_PATH="/usr/lib64/softhsm/libsofthsm.so"
    ```

- サポートアルゴリズムの確認

    ```bash
    pkcs11-tool --module $PKCS11_MODULE_PATH -M
    ```

- トークンの初期化

    ```bash
    pkcs11-tool --module $PKCS11_MODULE_PATH \
        --init-token \
        --slot 0 \
        --so-pin 1234 \
        --label "SoftHSM Token"
    ```

- User PIN の設定

    ```bash
    pkcs11-tool --module $PKCS11_MODULE_PATH \
        --init-pin \
        --so-pin 1234 \
        --pin 1234
    ```

- スロット番号の確認

    ```bash
    pkcs11-tool --module $PKCS11_MODULE_PATH -L
    ```

- 鍵作成

    - 対称鍵

        ```bash
        pkcs11-tool --module $PKCS11_MODULE_PATH \
            --keygen \
            --key-type AES:32 \
            --label "aeskey" \
            --login --pin 1234
        ```

    - RSA キーペア

        ```bash
        pkcs11-tool --module $PKCS11_MODULE_PATH \
            --keypairgen \
            --key-type rsa:4096 \
            --label "rsakey" \
            --login --pin 1234
        ```

    - ECDSA キーペア

        ```bash
        pkcs11-tool --module $PKCS11_MODULE_PATH \
            --keypairgen \
            --key-type EC:prime256v1 \
            --label "ecdsakey" \
            --login --pin 1234
        ```

    - EdDSA(ed25519) キーペア

        ```bash
        pkcs11-tool --module $PKCS11_MODULE_PATH \
            --keypairgen \
            --key-type EC:edwards25519 \
            --label "ed25519key" \
            --login --pin 1234
        ```

        !!! note
            **--key-type** には **curve25519** も指定できますが、こちらは ECDH の X25519 になります。

- 登録鍵一覧の取得

    ```bash
    pkcs11-tool --module $PKCS11_MODULE_PATH -O \
        --login \
        --pin 1234
    ```


## OpenSSL コマンドで利用する

!!! note
    openssl 3.0.x のように古い OpenSSL だと、環境によっては謎のコアダンプに悩まされることがあるようです。  
    可能なら OpenSSL 3.2+ など、より新しい版での検証を推奨します。

!!! warning
    以降の例では分かりやすさのため、PKCS#11 URI に `pin-value=...` を埋め込んでいます。  
    実運用ではシェル履歴やプロセス一覧に PIN が残りうるため、PIN の渡し方は環境に合わせて見直してください。

### PKCS11 Engine (OpenSSL 3 から非推奨)

#### インストール

- Debian 系 Linux (Debian Linux、Ubuntu Linux 他)

    ```bash
    sudo apt install -y libengine-pkcs11-openssl
    ```

- Redhat 系 Linux (Fedora、Alma Linux、Rocky Linux)

    ```bash
    sudo dnf install -y openssl-pkcs11
    ```

#### 使い方

- PKCS11 ライブラリパスの設定

    ```bash
    export PKCS11_MODULE_PATH="/usr/local/lib/softhsm/libsofthsm2.so"
    ```

- (例) 以降で使う変数

    ```bash
    export TOKEN_LABEL="SoftHSM Token"
    export PIN="1234"

    # 例: CA 用 / エンドエンティティ用に鍵ラベルを分ける
    export CA_RSA_KEY="pkcs11:token=${TOKEN_LABEL};object=ca-rsa;type=private;pin-value=${PIN}"
    export CA_EC_KEY="pkcs11:token=${TOKEN_LABEL};object=ca-ec-p256;type=private;pin-value=${PIN}"
    export CA_ED_KEY="pkcs11:token=${TOKEN_LABEL};object=ca-ed25519;type=private;pin-value=${PIN}"

    export USER_RSA_KEY="pkcs11:token=${TOKEN_LABEL};object=user-rsa;type=private;pin-value=${PIN}"
    export USER_EC_KEY="pkcs11:token=${TOKEN_LABEL};object=user-ec-p256;type=private;pin-value=${PIN}"
    export USER_ED_KEY="pkcs11:token=${TOKEN_LABEL};object=user-ed25519;type=private;pin-value=${PIN}"
    ```

    !!! note
        上記の鍵ラベルは例です。既存の鍵を使う場合は `pkcs11-tool -O` の出力に合わせて `object=...` を調整してください。

        鍵をまだ作っていない場合は、例えば以下のように作れます。

        ```bash
        pkcs11-tool --module "$PKCS11_MODULE_PATH" --login --pin "$PIN" \
            --keypairgen --key-type rsa:4096 --label "ca-rsa"
        pkcs11-tool --module "$PKCS11_MODULE_PATH" --login --pin "$PIN" \
            --keypairgen --key-type rsa:4096 --label "user-rsa"

        pkcs11-tool --module "$PKCS11_MODULE_PATH" --login --pin "$PIN" \
            --keypairgen --key-type EC:prime256v1 --label "ca-ec-p256"
        pkcs11-tool --module "$PKCS11_MODULE_PATH" --login --pin "$PIN" \
            --keypairgen --key-type EC:prime256v1 --label "user-ec-p256"

        pkcs11-tool --module "$PKCS11_MODULE_PATH" --login --pin "$PIN" \
            --keypairgen --key-type EC:edwards25519 --label "ca-ed25519"
        pkcs11-tool --module "$PKCS11_MODULE_PATH" --login --pin "$PIN" \
            --keypairgen --key-type EC:edwards25519 --label "user-ed25519"
        ```

#### 署名・署名検証

- テスト用データ

    ```bash
    printf 'hello pkcs11\n' > msg.txt
    ```

- RSA (PKCS#1 v1.5) 署名 / 検証

    ```bash
    # 公開鍵を取り出す（秘密鍵から公開鍵を導出して出力）
    openssl pkey \
        -engine pkcs11 -keyform engine \
        -in "$USER_RSA_KEY" \
        -pubout -out user-rsa-pub.pem

    # 署名
    openssl dgst -sha256 \
        -engine pkcs11 -keyform engine \
        -sign "$USER_RSA_KEY" \
        -out rsa_pkcs1.sig msg.txt

    # 検証
    openssl dgst -sha256 \
        -verify user-rsa-pub.pem \
        -signature rsa_pkcs1.sig msg.txt
    ```

- RSA (PSS) 署名 / 検証

    ```bash
    openssl dgst -sha256 \
        -sigopt rsa_padding_mode:pss \
        -sigopt rsa_mgf1_md:sha256 \
        -sigopt rsa_pss_saltlen:-1 \
        -engine pkcs11 -keyform engine \
        -sign "$USER_RSA_KEY" \
        -out rsa_pss.sig msg.txt

    openssl dgst -sha256 \
        -sigopt rsa_padding_mode:pss \
        -sigopt rsa_mgf1_md:sha256 \
        -sigopt rsa_pss_saltlen:-1 \
        -verify user-rsa-pub.pem \
        -signature rsa_pss.sig msg.txt
    ```

- ECDSA P-256 署名 / 検証

    ```bash
    openssl pkey \
        -engine pkcs11 -keyform engine \
        -in "$USER_EC_KEY" \
        -pubout -out user-ec-pub.pem

    openssl dgst -sha256 \
        -engine pkcs11 -keyform engine \
        -sign "$USER_EC_KEY" \
        -out ecdsa_p256.sig msg.txt

    openssl dgst -sha256 \
        -verify user-ec-pub.pem \
        -signature ecdsa_p256.sig msg.txt
    ```

- EdDSA ed25519 署名 / 検証

    ```bash
    openssl pkey \
        -engine pkcs11 -keyform engine \
        -in "$USER_ED_KEY" \
        -pubout -out user-ed25519-pub.pem

    openssl pkeyutl \
        -engine pkcs11 -keyform engine \
        -sign -inkey "$USER_ED_KEY" \
        -in msg.txt -out ed25519.sig

    openssl pkeyutl \
        -verify -pubin -inkey user-ed25519-pub.pem \
        -in msg.txt -sigfile ed25519.sig
    ```

#### 自己証明書の作成（CA用途を想定）

- RSA の自己証明書

    ```bash
    openssl req -new -x509 -days 3650 \
        -engine pkcs11 -keyform engine \
        -key "$CA_RSA_KEY" \
        -out ca-rsa.crt \
        -subj "/CN=Example RSA CA" \
        -addext "basicConstraints=critical,CA:TRUE" \
        -addext "keyUsage=critical,keyCertSign,cRLSign"
    ```

- ECDSA P-256 の自己証明書

    ```bash
    openssl req -new -x509 -days 3650 \
        -engine pkcs11 -keyform engine \
        -key "$CA_EC_KEY" \
        -out ca-ec-p256.crt \
        -subj "/CN=Example ECDSA P-256 CA" \
        -addext "basicConstraints=critical,CA:TRUE" \
        -addext "keyUsage=critical,keyCertSign,cRLSign"
    ```

- EdDSA ed25519 の自己証明書

    ```bash
    openssl req -new -x509 -days 3650 \
        -engine pkcs11 -keyform engine \
        -key "$CA_ED_KEY" \
        -out ca-ed25519.crt \
        -subj "/CN=Example Ed25519 CA" \
        -addext "basicConstraints=critical,CA:TRUE" \
        -addext "keyUsage=critical,keyCertSign,cRLSign"
    ```

#### CSR 作成

- RSA の CSR

    ```bash
    openssl req -new \
        -engine pkcs11 -keyform engine \
        -key "$USER_RSA_KEY" \
        -out user-rsa.csr \
        -subj "/CN=host1.example.com" \
        -addext "subjectAltName=DNS:host1.example.com,DNS:localhost,IP:127.0.0.1"
    ```

- ECDSA P-256 の CSR

    ```bash
    openssl req -new \
        -engine pkcs11 -keyform engine \
        -key "$USER_EC_KEY" \
        -out user-ec-p256.csr \
        -subj "/CN=host1.example.com" \
        -addext "subjectAltName=DNS:host1.example.com,DNS:localhost,IP:127.0.0.1"
    ```

- EdDSA ed25519 の CSR

    ```bash
    openssl req -new \
        -engine pkcs11 -keyform engine \
        -key "$USER_ED_KEY" \
        -out user-ed25519.csr \
        -subj "/CN=host1.example.com" \
        -addext "subjectAltName=DNS:host1.example.com,DNS:localhost,IP:127.0.0.1"
    ```

#### CA として CSR に署名して証明書発行

- RSA CA で RSA CSR に署名

    ```bash
    openssl x509 -req -days 365 \
        -in user-rsa.csr \
        -CA ca-rsa.crt \
        -CAkey "$CA_RSA_KEY" \
        -engine pkcs11 -CAkeyform engine \
        -CAcreateserial \
        -copy_extensions copy \
        -out user-rsa.crt

    openssl verify -CAfile ca-rsa.crt user-rsa.crt
    ```

- ECDSA P-256 CA で ECDSA CSR に署名

    ```bash
    openssl x509 -req -days 365 \
        -in user-ec-p256.csr \
        -CA ca-ec-p256.crt \
        -CAkey "$CA_EC_KEY" \
        -engine pkcs11 -CAkeyform engine \
        -CAcreateserial \
        -copy_extensions copy \
        -out user-ec-p256.crt

    openssl verify -CAfile ca-ec-p256.crt user-ec-p256.crt
    ```

- EdDSA ed25519 CA で Ed25519 CSR に署名

    ```bash
    openssl x509 -req -days 365 \
        -in user-ed25519.csr \
        -CA ca-ed25519.crt \
        -CAkey "$CA_ED_KEY" \
        -engine pkcs11 -CAkeyform engine \
        -CAcreateserial \
        -copy_extensions copy \
        -out user-ed25519.crt

    openssl verify -CAfile ca-ed25519.crt user-ed25519.crt
    ```

### PKCS11 Provider (OpenSSL 3 から推奨)

#### インストール

```bash
sudo apt install -y pkcs11-provider
```

#### 使い方

- (推奨) Provider を有効化する OpenSSL 設定ファイルを用意

    ```bash
    export PKCS11_MODULE_PATH="/usr/lib/softhsm/libsofthsm2.so"
    export OPENSSL_PKCS11_CONF="$PWD/openssl-pkcs11.cnf"

    cat > "$OPENSSL_PKCS11_CONF" <<EOF
openssl_conf = openssl_init

[openssl_init]
providers = provider_sect

[provider_sect]
default = default_sect
pkcs11 = pkcs11_sect

[default_sect]
activate = 1

[pkcs11_sect]
module = ${PKCS11_MODULE_PATH}
activate = 1
EOF

    openssl list -providers \
        -config "$OPENSSL_PKCS11_CONF" \
        -provider default -provider pkcs11
    ```

- (例) 以降で使う変数

    ```bash
    export TOKEN_LABEL="SoftHSM Token"
    export PIN="1234"

    export CA_RSA_KEY="pkcs11:token=${TOKEN_LABEL};object=ca-rsa;type=private;pin-value=${PIN}"
    export CA_EC_KEY="pkcs11:token=${TOKEN_LABEL};object=ca-ec-p256;type=private;pin-value=${PIN}"
    export CA_ED_KEY="pkcs11:token=${TOKEN_LABEL};object=ca-ed25519;type=private;pin-value=${PIN}"

    export USER_RSA_KEY="pkcs11:token=${TOKEN_LABEL};object=user-rsa;type=private;pin-value=${PIN}"
    export USER_EC_KEY="pkcs11:token=${TOKEN_LABEL};object=user-ec-p256;type=private;pin-value=${PIN}"
    export USER_ED_KEY="pkcs11:token=${TOKEN_LABEL};object=user-ed25519;type=private;pin-value=${PIN}"
    ```

#### 署名・署名検証

```bash
printf 'hello pkcs11\n' > msg.txt
```

- RSA (PKCS#1 v1.5) 署名 / 検証

    ```bash
    openssl pkey \
        -config "$OPENSSL_PKCS11_CONF" -provider default -provider pkcs11 \
        -in "$USER_RSA_KEY" \
        -pubout -out user-rsa-pub.pem

    openssl dgst -sha256 \
        -config "$OPENSSL_PKCS11_CONF" -provider default -provider pkcs11 \
        -sign "$USER_RSA_KEY" \
        -out rsa_pkcs1.sig msg.txt

    openssl dgst -sha256 \
        -verify user-rsa-pub.pem \
        -signature rsa_pkcs1.sig msg.txt
    ```

- RSA (PSS) 署名 / 検証

    ```bash
    openssl dgst -sha256 \
        -sigopt rsa_padding_mode:pss \
        -sigopt rsa_mgf1_md:sha256 \
        -sigopt rsa_pss_saltlen:-1 \
        -config "$OPENSSL_PKCS11_CONF" -provider default -provider pkcs11 \
        -sign "$USER_RSA_KEY" \
        -out rsa_pss.sig msg.txt

    openssl dgst -sha256 \
        -sigopt rsa_padding_mode:pss \
        -sigopt rsa_mgf1_md:sha256 \
        -sigopt rsa_pss_saltlen:-1 \
        -verify user-rsa-pub.pem \
        -signature rsa_pss.sig msg.txt
    ```

- ECDSA P-256 署名 / 検証

    ```bash
    openssl pkey \
        -config "$OPENSSL_PKCS11_CONF" -provider default -provider pkcs11 \
        -in "$USER_EC_KEY" \
        -pubout -out user-ec-pub.pem

    openssl dgst -sha256 \
        -config "$OPENSSL_PKCS11_CONF" -provider default -provider pkcs11 \
        -sign "$USER_EC_KEY" \
        -out ecdsa_p256.sig msg.txt

    openssl dgst -sha256 \
        -verify user-ec-pub.pem \
        -signature ecdsa_p256.sig msg.txt
    ```

- EdDSA ed25519 署名 / 検証

    ```bash
    openssl pkey \
        -config "$OPENSSL_PKCS11_CONF" -provider default -provider pkcs11 \
        -in "$USER_ED_KEY" \
        -pubout -out user-ed25519-pub.pem

    openssl pkeyutl \
        -config "$OPENSSL_PKCS11_CONF" -provider default -provider pkcs11 \
        -sign -inkey "$USER_ED_KEY" \
        -in msg.txt -out ed25519.sig

    openssl pkeyutl \
        -verify -pubin -inkey user-ed25519-pub.pem \
        -in msg.txt -sigfile ed25519.sig
    ```

#### 自己証明書の作成（CA用途を想定）

- RSA の自己証明書

    ```bash
    openssl req -new -x509 -days 3650 \
        -config "$OPENSSL_PKCS11_CONF" -provider default -provider pkcs11 \
        -key "$CA_RSA_KEY" \
        -out ca-rsa.crt \
        -subj "/CN=Example RSA CA" \
        -addext "basicConstraints=critical,CA:TRUE" \
        -addext "keyUsage=critical,keyCertSign,cRLSign"
    ```

- ECDSA P-256 の自己証明書

    ```bash
    openssl req -new -x509 -days 3650 \
        -config "$OPENSSL_PKCS11_CONF" -provider default -provider pkcs11 \
        -key "$CA_EC_KEY" \
        -out ca-ec-p256.crt \
        -subj "/CN=Example ECDSA P-256 CA" \
        -addext "basicConstraints=critical,CA:TRUE" \
        -addext "keyUsage=critical,keyCertSign,cRLSign"
    ```

- EdDSA ed25519 の自己証明書

    ```bash
    openssl req -new -x509 -days 3650 \
        -config "$OPENSSL_PKCS11_CONF" -provider default -provider pkcs11 \
        -key "$CA_ED_KEY" \
        -out ca-ed25519.crt \
        -subj "/CN=Example Ed25519 CA" \
        -addext "basicConstraints=critical,CA:TRUE" \
        -addext "keyUsage=critical,keyCertSign,cRLSign"
    ```

#### CSR 作成

- RSA の CSR

    ```bash
    openssl req -new \
        -config "$OPENSSL_PKCS11_CONF" -provider default -provider pkcs11 \
        -key "$USER_RSA_KEY" \
        -out user-rsa.csr \
        -subj "/CN=host1.example.com" \
        -addext "subjectAltName=DNS:host1.example.com,DNS:localhost,IP:127.0.0.1"
    ```

- ECDSA P-256 の CSR

    ```bash
    openssl req -new \
        -config "$OPENSSL_PKCS11_CONF" -provider default -provider pkcs11 \
        -key "$USER_EC_KEY" \
        -out user-ec-p256.csr \
        -subj "/CN=host1.example.com" \
        -addext "subjectAltName=DNS:host1.example.com,DNS:localhost,IP:127.0.0.1"
    ```

- EdDSA ed25519 の CSR

    ```bash
    openssl req -new \
        -config "$OPENSSL_PKCS11_CONF" -provider default -provider pkcs11 \
        -key "$USER_ED_KEY" \
        -out user-ed25519.csr \
        -subj "/CN=host1.example.com" \
        -addext "subjectAltName=DNS:host1.example.com,DNS:localhost,IP:127.0.0.1"
    ```

#### CA として CSR に署名して証明書発行

- RSA CA で RSA CSR に署名

    ```bash
    openssl x509 -req -days 365 \
        -in user-rsa.csr \
        -CA ca-rsa.crt \
        -CAkey "$CA_RSA_KEY" \
        -config "$OPENSSL_PKCS11_CONF" -provider default -provider pkcs11 \
        -CAcreateserial \
        -copy_extensions copy \
        -out user-rsa.crt

    openssl verify -CAfile ca-rsa.crt user-rsa.crt
    ```

- ECDSA P-256 CA で ECDSA CSR に署名

    ```bash
    openssl x509 -req -days 365 \
        -in user-ec-p256.csr \
        -CA ca-ec-p256.crt \
        -CAkey "$CA_EC_KEY" \
        -config "$OPENSSL_PKCS11_CONF" -provider default -provider pkcs11 \
        -CAcreateserial \
        -copy_extensions copy \
        -out user-ec-p256.crt

    openssl verify -CAfile ca-ec-p256.crt user-ec-p256.crt
    ```

- EdDSA ed25519 CA で Ed25519 CSR に署名

    ```bash
    openssl x509 -req -days 365 \
        -in user-ed25519.csr \
        -CA ca-ed25519.crt \
        -CAkey "$CA_ED_KEY" \
        -config "$OPENSSL_PKCS11_CONF" -provider default -provider pkcs11 \
        -CAcreateserial \
        -copy_extensions copy \
        -out user-ed25519.crt

    openssl verify -CAfile ca-ed25519.crt user-ed25519.crt
    ```


## サンプルコード

!!! note
    証明書操作（自己署名証明書・CSR・CA署名）のサンプルでは、`cryptography` ライブラリへの
    カスタムキーラッパーを介して PKCS#11 トークン上の鍵で署名します。
    `cryptography >= 42` で動作確認しています。

!!! warning
    `PKCS11RSAPrivateKey` 等のラッパークラスは `cryptography` の ABCを実装しますが、
    `private_numbers()` や `private_bytes()` は HSM 鍵を外部に出さないため
    `NotImplementedError` を送出します。証明書の **署名** 処理に限って使用してください。

### PyKCS11

#### インストール

```bash
pip install PyKCS11 cryptography asn1crypto
```

#### 鍵作成

```python
import ctypes
import PyKCS11
from asn1crypto.keys import ECDomainParameters, NamedCurve

PKCS11_MODULE_PATH = "/usr/local/lib/softhsm/libsofthsm2.so"
PIN = "1234"

lib = PyKCS11.PyKCS11Lib()
lib.load(PKCS11_MODULE_PATH)
session = lib.openSession(
    lib.getSlotList(tokenPresent=True)[0],
    PyKCS11.CKF_SERIAL_SESSION | PyKCS11.CKF_RW_SESSION,
)
session.login(PIN)

# ———— RSA 4096 ————
session.generateKeyPair(
    [
        (PyKCS11.CKA_TOKEN,           PyKCS11.CK_TRUE),
        (PyKCS11.CKA_VERIFY,          PyKCS11.CK_TRUE),
        (PyKCS11.CKA_LABEL,           "rsa-key"),
        (PyKCS11.CKA_ID,              (0x01,)),
        (PyKCS11.CKA_MODULUS_BITS,    4096),
        (PyKCS11.CKA_PUBLIC_EXPONENT, (0x01, 0x00, 0x01)),
    ],
    [
        (PyKCS11.CKA_TOKEN,     PyKCS11.CK_TRUE),
        (PyKCS11.CKA_SIGN,      PyKCS11.CK_TRUE),
        (PyKCS11.CKA_SENSITIVE, PyKCS11.CK_TRUE),
        (PyKCS11.CKA_LABEL,     "rsa-key"),
        (PyKCS11.CKA_ID,        (0x01,)),
    ],
    PyKCS11.Mechanism(PyKCS11.CKM_RSA_PKCS_KEY_PAIR_GEN),
)

# ———— ECDSA P-256 ————
ec_params = ECDomainParameters(name="named", value=NamedCurve("prime256v1")).dump()
session.generateKeyPair(
    [
        (PyKCS11.CKA_TOKEN,     PyKCS11.CK_TRUE),
        (PyKCS11.CKA_VERIFY,    PyKCS11.CK_TRUE),
        (PyKCS11.CKA_LABEL,     "ec-p256-key"),
        (PyKCS11.CKA_ID,        (0x02,)),
        (PyKCS11.CKA_KEY_TYPE,  PyKCS11.CKK_ECDSA),
        (PyKCS11.CKA_EC_PARAMS, ec_params),
    ],
    [
        (PyKCS11.CKA_TOKEN,     PyKCS11.CK_TRUE),
        (PyKCS11.CKA_SIGN,      PyKCS11.CK_TRUE),
        (PyKCS11.CKA_SENSITIVE, PyKCS11.CK_TRUE),
        (PyKCS11.CKA_LABEL,     "ec-p256-key"),
        (PyKCS11.CKA_ID,        (0x02,)),
    ],
    PyKCS11.MechanismECGENERATEKEYPAIR,
)

# ———— EdDSA ed25519 ————
ED25519_OID_DER = b"\x06\x03\x2b\x65\x70"  # OID 1.3.101.112
session.generateKeyPair(
    [
        (PyKCS11.CKA_TOKEN,           PyKCS11.CK_TRUE),
        (PyKCS11.CKA_VERIFY,          PyKCS11.CK_TRUE),
        (PyKCS11.CKA_KEY_TYPE,        PyKCS11.CKK_EC_EDWARDS),
        (PyKCS11.CKA_LABEL,           "ed25519-key"),
        (PyKCS11.CKA_ID,              (0x03,)),
        (PyKCS11.CKA_EC_PARAMS,       ED25519_OID_DER),
    ],
    [
        (PyKCS11.CKA_TOKEN,           PyKCS11.CK_TRUE),
        (PyKCS11.CKA_SIGN,            PyKCS11.CK_TRUE),
        (PyKCS11.CKA_SENSITIVE,       PyKCS11.CK_TRUE),
        (PyKCS11.CKA_EXTRACTABLE,     PyKCS11.CK_FALSE),
        (PyKCS11.CKA_KEY_TYPE,        PyKCS11.CKK_EC_EDWARDS),
        (PyKCS11.CKA_LABEL,           "ed25519-key"),
        (PyKCS11.CKA_ID,              (0x03,)),
    ],
    PyKCS11.Mechanism(PyKCS11.CKM_EC_EDWARDS_KEY_PAIR_GEN, None),
)

session.logout()
session.closeSession()
```

#### 署名・署名検証

```python
import ctypes
import PyKCS11
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding as asym_padding
from cryptography.hazmat.primitives.asymmetric.rsa import RSAPublicNumbers
from cryptography.hazmat.primitives.asymmetric.ec import (
    EllipticCurvePublicNumbers, SECP256R1, ECDSA,
)
from cryptography.hazmat.primitives.asymmetric.utils import (
    encode_dss_signature,
)
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PublicKey

PKCS11_MODULE_PATH = "/usr/local/lib/softhsm/libsofthsm2.so"
PIN = "1234"
DATA = b"hello pkcs11"


# ———— セッション取得 ————
lib = PyKCS11.PyKCS11Lib()
lib.load(PKCS11_MODULE_PATH)
session = lib.openSession(
    lib.getSlotList(tokenPresent=True)[0],
    PyKCS11.CKF_SERIAL_SESSION | PyKCS11.CKF_RW_SESSION,
)
session.login(PIN)


# ———— ヘルパー: 鍵ハンドル取得 ————
def _find(label, cls):
    return session.findObjects(
        [(PyKCS11.CKA_LABEL, label), (PyKCS11.CKA_CLASS, cls)]
    )[0]


# ———— ヘルパー: 公開鍵エクスポート ————
def rsa_pub_key(label):
    """RSA 公開鍵を cryptography オブジェクトとして取得"""
    h = _find(label, PyKCS11.CKO_PUBLIC_KEY)
    n_b, e_b = [
        bytes(v)
        for v in session.getAttributeValue(
            h, [PyKCS11.CKA_MODULUS, PyKCS11.CKA_PUBLIC_EXPONENT]
        )
    ]
    return RSAPublicNumbers(
        int.from_bytes(e_b, "big"), int.from_bytes(n_b, "big")
    ).public_key()


def ec_p256_pub_key(label):
    """ECDSA P-256 公開鍵を cryptography オブジェクトとして取得"""
    h = _find(label, PyKCS11.CKO_PUBLIC_KEY)
    pt = bytes(session.getAttributeValue(h, [PyKCS11.CKA_EC_POINT])[0])
    # CKA_EC_POINT は DER OCTET STRING: 04 41 <65-byte uncompressed point>
    raw = pt[2:]  # strip 04 41
    assert raw[0] == 0x04, "非圧縮点でなければなりません"
    x = int.from_bytes(raw[1:33], "big")
    y = int.from_bytes(raw[33:65], "big")
    return EllipticCurvePublicNumbers(x, y, SECP256R1()).public_key()


def ed25519_pub_key(label):
    """Ed25519 公開鍵を cryptography オブジェクトとして取得"""
    h = _find(label, PyKCS11.CKO_PUBLIC_KEY)
    pt = bytes(session.getAttributeValue(h, [PyKCS11.CKA_EC_POINT])[0])
    # DER OCTET STRING: 04 20 <32-byte raw key>
    raw = pt[2:]  # strip 04 20
    return Ed25519PublicKey.from_public_bytes(raw)


# ———— CK_RSA_PKCS_PSS_PARAMS (PSS 用) ————
class CK_RSA_PKCS_PSS_PARAMS(ctypes.Structure):
    _fields_ = [
        ("hashAlg", ctypes.c_ulong),
        ("mgf",     ctypes.c_ulong),
        ("sLen",    ctypes.c_ulong),
    ]


# ———— RSA (PKCS#1 v1.5) 署名 / 検証 ————
priv_rsa = _find("rsa-key", PyKCS11.CKO_PRIVATE_KEY)
sig_pkcs1 = bytes(
    session.sign(priv_rsa, DATA, PyKCS11.Mechanism(PyKCS11.CKM_SHA256_RSA_PKCS))
)
pub_rsa = rsa_pub_key("rsa-key")
pub_rsa.verify(sig_pkcs1, DATA, asym_padding.PKCS1v15(), hashes.SHA256())
print("RSA PKCS#1 v1.5: OK")

# ———— RSA (PSS) 署名 / 検証 ————
pss_p = CK_RSA_PKCS_PSS_PARAMS(
    hashAlg=PyKCS11.CKM_SHA256,       # 0x00000250
    mgf=PyKCS11.CKG_MGF1_SHA256,      # 0x00000002
    sLen=32,
)
mech_pss = PyKCS11.Mechanism(PyKCS11.CKM_SHA256_RSA_PKCS_PSS, pss_p)
sig_pss = bytes(session.sign(priv_rsa, DATA, mech_pss))
pub_rsa.verify(
    sig_pss, DATA,
    asym_padding.PSS(
        mgf=asym_padding.MGF1(hashes.SHA256()),
        salt_length=32,
    ),
    hashes.SHA256(),
)
print("RSA PSS: OK")

# ———— ECDSA P-256 署名 / 検証 ————
priv_ec = _find("ec-p256-key", PyKCS11.CKO_PRIVATE_KEY)
sig_raw = bytes(
    session.sign(priv_ec, DATA, PyKCS11.Mechanism(PyKCS11.CKM_ECDSA_SHA256))
)
# PKCS#11 ECDSA 署名は raw R||S (各 32 bytes)、cryptography は DER を要求
r = int.from_bytes(sig_raw[:32], "big")
s = int.from_bytes(sig_raw[32:], "big")
sig_ecdsa_der = encode_dss_signature(r, s)
pub_ec = ec_p256_pub_key("ec-p256-key")
pub_ec.verify(sig_ecdsa_der, DATA, ECDSA(hashes.SHA256()))
print("ECDSA P-256: OK")

# ———— EdDSA ed25519 署名 / 検証 ————
priv_ed = _find("ed25519-key", PyKCS11.CKO_PRIVATE_KEY)
sig_ed = bytes(
    session.sign(priv_ed, DATA, PyKCS11.Mechanism(PyKCS11.CKM_EDDSA))
)
pub_ed = ed25519_pub_key("ed25519-key")
pub_ed.verify(sig_ed, DATA)  # 例外なければ検証成功
print("EdDSA ed25519: OK")

session.logout()
session.closeSession()
```

#### 証明書操作（自己署名証明書・CSR・CA署名）

```python
"""
PyKCS11 + cryptography による X.509 証明書操作サンプル。

cryptography の CertificateBuilder / CertificateSigningRequestBuilder に
PyKCS11 でラップしたカスタム秘密鍵オブジェクトを渡して署名します。
"""
import ctypes
import datetime

import PyKCS11
from cryptography import x509
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding as asym_padding
from cryptography.hazmat.primitives.asymmetric.rsa import RSAPublicNumbers, RSAPrivateKey
from cryptography.hazmat.primitives.asymmetric.ec import (
    EllipticCurvePublicNumbers, SECP256R1, ECDSA, EllipticCurvePrivateKey,
)
from cryptography.hazmat.primitives.asymmetric.utils import encode_dss_signature
from cryptography.hazmat.primitives.asymmetric.ed25519 import (
    Ed25519PublicKey, Ed25519PrivateKey,
)
from cryptography.x509.oid import NameOID

PKCS11_MODULE_PATH = "/usr/local/lib/softhsm/libsofthsm2.so"
PIN = "1234"

lib = PyKCS11.PyKCS11Lib()
lib.load(PKCS11_MODULE_PATH)
session = lib.openSession(
    lib.getSlotList(tokenPresent=True)[0],
    PyKCS11.CKF_SERIAL_SESSION | PyKCS11.CKF_RW_SESSION,
)
session.login(PIN)


def _find(label, cls):
    return session.findObjects(
        [(PyKCS11.CKA_LABEL, label), (PyKCS11.CKA_CLASS, cls)]
    )[0]


# ======== 公開鍵エクスポートヘルパー（署名・検証サンプルと同じ） ========
def rsa_pub_key(label):
    h = _find(label, PyKCS11.CKO_PUBLIC_KEY)
    n_b, e_b = [
        bytes(v)
        for v in session.getAttributeValue(
            h, [PyKCS11.CKA_MODULUS, PyKCS11.CKA_PUBLIC_EXPONENT]
        )
    ]
    return RSAPublicNumbers(
        int.from_bytes(e_b, "big"), int.from_bytes(n_b, "big")
    ).public_key()


def ec_p256_pub_key(label):
    h = _find(label, PyKCS11.CKO_PUBLIC_KEY)
    pt = bytes(session.getAttributeValue(h, [PyKCS11.CKA_EC_POINT])[0])
    raw = pt[2:]
    x = int.from_bytes(raw[1:33], "big")
    y = int.from_bytes(raw[33:65], "big")
    return EllipticCurvePublicNumbers(x, y, SECP256R1()).public_key()


def ed25519_pub_key(label):
    h = _find(label, PyKCS11.CKO_PUBLIC_KEY)
    pt = bytes(session.getAttributeValue(h, [PyKCS11.CKA_EC_POINT])[0])
    return Ed25519PublicKey.from_public_bytes(pt[2:])


# ======== CKM_SHA256_RSA_PKCS PSS パラメータ ========
class CK_RSA_PKCS_PSS_PARAMS(ctypes.Structure):
    _fields_ = [
        ("hashAlg", ctypes.c_ulong),
        ("mgf",     ctypes.c_ulong),
        ("sLen",    ctypes.c_ulong),
    ]


# ======== カスタム秘密鍵ラッパー ========
class PKCS11RSAPrivateKey(RSAPrivateKey):
    """RSA HSM 秘密鍵を cryptography RSAPrivateKey として公開するラッパー"""
    def __init__(self, label, pub_key):
        self._label = label
        self._pub_key = pub_key

    def sign(self, data, padding, algorithm):
        priv = _find(self._label, PyKCS11.CKO_PRIVATE_KEY)
        if isinstance(padding, asym_padding.PKCS1v15):
            mech = PyKCS11.Mechanism(PyKCS11.CKM_SHA256_RSA_PKCS)
        elif isinstance(padding, asym_padding.PSS):
            pss_p = CK_RSA_PKCS_PSS_PARAMS(
                hashAlg=PyKCS11.CKM_SHA256,
                mgf=PyKCS11.CKG_MGF1_SHA256,
                sLen=32,
            )
            mech = PyKCS11.Mechanism(PyKCS11.CKM_SHA256_RSA_PKCS_PSS, pss_p)
        else:
            raise ValueError(f"非対応パディング: {padding}")
        return bytes(session.sign(priv, data, mech))

    @property
    def key_size(self):
        return self._pub_key.key_size

    def public_key(self):
        return self._pub_key

    def decrypt(self, ciphertext, padding):
        raise NotImplementedError

    def private_numbers(self):
        raise NotImplementedError

    def private_bytes(self, encoding, format, encryption_algorithm):
        raise NotImplementedError


class PKCS11ECPrivateKey(EllipticCurvePrivateKey):
    """ECDSA P-256 HSM 秘密鍵を cryptography EllipticCurvePrivateKey として公開するラッパー"""
    def __init__(self, label, pub_key):
        self._label = label
        self._pub_key = pub_key

    def sign(self, data, signature_algorithm):
        priv = _find(self._label, PyKCS11.CKO_PRIVATE_KEY)
        sig_raw = bytes(
            session.sign(priv, data, PyKCS11.Mechanism(PyKCS11.CKM_ECDSA_SHA256))
        )
        r = int.from_bytes(sig_raw[:32], "big")
        s = int.from_bytes(sig_raw[32:], "big")
        return encode_dss_signature(r, s)

    @property
    def curve(self):
        return SECP256R1()

    @property
    def key_size(self):
        return 256

    def public_key(self):
        return self._pub_key

    def exchange(self, algorithm, peer_public_key):
        raise NotImplementedError

    def private_numbers(self):
        raise NotImplementedError

    def private_bytes(self, encoding, format, encryption_algorithm):
        raise NotImplementedError


class PKCS11Ed25519PrivateKey(Ed25519PrivateKey):
    """Ed25519 HSM 秘密鍵を cryptography Ed25519PrivateKey として公開するラッパー"""
    def __init__(self, label, pub_key):
        self._label = label
        self._pub_key = pub_key

    def sign(self, data):
        priv = _find(self._label, PyKCS11.CKO_PRIVATE_KEY)
        return bytes(
            session.sign(priv, data, PyKCS11.Mechanism(PyKCS11.CKM_EDDSA))
        )

    def public_key(self):
        return self._pub_key

    def private_bytes(self, encoding, format, encryption_algorithm):
        raise NotImplementedError

    def private_bytes_raw(self):
        raise NotImplementedError


# ======== ラッパーキーオブジェクト生成 ========
rsa_key  = PKCS11RSAPrivateKey("rsa-key",    rsa_pub_key("rsa-key"))
ec_key   = PKCS11ECPrivateKey("ec-p256-key", ec_p256_pub_key("ec-p256-key"))
ed_key   = PKCS11Ed25519PrivateKey("ed25519-key", ed25519_pub_key("ed25519-key"))

NOW  = datetime.datetime.now(datetime.timezone.utc)
YEAR = datetime.timedelta(days=365)
TEN_YEARS = datetime.timedelta(days=3650)


# ======== 自己署名証明書（CA 用途） ========
def self_signed_ca_cert(key, cn, algo):
    name = x509.Name([x509.NameAttribute(NameOID.COMMON_NAME, cn)])
    return (
        x509.CertificateBuilder()
        .subject_name(name)
        .issuer_name(name)
        .public_key(key.public_key())
        .serial_number(x509.random_serial_number())
        .not_valid_before(NOW)
        .not_valid_after(NOW + TEN_YEARS)
        .add_extension(x509.BasicConstraints(ca=True, path_length=None), critical=True)
        .add_extension(
            x509.KeyUsage(
                digital_signature=True, key_cert_sign=True, crl_sign=True,
                content_commitment=False, key_encipherment=False,
                data_encipherment=False, key_agreement=False,
                encipher_only=False, decipher_only=False,
            ),
            critical=True,
        )
        .sign(key, algo)
    )


# RSA 自己証明書
ca_rsa_cert = self_signed_ca_cert(rsa_key, "Example RSA CA", hashes.SHA256())
with open("ca-rsa.crt", "wb") as f:
    f.write(ca_rsa_cert.public_bytes(serialization.Encoding.PEM))
print("自己署名証明書 (RSA): ca-rsa.crt")

# ECDSA P-256 自己証明書
ca_ec_cert = self_signed_ca_cert(ec_key, "Example ECDSA P-256 CA", hashes.SHA256())
with open("ca-ec-p256.crt", "wb") as f:
    f.write(ca_ec_cert.public_bytes(serialization.Encoding.PEM))
print("自己署名証明書 (ECDSA P-256): ca-ec-p256.crt")

# EdDSA ed25519 自己証明書（Ed25519 は algorithm=None）
ca_ed_cert = self_signed_ca_cert(ed_key, "Example Ed25519 CA", None)
with open("ca-ed25519.crt", "wb") as f:
    f.write(ca_ed_cert.public_bytes(serialization.Encoding.PEM))
print("自己署名証明書 (Ed25519): ca-ed25519.crt")


# ======== CSR 作成 ========
def make_csr(key, cn, sans, algo):
    name = x509.Name([x509.NameAttribute(NameOID.COMMON_NAME, cn)])
    return (
        x509.CertificateSigningRequestBuilder()
        .subject_name(name)
        .add_extension(
            x509.SubjectAlternativeName(sans),
            critical=False,
        )
        .sign(key, algo)
    )


SANS = [x509.DNSName("host1.example.com"), x509.IPAddress(__import__("ipaddress").IPv4Address("127.0.0.1"))]

# RSA CSR
csr_rsa = make_csr(rsa_key, "host1.example.com", SANS, hashes.SHA256())
with open("user-rsa.csr", "wb") as f:
    f.write(csr_rsa.public_bytes(serialization.Encoding.PEM))
print("CSR (RSA): user-rsa.csr")

# ECDSA P-256 CSR
csr_ec = make_csr(ec_key, "host1.example.com", SANS, hashes.SHA256())
with open("user-ec-p256.csr", "wb") as f:
    f.write(csr_ec.public_bytes(serialization.Encoding.PEM))
print("CSR (ECDSA P-256): user-ec-p256.csr")

# Ed25519 CSR
csr_ed = make_csr(ed_key, "host1.example.com", SANS, None)
with open("user-ed25519.csr", "wb") as f:
    f.write(csr_ed.public_bytes(serialization.Encoding.PEM))
print("CSR (Ed25519): user-ed25519.csr")


# ======== CA として CSR に署名して証明書発行 ========
def sign_csr(ca_cert, ca_key, csr, algo):
    return (
        x509.CertificateBuilder()
        .subject_name(csr.subject)
        .issuer_name(ca_cert.subject)
        .public_key(csr.public_key())
        .serial_number(x509.random_serial_number())
        .not_valid_before(NOW)
        .not_valid_after(NOW + YEAR)
        .add_extension(
            x509.BasicConstraints(ca=False, path_length=None), critical=True
        )
        # CSR の extensions をコピー
        .add_extension(
            csr.extensions.get_extension_for_class(x509.SubjectAlternativeName).value,
            critical=False,
        )
        .sign(ca_key, algo)
    )


# RSA CA が RSA CSR に署名
user_rsa_cert = sign_csr(ca_rsa_cert, rsa_key, csr_rsa, hashes.SHA256())
with open("user-rsa.crt", "wb") as f:
    f.write(user_rsa_cert.public_bytes(serialization.Encoding.PEM))
print("証明書発行 (RSA CA → RSA): user-rsa.crt")

# ECDSA CA が ECDSA CSR に署名
user_ec_cert = sign_csr(ca_ec_cert, ec_key, csr_ec, hashes.SHA256())
with open("user-ec-p256.crt", "wb") as f:
    f.write(user_ec_cert.public_bytes(serialization.Encoding.PEM))
print("証明書発行 (ECDSA CA → ECDSA): user-ec-p256.crt")

# Ed25519 CA が Ed25519 CSR に署名
user_ed_cert = sign_csr(ca_ed_cert, ed_key, csr_ed, None)
with open("user-ed25519.crt", "wb") as f:
    f.write(user_ed_cert.public_bytes(serialization.Encoding.PEM))
print("証明書発行 (Ed25519 CA → Ed25519): user-ed25519.crt")

session.logout()
session.closeSession()
```

---

### python-pkcs11

#### インストール

```bash
pip install python-pkcs11 cryptography
```

#### 鍵作成

```python
import pkcs11
from pkcs11 import Attribute, KeyType
from pkcs11.util.ec import encode_named_curve_parameters

PKCS11_MODULE_PATH = "/usr/local/lib/softhsm/libsofthsm2.so"
TOKEN_LABEL = "SoftHSM Token"
PIN = "1234"

# Ed25519 OID 1.3.101.112 の DER
ED25519_OID_DER = b"\x06\x03\x2b\x65\x70"

lib = pkcs11.lib(PKCS11_MODULE_PATH)
token = lib.get_token(token_label=TOKEN_LABEL)

with token.open(user_pin=PIN, rw=True) as session:
    # ———— RSA 4096 ————
    session.generate_keypair(
        KeyType.RSA, 4096,
        store=True,
        label="rsa-key",
        id=b"\x01",
    )

    # ———— ECDSA P-256 ————
    ec_params = encode_named_curve_parameters("prime256v1")
    session.generate_keypair(
        KeyType.EC, ec_params,
        store=True,
        label="ec-p256-key",
        id=b"\x02",
    )

    # ———— EdDSA ed25519 ————
    # KeyType.EC_EDWARDS = CKK_EC_EDWARDS (0x40, PKCS#11 v3.0)
    session.generate_keypair(
        KeyType.EC_EDWARDS, ED25519_OID_DER,
        store=True,
        label="ed25519-key",
        id=b"\x03",
    )

print("鍵作成完了")
```

#### 署名・署名検証

```python
import pkcs11
from pkcs11 import Attribute, KeyType, Mechanism, ObjectClass
from pkcs11.util.rsa import encode_rsa_public_key
from pkcs11.util.ec import encode_ec_public_key
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding as asym_padding
from cryptography.hazmat.primitives.asymmetric.ec import ECDSA
from cryptography.hazmat.primitives.asymmetric.utils import encode_dss_signature
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PublicKey
from cryptography.hazmat.primitives.serialization import load_der_public_key

PKCS11_MODULE_PATH = "/usr/local/lib/softhsm/libsofthsm2.so"
TOKEN_LABEL = "SoftHSM Token"
PIN = "1234"
DATA = b"hello pkcs11"

lib = pkcs11.lib(PKCS11_MODULE_PATH)
token = lib.get_token(token_label=TOKEN_LABEL)

with token.open(user_pin=PIN, rw=True) as session:

    # ———— 鍵オブジェクト取得 ————
    priv_rsa = session.get_key(
        label="rsa-key", key_type=KeyType.RSA, object_class=ObjectClass.PRIVATE_KEY
    )
    pub_rsa_p11 = session.get_key(
        label="rsa-key", key_type=KeyType.RSA, object_class=ObjectClass.PUBLIC_KEY
    )
    priv_ec = session.get_key(
        label="ec-p256-key", key_type=KeyType.EC, object_class=ObjectClass.PRIVATE_KEY
    )
    pub_ec_p11 = session.get_key(
        label="ec-p256-key", key_type=KeyType.EC, object_class=ObjectClass.PUBLIC_KEY
    )
    priv_ed = session.get_key(
        label="ed25519-key", key_type=KeyType.EC_EDWARDS, object_class=ObjectClass.PRIVATE_KEY
    )
    pub_ed_p11 = session.get_key(
        label="ed25519-key", key_type=KeyType.EC_EDWARDS, object_class=ObjectClass.PUBLIC_KEY
    )

    # ———— 公開鍵をエクスポート (cryptography 形式) ————
    # RSA / EC は util 関数で DER SubjectPublicKeyInfo を取得して変換
    pub_rsa  = load_der_public_key(encode_rsa_public_key(pub_rsa_p11))
    pub_ec   = load_der_public_key(encode_ec_public_key(pub_ec_p11))

    # Ed25519 は CKA_EC_POINT から raw bytes を取得
    ec_point = pub_ed_p11[Attribute.EC_POINT]  # DER OCTET STRING: 04 20 <32 bytes>
    pub_ed = Ed25519PublicKey.from_public_bytes(ec_point[2:])

    # ———— RSA (PKCS#1 v1.5) 署名 / 検証 ————
    sig_pkcs1 = priv_rsa.sign(DATA, mechanism=Mechanism.SHA256_RSA_PKCS)
    pub_rsa.verify(sig_pkcs1, DATA, asym_padding.PKCS1v15(), hashes.SHA256())
    print("RSA PKCS#1 v1.5: OK")

    # ———— RSA (PSS) 署名 / 検証 ————
    # mechanism_param = (hashAlg, mgf, sLen)
    sig_pss = priv_rsa.sign(
        DATA,
        mechanism=Mechanism.SHA256_RSA_PKCS_PSS,
        mechanism_param=(Mechanism.SHA256, pkcs11.MGF.SHA256, 32),
    )
    pub_rsa.verify(
        sig_pss, DATA,
        asym_padding.PSS(
            mgf=asym_padding.MGF1(hashes.SHA256()),
            salt_length=32,
        ),
        hashes.SHA256(),
    )
    print("RSA PSS: OK")

    # ———— ECDSA P-256 署名 / 検証 ————
    sig_raw = priv_ec.sign(DATA, mechanism=Mechanism.ECDSA_SHA256)
    # raw R||S → DER
    r = int.from_bytes(sig_raw[:32], "big")
    s = int.from_bytes(sig_raw[32:], "big")
    sig_ecdsa_der = encode_dss_signature(r, s)
    pub_ec.verify(sig_ecdsa_der, DATA, ECDSA(hashes.SHA256()))
    print("ECDSA P-256: OK")

    # ———— EdDSA ed25519 署名 / 検証 ————
    sig_ed = priv_ed.sign(DATA, mechanism=Mechanism.EDDSA)
    pub_ed.verify(sig_ed, DATA)
    print("EdDSA ed25519: OK")
```

#### 証明書操作（自己署名証明書・CSR・CA署名）

```python
"""
python-pkcs11 + cryptography による X.509 証明書操作サンプル。
PyKCS11 のサンプルと同じカスタムラッパーパターンを使用します。
"""
import datetime
import ipaddress

import pkcs11
from pkcs11 import Attribute, KeyType, Mechanism, ObjectClass
from pkcs11.util.rsa import encode_rsa_public_key
from pkcs11.util.ec import encode_ec_public_key
from cryptography import x509
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding as asym_padding
from cryptography.hazmat.primitives.asymmetric.ec import (
    SECP256R1, EllipticCurvePrivateKey,
)
from cryptography.hazmat.primitives.asymmetric.utils import encode_dss_signature
from cryptography.hazmat.primitives.asymmetric.rsa import RSAPrivateKey
from cryptography.hazmat.primitives.asymmetric.ed25519 import (
    Ed25519PublicKey, Ed25519PrivateKey,
)
from cryptography.hazmat.primitives.serialization import load_der_public_key
from cryptography.x509.oid import NameOID

PKCS11_MODULE_PATH = "/usr/local/lib/softhsm/libsofthsm2.so"
TOKEN_LABEL = "SoftHSM Token"
PIN = "1234"

lib = pkcs11.lib(PKCS11_MODULE_PATH)
token = lib.get_token(token_label=TOKEN_LABEL)

# ======== カスタム秘密鍵ラッパー（PyKCS11 版と同じインターフェース） ========
# session は with ブロック内で取得するため、close 後に使用不可

class P11RSAPrivateKey(RSAPrivateKey):
    def __init__(self, session, label, pub_key):
        self._s, self._label, self._pub = session, label, pub_key

    def sign(self, data, padding, algorithm):
        priv = self._s.get_key(label=self._label, key_type=KeyType.RSA, object_class=ObjectClass.PRIVATE_KEY)
        if isinstance(padding, asym_padding.PKCS1v15):
            return priv.sign(data, mechanism=Mechanism.SHA256_RSA_PKCS)
        elif isinstance(padding, asym_padding.PSS):
            return priv.sign(data, mechanism=Mechanism.SHA256_RSA_PKCS_PSS,
                             mechanism_param=(Mechanism.SHA256, pkcs11.MGF.SHA256, 32))
        raise ValueError(f"非対応パディング: {padding}")

    @property
    def key_size(self): return self._pub.key_size
    def public_key(self): return self._pub
    def decrypt(self, c, p): raise NotImplementedError
    def private_numbers(self): raise NotImplementedError
    def private_bytes(self, e, f, ea): raise NotImplementedError


class P11ECPrivateKey(EllipticCurvePrivateKey):
    def __init__(self, session, label, pub_key):
        self._s, self._label, self._pub = session, label, pub_key

    def sign(self, data, signature_algorithm):
        priv = self._s.get_key(label=self._label, key_type=KeyType.EC, object_class=ObjectClass.PRIVATE_KEY)
        sig_raw = priv.sign(data, mechanism=Mechanism.ECDSA_SHA256)
        r = int.from_bytes(sig_raw[:32], "big")
        s = int.from_bytes(sig_raw[32:], "big")
        return encode_dss_signature(r, s)

    @property
    def curve(self): return SECP256R1()
    @property
    def key_size(self): return 256
    def public_key(self): return self._pub
    def exchange(self, a, p): raise NotImplementedError
    def private_numbers(self): raise NotImplementedError
    def private_bytes(self, e, f, ea): raise NotImplementedError


class P11Ed25519PrivateKey(Ed25519PrivateKey):
    def __init__(self, session, label, pub_key):
        self._s, self._label, self._pub = session, label, pub_key

    def sign(self, data):
        priv = self._s.get_key(label=self._label, key_type=KeyType.EC_EDWARDS, object_class=ObjectClass.PRIVATE_KEY)
        return priv.sign(data, mechanism=Mechanism.EDDSA)

    def public_key(self): return self._pub
    def private_bytes(self, e, f, ea): raise NotImplementedError
    def private_bytes_raw(self): raise NotImplementedError


# ======== 証明書操作 ========
NOW = datetime.datetime.now(datetime.timezone.utc)
YEAR = datetime.timedelta(days=365)
TEN_YEARS = datetime.timedelta(days=3650)
SANS = [x509.DNSName("host1.example.com"), x509.IPAddress(ipaddress.IPv4Address("127.0.0.1"))]


def self_signed_ca_cert(key, cn, algo):
    name = x509.Name([x509.NameAttribute(NameOID.COMMON_NAME, cn)])
    return (
        x509.CertificateBuilder()
        .subject_name(name).issuer_name(name)
        .public_key(key.public_key())
        .serial_number(x509.random_serial_number())
        .not_valid_before(NOW).not_valid_after(NOW + TEN_YEARS)
        .add_extension(x509.BasicConstraints(ca=True, path_length=None), critical=True)
        .add_extension(
            x509.KeyUsage(digital_signature=True, key_cert_sign=True, crl_sign=True,
                          content_commitment=False, key_encipherment=False,
                          data_encipherment=False, key_agreement=False,
                          encipher_only=False, decipher_only=False),
            critical=True,
        )
        .sign(key, algo)
    )


def make_csr(key, cn, sans, algo):
    name = x509.Name([x509.NameAttribute(NameOID.COMMON_NAME, cn)])
    return (
        x509.CertificateSigningRequestBuilder()
        .subject_name(name)
        .add_extension(x509.SubjectAlternativeName(sans), critical=False)
        .sign(key, algo)
    )


def sign_csr(ca_cert, ca_key, csr, algo):
    return (
        x509.CertificateBuilder()
        .subject_name(csr.subject).issuer_name(ca_cert.subject)
        .public_key(csr.public_key())
        .serial_number(x509.random_serial_number())
        .not_valid_before(NOW).not_valid_after(NOW + YEAR)
        .add_extension(x509.BasicConstraints(ca=False, path_length=None), critical=True)
        .add_extension(
            csr.extensions.get_extension_for_class(x509.SubjectAlternativeName).value,
            critical=False,
        )
        .sign(ca_key, algo)
    )


with token.open(user_pin=PIN, rw=True) as session:
    # 公開鍵をエクスポート
    pub_rsa = load_der_public_key(encode_rsa_public_key(
        session.get_key(label="rsa-key", key_type=KeyType.RSA, object_class=ObjectClass.PUBLIC_KEY)
    ))
    pub_ec = load_der_public_key(encode_ec_public_key(
        session.get_key(label="ec-p256-key", key_type=KeyType.EC, object_class=ObjectClass.PUBLIC_KEY)
    ))
    ec_point = session.get_key(
        label="ed25519-key", key_type=KeyType.EC_EDWARDS, object_class=ObjectClass.PUBLIC_KEY
    )[Attribute.EC_POINT]
    pub_ed = Ed25519PublicKey.from_public_bytes(ec_point[2:])

    rsa_key = P11RSAPrivateKey(session, "rsa-key",     pub_rsa)
    ec_key  = P11ECPrivateKey(session,  "ec-p256-key", pub_ec)
    ed_key  = P11Ed25519PrivateKey(session, "ed25519-key", pub_ed)

    # ———— 自己署名証明書 ————
    ca_rsa_cert = self_signed_ca_cert(rsa_key, "Example RSA CA",        hashes.SHA256())
    ca_ec_cert  = self_signed_ca_cert(ec_key,  "Example ECDSA P-256 CA", hashes.SHA256())
    ca_ed_cert  = self_signed_ca_cert(ed_key,  "Example Ed25519 CA",     None)

    for name, cert in [("ca-rsa.crt", ca_rsa_cert), ("ca-ec-p256.crt", ca_ec_cert), ("ca-ed25519.crt", ca_ed_cert)]:
        with open(name, "wb") as f:
            f.write(cert.public_bytes(serialization.Encoding.PEM))
        print(f"自己署名証明書: {name}")

    # ———— CSR 作成 ————
    csr_rsa = make_csr(rsa_key, "host1.example.com", SANS, hashes.SHA256())
    csr_ec  = make_csr(ec_key,  "host1.example.com", SANS, hashes.SHA256())
    csr_ed  = make_csr(ed_key,  "host1.example.com", SANS, None)

    for name, csr in [("user-rsa.csr", csr_rsa), ("user-ec-p256.csr", csr_ec), ("user-ed25519.csr", csr_ed)]:
        with open(name, "wb") as f:
            f.write(csr.public_bytes(serialization.Encoding.PEM))
        print(f"CSR: {name}")

    # ———— CA として CSR に署名 ————
    user_rsa_cert = sign_csr(ca_rsa_cert, rsa_key, csr_rsa, hashes.SHA256())
    user_ec_cert  = sign_csr(ca_ec_cert,  ec_key,  csr_ec,  hashes.SHA256())
    user_ed_cert  = sign_csr(ca_ed_cert,  ed_key,  csr_ed,  None)

    for name, cert in [("user-rsa.crt", user_rsa_cert), ("user-ec-p256.crt", user_ec_cert), ("user-ed25519.crt", user_ed_cert)]:
        with open(name, "wb") as f:
            f.write(cert.public_bytes(serialization.Encoding.PEM))
        print(f"証明書発行: {name}")
```
