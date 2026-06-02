# Day 3 — Django プロジェクト初期設定 ＋ ユーザー認証（登録）

## このプロジェクト全体の流れ

```
Day 1  → Day 2  → [Day 3]  → Day 4  → Day 5〜7 → Day 8 → Day 9 → Day 10
Docker   paiza    Django     ログイン  タスク      カテゴリ 検索    提出
環境構築  学習     初期設定    認証      CRUD
                  +認証登録
                   ★今日
```

Day 1 で「動く箱（環境）」を作り、Day 2 で Django の概念を学びました。今日は**その箱の中に Django アプリを作り込む**最初の日です。

午前は「プロジェクトの骨格と DB 接続」、午後は「ユーザー登録機能」を実装します。

---

## この日のゴール

- `http://localhost:8000` で Django のウェルカムページが表示される
- `http://localhost:8000/accounts/register/` からユーザー登録ができ、DB に保存される

---

## この日の前提

- Day 1 の `feature/docker-setup` が main にマージ済みであること
- `docker compose up` で web と db が起動する状態であること

---

## 午前：Django プロジェクト初期設定

### 1. Django の全体構造を理解する

まず Django アプリ全体の構造を把握してから、個別の設定に入る。

```
ブラウザ
  │ URL（/tasks/ など）
  ▼
urls.py（ルーター）：URL を見てどのビューに渡すかを決める
  │
  ▼
views.py（処理）：データを取得して、テンプレートに渡す
  │           ↕
  │         models.py（DB とのやりとり）
  │           ↕
  │         PostgreSQL（データベース）
  ▼
templates/（HTML）：データを受け取って画面を作る
  │
  ▼
ブラウザにレスポンスを返す
```

この「URL → View → Model → Template」の流れを **MVT（Model-View-Template）** と呼ぶ。これが Django の中心的な考え方。

### 2. プロジェクトとアプリの違い

Django には「プロジェクト」と「アプリ」という2つの単位がある。

| 単位 | 役割 | 例 |
|------|------|-----|
| **プロジェクト** | アプリ全体の設定をまとめる箱 | `config/settings.py`、`config/urls.py` |
| **アプリ** | 機能ごとに分けたモジュール | `accounts`（認証）、`tasks`（タスク管理） |

```
taskboard/（プロジェクトルート）
├── config/（プロジェクト設定）
│   ├── settings.py   ← DB 設定、インストールアプリの一覧など
│   └── urls.py       ← ルートURL設定
├── accounts/（アプリ：認証機能）
├── tasks/（アプリ：タスク機能）
└── templates/（HTMLテンプレート）
```

### 3. settings.py の主要な設定項目

`settings.py` はプロジェクト全体の設定ファイル。最初に理解しておくべき項目：

```python
# インストール済みアプリの一覧
# 自分で作ったアプリはここに追加しないと認識されない
INSTALLED_APPS = [
    'django.contrib.admin',    # 管理画面
    'django.contrib.auth',     # 認証機能
    'django.contrib.contenttypes',
    'django.contrib.sessions', # セッション管理
    'django.contrib.messages', # フラッシュメッセージ
    'django.contrib.staticfiles', # CSS・画像ファイル
]

# DB の接続先
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',  # デフォルトは SQLite
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}

# テンプレートの検索場所
TEMPLATES = [
    {
        'DIRS': [],  # ここにテンプレートのディレクトリを追加する
        ...
    }
]
```

### 4. PostgreSQL への接続設定

デフォルトは SQLite（1ファイルの簡易 DB）だが、本番環境に近い PostgreSQL に切り替える。

切り替えに必要な変更：

```python
# settings.py の DATABASES を差し替える
import os

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME', 'taskboard'),
        'USER': os.environ.get('DB_USER', 'taskboard_user'),
        'PASSWORD': os.environ.get('DB_PASSWORD', 'password'),
        'HOST': os.environ.get('DB_HOST', 'db'),   # docker-compose のサービス名
        'PORT': os.environ.get('DB_PORT', '5432'),
    }
}
```

**なぜ `os.environ.get` を使うのか**：パスワードなどをコードに直書きすると GitHub にアップしたときに全世界に公開されてしまう。`.env` ファイルから読み込むことでシークレットを守る。

### 5. ハンズオン（午前）

#### Step 1：ブランチを作成する

```bash
git checkout main        # まず main に戻る
git pull origin main     # Day 1 のマージ後の最新を取得
git checkout -b feature/django-init
```

#### Step 2：Django プロジェクトを作成する

```bash
docker compose exec web django-admin startproject config .
```

このコマンドを実行すると以下のファイルが生成される：

```
taskboard/
├── manage.py          ← これから頻繁に使うコマンドラインツール
└── config/
    ├── __init__.py
    ├── settings.py    ← ここを編集していく
    ├── urls.py
    ├── wsgi.py
    └── asgi.py
```

