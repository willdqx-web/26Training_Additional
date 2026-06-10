# Day 15 — インデックス設計・最終整備

## このプロジェクト全体の流れ

```
Day 1〜10  → Day 11〜14 → [Day 15]
基礎完成     psql・JOIN   インデックス
             集計・全文検索 最終整備
                          ★今日（最終日）
```

全文検索まで実装しました。今日は **インデックス設計** を学び、Django の migration で管理します。最後に README を更新して、Day 11〜15 の成果を説明できる状態にします。

---

## この日のゴール

- インデックスがなぜ必要かを説明できる
- EXPLAIN ANALYZE でクエリの実行計画を（基本的に）読める
- Django の `Meta.indexes` でインデックスを migration 管理できる
- Day 11〜15 の成果をREADMEとGitHub履歴で説明できる状態にする

---

## この日の前提

- Day 14 の `feature/full-text-search` が main にマージ済み

---

## 午前：インデックスとEXPLAIN ANALYZE

### 1. インデックスとは

インデックスは「本の索引」と同じです。

```
インデックスなし（Seq Scan）：本を最初のページから順番に読んで探す
インデックスあり（Index Scan）：索引で「status = todo → 1, 3, 7ページ」と確認してから直接そのページを開く
```

**いつインデックスが必要か**

| 状況 | 判断 |
|------|------|
| WHERE で頻繁に絞り込むカラム | インデックスを追加 |
| ORDER BY でよく使うカラム | インデックスを追加 |
| JOIN の結合キー | 外部キー（FK）には Django が自動で追加 |
| ほとんど検索に使わないカラム | 不要（書き込みが遅くなるだけ） |
| テーブルが小さい（数十件程度） | インデックス不要（Seq Scan のほうが速いことも） |

**TaskBoard でインデックスが有効なカラム**

| カラム | 理由 |
|--------|------|
| `status` | 一覧表示でフィルタリングに毎回使われる |
| `due_date` | 期限順ソート・期限超過フィルタに使われる |
| `(created_by, status)` 複合 | 「自分のタスク && 特定ステータス」の複合条件で使われる |

### 2. EXPLAIN ANALYZE の読み方

`EXPLAIN ANALYZE` は「このSQLがどう実行されるか（実行計画）」を表示します。

```sql
EXPLAIN ANALYZE SELECT * FROM tasks_task WHERE status = 'todo';
```

出力例（インデックスなし）：

```
Seq Scan on tasks_task  (cost=0.00..1.06 rows=2 width=185)
                         (actual time=0.013..0.017 rows=2 loops=1)
  Filter: ((status)::text = 'todo'::text)
  Rows Removed by Filter: 4
Planning Time: 0.098 ms
Execution Time: 0.031 ms
```

**キーワード解説**

| キーワード | 意味 |
|-----------|------|
| `Seq Scan` | 全行スキャン（インデックス未使用） |
| `Index Scan` | インデックスを使ったスキャン |
| `cost=x..y` | 推定コスト（x: 初行取得, y: 全行取得） |
| `rows=N` | 推定行数 |
| `Execution Time` | 実際の実行時間（ms） |

インデックスを追加すると `Seq Scan` → `Index Scan` に変わります。

### ハンズオン（午前）

#### Step 1：psqlに接続する

```bash
docker compose exec db psql -U postgres -d taskboard
```

#### Step 2：EXPLAIN ANALYZE でインデックスなしの実行計画を確認する

```sql
EXPLAIN ANALYZE SELECT * FROM tasks_task WHERE status = 'todo';
```

`Seq Scan` が表示されることを確認する。

#### Step 3：手動でインデックスを作成して効果を確認する

```sql
-- インデックスを手動で作成
CREATE INDEX idx_task_status ON tasks_task(status);

-- 再度 EXPLAIN ANALYZE を実行
EXPLAIN ANALYZE SELECT * FROM tasks_task WHERE status = 'todo';
```

