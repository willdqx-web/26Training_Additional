# Day 5 — タスクモデル・マイグレーション

## この日のゴール

- `Task` モデルと `Category` モデルを作成し、マイグレーションが成功する
- Django 管理画面からタスクの CRUD が動作する
- Django シェルで ORM クエリを操作できる

---

## 1. Django のモデルとは

モデルは DB のテーブルを Python クラスとして表現したもの。

```python
class Task(models.Model):
    title = models.CharField(max_length=200)   # VARCHAR(200)
    created_at = models.DateTimeField(auto_now_add=True)  # TIMESTAMP
```

この1クラスが1テーブルに対応する。フィールド定義 = カラム定義。

---

## 2. 主要なフィールドの種類

| フィールド | DB 型 | 用途 |
|-----------|-------|------|
| `CharField(max_length=N)` | VARCHAR | 短いテキスト |
| `TextField` | TEXT | 長いテキスト |
| `IntegerField` | INT | 整数 |
| `DateField` | DATE | 日付 |
| `DateTimeField` | DATETIME | 日時 |
| `BooleanField` | BOOLEAN | True/False |
| `ForeignKey(Model, on_delete)` | INT + 外部キー | 他テーブルへの参照 |

**よく使うオプション**

```python
null=True        # DB に NULL を許可
blank=True       # フォームで空欄を許可（null と一緒に使うことが多い）
auto_now_add=True  # レコード作成時に現在時刻を自動セット（更新しない）
auto_now=True      # レコード更新時に現在時刻を自動セット
default=値       # デフォルト値を設定
```

---

## 3. ForeignKey と on_delete

```python
created_by = models.ForeignKey(
    settings.AUTH_USER_MODEL,  # 参照先のモデル
    on_delete=models.CASCADE,  # 参照先が削除されたときの動作
    related_name='tasks'       # 逆参照のための名前
)
```

**on_delete の選択肢**

| オプション | 動作 |
|-----------|------|
| `CASCADE` | 参照先が削除されたら一緒に削除 |
| `SET_NULL` | 参照先が削除されたら NULL にする（`null=True` も必要） |
| `PROTECT` | 参照先が削除されないよう保護（エラーを発生させる） |
| `SET_DEFAULT` | 参照先が削除されたらデフォルト値にする |

---

## 4. choices を使ったステータス管理

```python
class Task(models.Model):
    STATUS_CHOICES = [
        ('todo', '未着手'),
        ('in_progress', '進行中'),
        ('done', '完了'),
    ]
    status = models.CharField(
        max_length=20,
        choices=STATUS_CHOICES,
        default='todo'
    )
```

`choices` を設定すると：
- 管理画面でセレクトボックスになる
- `task.get_status_display()` で表示名（「未着手」など）を取得できる

---

## 5. マイグレーションの仕組み

```
モデル定義（models.py）
       ↓
python manage.py makemigrations  → migrations/0001_initial.py（差分ファイルを生成）
       ↓
python manage.py migrate         → DB にテーブルを作成
```

**重要なルール**

- `models.py` を変更したら必ず `makemigrations` → `migrate` をセットで実行する
- `migrations/` ディレクトリのファイルは Git にコミットする（チーム開発で差分共有が必要）
- 本番環境では `migrate` を手動実行してテーブルを更新する

---

## 6. Django ORM の基本クエリ

```python
# 全件取得
Task.objects.all()

# 条件フィルタ（AND 条件）
Task.objects.filter(status='todo')
Task.objects.filter(status='todo', created_by=user)

# 1件取得（存在しない場合は DoesNotExist 例外）
Task.objects.get(pk=1)

# 作成
task = Task.objects.create(title='テスト', created_by=user)

# 更新
task.title = '更新後'
task.save()

# 削除
task.delete()

# 件数
Task.objects.filter(status='done').count()

# 順序指定
Task.objects.all().order_by('-created_at')  # 作成日時の降順
```

---

## 7. ハンズオン

### Step 1：ブランチ作成

```bash
git checkout -b feature/task-crud
```

### Step 2：Task・Category モデルを作成

`tasks/models.py`：

```python
from django.conf import settings
from django.db import models


class Category(models.Model):
    name = models.CharField(max_length=100)
    created_by = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='categories'
    )

    def __str__(self):
        return self.name


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
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='created_tasks'
    )
    assigned_to = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name='assigned_tasks'
    )
    category = models.ForeignKey(
        Category,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name='tasks'
    )
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return self.title

    class Meta:
        ordering = ['-created_at']  # デフォルトの並び順
```

