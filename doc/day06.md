# Day 6 — タスク一覧・詳細ビュー

## この日のゴール

- ログイン後、自分が作成したタスクの一覧が表示される
- タスクをクリックすると詳細ページに遷移する

---

## 1. クラスベースビュー（CBV）の全体像

Django の CBV は「よくある処理のパターン」を継承するだけで実装できる設計になっている。

```
View（基底クラス）
 ├── ListView     → 一覧表示
 ├── DetailView   → 詳細表示
 ├── CreateView   → 作成フォーム
 ├── UpdateView   → 編集フォーム
 └── DeleteView   → 削除確認
```

---

## 2. ListView の仕組み

```python
class TaskListView(LoginRequiredMixin, ListView):
    model = Task                          # 対象モデル
    template_name = 'tasks/task_list.html'  # テンプレートのパス
    context_object_name = 'tasks'         # テンプレート内の変数名（デフォルトは object_list）
```

`ListView` は自動的に `model.objects.all()` を実行してテンプレートに渡す。

**表示データをカスタマイズするには `get_queryset()` をオーバーライド**

```python
def get_queryset(self):
    # ログインユーザーのタスクだけ返す
    return Task.objects.filter(created_by=self.request.user)
```

---

## 3. DetailView の仕組み

```python
class TaskDetailView(LoginRequiredMixin, DetailView):
    model = Task
    template_name = 'tasks/task_detail.html'
```

URL に `<pk>` を含めると、自動的に `Task.objects.get(pk=pk)` を実行して `object` という変数をテンプレートに渡す。

---

## 4. URL パターンの設計

```python
urlpatterns = [
    path('', TaskListView.as_view(), name='task_list'),
    path('<int:pk>/', TaskDetailView.as_view(), name='task_detail'),
]
```

`<int:pk>` は整数の URL パラメータを `pk` という名前でキャプチャする。

テンプレートでのリンク生成：

```html
<!-- 一覧 → 詳細へのリンク -->
<a href="{% url 'tasks:task_detail' task.pk %}">{{ task.title }}</a>
```

---

## 5. Django テンプレートの基本文法

```html
<!-- 変数の表示 -->
{{ task.title }}
{{ task.get_status_display }}

<!-- for ループ -->
{% for task in tasks %}
    <li>{{ task.title }}</li>
{% empty %}
    <li>タスクはありません</li>
{% endfor %}

<!-- if 分岐 -->
{% if task.status == 'done' %}
    <span class="badge bg-success">完了</span>
{% elif task.status == 'in_progress' %}
    <span class="badge bg-warning">進行中</span>
{% else %}
    <span class="badge bg-secondary">未着手</span>
{% endif %}

<!-- URL の生成 -->
<a href="{% url 'tasks:task_detail' task.pk %}">詳細</a>
```

---

## 6. ハンズオン

### Step 1：tasks アプリに URL を設定

`tasks/urls.py`（新規作成）：

```python
from django.urls import path
from .views import TaskListView, TaskDetailView

app_name = 'tasks'

urlpatterns = [
    path('', TaskListView.as_view(), name='task_list'),
    path('<int:pk>/', TaskDetailView.as_view(), name='task_detail'),
]
```

`config/urls.py` に追加：

```python
urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/', include('accounts.urls')),
    path('tasks/', include('tasks.urls')),
]
```

### Step 2：ビューを実装

`tasks/views.py`：

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
        # 自分のタスクだけアクセス可能にする（他人のタスク URL に直アクセスしても 404）
        return Task.objects.filter(created_by=self.request.user)
```

> **なぜ DetailView にも `get_queryset()` を書くのか**：URL に `/tasks/1/` のように他人のタスク ID を入力されたとき、クエリセットを絞っていないと他人のデータが見えてしまう（Insecure Direct Object Reference）。必ずログインユーザーのものだけを返すようにする。

### Step 3：一覧テンプレートを作成

`templates/tasks/task_list.html`：

```html
{% extends 'base.html' %}