`Index Scan` または `Bitmap Index Scan` に変わることを確認する。

> データ件数が少ない場合、PostgreSQL が Seq Scan を選ぶことがあります（小テーブルでは全スキャンの方が速い場合があるため）。`SET enable_seqscan = off;` で強制的に Index Scan にできます。

#### Step 4：手動インデックスを削除する

Django migration で管理するため、手動作成分は削除する：

```sql
DROP INDEX idx_task_status;
\q
```

---

## 午後：Django migration でインデックスを管理

### 3. Django の Meta.indexes

インデックスは Django の `Meta` クラスで定義し、`makemigrations` → `migrate` で管理します。これにより**インデックスの変更履歴がコードとして残ります**。

```python
class Task(models.Model):
    # ...フィールド定義...

    class Meta:
        indexes = [
            models.Index(fields=['status'], name='task_status_idx'),
            models.Index(fields=['due_date'], name='task_due_date_idx'),
            models.Index(fields=['created_by', 'status'], name='task_user_status_idx'),
        ]
```

`db_index=True` との違い：

| 方法 | 用途 |
|------|------|
| `db_index=True` | 単一カラムの基本インデックス |
| `Meta.indexes` | 複合インデックス・インデックス名の明示的指定 |
| `Meta.indexes` | GIN インデックス（全文検索用）などの特殊型 |

### ハンズオン（午後）

#### Step 5：ブランチを作成する

```bash
git checkout main
git pull origin main
git checkout -b feature/db-index
```

#### Step 6：Task モデルに Meta.indexes を追加する

`tasks/models.py` の **`Task` クラスに `class Meta` を追加する**だけでよい。`Category` クラスなど他のコードは変更しない。

> **注意**：以下のコードブロックは `Task` クラスの全体例として示しているが、`Category` モデルは含まれていない。ファイルを丸ごと書き換えると `Category` クラスが消えてしまう。既存の `tasks/models.py` を開き、`Task` クラスの末尾に `class Meta:` ブロックを追加（または既存の `Meta` があれば `indexes` だけ追記）する。

```python
# tasks/models.py の Task クラス末尾に追加する部分のみ示す

class Task(models.Model):
    STATUS_CHOICES = [
        ('todo', '未着手'),
        ('in_progress', '進行中'),
        ('done', '完了'),
    ]

    title = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='todo')
    due_date = models.DateField(null=True, blank=True)
    created_by = models.ForeignKey(
        'accounts.CustomUser',
        on_delete=models.CASCADE,
        related_name='created_tasks',
    )
    assigned_to = models.ForeignKey(
        'accounts.CustomUser',
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
        related_name='assigned_tasks',
    )
    category = models.ForeignKey(
        'Category',
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
    )
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        indexes = [
            models.Index(fields=['status'], name='task_status_idx'),
            models.Index(fields=['due_date'], name='task_due_date_idx'),
            # よく使われる複合条件（自分のタスク && 特定ステータス）のインデックス
            models.Index(fields=['created_by', 'status'], name='task_user_status_idx'),
        ]

    def __str__(self):
        return self.title
```

#### Step 7：migration を作成して適用する

```bash
docker compose exec web python manage.py makemigrations tasks
docker compose exec web python manage.py migrate
```

作成された migration ファイル（例：`tasks/migrations/000X_task_add_indexes.py`）の内容を確認する：

```python
# 自動生成される migration ファイルの一部
operations = [
    migrations.AddIndex(
        model_name='task',
        index=models.Index(fields=['status'], name='task_status_idx'),
    ),
    # ...
]
```

#### Step 8：psqlでインデックスが作成されたことを確認する

```bash
docker compose exec db psql -U postgres -d taskboard
```

```sql
\d tasks_task
```

`Indexes:` セクションに以下が表示されることを確認する：

```
Indexes:
    "tasks_task_pkey" PRIMARY KEY, btree (id)
    "task_due_date_idx" btree (due_date)
    "task_status_idx" btree (status)
    "task_user_status_idx" btree (created_by_id, status)
```

