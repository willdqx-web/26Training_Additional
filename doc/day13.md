# Day 13 — 集計機能実装：レポート画面（GROUP BY・ORM annotate）

## このプロジェクト全体の流れ

```
Day 1〜10  → Day 11〜12 → [Day 13] → Day 14〜15
基礎完成     psql・JOIN   集計         全文検索
             N+1解消      レポート      インデックス
                          ★今日
```

JOINでテーブルを結合できるようになりました。今日は**集計（GROUP BY）**を学び、「どのステータスのタスクが何件あるか」「誰に何件タスクが割り当てられているか」を表示するレポート画面を実装します。

---

## この日のゴール

- COUNT / GROUP BY / HAVING をpsqlで手打ちできる
- Django ORM の `annotate` / `aggregate` でDB集計ができる
- 集計データをテンプレートに渡してレポート画面を実装できる

---

## この日の前提

- Day 12 の `feature/select-related` が main にマージ済み

---

## 午前：GROUP BY をpsqlで実践

### 1. 集計関数とは

集計関数は「複数のレコードをまとめて1つの値を計算する」関数です。

| 関数 | 意味 | 例 |
|------|------|-----|
| `COUNT(*)` | 行数を数える | タスクの件数 |
| `COUNT(カラム)` | NULLを除いた件数 | 担当者ありタスクの件数 |
| `SUM(カラム)` | 合計 | 見積もり時間の合計（今回は不使用） |
| `MAX(カラム)` | 最大値 | 最も遅い期限日 |
| `MIN(カラム)` | 最小値 | 最も早い期限日 |

### 2. GROUP BY の仕組み

`GROUP BY` は「同じ値を持つレコードをグループ化してまとめる」操作です。

```
元データ（tasks_task）               GROUP BY status の結果
┌────┬──────────┬──────────┐        ┌──────────┬───────┐
│ id │ title    │ status   │        │ status   │ count │
├────┼──────────┼──────────┤        ├──────────┼───────┤
│  1 │ タスクA  │ todo     │  →     │ todo     │   3   │
│  2 │ タスクB  │ todo     │        │ in_prog  │   2   │
│  3 │ タスクC  │ todo     │        │ done     │   1   │
│  4 │ タスクD  │ in_prog  │        └──────────┴───────┘
│  5 │ タスクE  │ in_prog  │
│  6 │ タスクF  │ done     │
└────┴──────────┴──────────┘
```

```sql
SELECT status, COUNT(*) AS count
FROM tasks_task
GROUP BY status;
```

### 3. HAVING とは

`WHERE` はグループ化前の絞り込み、`HAVING` はグループ化後の絞り込みです。

```sql
-- 2件以上あるステータスのみ表示
SELECT status, COUNT(*) AS count
FROM tasks_task
GROUP BY status
HAVING COUNT(*) >= 2;
```

**WHERE vs HAVING 使い分け**

| 条件の対象 | 使うもの |
|-----------|---------|
| 個別レコード（グループ化前） | `WHERE` |
| 集計結果（グループ化後） | `HAVING` |

### ハンズオン（午前）

#### Step 1：psqlに接続する

```bash
docker compose exec db psql -U postgres -d taskboard
```

#### Step 2：ステータス別タスク数を集計する

```sql
-- ステータス別タスク数
SELECT status, COUNT(*) AS task_count
FROM tasks_task
GROUP BY status
ORDER BY task_count DESC;
```

#### Step 3：カテゴリ別タスク数を集計する

```sql
-- カテゴリ別タスク数（カテゴリなしはNULL）
SELECT c.name AS category, COUNT(t.id) AS task_count
FROM tasks_task t
LEFT JOIN tasks_category c ON t.category_id = c.id
GROUP BY c.name
ORDER BY task_count DESC;
```

#### Step 4：担当者別タスク数を集計する

```sql
-- 担当者別タスク数（担当者ありのみ）
SELECT u.username, COUNT(t.id) AS assigned_count
FROM tasks_task t
INNER JOIN accounts_customuser u ON t.assigned_to_id = u.id
GROUP BY u.username
ORDER BY assigned_count DESC;
```

#### Step 5：期限超過タスクを確認する

