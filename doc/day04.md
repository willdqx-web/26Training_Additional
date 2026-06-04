# Day 4 — ユーザー認証（ログイン・ログアウト・認可）

## このプロジェクト全体の流れ

```
Day 1  → Day 2  → Day 3  → [Day 4]  → Day 5〜7 → Day 8 → Day 9 → Day 10
Docker   paiza    Django    ログイン    タスク      カテゴリ 検索    提出
環境構築  学習     初期設定   認証        CRUD
                  +登録      ★今日
```

Day 3 でユーザー登録ができるようになりました。今日は「ログイン・ログアウト」と「ログインしていないユーザーへのアクセス制限」を実装し、認証フローを完成させます。

---

## この日のゴール

- ログインページでユーザー名・パスワードを入力してログインできる
- ログアウトボタンでセッションが終了する
- ログインしていないユーザーがタスク一覧（`/tasks/`）にアクセスすると、ログインページにリダイレクトされる

---

## この日の前提

- Day 3 の `feature/user-auth` が main にマージ済みであること
- ユーザー登録フォームからアカウントを作成できる状態であること

---

## 1. 認証の全体像：セッションとは何か

ブラウザとサーバーは「HTTP」でやりとりするが、HTTP はリクエストのたびに「誰が送ったか」を覚えていない。そのために「セッション」を使う。

```
[ログイン]
ブラウザ → サーバー：「ユーザー名 + パスワードを送る」
サーバー → DB：「このユーザーは存在するか？パスワードは正しいか？」
DB → サーバー：「OK」
サーバー → ブラウザ：「セッション ID（乱数）を Cookie に保存してね」

[その後のリクエスト]
ブラウザ → サーバー：「この Cookie のセッション ID が添付されています」
サーバー：「セッション ID を DB で確認 → ログイン済みと分かる」
```

**ポイント**：パスワードは最初の1回だけ送る。以降は「セッション ID」というランダムな文字列で認証する。セッション ID を知らない人はログイン状態を偽れない。

---

## 2. Django の認証フロー

```python
# authenticate()：ユーザー名とパスワードを照合する
from django.contrib.auth import authenticate, login

user = authenticate(username='taro', password='pass123')
# → User オブジェクト（一致した場合）または None（不一致の場合）

# login()：セッションにログイン情報を保存する
login(request, user)
# → この後の request.user は taro になる
```

Django には LoginView・LogoutView というビューが標準で用意されているため、上のコードを自分で書く必要はない。URL に接続するだけで動作する。

---

## 3. Django 標準の認証ビュー

```python
from django.contrib.auth import views as auth_views

urlpatterns = [
    # LoginView：フォーム表示（GET）とログイン処理（POST）を両方やってくれる
    path('login/', auth_views.LoginView.as_view(
        template_name='accounts/login.html'
    ), name='login'),

    # LogoutView：POST でセッションを破棄してリダイレクト
    path('logout/', auth_views.LogoutView.as_view(), name='logout'),
]
```

**なぜ LogoutView は POST なのか**：GET（リンクを踏むだけ）でログアウトできると、悪意のあるサイトからリンクを踏ませて強制ログアウトさせることができてしまう（CSRF 攻撃）。POST にすることで CSRF トークンが必要になり防げる。

---

## 4. ログイン後・ログアウト後のリダイレクト先

settings.py でリダイレクト先を設定する：

```python
LOGIN_URL = '/accounts/login/'           # 未認証ユーザーを送る先
LOGIN_REDIRECT_URL = '/tasks/'           # ログイン成功後に送る先
LOGOUT_REDIRECT_URL = '/accounts/login/' # ログアウト後に送る先
```

---

## 5. LoginRequiredMixin：ページへのアクセス制限

ログインしていないユーザーを弾く仕組み。

```python
from django.contrib.auth.mixins import LoginRequiredMixin
from django.views.generic import ListView

class TaskListView(LoginRequiredMixin, ListView):
    # LoginRequiredMixin は必ず最初（左側）に書く
    ...
```

未認証ユーザーがアクセスすると、`settings.py` の `LOGIN_URL` に自動リダイレクトされる。ビューの中に `if not request.user.is_authenticated:` を自分で書く必要はない。

---

## 6. テンプレートでのログイン状態の判定

テンプレートの中では `user` という変数が自動的に使えるようになっている（Django が自動でコンテキストに追加する）。

```html
{% if user.is_authenticated %}
    <!-- ログイン中のユーザーに見せる -->
    <span>{{ user.username }} さん</span>
    <form method="post" action="{% url 'accounts:logout' %}">
        {% csrf_token %}
        <button type="submit">ログアウト</button>
    </form>
{% else %}
    <!-- 未ログインユーザーに見せる -->
    <a href="{% url 'accounts:login' %}">ログイン</a>
{% endif %}
```

---

## 7. ハンズオン

### Step 1：ブランチを確認する

Day 3 の `feature/user-auth` ブランチで引き続き作業する（まだマージしていない場合）。

```bash
git checkout feature/user-auth
```

### Step 2：accounts/urls.py にログイン・ログアウトを追加する

```python
from django.contrib.auth import views as auth_views
from django.urls import path
from .views import RegisterView

app_name = 'accounts'

urlpatterns = [
    path('register/', RegisterView.as_view(), name='register'),
    path('login/', auth_views.LoginView.as_view(
        template_name='accounts/login.html'
    ), name='login'),
    path('logout/', auth_views.LogoutView.as_view(), name='logout'),
]
```

### Step 3：tasks スタブビューを作成する

`LoginRequiredMixin` の動作確認のため、最小限のビュー・URL・テンプレートを用意する。

`tasks/views.py` を以下の内容に書き換える：

