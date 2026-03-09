[GitHub Copilot CLI](https://github.com/features/copilot/cli?locale=ja)
===

## 前提条件

- 有効な GitHub Copilot サブスクリプション
    - [Copilot プランの確認](https://github.com/features/copilot/plans)
- Windows の場合、PowerShell v6 以降

---

## Windows

### WinGet を使用したインストール (推奨)

- インストール

    ```powershell
    winget install GitHub.Copilot
    ```

- バージョン確認

    ```powershell
    copilot --version
    ```

---

## macOS

### Homebrew を使用したインストール (推奨)

- Homebrew のインストール (未インストールの場合)

    ```bash
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    ```

- インストール

    ```bash
    brew install copilot-cli
    ```

- バージョン確認

    ```bash
    copilot --version
    ```

### インストールスクリプトを使用したインストール

- インストール (curl)

    ```bash
    curl -fsSL https://gh.io/copilot-install | bash
    ```

- インストール (wget)

    ```bash
    wget -qO- https://gh.io/copilot-install | bash
    ```

- `/usr/local/bin` にインストールする場合

    ```bash
    curl -fsSL https://gh.io/copilot-install | sudo bash
    ```

### npm を使用したインストール

前提条件: Node.js 22 以降

- nvm のインストール

    ```bash
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
    source ~/.bashrc
    ```

- Node.js のインストール

    ```bash
    nvm install --lts
    node -v
    npm -v
    ```

- インストール

    ```bash
    npm install -g @github/copilot
    ```

- バージョン確認

    ```bash
    copilot --version
    ```

---

## 認証

- Copilot CLI の起動

    ```bash
    copilot
    ```

    初回起動時に GitHub にログインしていない場合、`/login` スラッシュコマンドの使用を求められます。

    ```text
    /login
    ```

    表示された URL にブラウザでアクセスし、デバイスコードを入力して認証します。

### Personal Access Token による認証

GitHub の Fine-grained personal access token を使用して認証することもできます。

1. [Fine-grained personal access tokens](https://github.com/settings/personal-access-tokens/new) にアクセス
2. 「アクセス許可を追加」→「Copilot 要求」を選択
3. 「トークンの生成」をクリック
4. 生成されたトークンを環境変数に設定

    - Linux / macOS

        ```bash
        export GH_TOKEN=<生成されたトークン>
        ```

        `.bashrc` または `.zshrc` に追記して永続化することも可能です。

    - Windows (PowerShell)

        ```powershell
        $env:GH_TOKEN = "<生成されたトークン>"
        ```

---

## 使用例

```bash
copilot
```

```text
> どのようなお手伝いができますか？
```

ターミナル上で自然言語によってコマンドの提案や説明を受けることができます。
