# Day 9 — 検索・フィルタリング・ページネーション・フラッシュメッセージ

## この日のゴール

- キーワードでタスクを検索できる
- ステータスでタスクを絞り込める
- タスクが多い場合にページ分割される
- 操作完了後にフラッシュメッセージが表示される

---

## 午前：検索・フィルタリング

### 1. Q オブジェクトとは

複数の条件を OR や AND で組み合わせた複雑なクエリを書くための仕組み。

```python
from django.db.models import Q

# 通常の filter（すべての条件が AND）
Task.objects.filter(title__icontains='会議', status='todo')

# Q オブジェクトで OR 条件
Task.objects.filter(
    Q(title__icontains='会議') | Q(description__icontains='会議')
)

# Q オブジェクトで NOT 条件
Task.objects.filter(~Q(status='done'))

# AND と OR の組み合わせ
Task.objects.filter(
    Q(title__icontains=keyword) | Q(description__icontains=keyword),
    created_by=user  # これは AND として追加される
)
```

**主要なルックアップ**

| ルックアップ | 意味 | 例 |
|------------|------|-----|
| `exact` | 完全一致（デフォルト） | `status='todo'` |
| `icontains` | 大文字小文字を区別しない部分一致 | `title__icontains='会議'` |
| `startswith` | 前方一致 | `title__startswith='A'` |
| `in` | リストの中に含まれる | `status__in=['todo', 'in_progress']` |
| `gte` / `lte` | 以上 / 以下 | `due_date__lte=today` |
| `isnull` | NULL かどうか | `category__isnull=True` |

---

### 2. 複数フィルタの組み合わせ

検索・ステータスフィルタ・カテゴリフィルタを同時に扱う場合、クエリを段階的に絞り込む。

```python
def get_queryset(self):
    queryset = Task.objects.filter(created_by=self.request.user)

    # キーワード検索
    keyword = self.request.GET.get('q', '').strip()
    if keyword:
        queryset = queryset.filter(
            Q(title__icontains=keyword) | Q(description__icontains=keyword)
        )

    # ステータスフィルタ
    status = self.request.GET.get('status', '')
    if status:
        queryset = queryset.filter(status=status)

    # カテゴリフィルタ（Day 8 から継続）
    category_id = self.request.GET.get('category', '')
    if category_id:
        queryset = queryset.filter(category_id=category_id)

    return queryset
```

---

### ハンズオン（午前）

#### Step 1：ブランチ作成

```bash
git checkout -b feature/search-filter
```

#### Step 2：TaskListView の get_queryset を更新

`tasks/views.py` の `TaskListView` を更新：

```python
from django.db.models import Q
from .models import Category, Task


class TaskListView(LoginRequiredMixin, ListView):
    model = Task
    template_name = 'tasks/task_list.html'
    context_object_name = 'tasks'

    def get_queryset(self):
        queryset = Task.objects.filter(created_by=self.request.user)

        keyword = self.request.GET.get('q', '').strip()
        if keyword:
            queryset = queryset.filter(
                Q(title__icontains=keyword) | Q(description__icontains=keyword)
            )

        status = self.request.GET.get('status', '')
        if status:
            queryset = queryset.filter(status=status)

        category_id = self.request.GET.get('category', '')
        if category_id:
            queryset = queryset.filter(category_id=category_id)

        return queryset

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['categories'] = Category.objects.filter(created_by=self.request.user)
        context['status_choices'] = Task.STATUS_CHOICES
        # 現在の検索条件をテンプレートに渡す（フォームの値を維持するため）
        context['current_q'] = self.request.GET.get('q', '')
        context['current_status'] = self.request.GET.get('status', '')
        context['current_category'] = self.request.GET.get('category', '')
        return context
```

#### Step 3：タスク一覧テンプレートに検索フォームを追加

`templates/tasks/task_list.html` のフィルタフォームを更新（Day 8 のカテゴリフィルタと統合）：

```html
<form method="get" class="mb-3">
    <div class="row g-2">
        <div class="col-md-4">
            <input type="text" name="q" value="{{ current_q }}"
                   class="form-control form-control-sm" placeholder="キーワード検索...">
        </div>
        <div class="col-md-3">
            <select name="status" class="form-select form-select-sm">
                <option value="">すべてのステータス</option>
                {% for value, label in status_choices %}
                <option value="{{ value }}" {% if current_status == value %}selected{% endif %}>
                    {{ label }}
                </option>
                {% endfor %}
            </select>
        </div>
        <div class="col-md-3">
            <select name="category" class="form-select form-select-sm">
                <option value="">すべてのカテゴリ</option>
                {% for category in categories %}
                <option value="{{ category.pk }}"
                    {% if current_category == category.pk|stringformat:"s" %}selected{% endif %}>
                    {{ category.name }}
                </option>
                {% endfor %}
            </select>
        </div>
        <div class="col-md-2 d-flex gap-1">
            <button type="submit" class="btn btn-sm btn-outline-primary">検索</button>
            <a href="{% url 'tasks:task_list' %}" class="btn btn-sm btn-outline-secondary">クリア</a>
        </div>
    </div>
</form>
```

