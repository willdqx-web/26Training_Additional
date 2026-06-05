# Day 6 — タスク一覧・詳細ビュー

## このプロジェクト全体の流れ

```
Day 1  → Day 2  → Day 3  → Day 4  → Day 5 → [Day 6] → Day 7 → Day 8 → Day 9 → Day 10
Docker   paiza    Django   ログイン  タスク   一覧      CRUD    カテゴリ 検索    提出
環境構築  学習     初期設定  認証      モデル   詳細
                                              ★今日
```

Day 5 でデータの「設計図（モデル）」が完成しました。今日は「画面（ビューとテンプレート）」を作り、タスクの一覧と詳細をブラウザで表示できるようにします。

---

## この日のゴール

- ログイン後、`/tasks/` で自分が作成したタスクの一覧が表示される
- タスクをクリックすると `/tasks/<pk>/` で詳細ページに遷移する

---

## この日の前提

- Day 5 の `feature/task-crud` ブランチで続けて作業する
- 管理画面からタスクが作成できる状態であること

---

## 1. URL → View → Template の流れを理解する

Django のリクエスト処理を順番に追う：

```
ブラウザが http://localhost:8000/tasks/ にアクセス
        ↓
config/urls.py が /tasks/ を認識 → tasks/urls.py に転送
        ↓
tasks/urls.py が path('', TaskListView) と一致 → TaskListView を呼び出す
        ↓
TaskListView が DB からタスクを取得 → テンプレートに渡す
        ↓
templates/tasks/task_list.html でHTMLを生成
        ↓
ブラウザに HTML が返る
```

---

## 2. クラスベースビュー（CBV）の全体像

Django には「よくある処理のパターン」をあらかじめ用意したビューがある：

| ビュー | 何をするか | 主な処理 |
|--------|-----------|---------|
| `ListView` | 一覧表示 | `Model.objects.all()` を実行してテンプレートに渡す |
| `DetailView` | 詳細表示 | `Model.objects.get(pk=pk)` を実行してテンプレートに渡す |
| `CreateView` | 作成フォーム | フォーム表示（GET）と保存処理（POST）の両方を処理する |
| `UpdateView` | 編集フォーム | 既存データをフォームに表示し、更新を保存する |
| `DeleteView` | 削除確認 | 確認画面（GET）と削除処理（POST）の両方を処理する |

---

## 3. ListView の仕組み

```python
class TaskListView(LoginRequiredMixin, ListView):
    model = Task                             # このモデルのデータを取得する
    template_name = 'tasks/task_list.html'   # 使用するテンプレート
    context_object_name = 'tasks'            # テンプレート内で使う変数名
```

デフォルトでは `Task.objects.all()` を実行する。自分のタスクだけ表示したい場合は `get_queryset()` をオーバーライドする：

```python
def get_queryset(self):
    return Task.objects.filter(created_by=self.request.user)
```

---

## 4. DetailView の仕組みとセキュリティ

```python
class TaskDetailView(LoginRequiredMixin, DetailView):
    model = Task
    template_name = 'tasks/task_detail.html'

    def get_queryset(self):
        return Task.objects.filter(created_by=self.request.user)
```

> **なぜ DetailView にも `get_queryset()` を書くのか（重要）**
>
> `http://localhost:8000/tasks/5/` のように URL に ID を直接入力すると、他人のタスクの詳細が見えてしまう可能性がある。
> `get_queryset()` でログインユーザーのタスクだけに絞ることで、他人の ID を入力されても 404 エラーになる。
> これは「Insecure Direct Object Reference（IDOR）」と呼ばれる脆弱性の防止策。

---

## 5. URL パターンの設計

```python
urlpatterns = [
    path('', TaskListView.as_view(), name='task_list'),     # /tasks/
    path('<int:pk>/', TaskDetailView.as_view(), name='task_detail'),  # /tasks/1/
]
```

`<int:pk>` は「整数の URL パラメータを `pk` という名前でキャプチャする」という意味。`/tasks/1/` にアクセスすると `pk=1` が DetailView に渡される。

---

