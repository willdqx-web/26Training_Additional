# Day 8 — カテゴリ機能・UI 改善

## このプロジェクト全体の流れ

```
Day 1  → Day 2  → Day 3  → Day 4  → Day 5〜7 → [Day 8] → Day 9 → Day 10
Docker   paiza    Django   ログイン  タスク       カテゴリ   検索    提出
環境構築  学習     初期設定  認証      CRUD         UI改善
                                                  ★今日
```

タスク管理の中核機能が完成しました。今日は「カテゴリで整理する」機能と「コンポーネント分割によるテンプレートの整理」を実装します。

---

## この日のゴール

- カテゴリを作成・削除できる
- タスク作成フォームでカテゴリを選択できる
- タスク一覧でカテゴリ絞り込みができる
- ナビバーが `{% include %}` でコンポーネントとして分割されている

---

## この日の前提

- Day 7 の `feature/task-crud` が main にマージ済みであること
- タスクの CRUD が動作する状態であること

---

## 1. カテゴリ機能の全体像

```
カテゴリ管理（/tasks/categories/）
  ├── カテゴリ一覧の表示
  ├── カテゴリ追加（名前を入力して保存）
  └── カテゴリ削除

タスクとの連携
  ├── タスク作成・編集フォームにカテゴリ選択を追加
  └── タスク一覧でカテゴリ絞り込み（?category=1）
```

---

## 2. 関連モデルを持つフォーム

タスクのフォームにはカテゴリの選択肢（`ForeignKey`）がある。この選択肢を「ログインユーザーのカテゴリだけ」に絞るために `get_form()` をオーバーライドする（Day 7 で実装済み）。

```python
def get_form(self, form_class=None):
    form = super().get_form(form_class)
    form.fields['category'].queryset = Category.objects.filter(
        created_by=self.request.user
    )
    return form
```

---

## 3. クエリパラメータを使ったフィルタリング

URL に `?category=1` のようなパラメータを付けてフィルタリングする：

```
http://localhost:8000/tasks/?category=1   → カテゴリID=1のタスクだけ表示
http://localhost:8000/tasks/              → 全タスクを表示
```

ビュー側で `request.GET.get('category')` でパラメータを取得する：

```python
def get_queryset(self):
    queryset = Task.objects.filter(created_by=self.request.user)
    category_id = self.request.GET.get('category')
    if category_id:
        queryset = queryset.filter(category_id=category_id)
    return queryset
```

---

## 4. テンプレートの `{% include %}` タグ

共通のパーツを別ファイルに切り出して再利用する仕組み：

```html
<!-- base.html の中で使う -->
{% include 'components/_navbar.html' %}
```

メリット：
- `base.html` がシンプルになる
- ナビバーの変更が1ファイルだけで完結する
- 「部品」としての独立性が高まる

`_` プレフィックスは「直接 URL でアクセスするテンプレートではなく include 用のパーツ」という慣習的な命名。

---

## 5. ハンズオン

### Step 1：ブランチを作成する

```bash
git checkout main
git pull origin main
git checkout -b feature/category
```

### Step 2：カテゴリのビューを作成する

`tasks/views.py` に追加する：

```python
from .models import Category, Task


class CategoryListView(LoginRequiredMixin, ListView):
    model = Category
    template_name = 'tasks/category_list.html'
    context_object_name = 'categories'

    def get_queryset(self):
        return Category.objects.filter(created_by=self.request.user)


class CategoryCreateView(LoginRequiredMixin, CreateView):
    model = Category
    fields = ['name']
    template_name = 'tasks/category_form.html'
    success_url = reverse_lazy('tasks:category_list')

    def form_valid(self, form):
        form.instance.created_by = self.request.user
        return super().form_valid(form)


class CategoryDeleteView(LoginRequiredMixin, DeleteView):
    model = Category
    template_name = 'tasks/category_confirm_delete.html'
    success_url = reverse_lazy('tasks:category_list')

    def get_queryset(self):
        return Category.objects.filter(created_by=self.request.user)
```

### Step 3：カテゴリの URL を追加する

