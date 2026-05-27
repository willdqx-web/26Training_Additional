# Day 3 — Django プロジェクト初期設定 ＋ ユーザー認証（登録）

## この日のゴール

- `docker compose exec web python manage.py migrate` が成功し、`http://localhost:8000` でウェルカムページが表示される
- ユーザー登録フォームからアカウントが作成でき、DB に保存される

---

## 午前：Django プロジェクト初期設定

### 1. Django プロジェクトの構造

`django-admin startproject config .` を実行すると以下が生成される。

```
taskboard/
├── manage.py          # コマンドラインツール（runserver, migrate など）
└── config/
    ├── __init__.py
    ├── settings.py    # プロジェクト全体の設定
    ├── urls.py        # ルートURL設定
    ├── wsgi.py        # 本番デプロイ用
    └── asgi.py        # 非同期対応（今回は使わない）
```

`manage.py` はプロジェクトの起点。以降のコマンドはすべて `python manage.py <コマンド>` の形式。

### 2. settings.py の重要な設定

```python
# インストール済みアプリの一覧。自分で作ったアプリはここに追加する
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    # 自作アプリはここに追加
]

# DB の接続設定（デフォルトは SQLite）
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}
```

### 3. PostgreSQL に接続する設定

```python
# settings.py の DATABASES を以下に差し替える
import os

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME', 'taskboard'),
        'USER': os.environ.get('DB_USER', 'taskboard_user'),
        'PASSWORD': os.environ.get('DB_PASSWORD', 'password'),
        'HOST': os.environ.get('DB_HOST', 'db'),
        'PORT': os.environ.get('DB_PORT', '5432'),
    }
}
```

**なぜ `os.environ.get` を使うのか**：設定値をコードにハードコードせず、`.env` から読み込むため。パスワードを GitHub にコミットしてしまう事故を防ぐ。

### 4. .env の設定と python-dotenv の使い方

`.env` ファイルに実際の設定値を書く。

```
SECRET_KEY=django-insecure-your-secret-key
DEBUG=True
DB_NAME=taskboard
DB_USER=taskboard_user
DB_PASSWORD=password
DB_HOST=db
DB_PORT=5432
```

`settings.py` の先頭で読み込む。

```python
from dotenv import load_dotenv
load_dotenv()
```

### 5. ハンズオン（午前）

#### Step 1：ブランチ作成

```bash
git checkout -b feature/django-init
```

#### Step 2：Django プロジェクト作成

```bash
docker compose exec web django-admin startproject config .
```

実行後、`manage.py` と `config/` ディレクトリが生成される。

#### Step 3：settings.py を編集

`config/settings.py` の先頭に追加：

```python
import os
from dotenv import load_dotenv
load_dotenv()
```

`SECRET_KEY` を環境変数から読み込む：

```python
SECRET_KEY = os.environ.get('SECRET_KEY')
DEBUG = os.environ.get('DEBUG', 'False') == 'True'
```

`DATABASES` を PostgreSQL 用に変更（前述の設定を貼り付ける）。

`LANGUAGE_CODE` と `TIME_ZONE` を日本語設定に変更：

```python
LANGUAGE_CODE = 'ja'
TIME_ZONE = 'Asia/Tokyo'
```

#### Step 4：初期マイグレーション

```bash
docker compose exec web python manage.py migrate
```

成功すると以下のような出力が表示される。

```
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  ...
```

#### Step 5：起動確認

```bash
docker compose up
```

ブラウザで `http://localhost:8000` を開き、Django のロケットマークが表示されれば成功。

#### Step 6：PR 作成・マージ

```bash
git add config/ manage.py .env.example
git commit -m "feat: Djangoプロジェクト初期設定・PostgreSQL接続を追加"
git push origin feature/django-init
```

---

## 午後：ユーザー認証（登録機能）

### 6. なぜ AbstractUser をカスタムするのか