> **なぜ `__str__` を実装するのか**：管理画面やシェルでオブジェクトを表示したときに意味のある文字列が表示されるようにするため。

### Step 3：settings.py に tasks を追加

```python
INSTALLED_APPS = [
    ...
    'accounts',
    'tasks',
]
```

### Step 4：マイグレーション実行

```bash
docker compose exec web python manage.py makemigrations tasks
docker compose exec web python manage.py migrate
```

`migrations/0001_initial.py` が生成されたことを確認する。

### Step 5：管理画面に登録

`tasks/admin.py`：

```python
from django.contrib import admin
from .models import Category, Task


@admin.register(Task)
class TaskAdmin(admin.ModelAdmin):
    list_display = ('title', 'status', 'created_by', 'due_date', 'created_at')
    list_filter = ('status', 'category')
    search_fields = ('title', 'description')


@admin.register(Category)
class CategoryAdmin(admin.ModelAdmin):
    list_display = ('name', 'created_by')
```

管理画面（`http://localhost:8000/admin/`）にログインしてタスクを作成・編集・削除できることを確認。

### Step 6：Django シェルで ORM を練習

```bash
docker compose exec web python manage.py shell
```

シェルで以下を試す：

```python
from tasks.models import Task, Category
from accounts.models import CustomUser

# ユーザーを取得
user = CustomUser.objects.first()

# カテゴリを作成
cat = Category.objects.create(name='仕事', created_by=user)

# タスクを作成
task = Task.objects.create(
    title='ミーティング準備',
    description='資料を作成する',
    status='todo',
    created_by=user,
    category=cat
)

# 一覧取得
Task.objects.all()

# フィルタ
Task.objects.filter(status='todo')

# 関連オブジェクトの取得（ForeignKey の逆参照）
user.created_tasks.all()
cat.tasks.all()

# ステータスを更新
task.status = 'in_progress'
task.save()

# 表示名を取得
task.get_status_display()  # → '進行中'

# 削除
task.delete()
```

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `no such table` | マイグレーション未実行 | `migrate` を実行 |
| `column already exists` | マイグレーションの重複 | `migrations/` 内の不要ファイルを削除して `makemigrations` からやり直す |
| `django.db.utils.IntegrityError` | NOT NULL カラムに NULL を入れようとした | モデルの `null=True` か `default` を確認 |
| `RelatedObjectDoesNotExist` | ForeignKey 先が存在しない | 先に関連オブジェクトを作成する |

---

## この日の GitHub チェックポイント

> 詳細なルールは [github_flow.md](github_flow.md) を参照。

### いつブランチを切るか

**タイミング**：Day 4 の認証 PR がマージされた後、`main` から `feature/task-crud` を切る。

**理由**：Task モデルは `ForeignKey(settings.AUTH_USER_MODEL)` で CustomUser を参照する。認証機能がマージされた `main` をベースにしないと、ローカルの `accounts` アプリが存在しない状態でモデルを作ることになり、`makemigrations` が失敗する可能性がある。

### いつコミットするか

| タイミング | コミットの意味 | 理由 |
|-----------|--------------|------|
| `migrate` が成功したとき | モデル定義とマイグレーション完了 | migration ファイルは必ずモデル変更とセットでコミットする。「モデルを変えた」「migration を実行した」を別コミットにしない |
| 管理画面からタスクの作成・編集・削除が動いたとき | モデル動作確認完了 | Django shell だけでなく管理画面での確認もこの時点で済ませておく |

### コミットメッセージ例

```bash
# モデルと migration をセットでコミット
git commit -m "feat: TaskモデルとCategoryモデルを追加"

# 管理画面の設定を追加したとき
git commit -m "chore: TaskとCategoryをDjango管理画面に登録"
```

### いつ PR をマージするか

**条件**：`python manage.py migrate` が成功し、管理画面からタスクの CRUD が動作すること。

**理由**：`feature/task-crud` ブランチは Day 6・7 でも継続使用する（ビュー・テンプレートを追加していく）。このため「モデルが確実に動く」状態でコミットが積み重なっていることが重要。ただし **Day 5 の時点では PR はマージせず、ブランチを継続して Day 7 まで使う**。マージするのはタスク CRUD がすべて完成した Day 7 の終わりにする。