#### Step 3：settings.py を編集する

`config/settings.py` の先頭に以下を追加する（`from pathlib import Path` の下の行）：

```python
import os
from dotenv import load_dotenv
load_dotenv()
```

`SECRET_KEY` を環境変数から読み込む形に変更：

```python
SECRET_KEY = os.environ.get('SECRET_KEY')
DEBUG = os.environ.get('DEBUG', 'False') == 'True'
```

`DATABASES` を PostgreSQL 用に書き換える：

```python
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

日本語・タイムゾーン設定に変更：

```python
LANGUAGE_CODE = 'ja'
TIME_ZONE = 'Asia/Tokyo'
```

`TEMPLATES` の `DIRS` に templates フォルダを追加：

```python
TEMPLATES = [
    {
        ...
        'DIRS': [BASE_DIR / 'templates'],
        ...
    }
]
```

#### Step 4：tasks アプリを作成する

```bash
docker compose exec web python manage.py startapp tasks
```

`config/settings.py` の `INSTALLED_APPS` に追加：

```python
INSTALLED_APPS = [
    ...
    'tasks',
]
```

#### Step 5：初期マイグレーションを実行する

マイグレーションとは「モデルの定義を DB のテーブルに反映する作業」。最初の実行では Django 組み込みのテーブル（ユーザー管理など）が作られる。

```bash
docker compose exec web python manage.py migrate
```

成功すると以下のような出力が表示される：

```
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  ...
```

#### Step 6：起動確認

```bash
docker compose up
```

ブラウザで `http://localhost:8000` を開き、Django のロケットマーク（ウェルカムページ）が表示されれば成功。

#### Step 7：PR を作成してマージする

```bash
git add config/ manage.py tasks/
git commit -m "feat: DjangoプロジェクトをPostgreSQL接続で初期化"
git push origin feature/django-init
```

---

## 午後：ユーザー認証（登録機能）

### 6. ユーザー認証の全体像

「認証」とは「あなたは誰か」を確認する仕組み。このプロジェクトでは以下の流れで実装する：

```
[Day 3] ユーザー登録フォーム（新規アカウント作成）
[Day 4] ログイン・ログアウト（認証済みかどうかの確認）
         ↓
         ログイン済みユーザーだけがタスクページにアクセスできる
```

### 7. なぜ AbstractUser を最初に設定するのか

Django の標準ユーザーモデルは後から変更することが非常に難しい（マイグレーションが壊れる）。そのため、**プロジェクト開始時点でカスタムユーザーモデルを設定しておく**のが Django の定石。

```python
# settings.py に追加するだけでカスタムモデルを使うようになる
AUTH_USER_MODEL = 'accounts.CustomUser'
```

今は `AbstractUser` をそのまま継承するだけで中身は変えない。将来「プロフィール画像を追加したい」「電話番号を追加したい」という変更が発生したときに、このカスタムモデルを拡張すればよい。

### 8. クラスベースビュー（CBV）とは

Django のビューには2種類ある：

```python
# 関数ベースビュー（FBV）：シンプルだが定型処理が多い
def register(request):
    if request.method == 'POST':
        form = UserCreationForm(request.POST)
        if form.is_valid():
            form.save()
            return redirect('login')
    else:
        form = UserCreationForm()
    return render(request, 'accounts/register.html', {'form': form})

# クラスベースビュー（CBV）：上と同等の処理を数行で書ける
class RegisterView(CreateView):
    form_class = CustomUserCreationForm
    template_name = 'accounts/register.html'
    success_url = reverse_lazy('tasks:task_list')
```

CBV は「登録・編集・削除」などよくある処理のパターンが Django に組み込まれており、それを継承するだけで実装できる。このプロジェクトでは CBV を基本として使う。

### 9. ハンズオン（午後）

#### Step 1：ブランチを作成する

```bash
git checkout -b feature/user-auth
```

#### Step 2：accounts アプリを作成する

```bash
docker compose exec web python manage.py startapp accounts
```

`config/settings.py` の `INSTALLED_APPS` に追加：

```python
INSTALLED_APPS = [
    ...
    'accounts',
    'tasks',
]
```

#### Step 3：CustomUser モデルを作成する

`accounts/models.py` を以下の内容に書き換える：

```python
from django.contrib.auth.models import AbstractUser


class CustomUser(AbstractUser):
    pass  # 今は何も追加しない。後から拡張できるように継承しておく
```

`config/settings.py` に追加：

```python
AUTH_USER_MODEL = 'accounts.CustomUser'
```

マイグレーションを実行（CustomUser のテーブルを作る）：

```bash
docker compose exec web python manage.py makemigrations accounts
docker compose exec web python manage.py migrate
```