Django の標準ユーザーモデルをプロジェクト開始時に拡張しておく。**これは後から変更が非常に困難**なため、最初に設定することが鉄則。

```python
# NG：後からカスタムモデルを追加しようとすると migrate が壊れる
# OK：最初から AUTH_USER_MODEL を設定しておく
```

### 7. クラスベースビュー（CBV）とは

Django には2種類のビューがある。

| 種類 | 書き方 | 特徴 |
|------|--------|------|
| 関数ベースビュー（FBV） | `def my_view(request):` | シンプルで読みやすい |
| クラスベースビュー（CBV） | `class MyView(View):` | 共通処理を継承で再利用できる |

CBV を使うと、CRUD の定型処理（`CreateView`、`UpdateView` など）を数行で実装できる。

```python
# FBV の場合（フォーム処理を自分で書く）
def register(request):
    if request.method == 'POST':
        form = UserCreationForm(request.POST)
        if form.is_valid():
            form.save()
            return redirect('login')
    else:
        form = UserCreationForm()
    return render(request, 'accounts/register.html', {'form': form})

# CBV の場合（CreateView が同等の処理を自動でやってくれる）
class RegisterView(CreateView):
    form_class = CustomUserCreationForm
    template_name = 'accounts/register.html'
    success_url = reverse_lazy('login')
```

### 8. ハンズオン（午後）

#### Step 1：ブランチ作成

```bash
git checkout -b feature/user-auth
```

#### Step 2：accounts アプリを作成

```bash
docker compose exec web python manage.py startapp accounts
```

`config/settings.py` の `INSTALLED_APPS` に追加：

```python
INSTALLED_APPS = [
    ...
    'accounts',
]
```

#### Step 3：CustomUser モデルを作成

`accounts/models.py`：

```python
from django.contrib.auth.models import AbstractUser


class CustomUser(AbstractUser):
    pass  # 今は拡張なし。後から項目を追加できるように継承しておく
```

`config/settings.py` に追加：

```python
AUTH_USER_MODEL = 'accounts.CustomUser'
```

マイグレーションを実行：

```bash
docker compose exec web python manage.py makemigrations accounts
docker compose exec web python manage.py migrate
```

#### Step 4：登録フォームを作成

`accounts/forms.py`（新規作成）：

```python
from django.contrib.auth.forms import UserCreationForm
from .models import CustomUser


class CustomUserCreationForm(UserCreationForm):
    class Meta:
        model = CustomUser
        fields = ('username', 'email', 'password1', 'password2')
```

`UserCreationForm` は Django 標準のフォームで、パスワードの一致確認やバリデーションが組み込まれている。

#### Step 5：ビューを作成

`accounts/views.py`：

```python
from django.contrib.auth import login
from django.urls import reverse_lazy
from django.views.generic import CreateView
from .forms import CustomUserCreationForm


class RegisterView(CreateView):
    form_class = CustomUserCreationForm
    template_name = 'accounts/register.html'
    success_url = reverse_lazy('tasks:task_list')  # 登録後の遷移先

    def form_valid(self, form):
        response = super().form_valid(form)
        login(self.request, self.object)  # 登録後に自動ログイン
        return response
```

#### Step 6：URL を設定

`accounts/urls.py`（新規作成）：

```python
from django.urls import path
from .views import RegisterView

app_name = 'accounts'

urlpatterns = [
    path('register/', RegisterView.as_view(), name='register'),
]
```

`config/urls.py` に accounts の URL を追加：

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/', include('accounts.urls')),
]
```

#### Step 7：テンプレートを作成

`templates/base.html`（新規作成）：

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TaskBoard</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body>
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
        <div class="container">
            <a class="navbar-brand" href="/">TaskBoard</a>
        </div>
    </nav>
    <div class="container mt-4">
        {% block content %}{% endblock %}
    </div>
</body>
</html>
```

`templates/accounts/register.html`（新規作成）：