```python
from django.contrib.auth.mixins import LoginRequiredMixin
from django.views.generic import TemplateView


class TaskListView(LoginRequiredMixin, TemplateView):
    template_name = 'tasks/task_list.html'
```

`tasks/urls.py` を新規作成する：

```python
from django.urls import path
from .views import TaskListView

app_name = 'tasks'

urlpatterns = [
    path('', TaskListView.as_view(), name='task_list'),
]
```

`config/urls.py` に tasks の URL を追加する：

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/', include('accounts.urls')),
    path('tasks/', include('tasks.urls')),   # 追加
]
```

`templates/tasks/task_list.html` を新規作成する：

```html
{% extends 'base.html' %}

{% block content %}
<h2>タスク一覧（準備中）</h2>
<p>Day 6 で本実装します。</p>
{% endblock %}
```

### Step 4：settings.py にリダイレクト先を追加する

`config/settings.py` の末尾に追記：

```python
LOGIN_URL = '/accounts/login/'
LOGIN_REDIRECT_URL = '/tasks/'
LOGOUT_REDIRECT_URL = '/accounts/login/'
```

### Step 5：ログインテンプレートを作成する

`templates/accounts/login.html` を新規作成：

```html
{% extends 'base.html' %}

{% block content %}
<div class="row justify-content-center">
    <div class="col-md-6">
        <h2>ログイン</h2>
        <form method="post">
            {% csrf_token %}
            {{ form.as_p }}
            <button type="submit" class="btn btn-primary">ログイン</button>
        </form>
        <p class="mt-2">
            アカウントをお持ちでない方は <a href="{% url 'accounts:register' %}">こちら</a>
        </p>
    </div>
</div>
{% endblock %}
```

### Step 6：base.html のナビバーを更新する

`templates/base.html` の `<nav>` 部分を以下に置き換える：

```html
<nav class="navbar navbar-expand-lg navbar-dark bg-dark">
    <div class="container">
        <a class="navbar-brand" href="/">TaskBoard</a>
        <div class="navbar-nav ms-auto">
            {% if user.is_authenticated %}
                <span class="navbar-text text-white me-3">{{ user.username }}</span>
                <form method="post" action="{% url 'accounts:logout' %}" class="d-inline">
                    {% csrf_token %}
                    <button type="submit" class="btn btn-outline-light btn-sm">ログアウト</button>
                </form>
            {% else %}
                <a class="nav-link" href="{% url 'accounts:login' %}">ログイン</a>
                <a class="nav-link" href="{% url 'accounts:register' %}">登録</a>
            {% endif %}
        </div>
    </div>
</nav>
```

### Step 7：動作確認

以下の順番でブラウザ確認する：

1. `http://localhost:8000/accounts/login/` → ログインフォームが表示される
2. Day 3 で作成したユーザーでログイン → `/tasks/` に「タスク一覧（準備中）」ページが表示される
3. ナビバーにユーザー名が表示されている
4. ログアウトボタンを押す → ログインページにリダイレクトされる
5. ログインせずに `http://localhost:8000/tasks/` にアクセス → `http://localhost:8000/accounts/login/?next=/tasks/` にリダイレクトされる

### Step 8：PR を作成してマージする

```bash
git add accounts/ templates/ tasks/ config/
git commit -m "feat: ログイン・ログアウト・アクセス制御を実装"
git push origin feature/user-auth
```

PR のコメントに動作確認のスクリーンショット（ログインページ・ナビバー）を添付してからマージする。

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `NoReverseMatch: accounts:login` | URL の `app_name` が設定されていない | `accounts/urls.py` に `app_name = 'accounts'` を追加 |
| ログアウトが GET でも動いてしまう | Django 5.x の設定 | `LogoutView` は POST のみ有効。`<form method="post">` で実装する |
| ログイン後に `/accounts/profile/` に飛ぶ | `LOGIN_REDIRECT_URL` が未設定 | `settings.py` に `LOGIN_REDIRECT_URL = '/tasks/'` を追加 |
| `403 Forbidden` | CSRF トークン漏れ | form タグの中に `{% csrf_token %}` が書かれているか確認 |

---

## この日の GitHub チェックポイント

> 詳細なルールは [github_flow.md](github_flow.md) を参照。

### いつブランチを切るか

Day 3 の `feature/user-auth` ブランチを継続して使う。登録・ログイン・ログアウトは「認証機能」という1つのまとまりとして1ブランチで管理する。

**理由**：「登録だけ動く状態」は機能として半完成。ログイン・ログアウトまで揃って初めて「認証機能が完成」と言える。途中でブランチを分けると「認証が半分しか動かない状態の PR」が生まれてしまう。

### いつコミットするか

| タイミング | コミットの意味 | 理由 |
|-----------|--------------|------|
| ログインフォームが表示されたとき | ログインページ完成 | URL・テンプレートの疎通確認ができた最小単位 |
| ログイン → ログアウトの全フローが動いたとき | 認証フロー完成 | Day 4 のゴールが達成された状態 |

### コミットメッセージ例

```bash
git commit -m "feat: ログイン・ログアウト機能を実装"
git commit -m "feat: ナビバーにログイン状態の表示を追加"

# まとめる場合
git commit -m "feat: 認証フロー（登録・ログイン・ログアウト）を完成"
```

### いつ PR をマージするか

**条件**：「登録 → ログイン → ログアウト」の全フローをブラウザで確認済みで、PR にスクリーンショットのコメントがあること。

**理由**：Day 5 以降のタスク機能には `LoginRequiredMixin` がかかる。ログインが壊れているとタスク一覧を一切確認できなくなり、機能の動作確認が常にログインページにリダイレクトされてしまう。
