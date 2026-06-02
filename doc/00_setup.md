# 事前準備：開発ツールのインストール

> Day 1 の作業を始める前に、このドキュメントの手順をすべて完了させてください。

---

## このプロジェクトで作るもの

10日間で「タスク管理アプリ（TaskBoard）」を構築します。

```
┌──────────────────────────────────────────────┐
│  ブラウザ（Chrome / Edge など）               │
│  http://localhost:8000                        │
└───────────────────┬──────────────────────────┘
                    │ HTTP リクエスト
┌───────────────────▼──────────────────────────┐
│  Django（Web サーバー）                        │
│  ・ログイン・ログアウト                        │
│  ・タスクの作成・編集・削除                    │
│  ・カテゴリ管理・検索                         │
└───────────────────┬──────────────────────────┘
                    │ データの読み書き
┌───────────────────▼──────────────────────────┐
│  PostgreSQL（データベース）                    │
│  ・ユーザー情報                               │
│  ・タスクデータ                               │
└──────────────────────────────────────────────┘

※ Django と PostgreSQL は Docker コンテナの中で動く
```

### 10日間のスケジュール

| Day | 内容 | 完成する状態 |
|-----|------|------------|
| **Day 1** | Docker 開発環境構築 | `docker compose up` で環境が起動する |
| **Day 2** | paiza 学習（Django 基礎） | Django の MVT を理解する |
| **Day 3** | Django 初期設定 + 認証（登録） | DB 接続・ユーザー登録が動く |
| **Day 4** | ログイン・ログアウト | 認証フロー全体が完成する |
| **Day 5** | タスクモデル・マイグレーション | DB にタスクテーブルが作られる |
| **Day 6** | タスク一覧・詳細ビュー | タスクの一覧・詳細ページが表示される |
| **Day 7** | タスク作成・編集・削除 | タスクの CRUD が動作する |
| **Day 8** | カテゴリ機能・UI 改善 | カテゴリ管理・フィルタリングが動く |
| **Day 9** | 検索・ページネーション | 検索・絞り込み・ページ分割が動く |
| **Day 10** | テスト・README・最終確認 | GitHub に提出できる状態になる |

---

## インストールするツール一覧

| ツール | 役割 | インストール済みか確認するコマンド |
|--------|------|--------------------------------|
| Docker Desktop | アプリの実行環境をコンテナで管理する | `docker --version` |
| Git | バージョン管理（変更履歴の記録） | `git --version` |
| VS Code | コードを書くエディタ | — |

---

## 1. Docker Desktop のインストール

### Docker Desktop とは

「コンテナ」と呼ばれる軽量な仮想環境を作るツール。このプロジェクトでは Django と PostgreSQL を別々のコンテナで動かし、`docker compose up` 1コマンドで両方まとめて起動できるようにします。

### インストール手順（Windows）

1. 以下の公式サイトからインストーラーをダウンロードする
   ```
   https://www.docker.com/products/docker-desktop/
   ```

2. ダウンロードした `Docker Desktop Installer.exe` を実行する

3. インストール中に「Use WSL 2 instead of Hyper-V」のチェックが表示される場合は、チェックを入れたままにする

4. インストール完了後、PC を再起動する

5. Docker Desktop を起動する（タスクバーにクジラのアイコンが表示されれば起動中）

### インストール確認

コマンドプロンプト（またはPowerShell）を開いて以下を実行する。

```cmd
docker --version
docker compose version
```

出力例：
```
Docker version 26.0.0, build abc1234
Docker Compose version v2.26.0
```

バージョン番号が表示されれば成功。

---

## 2. Git のインストール

### Git とは

ファイルの変更履歴を記録するツール。「いつ・誰が・何を変えたか」を管理できます。このプロジェクトでは GitHub と組み合わせてチーム開発の証跡を残します。

### インストール手順（Windows）

1. 以下の公式サイトからインストーラーをダウンロードする
   ```
   https://git-scm.com/download/win
   ```

2. ダウンロードした `Git-x.x.x-64-bit.exe` を実行する

