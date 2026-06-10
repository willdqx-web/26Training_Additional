# Day 12 — SQL実践：JOIN（複数テーブル結合）とN+1解消

## このプロジェクト全体の流れ

```
Day 1〜10  → Day 11    → [Day 12] → Day 13〜15
基礎完成     psql入門    JOIN        集計・全文検索
             debug-toolbar N+1解消   インデックス
                          ★今日
```

昨日は SELECT 単体を学びました。今日は **複数テーブルを結合（JOIN）** する方法を学び、Django ORM が自動生成する JOIN を意識したコードの書き方（`select_related`）を実装します。

---

## この日のゴール

- INNER JOIN / LEFT JOIN の違いを説明できる
- ForeignKey がSQL上でどう見えるかを理解できる
- psqlで複数テーブルを結合したSELECTを手打ちできる
- N+1問題を発見し、`select_related` で解消できる

---

## この日の前提

- Day 11 の `feature/postgres-basics` が main にマージ済み
- django-debug-toolbar が動作している

---

## 午前：JOIN をpsqlで実践

### 1. JOIN とは何か

JOIN は「2つのテーブルを共通のキーで結合して1つの結果として取り出す」操作です。

TaskBoardでは次のリレーションがあります：

```
tasks_task
  ├── created_by_id → accounts_customuser.id
  ├── assigned_to_id → accounts_customuser.id（NULL の場合あり）
  └── category_id → tasks_category.id（NULL の場合あり）
```

### 2. INNER JOIN vs LEFT JOIN

**INNER JOIN**：両方のテーブルに一致するレコードのみを返す

```sql
SELECT t.title, u.username
FROM tasks_task t
INNER JOIN accounts_customuser u ON t.created_by_id = u.id;
```

```
結果：created_by_id が NULL のタスクは除外される（が、created_by は NOT NULL なので今回は差が出ない）
```

**LEFT JOIN**：左テーブルのレコードを全部返し、右テーブルに一致しない場合はNULLで補完

```sql
SELECT t.title, c.name AS category_name
FROM tasks_task t
LEFT JOIN tasks_category c ON t.category_id = c.id;
```

```
結果：
  設計書を書く | バックエンド
  レビューする  | NULL          ← カテゴリ未設定のタスクも含まれる
  デプロイ準備  | インフラ
```

**選び方の判断基準**

| 状況 | 使うべきJOIN |
|------|------------|
| 紐付いていないレコードを除外したい | INNER JOIN |
| 紐付いていないレコードも表示したい | LEFT JOIN |
| ForeignKeyが `null=True` のフィールドを結合する | LEFT JOIN（NULLレコードを落とさないため） |

### 3. テーブル名の別名（AS）

SQL が長くなると `tasks_task` を毎回書くのが大変です。`AS` で別名をつけます：

```sql
SELECT t.title, u.username AS creator, c.name AS category
FROM tasks_task AS t
LEFT JOIN accounts_customuser AS u ON t.created_by_id = u.id
LEFT JOIN tasks_category AS c ON t.category_id = c.id;
```

`AS` は省略可：`tasks_task t` と書いても同じ。

### ハンズオン（午前）

#### Step 1：psqlに接続する

```bash
docker compose exec db psql -U postgres -d taskboard
```

#### Step 2：INNER JOINでタスクと作成者を結合する

```sql
-- タスクのタイトルと作成者のユーザー名を一緒に取得
SELECT t.id, t.title, t.status, u.username AS created_by
FROM tasks_task t
INNER JOIN accounts_customuser u ON t.created_by_id = u.id
ORDER BY t.created_at DESC;
```

#### Step 3：LEFT JOINでカテゴリなしのタスクも取得する

```sql
-- カテゴリなし（NULL）のタスクも含めて取得
SELECT t.title, c.name AS category_name
FROM tasks_task t
LEFT JOIN tasks_category c ON t.category_id = c.id;
```

#### Step 4：3テーブルを結合する

```sql
-- タスク + 作成者 + 担当者 + カテゴリ
SELECT
    t.id,
    t.title,
    t.status,
    creator.username AS created_by,
    assignee.username AS assigned_to,
    c.name AS category
FROM tasks_task t
INNER JOIN accounts_customuser creator ON t.created_by_id = creator.id
LEFT JOIN accounts_customuser assignee ON t.assigned_to_id = assignee.id
LEFT JOIN tasks_category c ON t.category_id = c.id
ORDER BY t.created_at DESC;
```

> `assigned_to_id` と `category_id` は NULL になりえるので LEFT JOIN を使う。`created_by_id` は NOT NULL なので INNER JOIN でよい。

---

## 午後：N+1問題とselect_related

### 4. N+1問題とは

タスク一覧で「タスクごとに担当者名」を表示しようとすると、こんなことが起きます：

```
1回目の SQL: SELECT * FROM tasks_task              → 10件取得
2回目の SQL: SELECT * FROM accounts_customuser WHERE id = 1  → タスク1の担当者
3回目の SQL: SELECT * FROM accounts_customuser WHERE id = 2  → タスク2の担当者
...（10回繰り返す）
合計: 1 + 10 = 11回のSQL発行
```

これを「N+1問題」と呼びます。タスクが100件なら101回のSQLが発行されます。

### 5. select_related で解消する

`select_related` を使うと、DjangoはJOINを使って1回のSQLで全データを取得します：

```python
# Before（N+1が発生）
tasks = Task.objects.filter(created_by=request.user)

# After（JOIN 1回で取得）
tasks = Task.objects.filter(created_by=request.user).select_related(
    'created_by', 'assigned_to', 'category'
)
```

生成されるSQL：