`tasks/urls.py` に追加する：

```python
from .views import (
    TaskListView, TaskDetailView,
    TaskCreateView, TaskUpdateView, TaskDeleteView,
    TaskStatusUpdateView,
    CategoryListView, CategoryCreateView, CategoryDeleteView,
)

urlpatterns = [
    # タスク
    path('', TaskListView.as_view(), name='task_list'),
    path('<int:pk>/', TaskDetailView.as_view(), name='task_detail'),
    path('create/', TaskCreateView.as_view(), name='task_create'),
    path('<int:pk>/update/', TaskUpdateView.as_view(), name='task_update'),
    path('<int:pk>/delete/', TaskDeleteView.as_view(), name='task_delete'),
    path('<int:pk>/status/', TaskStatusUpdateView.as_view(), name='task_status_update'),
    # カテゴリ
    path('categories/', CategoryListView.as_view(), name='category_list'),
    path('categories/create/', CategoryCreateView.as_view(), name='category_create'),
    path('categories/<int:pk>/delete/', CategoryDeleteView.as_view(), name='category_delete'),
]
```

### Step 4：カテゴリのテンプレートを作成する

`templates/tasks/category_list.html`：

```html
{% extends 'base.html' %}

{% block content %}
<div class="d-flex justify-content-between align-items-center mb-3">
    <h2>カテゴリ一覧</h2>
    <a href="{% url 'tasks:category_create' %}" class="btn btn-primary">+ カテゴリ追加</a>
</div>

{% if categories %}
<ul class="list-group">
    {% for category in categories %}
    <li class="list-group-item d-flex justify-content-between align-items-center">
        {{ category.name }}
        <a href="{% url 'tasks:category_delete' category.pk %}" class="btn btn-sm btn-outline-danger">削除</a>
    </li>
    {% endfor %}
</ul>
{% else %}
<div class="alert alert-info">カテゴリがありません。</div>
{% endif %}
{% endblock %}
```

`templates/tasks/category_form.html`：

```html
{% extends 'base.html' %}

{% block content %}
<h2>カテゴリを追加</h2>
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit" class="btn btn-primary">追加</button>
    <a href="{% url 'tasks:category_list' %}" class="btn btn-secondary">キャンセル</a>
</form>
{% endblock %}
```

`templates/tasks/category_confirm_delete.html`：

```html
{% extends 'base.html' %}

{% block content %}
<div class="card border-danger">
    <div class="card-header bg-danger text-white">カテゴリの削除</div>
    <div class="card-body">
        <p>「{{ object.name }}」を削除してもよいですか？</p>
        <p class="text-muted small">このカテゴリに紐付いたタスクのカテゴリは「未設定」になります。</p>
        <form method="post">
            {% csrf_token %}
            <button type="submit" class="btn btn-danger">削除する</button>
            <a href="{% url 'tasks:category_list' %}" class="btn btn-secondary">キャンセル</a>
        </form>
    </div>
</div>
{% endblock %}
```

### Step 5：TaskListView にフィルタリングを追加する

`tasks/views.py` の `TaskListView` を更新する：

```python
class TaskListView(LoginRequiredMixin, ListView):
    model = Task
    template_name = 'tasks/task_list.html'
    context_object_name = 'tasks'

    def get_queryset(self):
        queryset = Task.objects.filter(created_by=self.request.user)
        category_id = self.request.GET.get('category')
        if category_id:
            queryset = queryset.filter(category_id=category_id)
        return queryset

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['categories'] = Category.objects.filter(created_by=self.request.user)
        context['selected_category'] = self.request.GET.get('category', '')
        return context
```

`templates/tasks/task_list.html` のテーブルの上に追加する：

```html
<!-- カテゴリフィルタ -->
<div class="mb-3">
    <form method="get" class="d-inline-flex gap-2 align-items-center">
        <label class="form-label mb-0">カテゴリ：</label>
        <select name="category" class="form-select form-select-sm" style="width:auto">
            <option value="">すべて</option>
            {% for category in categories %}
            <option value="{{ category.pk }}"
                {% if selected_category == category.pk|stringformat:"s" %}selected{% endif %}>
                {{ category.name }}
            </option>
            {% endfor %}
        </select>
        <button type="submit" class="btn btn-sm btn-outline-secondary">絞り込み</button>
        {% if selected_category %}
        <a href="{% url 'tasks:task_list' %}" class="btn btn-sm btn-link">クリア</a>
        {% endif %}
    </form>
</div>
```

