# Docker ターミナルコマンド一覧

> このプロジェクトで使用する Docker コマンドをまとめた参照資料です。
> 困ったときにここを確認してください。

---

## コマンドの構造を理解する

```
docker compose   exec    web    python manage.py migrate
  ↑              ↑       ↑      ↑
docker composeに execという  webサービスの  コンテナ内で実行する
コマンドを送る    操作を指示  コンテナを対象  コマンド
```

---

## 1. 起動・停止

### コンテナを起動する

```bash
docker compose up
```
- フォアグラウンドで起動（ログがターミナルに流れる）
- `Ctrl + C` で停止できる

```bash
docker compose up -d
```
- バックグラウンドで起動（ターミナルが返ってくる）
- ログを見たいときは別途 `docker compose logs` を使う

```bash
docker compose up --build
```
- イメージを**再ビルドしてから**起動する
- `requirements.txt` を変更したあとは必ずこれを実行する

```bash
docker compose up --build -d
```
- 再ビルド ＋ バックグラウンド起動

---

### コンテナを停止する

```bash
docker compose down
```
- コンテナを停止して削除する
- DB のデータ（ボリューム）は残る

```bash
docker compose down -v
```
- コンテナ＋ボリュームをすべて削除する
- **DB のデータが消える** → マイグレーションのやり直しが必要になる
- 「DBをまっさらな状態にしたい」ときに使う

```bash
docker compose stop
```
- コンテナを停止するが、削除はしない（`start` で再開できる）

```bash
docker compose start
```
- 停止中のコンテナを再開する（`stop` の対）

```bash
docker compose restart
```
- コンテナを再起動する（停止 → 起動を一括）
- 設定ファイルを変更したあとに使う

---

## 2. 状態確認

### 起動状態を確認する

```bash
docker compose ps
```

出力例：
```
NAME              IMAGE        COMMAND                  STATUS    PORTS
taskboard-web-1   taskboard    "python manage.py ru…"  Up        0.0.0.0:8000->8000/tcp
taskboard-db-1    postgres:16  "docker-entrypoint.s…"  Up        5432/tcp
```

- `Up`：正常に起動中
- `Exit`：停止している（エラーの場合も）

---

### ログを確認する

```bash
docker compose logs
```
- 全サービスのログをまとめて表示する

```bash
docker compose logs web
```
- `web` サービス（Django）のログだけ表示する

```bash
docker compose logs db
```
- `db` サービス（PostgreSQL）のログだけ表示する

```bash
docker compose logs -f web
```
- ログをリアルタイムで流し続ける（`-f` = follow）
- `Ctrl + C` で停止

```bash
docker compose logs --tail=50 web
```
- 直近50行だけ表示する（ログが長いときに使う）

---

## 3. コンテナ内でコマンドを実行する

### Django のコマンドを実行する

```bash
docker compose exec web python manage.py migrate
```
- マイグレーションを実行する

```bash
docker compose exec web python manage.py makemigrations
```
- マイグレーションファイルを生成する

```bash
docker compose exec web python manage.py makemigrations アプリ名
```
- 特定のアプリのマイグレーションを生成する（例：`accounts`、`tasks`）

```bash
docker compose exec web python manage.py createsuperuser
```
- 管理画面用のスーパーユーザーを作成する

```bash
docker compose exec web python manage.py shell
```
- Django シェルを起動する（Python の対話モードで Django の ORM が使える）

```bash
docker compose exec web python manage.py test
```
- テストを全件実行する

```bash
docker compose exec web python manage.py test tasks
```
- `tasks` アプリのテストだけ実行する

```bash
docker compose exec web python manage.py collectstatic
```
- 静的ファイルを収集する（本番デプロイ時に使う）

---

### コンテナ内に入る（シェルを起動する）

```bash
docker compose exec web bash
```
- `web` コンテナの中に `bash` シェルで入る
- コンテナ内のファイルを直接確認したいときに使う
- `exit` で抜ける

```bash
docker compose exec db bash
```
- `db` コンテナの中に入る（PostgreSQL コンテナ）

---

### PostgreSQL を直接操作する