```sql
-- 今日以降に期限が切れている未完了タスク
SELECT id, title, due_date, status
FROM tasks_task
WHERE due_date < CURRENT_DATE
  AND status IN ('todo', 'in_progress');
```

#### Step 6：Django shell で ORM の集計と比較する

```bash
docker compose exec web python manage.py shell
```

```python
from django.db.models import Count
from tasks.models import Task

# ステータス別（SQL: SELECT status, COUNT(*) FROM tasks_task GROUP BY status）
status_stats = Task.objects.values('status').annotate(count=Count('id'))
print(list(status_stats))
# → [{'status': 'todo', 'count': 3}, {'status': 'in_progress', 'count': 2}, ...]

# 生成SQLを確認
print(str(status_stats.query))

# aggregate（単一の集計値を取得）
from django.db.models import Count
total = Task.objects.aggregate(total=Count('id'))
print(total)
# → {'total': 6}

exit()
```

**`annotate` vs `aggregate` の違い**

| メソッド | 返り値 | 使う場面 |
|---------|--------|---------|
| `annotate` | QuerySet（複数行） | GROUP BY の結果 |
| `aggregate` | dict（1つの値） | 全体集計（件数・合計など） |

---

## 午後：レポート画面の実装

### 4. レポート画面の設計

URL: `/tasks/report/`

表示する情報：

```
┌─────────────────────────────────────────┐
│ タスクレポート                            │
├──────────────────────┬──────────────────┤
│ ステータス別          │ 件数             │
│ 未着手（todo）        │ 5               │
│ 進行中（in_progress） │ 3               │
│ 完了（done）          │ 10              │
├──────────────────────┼──────────────────┤
│ カテゴリ別            │ 件数             │
│ バックエンド          │ 7               │
│ フロントエンド        │ 4               │
│ （未設定）            │ 7               │
├──────────────────────┴──────────────────┤
│ 期限超過タスク数：2件                     │
└─────────────────────────────────────────┘
```

### ハンズオン（午後）

#### Step 7：ブランチを作成する

```bash
git checkout main
git pull origin main
git checkout -b feature/report-screen
```

#### Step 8：TaskReportView を追加する

`tasks/views.py` に追加する：

```python
from datetime import date
from django.db.models import Count
from django.views.generic import TemplateView


class TaskReportView(LoginRequiredMixin, TemplateView):
    template_name = 'tasks/task_report.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        user = self.request.user

        # ステータス別集計
        context['status_stats'] = (
            Task.objects.filter(created_by=user)
            .values('status')
            .annotate(count=Count('id'))
            .order_by('status')
        )

        # カテゴリ別集計（カテゴリなし = None を「未設定」として扱う）
        context['category_stats'] = (
            Task.objects.filter(created_by=user)
            .values('category__name')
            .annotate(count=Count('id'))
            .order_by('-count')
        )

        # 担当者別集計（担当者ありのみ）
        context['assignee_stats'] = (
            Task.objects.filter(created_by=user, assigned_to__isnull=False)
            .values('assigned_to__username')
            .annotate(count=Count('id'))
            .order_by('-count')
        )

        # 期限超過タスク数
        context['overdue_count'] = Task.objects.filter(
            created_by=user,
            due_date__lt=date.today(),
            status__in=['todo', 'in_progress'],
        ).count()

        # 全体集計
        context['total_count'] = Task.objects.filter(created_by=user).count()

        return context
```

#### Step 9：URLを追加する

`tasks/urls.py` の既存ファイルに **2箇所だけ追記** する（既存の内容は消さない）：

```python
# ① import の行に TaskReportView を追加する
from .views import (
    TaskListView,
    TaskDetailView,
    TaskCreateView,
    TaskUpdateView,
    TaskDeleteView,
    TaskStatusUpdateView,
    TaskReportView,      # ← 追加
)

# ② urlpatterns リストの末尾に追加する
urlpatterns = [
    # ...既存のURLパターンはそのまま残す...
    path('report/', TaskReportView.as_view(), name='task_report'),  # ← 追加
]
```

> **注意**：`urlpatterns = [...]` をそのまま書き換えると既存URLが消える。必ず既存のリストの末尾に `path(...)` の1行を追加する形にする。