### Step 6：ナビバーをコンポーネント分割する

`templates/components/` フォルダを作成し、`templates/components/_navbar.html` を新規作成する：

```html
<nav class="navbar navbar-expand-lg navbar-dark bg-dark">
    <div class="container">
        <a class="navbar-brand" href="{% url 'tasks:task_list' %}">TaskBoard</a>
        <div class="navbar-nav me-auto">
            {% if user.is_authenticated %}
            <a class="nav-link" href="{% url 'tasks:task_list' %}">タスク</a>
            <a class="nav-link" href="{% url 'tasks:category_list' %}">カテゴリ</a>
            {% endif %}
        </div>
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

`templates/base.html` の `<nav>` タグを `{% include %}` に置き換える：

```html
<body>
    {% include 'components/_navbar.html' %}
    <div class="container mt-4">
        {% block content %}{% endblock %}
    </div>
</body>
```

### Step 7：動作確認

1. `/tasks/categories/` でカテゴリを作成・削除できる
2. タスク作成フォームでカテゴリが選択できる
3. タスク一覧のセレクトボックスでカテゴリ絞り込みができる（別カテゴリのタスクが非表示になる）
4. ナビバーに「タスク」「カテゴリ」のリンクが表示されている

### Step 8：PR を作成してマージする

```bash
git add tasks/ templates/
git commit -m "feat: カテゴリ機能とUIコンポーネント分割を実装"
git push origin feature/category
```

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| カテゴリ削除後にタスクのカテゴリが消える | 正常動作（`on_delete=SET_NULL`） | 仕様通り。タスクの `category` が `None` になる |
| フィルタが効かない | パラメータ名のミス | `request.GET.get('category')` と `<select name="category">` の名前が一致しているか確認 |
| `{% include %}` でテンプレートが見つからない | パスのミス | `templates/` からの相対パスで `'components/_navbar.html'` と書いているか確認 |

---

## この日の GitHub チェックポイント

> 詳細なルールは [github_flow.md](github_flow.md) を参照。

### いつブランチを切るか

Day 7 のマージ後の `main` から `feature/category` を切る。

**理由**：カテゴリ機能はタスク CRUD とは別の機能単位。タスク CRUD のブランチに継続して追加すると「タスクの PR」にカテゴリ関連の差分が混入し、「何をした PR か」が分かりにくくなる。

### いつコミットするか

| タイミング | コミットの意味 | 理由 |
|-----------|--------------|------|
| カテゴリの作成・削除が動いたとき | Category CRUD 完成 | カテゴリ機能の最小動作単位 |
| タスクフォームのカテゴリ選択が動いたとき | 連携完成 | ForeignKey 経由の選択肢が保存できる状態 |
| フィルタが動いたとき | フィルタリング完成 | カテゴリ機能の仕上げ |
| ナビバーが `{% include %}` に分割されたとき | UI コンポーネント整理 | テンプレートのリファクタリングは機能変更と別コミットにする |

### コミットメッセージ例

```bash
git commit -m "feat: Categoryモデルと管理CRUD機能を追加"
git commit -m "feat: タスクフォームにカテゴリ選択を追加"
git commit -m "feat: タスク一覧にカテゴリフィルタリングを実装"
git commit -m "refactor: ナビバーをincludeコンポーネントに分割"
```

### いつ PR をマージするか

**条件**：カテゴリの CRUD・タスクへの紐付け・一覧フィルタリングがすべて動作確認済みであること。

**理由**：Day 9 で検索・フィルタリングを拡張するとき、カテゴリフィルタが動いていることを前提にする。中途半端な状態でマージすると Day 9 の検索実装と混在して「どちらのバグか」の切り分けが難しくなる。