```bash
docker compose exec db psql -U taskboard_user -d taskboard
```
- PostgreSQL のインタラクティブモードに入る
- `\l`：データベース一覧
- `\dt`：テーブル一覧
- `SELECT * FROM tasks_task;`：タスクテーブルを確認
- `\q`：終了

---

## 4. ビルド関連

### イメージをビルドする

```bash
docker compose build
```
- イメージを再ビルドする（コンテナは起動しない）

```bash
docker compose build web
```
- `web` サービスだけビルドする

```bash
docker compose build --no-cache
```
- キャッシュを使わずにビルドする（ビルドに問題があるときの切り分けに使う）

---

## 5. 掃除・リセット

### 不要なリソースを削除する

```bash
docker image prune
```
- 使われていないイメージを削除する（ディスク容量を節約する）

```bash
docker system prune
```
- 使われていないコンテナ・イメージ・ネットワークをまとめて削除する

```bash
docker system prune -a
```
- 停止中のコンテナを含めてすべてのキャッシュを削除する
- **注意**：次回起動時に全イメージの再ダウンロードが必要になる

---

## 6. よく使うワークフロー

### 毎日の開発開始時

```bash
docker compose up -d
```

---

### requirements.txt を変更したとき

```bash
docker compose down
docker compose up --build -d
```

---

### マイグレーションを実行するとき

```bash
# モデルを変更した場合は makemigrations → migrate の順
docker compose exec web python manage.py makemigrations
docker compose exec web python manage.py migrate
```

---

### DB をまっさらにしてやり直したいとき

```bash
docker compose down -v                          # ボリューム（DBデータ）も削除
docker compose up -d                            # 再起動
docker compose exec web python manage.py migrate  # テーブルを再作成
docker compose exec web python manage.py createsuperuser  # 管理ユーザーを再作成
```

---

### コンテナが止まって原因を調べたいとき

```bash
docker compose ps          # まず状態を確認
docker compose logs web    # web のログを確認
docker compose logs db     # db のログを確認
```

---

### テストを実行してログを確認したいとき

```bash
docker compose exec web python manage.py test tasks -v 2
```
- `-v 2`：詳細モード（各テストの名前と結果が表示される）

---

## 7. コマンドのショートカット早見表

| やりたいこと | コマンド |
|------------|---------|
| 起動（バックグラウンド） | `docker compose up -d` |
| 起動（ログを見ながら） | `docker compose up` |
| 再ビルドして起動 | `docker compose up --build -d` |
| 停止（データ保持） | `docker compose down` |
| 停止（データ削除） | `docker compose down -v` |
| 起動状態を確認 | `docker compose ps` |
| ログを見る（リアルタイム） | `docker compose logs -f web` |
| マイグレーション実行 | `docker compose exec web python manage.py migrate` |
| Django シェル起動 | `docker compose exec web python manage.py shell` |
| テスト実行 | `docker compose exec web python manage.py test` |
| コンテナ内に入る | `docker compose exec web bash` |
| イメージ再ビルド（キャッシュなし） | `docker compose build --no-cache` |

---

## 8. エラーが出たときの確認手順

```
Step 1: docker compose ps でコンテナの状態を確認する
         ↓ Exit になっているサービスがある場合
Step 2: docker compose logs [サービス名] でログを確認する
         ↓ エラーメッセージを確認する
Step 3: よくあるエラーの対処表（下記）を参照する
```

### よくあるエラーと対処

| エラーメッセージ | 原因 | 対処 |
|--------------|------|------|
| `port is already allocated` | ポートが別のプロセスで使用中 | Docker Desktop を再起動するか、他のコンテナを停止する |
| `no such service: web` | サービス名が間違っている | `docker compose ps` でサービス名を確認する |
| `Cannot connect to the Docker daemon` | Docker Desktop が起動していない | Docker Desktop を起動する |
| `ERROR: Service 'web' failed to build` | Dockerfile に問題がある | `docker compose logs` でビルドエラーを確認する |
| `FATAL: database "taskboard" does not exist` | DB が初期化されていない | `docker compose down -v` して再起動後に `migrate` を実行 |
| `django.db.utils.OperationalError: could not connect` | DB コンテナが起動していない | `docker compose ps` で db の状態を確認、`docker compose up -d db` で再起動 |
