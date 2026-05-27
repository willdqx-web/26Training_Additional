# Day 1 — Docker 開発環境構築

## この日のゴール

`docker compose up` 1コマンドで Django（Python）と PostgreSQL が同時に起動する開発環境を作る。

---

## 1. Docker の基礎知識

### イメージとコンテナ

| 概念 | 説明 | 例え |
|------|------|------|
| **イメージ** | 環境の設計図（読み取り専用） | アプリのインストーラー |
| **コンテナ** | イメージから起動した実行中の環境 | インストール後に動いているアプリ |

```
Dockerfile → (build) → イメージ → (run) → コンテナ
```

### なぜ Docker を使うのか

- 「自分の PC では動いた」問題をなくす
- `git clone` → `docker compose up` だけで誰でも同じ環境を再現できる
- 本番環境との差異を最小化できる

---

## 2. Dockerfile の書き方

`Dockerfile` はイメージの作り方を記述するファイル。

```dockerfile
# ベースイメージを指定（Python 3.12 の公式イメージ）
FROM python:3.12-slim

# コンテナ内の作業ディレクトリを設定
WORKDIR /app

# 依存パッケージのファイルだけ先にコピー（キャッシュ効率化）
COPY requirements.txt .

# パッケージをインストール
RUN pip install --no-cache-dir -r requirements.txt

# アプリのコードをコピー
COPY . .
```

**主要な命令**

| 命令 | 役割 |
|------|------|
| `FROM` | ベースとなるイメージを指定 |
| `WORKDIR` | 以降のコマンドを実行するディレクトリ |
| `COPY src dst` | ホストのファイルをコンテナへコピー |
| `RUN` | イメージビルド時に実行するコマンド |
| `CMD` | コンテナ起動時のデフォルトコマンド |

---

## 3. docker-compose.yml の書き方

複数のコンテナをまとめて管理するファイル。

```yaml
services:
  web:                              # サービス名（任意）
    build: .                        # Dockerfile のある場所
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/app                      # ホストのコードをコンテナにマウント
    ports:
      - "8000:8000"                 # ホスト:コンテナ のポートマッピング
    env_file:
      - .env                        # 環境変数ファイル
    depends_on:
      - db                          # db が起動してから web を起動

  db:
    image: postgres:16              # PostgreSQL の公式イメージ
    volumes:
      - postgres_data:/var/lib/postgresql/data  # DB データを永続化
    environment:
      POSTGRES_DB: taskboard
      POSTGRES_USER: taskboard_user
      POSTGRES_PASSWORD: password

volumes:
  postgres_data:                    # 名前付きボリュームの宣言
```

**重要な概念**

- **volumes（マウント）**：ホスト側のファイル変更がリアルタイムでコンテナに反映される。コードを編集したら即反映される理由。
- **depends_on**：サービスの起動順序を制御する。ただし「DB が接続受付可能になるまで待つ」は保証しない点に注意。
- **ports**：`"8000:8000"` は「ホストの 8000 番ポートへのアクセスをコンテナの 8000 番へ転送」する意味。

---

## 4. ハンズオン

### Step 1：リポジトリの作成

GitHub で `taskboard` リポジトリを作成し、ローカルにクローンする。

```bash
git clone https://github.com/<あなたのユーザー名>/taskboard.git
cd taskboard
```

### Step 2：ブランチを作成

```bash
git checkout -b feature/docker-setup
```

### Step 3：requirements.txt を作成

```
Django==5.0.6
psycopg2-binary==2.9.9
python-dotenv==1.0.1
```

### Step 4：Dockerfile を作成

プロジェクトルートに `Dockerfile` を作成する。

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
```

### Step 5：docker-compose.yml を作成

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

### Step 6：.env ファイルを作成

```
SECRET_KEY=django-insecure-xxxxxx-ここは後で変更する
DEBUG=True
DATABASE_URL=postgres://taskboard_user:password@db:5432/taskboard
```

**.gitignore に .env を追加する**

```
.env
__pycache__/
*.pyc
.DS_Store
```

### Step 7：起動確認

```bash
# イメージのビルドと起動
docker compose up --build