3. インストール中の設定（すべてデフォルトで OK）
   - エディタ：VS Code を選ぶ（後でインストールする場合は「Use Vim」のままでも可）
   - `Adjusting your PATH environment`：「Git from the command line and also from 3rd-party software」を選ぶ
   - その他：すべてデフォルトのまま「Next」を押す

4. 「Finish」でインストール完了

### インストール確認

```cmd
git --version
```

出力例：
```
git version 2.44.0.windows.1
```

### Git の初期設定

インストール後、コミットに表示される名前とメールアドレスを設定する（GitHub のアカウントと同じにする）。

```cmd
git config --global user.name "あなたの名前"
git config --global user.email "your-email@example.com"
```

設定を確認：

```cmd
git config --list
```

`user.name` と `user.email` が表示されれば完了。

---

## 3. GitHub アカウントの作成と SSH 設定

### GitHub とは

Git で管理したコードをインターネット上に保存・共有できるサービス。このプロジェクトでは提出物をここに置きます。

### アカウント作成

1. `https://github.com` にアクセスする
2. 「Sign up」からアカウントを作成する（メールアドレス・パスワード・ユーザー名を設定）
3. 確認メールのリンクをクリックしてアカウントを有効化する

### SSH キーの設定（GitHub への接続設定）

SSH キーを設定すると、毎回パスワードを入力せずに GitHub と通信できるようになります。

**Step 1：SSH キーを生成する**

```cmd
ssh-keygen -t ed25519 -C "your-email@example.com"
```

「Enter file in which to save the key」と聞かれたら、そのまま Enter を押す（デフォルトの場所に保存される）。

パスフレーズは空欄でも可（Enter を2回押す）。

**Step 2：公開鍵の内容をコピーする**

```cmd
type C:\Users\あなたのユーザー名\.ssh\id_ed25519.pub
```

表示されたテキスト（`ssh-ed25519 AAAA...` から始まる1行）をすべてコピーする。

**Step 3：GitHub に公開鍵を登録する**

1. GitHub にログインし、右上のアイコン → 「Settings」を開く
2. 左メニューの「SSH and GPG keys」をクリック
3. 「New SSH key」をクリック
4. Title に「自分のPC」など分かりやすい名前を入力
5. Key に Step 2 でコピーした内容を貼り付ける
6. 「Add SSH key」をクリック

**Step 4：接続確認**

```cmd
ssh -T git@github.com
```

「Hi ユーザー名! You've successfully authenticated...」と表示されれば成功。

---

## 4. VS Code のインストール

### VS Code とは

Microsoft が開発した無料のコードエディタ。拡張機能で Python・Django の開発が快適になります。

### インストール手順

1. 以下の公式サイトからダウンロードする
   ```
   https://code.visualstudio.com/
   ```

2. ダウンロードした `VSCodeUserSetup-x64-x.x.x.exe` を実行する

3. 「Add to PATH」のチェックが表示された場合はチェックを入れる

4. インストール完了後、VS Code を起動する

### おすすめ拡張機能

VS Code を起動し、左サイドバーの拡張機能アイコン（四角が4つ）をクリックして以下を検索・インストールする。

| 拡張機能名 | 役割 |
|-----------|------|
| `Python` (Microsoft) | Python のシンタックスハイライト・補完 |
| `Django` (Baptiste Darthenay) | Django テンプレートのハイライト |
| `Docker` (Microsoft) | Docker ファイルの補完 |
| `GitLens` | Git 履歴の可視化 |

---

## 5. 全ツールの最終確認

以下のコマンドをすべて実行し、バージョンが表示されることを確認する。

```cmd
docker --version
docker compose version
git --version
git config user.name
git config user.email
```

すべて出力されれば事前準備は完了です。Day 1 の作業に進んでください。

---

## インストールでよくある問題

| 問題 | 原因 | 対処 |
|------|------|------|
| `docker: command not found` | Docker Desktop が起動していない | Docker Desktop をタスクバーから起動する |
| `WSL 2 is not installed` | WSL が無効 | PowerShell を管理者で開き `wsl --install` を実行後に再起動 |
| `git: command not found` | PATH が通っていない | Git を再インストールして「Add Git to PATH」オプションを確認 |
| SSH 接続で `Permission denied` | 公開鍵が登録されていない | Step 3 の公開鍵登録をやり直す |
