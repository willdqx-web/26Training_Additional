# Day 11 — PostgreSQL基礎理解：RDBMSとpsql入門

## このプロジェクト全体の流れ

```
Day 1  → Day 2  → Day 3〜4 → Day 5〜7 → Day 8〜9 → Day 10 → [Day 11] → Day 12〜15
Docker   paiza    認証       タスク      検索・      提出      psql       SQL深化
環境構築  学習               CRUD        フィルタ    準備      入門       ★今日
```

Day 10 でアプリの基本機能が完成しました。Day 11〜15 は「PostgreSQL を本当に理解している」ことを証明するフェーズです。まず今日は **PostgreSQL とは何か** を理解し、psql CLI でデータを直接触ります。

---

## この日のゴール

- RDBMSとPostgreSQLの役割を自分の言葉で説明できる
- psqlでTaskBoardのテーブル構造と中身を確認できる
- SELECT / WHERE / ORDER BY をpsqlで手打ちして結果を確認できる
- Django ORM が内部でどんなSQLを生成しているか確認できる（django-debug-toolbar）

---

## この日の前提

- Day 10 の全チェックリストが完了していること
- TaskBoard が `docker compose up` で起動できること

---

## 午前：RDBMSとは何か・psqlで触れる

### 1. RDBMS とは

RDBMS（Relational DataBase Management System）は「表の形でデータを管理するシステム」です。

```
tasks_task テーブル
┌────┬──────────────┬──────────┬────────────┐
│ id │ title        │ status   │created_by_id│
├────┼──────────────┼──────────┼────────────┤
│  1 │ 設計書を書く  │ todo     │     1      │
│  2 │ レビューする  │ in_prog  │     1      │
│  3 │ デプロイ準備  │ done     │     2      │
└────┴──────────────┴──────────┴────────────┘
```

**キーワード整理**

| 用語 | 意味 |
|------|------|
| テーブル | Excelの「シート」に相当。1つのモデル = 1つのテーブル |
| カラム（列） | フィールド。`title`, `status` など |
| レコード（行） | 1件のデータ |
| PK（主キー） | 各行を一意に識別するID。DjangoのAutoFieldがこれ |
| FK（外部キー） | 別テーブルのPKを参照するカラム。DjangoのForeignKey |

**PostgreSQL が SQLite と違う点**

| 比較 | SQLite | PostgreSQL |
|------|--------|------------|
| 動作形態 | ファイル1つ | サーバープロセス |
| 同時接続 | 弱い | 複数クライアントに強い |
| 全文検索 | LIKE のみ | tsvector/tsquery |
| 型の厳密さ | 緩い | 厳格（型エラーが出る） |
| 実務利用 | 小規模・組み込み | 大規模Webアプリ |

### 2. psql の使い方

psql は PostgreSQL に付属するコマンドラインクライアントです。Docker 内の PostgreSQL に接続するには：

```bash
docker compose exec db psql -U postgres -d taskboard
```

接続できると `taskboard=#` というプロンプトが表示されます。

**psqlの基本コマンド（バックスラッシュコマンド）**

| コマンド | 意味 |
|---------|------|
| `\l` | データベース一覧を表示 |
| `\c データベース名` | 別のDBに接続する |
| `\dt` | 現在のDBのテーブル一覧 |
| `\d テーブル名` | テーブルの定義（カラム・型・制約）を表示 |
| `\q` | psqlを終了 |

### ハンズオン（午前）

#### Step 1：ブランチを作成する

```bash
git checkout main
git pull origin main
git checkout -b feature/postgres-basics
```

#### Step 2：psqlで接続してテーブルを確認する

> **接続前の確認**：DB名は `.env` の `POSTGRES_DB` の値を使う。`POSTGRES_DB=taskboard` であれば以下のコマンドでOK。値が違う場合は `-d taskboard` の部分を書き換える。

```bash
# .env のDB名を確認してから接続する
docker compose exec db psql -U postgres -d taskboard
```

```sql
-- テーブル一覧
\dt

-- tasks_task テーブルの定義を確認
\d tasks_task

-- accounts_customuser テーブルの定義を確認
\d accounts_customuser
```

出力例（`\d tasks_task` の一部）：

```
               Table "public.tasks_task"
   Column    |          Type          | Nullable |
-------------+------------------------+----------+
 id          | bigint                 | not null |
 title       | character varying(200) | not null |
 status      | character varying(20)  | not null |
 due_date    | date                   |          |
 created_by_id | bigint               | not null |
...
Indexes:
    "tasks_task_pkey" PRIMARY KEY, btree (id)
Foreign-key constraints:
    "tasks_task_created_by_id_fkey" FOREIGN KEY (created_by_id) ...
```

#### Step 3：SELECT を手打ちする