#### Step 10：レポートテンプレートを作成する

`templates/tasks/task_report.html` を作成する：

```html
{% extends 'base.html' %}

{% block content %}
<h2 class="mb-4">タスクレポート</h2>

<div class="row mb-4">
    <!-- ステータス別 -->
    <div class="col-md-4">
        <div class="card">
            <div class="card-header fw-bold">ステータス別タスク数</div>
            <ul class="list-group list-group-flush">
                {% for stat in status_stats %}
                <li class="list-group-item d-flex justify-content-between">
                    <span>
                        {% if stat.status == 'todo' %}未着手
                        {% elif stat.status == 'in_progress' %}進行中
                        {% elif stat.status == 'done' %}完了
                        {% endif %}
                    </span>
                    <span class="badge bg-primary rounded-pill">{{ stat.count }}</span>
                </li>
                {% endfor %}
            </ul>
        </div>
    </div>

    <!-- カテゴリ別 -->
    <div class="col-md-4">
        <div class="card">
            <div class="card-header fw-bold">カテゴリ別タスク数</div>
            <ul class="list-group list-group-flush">
                {% for stat in category_stats %}
                <li class="list-group-item d-flex justify-content-between">
                    <span>{{ stat.category__name|default:"（未設定）" }}</span>
                    <span class="badge bg-secondary rounded-pill">{{ stat.count }}</span>
                </li>
                {% endfor %}
            </ul>
        </div>
    </div>

    <!-- 担当者別 -->
    <div class="col-md-4">
        <div class="card">
            <div class="card-header fw-bold">担当者別タスク数</div>
            <ul class="list-group list-group-flush">
                {% for stat in assignee_stats %}
                <li class="list-group-item d-flex justify-content-between">
                    <span>{{ stat.assigned_to__username }}</span>
                    <span class="badge bg-info rounded-pill">{{ stat.count }}</span>
                </li>
                {% empty %}
                <li class="list-group-item text-muted">担当者なし</li>
                {% endfor %}
            </ul>
        </div>
    </div>
</div>

<!-- サマリー -->
<div class="row">
    <div class="col-md-6">
        <div class="card border-danger">
            <div class="card-body">
                <h5 class="card-title text-danger">期限超過タスク</h5>
                <p class="display-6 text-danger">{{ overdue_count }} 件</p>
            </div>
        </div>
    </div>
    <div class="col-md-6">
        <div class="card">
            <div class="card-body">
                <h5 class="card-title">総タスク数</h5>
                <p class="display-6">{{ total_count }} 件</p>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

#### Step 11：ナビゲーションバーにリンクを追加する

`templates/_navbar.html` または `templates/base.html` のナビゲーションに追加する：

```html
<li class="nav-item">
    <a class="nav-link" href="{% url 'tasks:task_report' %}">レポート</a>
</li>
```

#### Step 12：動作確認

1. `http://localhost:8000/tasks/report/` にアクセスする
2. ステータス別・カテゴリ別・担当者別の集計が表示されることを確認する
3. DjDT でSQLを確認し、`GROUP BY` が使われていることを確認する

#### Step 13：PR を作成してマージする

```bash
git add tasks/views.py tasks/urls.py templates/
git commit -m "feat: タスク集計レポート画面を追加"
git push origin feature/report-screen
```

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `category__name` が `KeyError` | `.values()` に存在しないフィールドを指定 | フィールド名を `tasks_task` のカラム名と一致させる |
| カテゴリなしが `None` で表示される | `category__name` が NULL | テンプレートで `\|default:"（未設定）"` を使う |
| `overdue_count` が常に0 | due_date に値なし、またはtimezone問題 | まずpsqlで `SELECT * FROM tasks_task WHERE due_date IS NOT NULL;` で確認 |

---

## この日の GitHub チェックポイント

### コミットメッセージ例

```bash
git commit -m "feat: タスク集計レポート画面を追加"
```

### いつ PR をマージするか

**条件**：`/tasks/report/` でステータス別・カテゴリ別集計が正しく表示されること。

**理由**：レポート画面の動作確認がついたら、次のDay14（全文検索）へ進む。