#### Step 4：Django シェルで Q オブジェクトの動作を確認

```bash
docker compose exec web python manage.py shell
```

```python
from django.db.models import Q
from tasks.models import Task
from accounts.models import CustomUser

user = CustomUser.objects.first()

# キーワード検索のシミュレーション
keyword = '会議'
Task.objects.filter(
    Q(title__icontains=keyword) | Q(description__icontains=keyword),
    created_by=user
)

# 複合フィルタ
Task.objects.filter(
    created_by=user,
    status__in=['todo', 'in_progress']
).order_by('due_date')
```

---

## 午後：ページネーション・フラッシュメッセージ

### 3. Paginator の仕組み

```python
from django.core.paginator import Paginator

tasks = Task.objects.all()
paginator = Paginator(tasks, 10)     # 1ページ10件
page_obj = paginator.get_page(1)    # 1ページ目を取得

page_obj.object_list   # このページのオブジェクト一覧
page_obj.has_next()    # 次のページがあるか
page_obj.has_previous() # 前のページがあるか
page_obj.number        # 現在のページ番号
page_obj.paginator.num_pages  # 総ページ数
```

ListView の場合は `paginate_by` を設定するだけで自動的に処理される。

```python
class TaskListView(LoginRequiredMixin, ListView):
    ...
    paginate_by = 10  # これだけでページネーション有効
```

テンプレートでは `page_obj` が自動的に使えるようになる。

---

### 4. Django メッセージフレームワーク

操作の成功・失敗・警告を次のリクエストに1回だけ表示する仕組み（フラッシュメッセージ）。

**ビューでメッセージを設定**

```python
from django.contrib import messages

# 成功メッセージ
messages.success(request, 'タスクを作成しました。')

# 警告メッセージ
messages.warning(request, '期限が過ぎています。')

# エラーメッセージ
messages.error(request, '削除に失敗しました。')
```

**テンプレートで表示**

```html
{% for message in messages %}
<div class="alert alert-{{ message.tags }} alert-dismissible fade show" role="alert">
    {{ message }}
    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
</div>
{% endfor %}
```

Django のメッセージレベルと Bootstrap の色クラスの対応：

| メッセージレベル | tags | Bootstrap クラス |
|--------------|------|----------------|
| `messages.success` | `success` | `alert-success`（緑） |
| `messages.info` | `info` | `alert-info`（青） |
| `messages.warning` | `warning` | `alert-warning`（黄） |
| `messages.error` | `error` | `alert-danger`（赤） |

---

### ハンズオン（午後）

#### Step 5：ListView にページネーションを追加

`tasks/views.py` の `TaskListView` に `paginate_by` を追加：

```python
class TaskListView(LoginRequiredMixin, ListView):
    ...
    paginate_by = 10
```

テンプレートの下部にページネーションリンクを追加：

```html
<!-- task_list.html のテーブルの下に追加 -->
{% if is_paginated %}
<nav aria-label="ページナビゲーション">
    <ul class="pagination justify-content-center">
        {% if page_obj.has_previous %}
        <li class="page-item">
            <a class="page-link" href="?{% if current_q %}q={{ current_q }}&{% endif %}{% if current_status %}status={{ current_status }}&{% endif %}page={{ page_obj.previous_page_number }}">前へ</a>
        </li>
        {% endif %}

        {% for num in page_obj.paginator.page_range %}
        <li class="page-item {% if page_obj.number == num %}active{% endif %}">
            <a class="page-link" href="?{% if current_q %}q={{ current_q }}&{% endif %}{% if current_status %}status={{ current_status }}&{% endif %}page={{ num }}">{{ num }}</a>
        </li>
        {% endfor %}

        {% if page_obj.has_next %}
        <li class="page-item">
            <a class="page-link" href="?{% if current_q %}q={{ current_q }}&{% endif %}{% if current_status %}status={{ current_status }}&{% endif %}page={{ page_obj.next_page_number %}">次へ</a>
        </li>
        {% endif %}
    </ul>
</nav>
{% endif %}
```

#### Step 6：ビューにフラッシュメッセージを追加

`tasks/views.py` の各ビューを更新：