#### Step 4：登録フォームを作成する

`accounts/forms.py` を新規作成する：

```python
from django.contrib.auth.forms import UserCreationForm
from .models import CustomUser


class CustomUserCreationForm(UserCreationForm):
    class Meta:
        model = CustomUser
        fields = ('username', 'email', 'password1', 'password2')
```

`UserCreationForm` は Django 標準のフォームで、パスワードの一致確認・強度チェックなどのバリデーションが最初から組み込まれている。

#### Step 5：ビューを作成する

`accounts/views.py` を以下の内容に書き換える：

```python
from django.contrib.auth import login
from django.urls import reverse_lazy
from django.views.generic import CreateView
from .forms import CustomUserCreationForm


class RegisterView(CreateView):
    form_class = CustomUserCreationForm
    template_name = 'accounts/register.html'
    success_url = reverse_lazy('tasks:task_list')

    def form_valid(self, form):
        response = super().form_valid(form)
        login(self.request, self.object)  # 登録後に自動ログイン
        return response
```

#### Step 6：URL を設定する

`accounts/urls.py` を新規作成する：

```python
from django.urls import path
from .views import RegisterView

app_name = 'accounts'

urlpatterns = [
    path('register/', RegisterView.as_view(), name='register'),
]
```

`config/urls.py` に accounts の URL を追加する：

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/', include('accounts.urls')),
]
```

#### Step 7：テンプレートを作成する

`templates/` フォルダを作成し、`templates/base.html` を新規作成する：

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TaskBoard</title>
    <!-- Bootstrap 5 でスタイルを適用 -->
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

`templates/accounts/` フォルダを作成し、`templates/accounts/register.html` を新規作成する：

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

#### Step 8：動作確認

管理画面用のスーパーユーザーを作成しておく：

```bash
docker compose exec web python manage.py createsuperuser
```

1. `http://localhost:8000/accounts/register/` を開く
2. ユーザー名・メール・パスワードを入力して「登録」をクリック
3. エラーなく遷移すれば成功
4. `http://localhost:8000/admin/` にスーパーユーザーでログインし、「Users」にアカウントが作られていることを確認する

#### Step 9：PR を作成してマージする

```bash
git add accounts/ templates/ config/
git commit -m "feat: ユーザー登録フォームとカスタムUserモデルを実装"
git push origin feature/user-auth
```

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `django.db.utils.OperationalError` | DB 接続失敗 | `.env` の DB 設定と `docker-compose.yml` の `environment` が一致しているか確認 |
| `AUTH_USER_MODEL refers to model not installed` | `INSTALLED_APPS` に `accounts` がない | `settings.py` の `INSTALLED_APPS` に `'accounts'` を追加 |
| `Table doesn't exist` | マイグレーション未実行 | `python manage.py migrate` を実行 |
| `TemplateDoesNotExist` | テンプレートのパス設定ミス | `TEMPLATES` の `DIRS` に `BASE_DIR / 'templates'` が設定されているか確認 |

---

## この日の GitHub チェックポイント

> 詳細なルールは [github_flow.md](github_flow.md) を参照。

### いつブランチを切るか

- **午前**：Day 1 のマージ後の `main` から `feature/django-init` を切る
- **午後**：`feature/django-init` のマージ後の `main` から `feature/user-auth` を切る

**理由**：「Django 初期設定」と「認証機能」は独立した機能。初期設定でマイグレーションを実行するため、この PR がマージされた `main` をベースに認証ブランチを切ることで、`AUTH_USER_MODEL` の設定が確実に含まれた状態で作業できる。

### いつコミットするか

| タイミング | コミットの意味 | 理由 |
|-----------|--------------|------|
| `migrate` 成功・ウェルカムページ表示 | Django + PostgreSQL 接続完了 | この状態でコミットすると「DB につながって動く」最小の安全地点になる |
| `makemigrations accounts` + `migrate` 完了 | カスタムユーザーモデル追加 | migration ファイルはモデル変更とセットでコミットする |
| 登録フォームからユーザーが作成できたとき | 登録機能完成 | 動作確認済みの状態を記録する |

### コミットメッセージ例

```bash
git commit -m "feat: DjangoプロジェクトをPostgreSQL接続で初期化"
git commit -m "feat: AbstractUserを継承したCustomUserモデルを追加"
git commit -m "feat: ユーザー登録フォームとビューを実装"
```

### いつ PR をマージするか

- `feature/django-init`：ウェルカムページが表示されること
- `feature/user-auth`：登録フォームからユーザーが DB に保存されること

**理由**：`feature/django-init` は次の認証ブランチのベース。不安定な状態でマージすると、認証実装の途中で環境の問題に気づいてしまい、どちらのバグか分からなくなる。
