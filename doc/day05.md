# Day 5 — タスクモデル・マイグレーション

## このプロジェクト全体の流れ

```
Day 1  → Day 2  → Day 3  → Day 4  → [Day 5] → Day 6 → Day 7 → Day 8 → Day 9 → Day 10
Docker   paiza    Django   ログイン   タスク     一覧    CRUD    カテゴリ 検索    提出
環境構築  学習     初期設定  認証       モデル     詳細
                                      ★今日
```

Day 4 まで「誰が使うか（ユーザー認証）」を作りました。今日からは「何を管理するか（タスクデータ）」を作ります。

今日はデータの「設計図（モデル）」と「DB テーブルの作成（マイグレーション）」に集中します。画面はまだ作りません。

---

## この日のゴール

- `Task` モデルと `Category` モデルを作成し、`migrate` が成功する
- Django 管理画面からタスクの作成・編集・削除ができる
- Django シェルで ORM クエリを実行できる

---

## この日の前提

- Day 4 の `feature/user-auth` が main にマージ済みであること
- ログイン・ログアウトが動作する状態であること

---

## 1. データとモデルの全体像

### このアプリに必要なデータを整理する

```
[ユーザー]
  id, username, email, password
  （Day 3 で作成済み）

[カテゴリ]
  id, name, 作成者（どのユーザーが作ったか）

[タスク]
  id, タイトル, 説明, ステータス, 期限,
  作成者（ユーザーと紐づく）, 担当者（ユーザーと紐づく）, カテゴリ（カテゴリと紐づく）
  作成日時, 更新日時
```

### DB テーブルのイメージ

```
usersテーブル          categoriesテーブル         tasksテーブル
┌────┬────────┐    ┌────┬──────┬──────────┐    ┌────┬────────────┬────────┬──────┬──────────┐
│ id │username│    │ id │ name │created_by│    │ id │   title    │ status │cat_id│created_by│
├────┼────────┤    ├────┼──────┼──────────┤    ├────┼────────────┼────────┼──────┼──────────┤
│  1 │ taro   │◄───│  1 │ 仕事 │    1     │◄───│  1 │ 資料作成   │  todo  │  1   │    1     │
│  2 │ hanako │    │  2 │ 趣味 │    2     │    │  2 │ 会議準備   │  done  │  1   │    1     │
└────┴────────┘    └────┴──────┴──────────┘    └────┴────────────┴────────┴──────┴──────────┘
```

矢印（`◄───`）が ForeignKey（外部キー）の参照関係。

### モデルとは

「テーブルの設計図を Python クラスで表現したもの」。

```python
class Task(models.Model):       # クラス定義 = テーブル定義
    title = models.CharField(max_length=200)   # カラム定義
    status = models.CharField(...)
    created_at = models.DateTimeField(auto_now_add=True)
```

このクラスを書くと、Django が自動で SQL（`CREATE TABLE tasks ...`）を生成してくれる。

---

## 2. 主要なフィールドの種類

| フィールド | DB の型 | 主な用途 |
|-----------|---------|---------|
| `CharField(max_length=N)` | VARCHAR | 短いテキスト（タイトル、名前など） |
| `TextField` | TEXT | 長いテキスト（説明文など） |
| `DateField` | DATE | 日付（期限など） |
| `DateTimeField` | DATETIME | 日時（作成日時など） |
| `ForeignKey(Model, on_delete)` | INT + 外部キー | 他テーブルへの参照 |

**よく使うオプション**

```python
null=True        # DB に NULL を許可する（省略可能なフィールド）
blank=True       # フォームで空欄を許可する（null と組み合わせて使う）
default=値       # 値が指定されなかったときのデフォルト値
auto_now_add=True  # レコード作成時に現在時刻を自動セット（変更不可）
auto_now=True      # レコード更新時に現在時刻を自動更新
```

---

## 3. ForeignKey と on_delete

ForeignKey は「このテーブルのカラムは、別のテーブルの行を指す」という設定。

```python
created_by = models.ForeignKey(
    settings.AUTH_USER_MODEL,  # 参照先のモデル（ユーザーモデル）
    on_delete=models.CASCADE,  # 参照先（ユーザー）が削除されたときの動作
    related_name='created_tasks'  # user.created_tasks.all() で逆引きできるようになる
)
```

**on_delete の選択肢と使い分け**

| オプション | 動作 | 使い所 |
|-----------|------|--------|
| `CASCADE` | 参照先が削除されたら一緒に削除 | ユーザーが削除されたらそのタスクも削除したい場合 |
| `SET_NULL` | 参照先が削除されたら NULL にする | カテゴリが削除されてもタスクは残したい場合 |
| `PROTECT` | 参照先が削除されないよう保護する | 参照されているデータは消させたくない場合 |

---

## 4. choices を使ったステータス管理

```python
class Task(models.Model):
    STATUS_CHOICES = [
        ('todo', '未着手'),         # ('DBに保存する値', '画面に表示する値')
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
- `task.get_status_display()` で「未着手」「進行中」などの表示名を取得できる

---

## 5. マイグレーションの仕組み

```
models.py（Pythonでの設計図）
         ↓
python manage.py makemigrations   → migrations/0001_initial.py が生成される
（差分を検出して「変更指示書」を作る）
         ↓