```python
from django.contrib import messages


class TaskCreateView(LoginRequiredMixin, CreateView):
    ...
    def form_valid(self, form):
        form.instance.created_by = self.request.user
        messages.success(self.request, f'タスク「{form.instance.title}」を作成しました。')
        return super().form_valid(form)


class TaskUpdateView(LoginRequiredMixin, UpdateView):
    ...
    def form_valid(self, form):
        messages.success(self.request, f'タスク「{form.instance.title}」を更新しました。')
        return super().form_valid(form)


class TaskDeleteView(LoginRequiredMixin, DeleteView):
    ...
    def form_valid(self, form):
        messages.success(self.request, f'タスク「{self.object.title}」を削除しました。')
        return super().form_valid(form)


class TaskStatusUpdateView(LoginRequiredMixin, View):
    def post(self, request, pk):
        task = get_object_or_404(Task, pk=pk, created_by=request.user)
        new_status = request.POST.get('status')
        if new_status in dict(Task.STATUS_CHOICES):
            task.status = new_status
            task.save()
            messages.success(request, f'ステータスを「{task.get_status_display()}」に変更しました。')
        return redirect('tasks:task_detail', pk=pk)
```

#### Step 7：base.html にメッセージ表示エリアを追加

`templates/base.html` のコンテナ内、`{% block content %}` の上に追加：

```html
<div class="container mt-4">
    {% for message in messages %}
    <div class="alert alert-{{ message.tags }} alert-dismissible fade show" role="alert">
        {{ message }}
        <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
    </div>
    {% endfor %}

    {% block content %}{% endblock %}
</div>
```

Bootstrap の JS も追加（`</body>` の直前）：

```html
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
```

#### Step 8：動作確認

1. タスクを作成 → 一覧ページに「タスクを作成しました」の緑メッセージが表示される
2. タスクを編集 → 詳細ページに更新成功メッセージが表示される
3. ステータスを変更 → 詳細ページに変更メッセージが表示される
4. タスクを10件以上作成して `http://localhost:8000/tasks/` → ページネーションが表示される
5. 検索フォームにキーワードを入力 → 該当タスクだけが表示される

#### Step 9：PR 作成・マージ

```bash
git add tasks/ templates/
git commit -m "feat: 検索・フィルタリング・ページネーション・フラッシュメッセージを実装"
git push origin feature/search-filter
```

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| 検索後にページネーションリンクで検索条件がリセットされる | クエリパラメータを引き継いでいない | ページネーションリンクに `q={{ current_q }}&` を追加 |
| メッセージが表示されない | `INSTALLED_APPS` に `django.contrib.messages` がない | 標準では含まれているが `settings.py` を確認 |
| `alert-error` のクラスで CSS が当たらない | Bootstrap は `alert-danger` を使う | `MESSAGE_TAGS` を設定する |

**`MESSAGE_TAGS` の設定（エラーメッセージの CSS 対応）**

`settings.py` に追加：

```python
from django.contrib.messages import constants as messages

MESSAGE_TAGS = {
    messages.ERROR: 'danger',
}
```

---

## この日の GitHub チェックポイント

> 詳細なルールは [github_flow.md](github_flow.md) を参照。

### いつブランチを切るか

**タイミング**：Day 8 の `feature/category` がマージされた後、`main` から `feature/search-filter` を切る。

**理由**：検索・ページネーション・フラッシュメッセージは「UX 改善」という独立した機能単位。カテゴリフィルタ（Day 8）の変更が確定した `main` をベースにしないと、フィルタとの組み合わせ動作確認がやりにくい。

### いつコミットするか

| タイミング | コミットの意味 | 理由 |
|-----------|--------------|------|
| キーワード検索が動いたとき | Q オブジェクトによる検索完成 | 検索ロジック（views.py）とフォーム（template）がセットで動いた状態 |
| ステータスフィルタが動いたとき | フィルタリング完成 | 検索と組み合わせた複合フィルタが動く状態 |
| ページネーションが表示されたとき | ページ分割完成 | `paginate_by` の設定とテンプレートのページリンクが揃った状態 |
| フラッシュメッセージが表示されたとき | UX フィードバック完成 | 各 CRUD 操作後にメッセージが出る状態 |

### コミットメッセージ例

```bash
git commit -m "feat: キーワード検索とステータスフィルタリングを実装"
git commit -m "feat: タスク一覧にページネーションを追加"
git commit -m "feat: CRUD操作後のフラッシュメッセージを追加"
```

### いつ PR をマージするか

**条件**：検索・ステータスフィルタ・ページネーション・フラッシュメッセージがすべて動作確認済みであること。検索条件をまたいだページネーションが正しく動作すること（検索後に「次へ」で検索条件がリセットされないこと）。

**理由**：Day 10 では README とテストを書くが、その際「アプリの全機能が動く」状態を前提にゼロ起動テストを行う。この PR がマージされていれば Day 10 の最終確認がスムーズに進む。