# 別ターミナルで確認
docker compose ps           # サービスの稼働状態を確認
docker compose exec web python --version   # Python のバージョン確認
docker compose exec db psql -U taskboard_user -d taskboard -c "\l"  # DB 一覧確認
```

`docker compose ps` の出力例：

```
NAME              IMAGE        STATUS
taskboard-web-1   taskboard    Up
taskboard-db-1    postgres:16  Up
```

両サービスが `Up` になっていれば成功。

### Step 8：.env.example を作成

実際の値を書かないダミーファイルをリポジトリに含める。

```
SECRET_KEY=your-secret-key-here
DEBUG=True
DATABASE_URL=postgres://USER:PASSWORD@db:5432/DBNAME
```

### Step 9：PR を作成してマージ

```bash
git add Dockerfile docker-compose.yml requirements.txt .env.example .gitignore
git commit -m "feat: Docker開発環境を構築"
git push origin feature/docker-setup
```

GitHub でプルリクエストを作成し、PR の説明欄に以下を記載する。

```
## 概要
Docker + PostgreSQL の開発環境を構築しました。

## 確認方法
1. `.env.example` をコピーして `.env` を作成
2. `docker compose up --build` を実行
3. `docker compose ps` で両サービスが Up になっていることを確認
```

自己レビューコメントを1件追加してからマージする。

---

## 5. よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `port is already allocated` | 8000番ポートが使用中 | `lsof -i :8000` で確認してプロセスを終了 |
| `no such file: requirements.txt` | ファイルがない | `requirements.txt` が Dockerfile と同じ場所にあるか確認 |
| `db コンテナが exit` | PostgreSQL の設定ミス | `docker compose logs db` でエラー内容を確認 |
| `permission denied` | Docker デーモン未起動 | Docker Desktop を起動する |

---

## 6. 便利な Docker コマンド

```bash
docker compose up          # 起動（フォアグラウンド）
docker compose up -d       # 起動（バックグラウンド）
docker compose down        # 停止・コンテナ削除
docker compose down -v     # 停止・コンテナ・ボリューム削除（DB データも消える）
docker compose logs web    # web サービスのログを見る
docker compose exec web bash   # web コンテナの中に入る
docker compose build       # イメージを再ビルド
```

---

## この日の GitHub チェックポイント

> 詳細なルールは [github_flow.md](github_flow.md) を参照。

### いつブランチを切るか

**タイミング**：リポジトリを作成した直後、最初のコードを書く前。

**理由**：Docker の設定ファイル群（Dockerfile / docker-compose.yml）は「環境構築」という独立した1機能。後から認証やモデルを実装するブランチと混在させると、「環境を変えたのか機能を追加したのか」がコミット履歴から判別できなくなる。

```bash
git checkout -b feature/docker-setup   # main から切る
```

### いつコミットするか

| タイミング | コミットの意味 | 理由 |
|-----------|--------------|------|
| `docker compose up` が成功し両サービスが `Up` になったとき | 環境構築の完了 | 「動く状態」でコミットすることで、以降の変更で壊れたとき安全にここまで戻れる |
| `.env.example` と `.gitignore` を整備したとき | セキュリティ設定の完了 | 環境構築とシークレット管理は別の関心事なので分けてもよい。まとめてもよい |

### コミットメッセージ例

```bash
# ファイル構成が整ったとき
git commit -m "chore: DockerfileとDockerCompose設定を追加"

# 動作確認まで含めて完了したとき（まとめる場合）
git commit -m "feat: Docker開発環境を構築（web + PostgreSQL）"

# .gitignore / .env.example を別コミットにする場合
git commit -m "chore: .gitignoreと.env.exampleを追加"
```

### いつ PR をマージするか

**条件**：`docker compose ps` で `web` と `db` が両方 `Up` になること。

**理由**：次の Day 3 では `docker compose exec web python manage.py migrate` を実行する。DB サービスが起動できない状態で main にマージすると、Day 3 のブランチを切った時点から詰まる。「DB が動く」ことが確認できてからマージする。
