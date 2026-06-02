# Day 1 — Docker 開発環境構築

## このプロジェクト全体の流れ

```
[Day 1]  → Day 2  → Day 3  → Day 4  → Day 5〜7 → Day 8 → Day 9 → Day 10
Docker     paiza    Django   ログイン  タスク      カテゴリ 検索    提出
環境構築   学習     初期設定  認証      CRUD                ページ
 ★今日                                                     ネーション
```

まず「アプリを動かす箱（環境）」を作ります。環境が整わないと、その後の Django の学習をどれだけ進めても動作確認ができません。Day 1 は地味に見えますが、全体の土台になる最重要ステップです。

---

## この日のゴール

- `docker compose up` 1コマンドで Django（Python）と PostgreSQL が同時に起動する
- `http://localhost:8000` でアクセスできる環境が整っている

---

## この日の前提

- `doc/00_setup.md` の手順が完了していること（Docker Desktop・Git・VS Code・GitHub アカウントの準備）
- GitHub に `taskboard` リポジトリを作成済みであること

---

## 1. 全体像：Docker とは何か、なぜ使うのか

### 「環境の問題」とは

ソフトウェア開発でよくある問題：

```
自分のPCでは動く → チームメンバーのPCでは動かない
本番サーバーでは動く → 開発PCでは動かない
```

原因は Python のバージョン、インストールされているパッケージ、OS の違いなど。これを解決するのが Docker。

### Docker の仕組み

Docker は「コンテナ」という軽量な仮想環境を作るツール。

```
あなたのPC（Windows / Mac / Linux）
├── Docker Desktop
│   ├── コンテナ A：Django（Python 3.12）
│   │   ・requirements.txt に書いたパッケージがすべて入っている
│   │   ・どの PC でも同じ状態
│   └── コンテナ B：PostgreSQL 16
│       ・データベースサーバー
│       ・データはボリュームに永続化
└── あなたのコード（ホスト側）
    └── コンテナ A にマウント（リアルタイムで反映）
```

### なぜ仮想マシン（VMware など）ではなくコンテナなのか

| 比較 | 仮想マシン | コンテナ |
|------|-----------|---------|
| 起動時間 | 数分 | 数秒 |
| ディスク使用量 | 数十GB | 数百MB |
| OS | 丸ごとの OS | ホスト OS のカーネルを共有 |
| 用途 | 完全な環境の分離 | アプリの実行環境の管理 |

Web アプリ開発では起動が速くて軽いコンテナが主流。

---

## 2. Dockerfile の役割と書き方

`Dockerfile` は「コンテナの作り方を記した設計図」。このファイルを元に Docker がイメージ（設計図の実体化）を作る。

```
Dockerfile（設計図）→ イメージ（インストーラー）→ コンテナ（実際に動くもの）
```

```dockerfile
# ベースイメージ：Python 3.12 が入った Linux 環境から始める
FROM python:3.12-slim

# コンテナ内の作業ディレクトリを /app に設定
# （以降のコマンドはすべてこのディレクトリで実行される）
WORKDIR /app

# requirements.txt だけ先にコピーしてパッケージをインストール
# （コードを変えるたびに pip install が走らないようにキャッシュを使う）
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# アプリのコード全体をコピー
COPY . .
```

**各命令の役割**

| 命令 | 何をするか |
|------|-----------|
| `FROM` | どの Linux 環境から始めるかを指定（Python 公式イメージを使う） |
| `WORKDIR` | 「作業フォルダ」を設定。ここを基点にファイルが置かれる |
| `COPY 元 先` | ホスト（PC）のファイルをコンテナにコピー |
| `RUN` | イメージを作るときに実行するコマンド（pip install など） |

---

## 3. docker-compose.yml の役割と書き方

`docker-compose.yml` は「複数のコンテナをまとめて管理する設定ファイル」。

このプロジェクトでは Django コンテナと PostgreSQL コンテナの2つを同時に起動する必要があるため、compose を使う。