#### Step 9：EXPLAIN ANALYZE で効果を確認する

```sql
EXPLAIN ANALYZE SELECT * FROM tasks_task WHERE status = 'todo';
```

#### Step 10：README を更新する

`README.md` に PostgreSQL に関するセクションを追加する：

```markdown
## データベースについて

本プロジェクトは SQLite ではなく PostgreSQL 16 を使用しています。

### PostgreSQL を選んだ理由

| 機能 | SQLite | PostgreSQL（本プロジェクト） |
|------|--------|--------------------------|
| 全文検索 | LIKE のみ | SearchVector / tsquery |
| インデックス | 基本的なもの | 複合インデックス・GINインデックス |
| 同時アクセス | 弱い | 複数ユーザーの同時書き込みに対応 |

### 実装した PostgreSQL 固有機能

- **全文検索**（`SearchVector` / `SearchQuery`）：タスクのタイトル・説明文を全文検索
- **インデックス設計**：`status`・`due_date`・複合インデックスを migration で管理
- **N+1解消**：`select_related` でJOINクエリを生成し、SQL発行数を最小化
- **集計クエリ**：`annotate` / `aggregate` でステータス別・カテゴリ別タスク数を集計
```

#### Step 11：PR を作成してマージする

```bash
git add tasks/models.py tasks/migrations/ README.md
git commit -m "feat: インデックス設計をmigrationで管理・READMEにPostgreSQL説明を追加"
git push origin feature/db-index
```

---

## 最終チェックリスト（Day 11〜15）

- [ ] psqlで接続し、`\d tasks_task` でテーブル構造を確認できる
- [ ] psqlでSELECT / WHERE / ORDER BY / JOIN / GROUP BY を手打ちできる
- [ ] django-debug-toolbar が動作し、SQL発行状況を確認できる
- [ ] N+1問題を select_related で解消している
- [ ] `/tasks/report/` でレポート画面が表示される
- [ ] SearchVector を使った全文検索が動作する
- [ ] インデックスが `Meta.indexes` で定義され migration 管理されている
- [ ] EXPLAIN ANALYZE の出力を読んで `Seq Scan` / `Index Scan` の違いを説明できる
- [ ] README に PostgreSQL採用理由と実装概要が記載されている
- [ ] GitHub に Day 11〜15 のブランチ・PRが残っている

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `django.db.utils.ProgrammingError: index "task_status_idx" already exists` | psqlで手動作成したインデックスが残っている | psqlで `DROP INDEX task_status_idx;` してから再migrate |
| `Meta.indexes` を追加しても migrate 不要と言われる | すでに同じインデックスがある | `\d tasks_task` でインデックスを確認し、名前の重複を解消する |
| EXPLAIN で `Index Scan` にならない | データ件数が少なく Seq Scan の方が速い | `SET enable_seqscan = off;` で強制確認、または件数を増やす |

---

## この日の GitHub チェックポイント

### コミットメッセージ例

```bash
git commit -m "feat: インデックス設計をmigrationで管理・READMEにPostgreSQL説明を追加"
```

### プロジェクト全体の振り返り

Day 1〜15 で作った成果物を確認する：

```
main ブランチの PR 履歴
├── feature/docker-setup         Day 1
├── feature/django-init          Day 3 午前
├── feature/user-auth            Day 3〜4
├── feature/task-crud            Day 5〜7
├── feature/category             Day 8
├── feature/search-filter        Day 9
├── feature/readme-tests         Day 10
├── feature/postgres-basics      Day 11  ← debug-toolbar
├── feature/select-related       Day 12  ← N+1解消
├── feature/report-screen        Day 13  ← 集計レポート
├── feature/full-text-search     Day 14  ← PostgreSQL全文検索
└── feature/db-index             Day 15  ← インデックス設計
```

この PR 履歴が「3週間でPostgreSQLを含む実務開発を体験した」証跡になります。