```html
{% extends 'base.html' %}

{% block content %}
<div class="row justify-content-center">
    <div class="col-md-6">
        <h2>アカウント登録</h2>
        <form method="post">
            {% csrf_token %}
            {{ form.as_p }}
            <button type="submit" class="btn btn-primary">登録</button>
        </form>
        <p class="mt-2">
            すでにアカウントをお持ちの方は <a href="{% url 'accounts:login' %}">こちら</a>
        </p>
    </div>
</div>
{% endblock %}
```

`settings.py` にテンプレートの検索パスを追加：

```python
TEMPLATES = [
    {
        ...
        'DIRS': [BASE_DIR / 'templates'],
        ...
    }
]
```

#### Step 8：動作確認

1. `http://localhost:8000/accounts/register/` を開く
2. ユーザー名・メール・パスワードを入力して登録
3. エラーなく遷移すれば成功
4. 管理画面（`http://localhost:8000/admin/`）でユーザーが作成されたか確認

管理画面にログインするためのスーパーユーザーを作成：

```bash
docker compose exec web python manage.py createsuperuser
```

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `django.db.utils.OperationalError` | DB 接続失敗 | `.env` の DB 設定と docker-compose.yml の environment を確認 |
| `AUTH_USER_MODEL refers to model not installed` | INSTALLED_APPS に accounts がない | `INSTALLED_APPS` に `'accounts'` を追加 |
| `Table doesn't exist` | マイグレーション未実行 | `python manage.py migrate` を実行 |
| `TemplateDoesNotExist` | テンプレートのパス設定ミス | `TEMPLATES` の `DIRS` に `BASE_DIR / 'templates'` が設定されているか確認 |

---

## この日の GitHub チェックポイント

> 詳細なルールは [github_flow.md](github_flow.md) を参照。

### いつブランチを切るか

**午前（Django 初期設定）**：Day 1 の `feature/docker-setup` がマージされた後、`main` から `feature/django-init` を切る。

**午後（ユーザー認証）**：Django 初期設定の PR をマージした後、`main` から `feature/user-auth` を切る。

**理由**：「Django の初期設定」と「認証機能の実装」は独立した機能。1ブランチにまとめると PR の差分が大きくなり、「何をしたブランチか」が分かりにくくなる。また、初期設定で `AUTH_USER_MODEL` を設定しているため、マージ順序に依存関係があり、ブランチを分けることでその順序が明確になる。

### いつコミットするか

| タイミング | コミットの意味 | 理由 |
|-----------|--------------|------|
| `migrate` が成功し、ウェルカムページが表示されたとき | Django + PostgreSQL 接続完了 | 「DB につながって動く」最小の完成状態。ここでコミットしないと認証実装の途中でDB設定の問題と混在してしまう |
| `CustomUser` モデルと `makemigrations` が通ったとき | カスタムユーザーモデル追加完了 | モデル変更は migration ファイルとセットでコミットする（migration ファイルは必ずコミットする） |
| 登録フォームからユーザーが作成できたとき | 登録機能完了 | 動作確認済みの状態を記録する |

### コミットメッセージ例

```bash
# Django 初期設定
git commit -m "feat: DjangoプロジェクトをPostgreSQL接続で初期化"

# カスタムユーザーモデル
git commit -m "feat: AbstractUserを継承したCustomUserモデルを追加"

# 登録機能
git commit -m "feat: ユーザー登録フォームとビューを実装"
```

### いつ PR をマージするか

**`feature/django-init` のマージ条件**：`http://localhost:8000` でウェルカムページが表示されること。

**`feature/user-auth` のマージ条件**：登録フォームから新規ユーザーが作成でき、管理画面で確認できること。

**理由**：`feature/django-init` は次の `feature/user-auth` のベースになる。ウェルカムページが表示されない（DB 接続が壊れているなど）状態でマージすると、認証実装のブランチを切った時点から動かない環境で作業することになる。