```yaml
services:           # 起動するコンテナの一覧
  web:              # コンテナの名前（任意）
    build: .        # このディレクトリの Dockerfile からイメージを作る
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/app      # ホストのコードをコンテナの /app にマウント
    ports:
      - "8000:8000" # ホストの 8000番 → コンテナの 8000番 に転送
    env_file:
      - .env        # 環境変数ファイルを読み込む
    depends_on:
      - db          # db コンテナが起動してから web を起動する

  db:
    image: postgres:16    # PostgreSQL の公式イメージを使う（Dockerfile は不要）
    volumes:
      - postgres_data:/var/lib/postgresql/data  # DB データを永続化
    environment:
      POSTGRES_DB: taskboard
      POSTGRES_USER: taskboard_user
      POSTGRES_PASSWORD: password

volumes:
  postgres_data:    # 名前付きボリュームの宣言（コンテナを消してもデータが残る）
```

**重要な概念の補足**

- **volumes（マウント）**：ホスト側でコードを編集すると、コンテナ内にも即座に反映される。コンテナを再起動しなくていい理由。
- **ports（ポートマッピング）**：`"8000:8000"` は「PC 側の 8000 番ポートへのアクセスをコンテナの 8000 番に転送する」という意味。ブラウザで `http://localhost:8000` にアクセスできる理由。
- **volumes（永続化）**：コンテナは停止・削除するとデータが消える。ボリュームを使うと PostgreSQL のデータが PC 側に残り続ける。

---

## 4. ハンズオン

### Step 1：GitHub でリポジトリを作成する

1. GitHub（`https://github.com`）にログインする
2. 右上の「+」ボタン → 「New repository」をクリック
3. 以下を設定する：
   - Repository name: `taskboard`
   - Public を選択（提出物として公開するため）
   - 「Add a README file」にチェックを入れる
4. 「Create repository」をクリック

### Step 2：ローカルにクローンする

VS Code のターミナル（`Ctrl + `` ` ``）またはコマンドプロンプトで実行する。

```bash
git clone git@github.com:<あなたのユーザー名>/taskboard.git
cd taskboard
```

クローンすると `taskboard` フォルダが作られ、中に `README.md` がある状態になる。

### Step 3：作業ブランチを作成する

```bash
git checkout -b feature/docker-setup
```

`main` に直接コードを書かず、機能ごとにブランチを切る（GitHub Flow）。詳細は [github_flow.md](github_flow.md) を参照。

### Step 4：requirements.txt を作成する

`taskboard/` フォルダに `requirements.txt` を新規作成し、以下を書く。

```
Django==5.0.6
psycopg2-binary==2.9.9
python-dotenv==1.0.1
```

各パッケージの役割：
- `Django`：Web フレームワーク本体
- `psycopg2-binary`：Django が PostgreSQL と通信するためのドライバ
- `python-dotenv`：`.env` ファイルから環境変数を読み込むライブラリ

### Step 5：Dockerfile を作成する

`taskboard/` フォルダに `Dockerfile`（拡張子なし）を新規作成し、以下を書く。

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
```

### Step 6：docker-compose.yml を作成する

`taskboard/` フォルダに `docker-compose.yml` を新規作成し、以下を書く。

```yaml
services:
  web:
    build: .
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      - db

  db:
    image: postgres:16
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: taskboard
      POSTGRES_USER: taskboard_user
      POSTGRES_PASSWORD: password

volumes:
  postgres_data:
```

### Step 7：.env ファイルを作成する

`.env` はシークレット情報を管理するファイル。**Git にコミットしてはいけない**。

```
SECRET_KEY=django-insecure-xxxxxx-ここは後で変更する
DEBUG=True
DB_NAME=taskboard
DB_USER=taskboard_user
DB_PASSWORD=password
DB_HOST=db
DB_PORT=5432
```

次に `.env.example`（他の人が参考にするテンプレート）を作成する。実際の値は書かない。

```
SECRET_KEY=your-secret-key-here
DEBUG=True
DB_NAME=taskboard
DB_USER=taskboard_user
DB_PASSWORD=password
DB_HOST=db
DB_PORT=5432
```

### Step 8：.gitignore を作成する

`.gitignore` に書いたファイル・フォルダは Git の管理対象外になる。`.env` を必ず含める。

```
.env
__pycache__/
*.pyc
*.pyo
.DS_Store
*.sqlite3
```

### Step 9：ファイル構成を確認する

この時点のフォルダ構成：

```
taskboard/
├── .env               ← コミットしない（.gitignore で除外）
├── .env.example       ← コミットする
├── .gitignore
├── Dockerfile
├── README.md
├── docker-compose.yml
└── requirements.txt
```

### Step 10：起動確認

```bash
# イメージをビルドしてコンテナを起動
docker compose up --build
```

初回は Python イメージのダウンロードと pip install が走るため、数分かかる。

別ターミナルを開いて起動状態を確認：

```bash
docker compose ps
```

期待する出力：

```
NAME              IMAGE        STATUS
taskboard-web-1   taskboard    Up
taskboard-db-1    postgres:16  Up
```

両方 `Up` になっていれば成功。

Python のバージョン確認：

```bash
docker compose exec web python --version
```

出力例：`Python 3.12.x`

DB の接続確認：

```bash
docker compose exec db psql -U taskboard_user -d taskboard -c "\l"
```

データベースの一覧が表示されれば接続成功。

### Step 11：PR を作成してマージする

```bash
git add Dockerfile docker-compose.yml requirements.txt .env.example .gitignore
git commit -m "feat: Docker開発環境を構築（web + PostgreSQL）"
git push origin feature/docker-setup
```

GitHub でプルリクエストを作成する。PR の説明欄に以下を記載：

```markdown
## 概要
Docker + PostgreSQL の開発環境を構築しました。