python manage.py migrate          → DB に実際にテーブルが作られる
（変更指示書を実行する）
```

**大切なルール**
- `models.py` を変更したら必ず `makemigrations` → `migrate` の順で実行する
- `migrations/` フォルダ内のファイルは Git にコミットする（チーム全員が同じ DB 構造にするため）

---

## 6. Django ORM の基本クエリ

```python
# 全件取得
Task.objects.all()

# 条件フィルタ（AND 条件）
Task.objects.filter(status='todo')
Task.objects.filter(status='todo', created_by=user)

# 1件取得（存在しない場合は DoesNotExist 例外が発生）
Task.objects.get(pk=1)

# 作成
Task.objects.create(title='テスト', created_by=user)

# 更新
task = Task.objects.get(pk=1)
task.title = '更新後のタイトル'
task.save()

# 削除
task.delete()

# 件数
Task.objects.filter(status='done').count()

# 並び順指定（-をつけると降順）
Task.objects.all().order_by('-created_at')
```

---

## 7. ハンズオン

### Step 1：ブランチを作成する

```bash
git checkout main
git pull origin main
git checkout -b feature/task-crud
```

### Step 2：Task・Category モデルを作成する

`tasks/models.py` を以下の内容に書き換える：

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
        ordering = ['-created_at']  # デフォルトの並び順：作成日時の降順
```

> **なぜ `__str__` を実装するのか**：管理画面やデバッグ時に `<Task object (1)>` ではなくタイトルが表示されるようになる。実務では `__str__` の実装は必須とされることが多い。

### Step 3：マイグレーションを実行する

```bash
docker compose exec web python manage.py makemigrations tasks
docker compose exec web python manage.py migrate
```

成功すると `tasks/migrations/0001_initial.py` が生成される。このファイルを Git にコミットする。

### Step 4：管理画面に登録する

`tasks/admin.py` を以下の内容に書き換える：

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

管理画面（`http://localhost:8000/admin/`）にスーパーユーザーでログインし、以下を確認する：
- 「Categories」からカテゴリを作成できる
- 「Tasks」からタスクを作成・編集・削除できる

### Step 5：Django シェルで ORM を練習する

```bash
docker compose exec web python manage.py shell
```

シェルが起動したら以下を1行ずつ試す：

```python
from tasks.models import Task, Category
from accounts.models import CustomUser

# ユーザーを取得する
user = CustomUser.objects.first()

# カテゴリを作成する
cat = Category.objects.create(name='仕事', created_by=user)
print(cat)  # → 仕事

# タスクを作成する
task = Task.objects.create(
    title='ミーティング準備',
    description='資料を作成する',
    status='todo',
    created_by=user,
    category=cat
)

# 一覧取得する
Task.objects.all()

# ステータスでフィルタする
Task.objects.filter(status='todo')

# ForeignKey の逆参照（user のタスク一覧）
user.created_tasks.all()

# 表示名を取得する
task.get_status_display()   # → '未着手'

# ステータスを更新する
task.status = 'in_progress'
task.save()
task.refresh_from_db()      # DB から最新の値を取得しなおす
print(task.status)          # → 'in_progress'

# 終了
exit()
```

### Step 6：コミットしておく

```bash
git add tasks/
git commit -m "feat: TaskモデルとCategoryモデルを追加"
```

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `no such table` | マイグレーション未実行 | `python manage.py migrate` を実行する |
| `relation already exists` | マイグレーションが重複している | `migrations/` 内の不要ファイルを確認する |
| `django.db.utils.IntegrityError` | NOT NULL カラムに NULL を入れようとした | モデルの `null=True` または `default` を確認する |
| `tasks` が管理画面に表示されない | `INSTALLED_APPS` に `'tasks'` がない | `settings.py` を確認する |

---

## この日の GitHub チェックポイント

> 詳細なルールは [github_flow.md](github_flow.md) を参照。

### いつブランチを切るか

Day 4 のマージ後の `main` から `feature/task-crud` を切る。このブランチは Day 7 まで継続して使う。

**理由**：Task モデルは `ForeignKey(settings.AUTH_USER_MODEL)` で CustomUser を参照する。認証機能がマージされた `main` を起点にしないと、`accounts` アプリが存在しない状態でマイグレーションを実行することになる。

### いつコミットするか

| タイミング | コミットの意味 | 理由 |
|-----------|--------------|------|
| `migrate` 成功・管理画面でタスク作成確認 | モデル定義とマイグレーション完了 | migration ファイルはモデル変更とセットでコミットする。別々にコミットすると「モデルと migration がずれた状態」が履歴に残る |

### コミットメッセージ例

```bash
git commit -m "feat: TaskモデルとCategoryモデルを追加"
git commit -m "chore: TaskとCategoryをDjango管理画面に登録"
```

### いつ PR をマージするか

**Day 5 の時点ではマージしない。**

`feature/task-crud` ブランチは Day 6（一覧・詳細）と Day 7（CRUD）も同じブランチで続ける。タスク機能全体が完成した Day 7 の終わりにまとめてマージする。

**理由**：「モデルだけある（画面がない）」状態を main に入れると、main の状態が「機能として使えないが DB 構造だけ変わった」という中途半端な状態になる。