```sql
SELECT tasks_task.*, creator.*, assignee.*, category.*
FROM tasks_task
INNER JOIN accounts_customuser creator ON tasks_task.created_by_id = creator.id
LEFT OUTER JOIN accounts_customuser assignee ON tasks_task.assigned_to_id = assignee.id
LEFT OUTER JOIN tasks_category ON tasks_task.category_id = tasks_category.id
WHERE tasks_task.created_by_id = 1
```

**`select_related` vs `prefetch_related`**

| 方式 | 仕組み | 使う場面 |
|------|--------|---------|
| `select_related` | SQLのJOIN（1クエリ） | ForeignKey・OneToOne（単数関係） |
| `prefetch_related` | 別クエリ後にPythonで結合 | ManyToMany・逆参照（複数関係） |

今回の Task → User, Task → Category は ForeignKey なので `select_related` を使います。

### ハンズオン（午後）

#### Step 5：N+1 を debug-toolbar で確認する

> **前提確認**：N+1は「テンプレートで関連オブジェクトのフィールドにアクセスしたとき」に発生する。一覧テンプレート（`task_list.html`）に `{{ task.assigned_to.username }}` や `{{ task.category.name }}` の表示がある場合のみ観測できる。
>
> もし一覧テンプレートに担当者・カテゴリの表示がない場合は、ここで一旦 Step 7 に進んで `select_related` を追加した上でDjDTのSQL件数が変わらないことを確認するだけでよい。

1. `http://localhost:8000/tasks/` を開く
2. DjDT の「SQL」パネルを開く
3. `SELECT ... FROM accounts_customuser WHERE id = N` が繰り返されているのを確認する（表示がない場合は件数が少なく、繰り返しがなくて正常）

#### Step 6：ブランチを作成する

```bash
git checkout main
git pull origin main
git checkout -b feature/select-related
```

#### Step 7：TaskListView に select_related を適用する

`tasks/views.py` の `TaskListView.get_queryset()` を更新する：

```python
class TaskListView(LoginRequiredMixin, ListView):
    model = Task
    template_name = 'tasks/task_list.html'
    context_object_name = 'tasks'
    paginate_by = 10

    def get_queryset(self):
        queryset = Task.objects.filter(
            created_by=self.request.user
        ).select_related(
            'created_by', 'assigned_to', 'category'
        )

        keyword = self.request.GET.get('q', '').strip()
        if keyword:
            queryset = queryset.filter(
                Q(title__icontains=keyword) | Q(description__icontains=keyword)
            )
        # ... 既存のフィルタ処理を継続
        return queryset
```

同様に `TaskDetailView` にも追加する：

```python
class TaskDetailView(LoginRequiredMixin, DetailView):
    model = Task
    template_name = 'tasks/task_detail.html'

    def get_queryset(self):
        return Task.objects.filter(
            created_by=self.request.user
        ).select_related('created_by', 'assigned_to', 'category')
```

#### Step 8：SQL件数が減ったことを確認する

1. `http://localhost:8000/tasks/` を再度開く
2. DjDT の「SQL」パネルで件数が減っていることを確認する
3. 実行されたSQLに JOIN が含まれていることを確認する

#### Step 9：Django shell でSQL件数を比較する

`django.db.connection.queries` は `DEBUG=True` のときだけ記録される。設定ファイルで `DEBUG=True` になっていることを確認してから実行する。

```bash
docker compose exec web python manage.py shell
```

```python
from django.db import connection, reset_queries
from tasks.models import Task
from accounts.models import CustomUser

user = CustomUser.objects.first()

# ── N+1 パターン ──
reset_queries()
tasks = list(Task.objects.filter(created_by=user))  # list() で即評価
for t in tasks:
    _ = str(t.assigned_to)  # ← 関連オブジェクトにアクセスするたびSQLが発行される

print(f"SQL件数（N+1）: {len(connection.queries)}")
# 例：タスクが5件なら 1 + 5 = 6件

# ── select_related パターン ──
reset_queries()
tasks_opt = list(Task.objects.filter(created_by=user).select_related('assigned_to'))
for t in tasks_opt:
    _ = str(t.assigned_to)  # ← 追加SQLは発行されない

print(f"SQL件数（select_related）: {len(connection.queries)}")
# 例：1件（JOINで一括取得）

exit()
```

> `connection.queries` が空の場合は `DEBUG=False` になっている。`docker compose exec web python -c "import django; django.setup(); from django.conf import settings; print(settings.DEBUG)"` で確認する。

#### Step 10：PR を作成してマージする

```bash
git add tasks/views.py
git commit -m "perf: select_related でN+1問題を解消"
git push origin feature/select-related
```

PRのコメントに「select_related 前後のSQL件数」をスクリーンショットで添付してマージする。

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `SELECT` でテーブル名が見つからない | Djangoのテーブル名は `アプリ名_モデル名` | `\dt` で正確なテーブル名を確認する |
| LEFT JOINしたのに NULL が出ない | 元データに全件関連がある | テスト用に担当者なしのタスクを作って確認する |
| `select_related` でエラーが出る | フィールド名が間違っている | モデルのフィールド名を `models.py` で確認する |
| SQL件数が変わらない | キャッシュが効いている | `django.db.reset_queries()` 後に再計測する |

---

## この日の GitHub チェックポイント

### コミットメッセージ例

```bash
git commit -m "perf: select_related でN+1問題を解消"
```

### いつ PR をマージするか

**条件**：DjDT でselect_related 適用後のSQL件数が減ったことを確認済みであること。

**理由**：N+1解消は次の集計・検索フェーズでも有効。レポート画面を実装する前にクリーンな状態にしておく。