## 変更内容
- Dockerfile を作成（Python 3.12 ベース）
- docker-compose.yml を作成（web + db の2サービス構成）
- requirements.txt を作成
- .env.example と .gitignore を追加

## 確認方法
1. `.env.example` をコピーして `.env` を作成
2. `docker compose up --build` を実行
3. `docker compose ps` で両サービスが Up になることを確認
```

Files Changed タブで差分を確認し、`.env` がコミットに含まれていないことを確認してからマージする。

---

## 5. よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `Docker Desktop is not running` | Docker が起動していない | タスクバーのクジラアイコンから Docker Desktop を起動する |
| `port is already allocated` | 8000番ポートが別のプロセスで使用中 | `taskkill /f /im python.exe` などで他のプロセスを終了する |
| `no such file: requirements.txt` | ファイルが存在しない | `requirements.txt` が `Dockerfile` と同じ階層にあるか確認 |
| `db コンテナが exit` | PostgreSQL の設定ミス | `docker compose logs db` でエラー内容を確認する |
| イメージのビルドが途中で止まる | ネットワークの問題 | `docker compose build --no-cache` で再試行する |

---

## 6. 便利な Docker コマンド

```bash
docker compose up          # コンテナを起動（フォアグラウンド）
docker compose up -d       # コンテナを起動（バックグラウンド）
docker compose down        # コンテナを停止・削除
docker compose down -v     # コンテナを停止・削除（DB データも削除）
docker compose logs web    # web コンテナのログを表示
docker compose exec web bash   # web コンテナの中に入る
docker compose build       # イメージを再ビルド
docker compose ps          # コンテナの起動状態を確認
```

---

## この日の GitHub チェックポイント

> 詳細なルールは [github_flow.md](github_flow.md) を参照。

### いつブランチを切るか

**タイミング**：リポジトリをクローンした直後、最初のファイルを作る前。

**理由**：Docker の設定は「環境構築」という独立した1機能。後から認証やモデルを実装するブランチと混在させると、「環境を変えたのか機能を追加したのか」が履歴から判別できなくなる。

```bash
git checkout -b feature/docker-setup   # main から切る
```

### いつコミットするか

| タイミング | コミットの意味 | 理由 |
|-----------|--------------|------|
| `docker compose up` が成功し両サービスが `Up` になったとき | 環境構築の完了 | 「動く状態」でコミットすると、以降で壊れたときにここまで安全に戻れる |
| `.env.example` と `.gitignore` を整備したとき | セキュリティ設定の完了 | シークレット管理の設定も機能の一部 |

### コミットメッセージ例

```bash
git commit -m "chore: DockerfileとDockerCompose設定を追加"
git commit -m "chore: .gitignoreと.env.exampleを追加"

# まとめる場合
git commit -m "feat: Docker開発環境を構築（web + PostgreSQL）"
```

### いつ PR をマージするか

**条件**：`docker compose ps` で `web` と `db` が両方 `Up` になること。

**理由**：次の Day 3 では `docker compose exec web python manage.py migrate` を実行する。DB が起動できない状態でマージすると、Day 3 の作業開始から詰まる。