> **データがない場合**：ブラウザで `http://localhost:8000/tasks/` からタスクを数件作成してから実行する。テーブルが空だと SELECT の結果が空になり確認しにくい。

```sql
-- 全タスクを取得
SELECT * FROM tasks_task;

-- タイトルとステータスだけ取得
SELECT id, title, status FROM tasks_task;

-- ステータスで絞り込む
SELECT id, title, status FROM tasks_task WHERE status = 'todo';

-- 作成日の降順で並べる
SELECT id, title, status FROM tasks_task ORDER BY created_at DESC;

-- 最新3件だけ取得
SELECT id, title FROM tasks_task ORDER BY created_at DESC LIMIT 3;
```

> ForeignKey（`created_by_id`）には実際の User の id（数値）が格納されている。Djangoが `Task.created_by.username` と書けるのは、ORM がSQLの JOIN を自動生成してくれているから。

#### Step 4：psqlを終了する

```sql
\q
```

---

## 午後：django-debug-toolbar でORM生成SQLを可視化

### 3. ORM が生成するSQLを見る重要性

Django ORM は便利ですが、内部で何本のSQLを発行しているか見えません。`django-debug-toolbar` をインストールすると、**ブラウザ画面の右にデバッグパネル**が表示され、実行されたSQLをリアルタイムで確認できます。

```
ブラウザ
┌──────────────────────────────┬───────────────┐
│  タスク一覧                   │  DjDT Panel   │
│                               │ SQL: 3 queries│
│  ・設計書を書く（todo）        │ 0.52ms total  │
│  ・レビューする（in_progress）│ [▼ SQLを展開] │
└──────────────────────────────┴───────────────┘
```

### 4. django-debug-toolbar の設定

#### Step 5：パッケージを追加する

`requirements.txt` に追記する：

```
django-debug-toolbar==4.4.6
```

コンテナを再ビルドする：

```bash
docker compose down
docker compose build
docker compose up -d
```

#### Step 6：settings.py を設定する

`config/settings.py` に追記する：

```python
# 開発環境でのみ有効
if DEBUG:
    INSTALLED_APPS += ['debug_toolbar']
    MIDDLEWARE.insert(0, 'debug_toolbar.middleware.DebugToolbarMiddleware')
    INTERNAL_IPS = ['127.0.0.1']
```

#### Step 7：urls.py を設定する

`config/urls.py` を更新する：

```python
from django.conf import settings

urlpatterns = [
    # 既存のURLパターン
]

if settings.DEBUG:
    import debug_toolbar
    urlpatterns = [
        path('__debug__/', include(debug_toolbar.urls)),
    ] + urlpatterns
```

#### Step 8：動作確認

1. `http://localhost:8000/tasks/` にアクセスする
2. 右端に DjDT アイコン（▶）が表示される
3. 「SQL」パネルをクリックし、実行されたSQLを確認する

#### Step 9：Django shell で `.query` を確認する

```bash
docker compose exec web python manage.py shell
```

```python
from tasks.models import Task

# クエリオブジェクトの生成SQL を文字列で確認
qs = Task.objects.all()
print(str(qs.query))
# → SELECT "tasks_task"."id", "tasks_task"."title", ... FROM "tasks_task"

# フィルタをかけたときのSQL
qs2 = Task.objects.filter(status='todo').order_by('-created_at')
print(str(qs2.query))

exit()
```

#### Step 10：PR を作成してマージする

```bash
git add requirements.txt config/settings.py config/urls.py
git commit -m "feat: django-debug-toolbar を追加してORM生成SQLを可視化"
git push origin feature/postgres-basics
```

GitHub でPRを作成し、自己レビューコメントに「確認できたSQL例」を貼り付けてマージする。

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `psql: command not found` | psql がない | `docker compose exec db psql` を使う（コンテナ内のpsqlを使う） |
| DjDT パネルが表示されない | `INTERNAL_IPS` が未設定 | `INTERNAL_IPS = ['127.0.0.1']` を settings.py に追加 |
| `django.db.utils.OperationalError` | コンテナ再起動が必要 | `docker compose restart web` を実行 |
| SQLが `None` になる | `qs.query` はまだSQL未発行の状態 | `str(qs.query)` で文字列化するか `list(qs)` で評価後に確認 |

---

## この日の GitHub チェックポイント

### いつコミットするか

| タイミング | コミットの意味 |
|-----------|--------------|
| debug-toolbar が表示されたとき | DjDT組み込み完成 |

### コミットメッセージ例

```bash
git commit -m "feat: django-debug-toolbar を追加してORM生成SQLを可視化"
```

### いつ PR をマージするか

**条件**：ブラウザでDjDT SQLパネルが表示でき、タスク一覧ページのSQL発行数が確認できること。

**理由**：debug-toolbar は Day 12〜15 での学習の基盤ツール。先にマージしておくことで、以降のPRでの動作確認が楽になる。
