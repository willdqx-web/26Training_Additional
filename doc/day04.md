# Day 4 — ユーザー認証（ログイン・ログアウト・認可）

## この日のゴール

- ログイン・ログアウトが動作する
- ログインしていないユーザーがタスク一覧にアクセスするとログインページにリダイレクトされる

---

## 1. Django の認証フロー

```
ユーザーがログインフォームを送信
       ↓
authenticate(username, password) → User オブジェクト or None
       ↓
login(request, user) → セッションに認証情報を保存
       ↓
以降のリクエストで request.user.is_authenticated が True になる
```

セッションとは：ログイン状態をサーバー側で管理する仕組み。ブラウザには「セッションID」だけを Cookie で渡す。

---

## 2. Django 標準の認証ビュー

Django には認証用のビューがあらかじめ用意されている。自分でゼロから書く必要はない。

```python
from django.contrib.auth import views as auth_views

urlpatterns = [
    path('login/', auth_views.LoginView.as_view(), name='login'),
    path('logout/', auth_views.LogoutView.as_view(), name='logout'),
]
```

`LoginView` はデフォルトで `registration/login.html` というテンプレートを探す。パスを変えたい場合：

```python
auth_views.LoginView.as_view(template_name='accounts/login.html')
```

---

## 3. ログイン後・ログアウト後のリダイレクト先

`settings.py` に以下を追加する。

```python
LOGIN_URL = '/accounts/login/'          # 未認証ユーザーをリダイレクトする先
LOGIN_REDIRECT_URL = '/tasks/'          # ログイン成功後のリダイレクト先
LOGOUT_REDIRECT_URL = '/accounts/login/'  # ログアウト後のリダイレクト先
```

---

## 4. LoginRequiredMixin によるアクセス制御

ログインしていないユーザーにはアクセスさせたくないビューに `LoginRequiredMixin` を付ける。

```python
from django.contrib.auth.mixins import LoginRequiredMixin
from django.views.generic import ListView

class TaskListView(LoginRequiredMixin, ListView):
    # LoginRequiredMixin は必ず最初に書く（MRO の順番に従う）
    ...
```

未認証ユーザーがアクセスすると `settings.py` の `LOGIN_URL` へ自動リダイレクトされる。

---

## 5. テンプレートでのログイン状態の判定

```html
{% if user.is_authenticated %}
    <span>{{ user.username }} さん</span>
    <a href="{% url 'accounts:logout' %}">ログアウト</a>
{% else %}
    <a href="{% url 'accounts:login' %}">ログイン</a>
    <a href="{% url 'accounts:register' %}">登録</a>
{% endif %}
```

`user` はテンプレート内でデフォルトで使えるコンテキスト変数（`django.contrib.auth.context_processors.auth` が有効な場合）。

---

## 6. ハンズオン

### Step 1：accounts/urls.py にログイン・ログアウトを追加

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

### Step 2：settings.py にリダイレクト先を追加

`config/settings.py` に追記：

```python
LOGIN_URL = '/accounts/login/'
LOGIN_REDIRECT_URL = '/tasks/'
LOGOUT_REDIRECT_URL = '/accounts/login/'
```

### Step 3：ログインテンプレートを作成

`templates/accounts/login.html`：

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

### Step 4：base.html のナビゲーションバーを更新

`templates/base.html` の `<nav>` 部分を更新：

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

> **ポイント**：ログアウトは GET ではなく POST で実装する。GET で実装すると CSRF 攻撃のリスクがある（リンクを踏むだけでログアウトさせられる）。

### Step 5：動作確認

以下のフローをブラウザで確認する。

1. `http://localhost:8000/accounts/login/` にアクセス → ログインフォームが表示される
2. Day 3 で作成したユーザーでログイン → `/tasks/` にリダイレクトされる（まだ404で問題なし）
3. ナビバーにユーザー名が表示されている
4. ログアウトボタンを押す → ログインページにリダイレクトされる
5. ログインせずに `http://localhost:8000/tasks/` にアクセス → ログインページにリダイレクトされる

### Step 6：PR 作成・マージ

```bash
git add accounts/ templates/ config/settings.py
git commit -m "feat: ログイン・ログアウト・アクセス制御を実装"
git push origin feature/user-auth
```

PR のコメントに動作確認のスクリーンショットを添付してマージする。

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `NoReverseMatch: accounts:login` | URL の `app_name` が設定されていない | `accounts/urls.py` に `app_name = 'accounts'` を追加 |
| ログアウトが GET でも動いてしまう | Django バージョンによる挙動差異 | Django 5.x では POST のみ有効。上記の form 形式で実装する |
| ログイン後に `/accounts/profile/` に飛ぶ | `LOGIN_REDIRECT_URL` が未設定 | `settings.py` に `LOGIN_REDIRECT_URL` を追加 |
| `403 Forbidden` | CSRF トークン漏れ | form に `{% csrf_token %}` があるか確認 |

---

## この日の GitHub チェックポイント

> 詳細なルールは [github_flow.md](github_flow.md) を参照。

### いつブランチを切るか

**タイミング**：Day 3 の `feature/user-auth` PR がマージされた後、`main` から新しいブランチを切る。ただしログイン・ログアウトは登録機能と同じ「認証」の文脈なので、Day 3 の `feature/user-auth` ブランチを継続して使っても構わない。

**理由**：登録・ログイン・ログアウトは「認証機能」という1つの機能セットとみなせる。Day 3 でブランチを切り、Day 4 で継続してコミットし、全認証フローが完成したときに1つの PR にまとめるのが自然な粒度。

### いつコミットするか

| タイミング | コミットの意味 | 理由 |
|-----------|--------------|------|
| ログインフォームが表示されたとき | ログインページ完成 | テンプレートとURLの疎通確認ができた最小単位 |
| ログイン → ログアウトの一連フローが動いたとき | 認証フロー完成 | 「登録 → ログイン → ログアウト」がすべて動く状態が Day 4 のゴール |

### コミットメッセージ例

```bash
# ログイン・ログアウト URL・テンプレートを追加したとき
git commit -m "feat: ログイン・ログアウト機能を実装"

# ナビバーを更新したとき
git commit -m "feat: ナビバーにログイン状態の表示を追加"

# リダイレクト設定を追加したとき（まとめる場合）
git commit -m "feat: 認証フロー（登録・ログイン・ログアウト）を完成"
```

### いつ PR をマージするか

**条件**：「登録 → ログイン → ログアウト」の全フローをブラウザで確認済みであること。PR のコメントに確認済みのスクリーンショットを添付していること。

**理由**：Day 5 以降はタスク機能の実装になるが、タスク一覧には `LoginRequiredMixin` がかかっている。ログイン機能が動いていない状態でマージすると、Day 5 でタスク一覧の動作確認が常にログインページにリダイレクトされてしまい、機能の確認がしにくくなる。