## 6. Django テンプレートの基本文法

```html
<!-- 変数の表示 -->
{{ task.title }}
{{ task.get_status_display }}    <!-- choices の表示名を取得 -->

<!-- for ループ -->
{% for task in tasks %}
    <li>{{ task.title }}</li>
{% empty %}
    <li>タスクはありません</li>  <!-- tasks が空のときに表示 -->
{% endfor %}

<!-- if 分岐 -->
{% if task.status == 'done' %}
    <span class="badge bg-success">完了</span>
{% elif task.status == 'in_progress' %}
    <span class="badge bg-warning">進行中</span>
{% else %}
    <span class="badge bg-secondary">未着手</span>
{% endif %}

<!-- URL の生成（ハードコードしない） -->
<a href="{% url 'tasks:task_detail' task.pk %}">詳細</a>

<!-- フィルタの使用 -->
{{ task.due_date|default:'-' }}        <!-- None のとき '-' を表示 -->
{{ task.created_at|date:"Y年m月d日" }}  <!-- 日付フォーマット -->
```

---

## 7. ハンズオン

### Step 1：ブランチを確認する

Day 5 から継続の `feature/task-crud` ブランチで作業する。

```bash
git checkout feature/task-crud
```

### Step 2：tasks/urls.py を更新する

> **注意**：`config/urls.py` への `tasks/` パスの追加は Day 4 で実施済みのため、本 Step では不要。

`tasks/urls.py` を以下の内容に書き換える（Day 4 のスタブに `task_detail` を追加）：

```python
from django.urls import path
from .views import TaskListView, TaskDetailView

app_name = 'tasks'

urlpatterns = [
    path('', TaskListView.as_view(), name='task_list'),
    path('<int:pk>/', TaskDetailView.as_view(), name='task_detail'),
]
```

`config/urls.py` は Day 4 で設定済みのため変更不要。念のため以下の状態になっていることを確認する：

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/', include('accounts.urls')),
    path('tasks/', include('tasks.urls')),  # Day 4 で追加済み。重複追加不要
]
```

### Step 3：tasks/views.py を実装する

```python
from django.contrib.auth.mixins import LoginRequiredMixin
from django.views.generic import ListView, DetailView
from .models import Task


class TaskListView(LoginRequiredMixin, ListView):
    model = Task
    template_name = 'tasks/task_list.html'
    context_object_name = 'tasks'

    def get_queryset(self):
        return Task.objects.filter(created_by=self.request.user)


class TaskDetailView(LoginRequiredMixin, DetailView):
    model = Task
    template_name = 'tasks/task_detail.html'
    context_object_name = 'task'

    def get_queryset(self):
        # 自分のタスクだけアクセス可能にする（他人のタスク URL に直アクセスすると 404）
        return Task.objects.filter(created_by=self.request.user)
```

### Step 4：一覧テンプレートを作成する

`templates/tasks/` フォルダを作成し、`templates/tasks/task_list.html` を新規作成する：

```html
{% extends 'base.html' %}