{% block content %}
<div class="d-flex justify-content-between align-items-center mb-3">
    <h2>タスク一覧</h2>
    <a href="{% url 'tasks:task_create' %}" class="btn btn-primary">+ 新規タスク</a>
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

### Step 4：詳細テンプレートを作成

`templates/tasks/task_detail.html`：

```html
{% extends 'base.html' %}

{% block content %}
<div class="card">
    <div class="card-header d-flex justify-content-between">
        <h3>{{ task.title }}</h3>
        <div>
            <a href="{% url 'tasks:task_update' task.pk %}" class="btn btn-sm btn-outline-secondary">編集</a>
            <a href="{% url 'tasks:task_delete' task.pk %}" class="btn btn-sm btn-outline-danger">削除</a>
        </div>
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

### Step 5：動作確認

1. `http://localhost:8000/tasks/` を開く → 一覧ページが表示される
2. Day 5 で管理画面から作成したタスクが表示されている
3. タスクのタイトルをクリック → 詳細ページに遷移する
4. ログアウト後に `http://localhost:8000/tasks/` → ログインページにリダイレクト

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `NoReverseMatch: tasks:task_create` | URL が未定義 | Day 7 で追加する URL。今はリンクをコメントアウトしておく |
| `TemplateDoesNotExist: tasks/task_list.html` | テンプレートのパスが間違っている | `templates/tasks/` ディレクトリが存在するか確認 |
| 一覧に何も表示されない | ログインユーザーのタスクがない | 管理画面でログインユーザーが作成者のタスクを作成する |
| 詳細ページで 404 | 他人のタスク ID にアクセス | `get_queryset()` の絞り込みが効いている。正しい動作 |

---

## Step 6：PR を作成する（コミット・プッシュ）

```bash
git add tasks/views.py tasks/urls.py templates/tasks/
git commit -m "feat: タスク一覧・詳細ビューとテンプレートを実装"
git push origin feature/task-crud
```

GitHub で PR を作成し、以下を記載する。

```
## 概要
タスク一覧・詳細ページを実装しました。

## 変更内容
- `ListView` でログインユーザーのタスクのみ表示
- `DetailView` で詳細ページを実装
- `LoginRequiredMixin` でアクセス制御を追加

## 確認方法
1. ログイン後に `http://localhost:8000/tasks/` を開く
2. 管理画面で作成したタスクが一覧に表示されることを確認
3. タスクをクリックして詳細ページに遷移することを確認
```

> この時点では PR をマージしない。Day 7 で CRUD が完成してからまとめてマージする。

---

## この日の GitHub チェックポイント

> 詳細なルールは [github_flow.md](github_flow.md) を参照。

### いつブランチを切るか

**タイミング**：Day 5 から継続して `feature/task-crud` ブランチを使う。新しいブランチは切らない。

**理由**：タスク機能（モデル → 一覧・詳細 → CRUD → ステータス変更）は「タスク管理機能」という1つの機能セット。Day 5〜7 にわたって1つのブランチで開発し、Day 7 に完成したときに1つの PR にまとめる。機能の途中でマージすると「未完成の機能が main に入る」状態になる。

### いつコミットするか

| タイミング | コミットの意味 | 理由 |
|-----------|--------------|------|
| 一覧ページがブラウザで表示されたとき | 一覧ビュー完成 | ListView + template + URL の3点が揃い動作した状態 |
| 詳細ページに遷移できたとき | 詳細ビュー完成 | 一覧と詳細は独立した画面なので別コミットにしてもよい |

### コミットメッセージ例

```bash
git commit -m "feat: タスク一覧ビューを実装"
git commit -m "feat: タスク詳細ビューを実装"

# まとめる場合
git commit -m "feat: タスク一覧・詳細ビューとテンプレートを実装"
```

### いつ PR をマージするか

**Day 6 の時点ではマージしない。**

**理由**：一覧・詳細だけでは「タスクを作成する手段」がない（管理画面経由でしか作れない）。実用的な機能として完結していない状態を main に入れても意味がなく、PR の粒度として中途半端。Day 7 でCRUDが完成してからまとめてマージする。