{% block content %}
<div class="d-flex justify-content-between align-items-center mb-3">
    <h2>タスク一覧</h2>
    {# + 新規タスク ボタンは Day 7 で追加 #}
</div>

{% if tasks %}
<table class="table table-hover">
    <thead>
        <tr>
            <th>タイトル</th>
            <th>ステータス</th>
            <th>カテゴリ</th>
            <th>期限</th>
        </tr>
    </thead>
    <tbody>
        {% for task in tasks %}
        <tr>
            <td>
                <a href="{% url 'tasks:task_detail' task.pk %}">{{ task.title }}</a>
            </td>
            <td>
                {% if task.status == 'done' %}
                    <span class="badge bg-success">{{ task.get_status_display }}</span>
                {% elif task.status == 'in_progress' %}
                    <span class="badge bg-warning text-dark">{{ task.get_status_display }}</span>
                {% else %}
                    <span class="badge bg-secondary">{{ task.get_status_display }}</span>
                {% endif %}
            </td>
            <td>{{ task.category.name|default:'-' }}</td>
            <td>{{ task.due_date|default:'-' }}</td>
        </tr>
        {% endfor %}
    </tbody>
</table>
{% else %}
<div class="alert alert-info">タスクがありません。新規タスクを作成してください。</div>
{% endif %}
{% endblock %}
```

### Step 5：詳細テンプレートを作成する

`templates/tasks/task_detail.html` を新規作成する：

```html
{% extends 'base.html' %}

{% block content %}
<div class="card">
    <div class="card-header">
        <h3>{{ task.title }}</h3>
        {# 編集・削除ボタンは Day 7 で追加 #}
    </div>
    <div class="card-body">
        <dl class="row">
            <dt class="col-sm-3">ステータス</dt>
            <dd class="col-sm-9">{{ task.get_status_display }}</dd>

            <dt class="col-sm-3">カテゴリ</dt>
            <dd class="col-sm-9">{{ task.category.name|default:'-' }}</dd>

            <dt class="col-sm-3">期限</dt>
            <dd class="col-sm-9">{{ task.due_date|default:'-' }}</dd>

            <dt class="col-sm-3">説明</dt>
            <dd class="col-sm-9">{{ task.description|linebreaks|default:'-' }}</dd>

            <dt class="col-sm-3">作成日時</dt>
            <dd class="col-sm-9">{{ task.created_at|date:"Y年m月d日 H:i" }}</dd>
        </dl>
    </div>
</div>
<a href="{% url 'tasks:task_list' %}" class="btn btn-link mt-2">← 一覧に戻る</a>
{% endblock %}
```

### Step 6：動作確認

1. `http://localhost:8000/tasks/` を開く
   - 管理画面から作成したタスクが一覧に表示される（「+ 新規タスク」ボタンは Day 7 で追加）
2. タスクのタイトルをクリック → 詳細ページに遷移する
3. ログアウト後に `http://localhost:8000/tasks/` にアクセス → ログインページにリダイレクトされる

### Step 7：コミットしておく

```bash
git add tasks/ templates/tasks/
git commit -m "feat: タスク一覧・詳細ビューを実装"
git push origin feature/task-crud
```

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `NoReverseMatch: tasks:task_create` | URL が未定義 | Day 7 で追加する URL。今は一覧の `+ 新規タスク` リンクをコメントアウトしておく |
| `TemplateDoesNotExist: tasks/task_list.html` | テンプレートのパスが間違っている | `templates/tasks/` ディレクトリが存在するか確認する |
| 一覧に何も表示されない | ログインユーザーが作成したタスクがない | 管理画面でログイン中ユーザーを `created_by` に指定してタスクを作成する |
| 詳細ページで 404 | 他人のタスク ID でアクセスした | `get_queryset()` の絞り込みが効いている。正しい動作 |

---

## この日の GitHub チェックポイント

> 詳細なルールは [github_flow.md](github_flow.md) を参照。

### いつブランチを切るか

Day 5 から継続の `feature/task-crud` ブランチを使う。新しいブランチは切らない。

**理由**：一覧・詳細だけでは「タスクを自分で作る方法がない」（管理画面経由でしか作れない）。機能として半完成なため、Day 7 で CRUD が揃ってからまとめてマージする。

### いつコミットするか

| タイミング | コミットの意味 | 理由 |
|-----------|--------------|------|
| 一覧ページがブラウザで表示されたとき | ListView 完成 | URL・View・Template の3点が揃って動いた状態 |
| 詳細ページに遷移できたとき | DetailView 完成 | 一覧・詳細は別々の画面なので別コミットでも可 |

### コミットメッセージ例

```bash
git commit -m "feat: タスク一覧ビューを実装"
git commit -m "feat: タスク詳細ビューを実装"

# まとめる場合
git commit -m "feat: タスク一覧・詳細ビューとテンプレートを実装"
```

### いつ PR をマージするか

**Day 6 の時点ではマージしない。**

**理由**：一覧・詳細しかない状態では「タスクを自分で作れない」ため、機能として完結していない。Day 7 で CRUD が全部揃ってから1つの PR にまとめてマージする。
