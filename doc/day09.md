# Day 9 — 検索・フィルタリング・ページネーション・フラッシュメッセージ

## このプロジェクト全体の流れ

```
Day 1  → Day 2  → Day 3  → Day 4  → Day 5〜7 → Day 8 → [Day 9] → Day 10
Docker   paiza    Django   ログイン  タスク      カテゴリ  検索      提出
環境構築  学習     初期設定  認証      CRUD        UI改善    ページ
                                                          ネーション
                                                           ★今日
```

カテゴリ機能まで実装し、アプリの主要機能が揃いました。今日は「使いやすさ（UX）」を改善する機能を追加します。

---

## この日のゴール

- タイトル・説明文のキーワード検索ができる
- ステータス・カテゴリでの絞り込みができる
- タスクが10件を超えたらページ分割される
- 操作完了後にフラッシュメッセージが表示される

---

## この日の前提

- Day 8 の `feature/category` が main にマージ済みであること
- カテゴリ機能が動作する状態であること

---

## 午前：検索・フィルタリング

### 1. 検索機能の全体像

```
ユーザーが検索フォームに「会議」と入力して送信
        ↓
ブラウザが /tasks/?q=会議 にアクセス
        ↓
ビューが request.GET.get('q') で「会議」を取得
        ↓
DB で title に「会議」を含む、または description に「会議」を含むタスクを検索
        ↓
一覧テンプレートに結果を渡す
```

### 2. Q オブジェクトとは何か

通常の `filter()` は AND 条件でしか書けない：

```python
# AND 条件（すべての条件を満たすもの）
Task.objects.filter(title__icontains='会議', status='todo')
```

OR 条件（どちらかを満たすもの）を書くには Q オブジェクトを使う：

```python
from django.db.models import Q

# OR 条件：タイトルに「会議」を含む OR 説明に「会議」を含む
Task.objects.filter(
    Q(title__icontains='会議') | Q(description__icontains='会議')
)
```

**演算子の意味**

| 演算子 | 意味 | 例 |
|--------|------|-----|
| `\|` | OR（どちらか一方） | `Q(A) \| Q(B)` |
| `&` | AND（両方） | `Q(A) & Q(B)` |
| `~` | NOT（否定） | `~Q(A)` |

**よく使うルックアップ**

| ルックアップ | 意味 | 例 |
|------------|------|-----|
| `icontains` | 大文字小文字を区別しない部分一致 | `title__icontains='会議'` |
| `exact` | 完全一致（デフォルト） | `status='todo'` |
| `in` | リストの中に含まれる | `status__in=['todo', 'in_progress']` |
| `isnull` | NULL かどうか | `category__isnull=True` |

### 3. 複数フィルタの段階的な絞り込み

検索・ステータスフィルタ・カテゴリフィルタを同時に扱う場合、クエリを段階的に絞り込む：

```python
def get_queryset(self):
    queryset = Task.objects.filter(created_by=self.request.user)

    # Step 1: キーワード検索
    keyword = self.request.GET.get('q', '').strip()
    if keyword:
        queryset = queryset.filter(
            Q(title__icontains=keyword) | Q(description__icontains=keyword)
        )

    # Step 2: ステータスフィルタ
    status = self.request.GET.get('status', '')
    if status:
        queryset = queryset.filter(status=status)

    # Step 3: カテゴリフィルタ（Day 8 から継続）
    category_id = self.request.GET.get('category', '')
    if category_id:
        queryset = queryset.filter(category_id=category_id)

    return queryset
```

### ハンズオン（午前）

#### Step 1：ブランチを作成する

```bash
git checkout main
git pull origin main
git checkout -b feature/search-filter
```

#### Step 2：TaskListView を更新する

`tasks/views.py` の `TaskListView` を書き換える：

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
        # 現在の検索条件をテンプレートに渡す（フォームの入力値を維持するため）
        context['current_q'] = self.request.GET.get('q', '')
        context['current_status'] = self.request.GET.get('status', '')
        context['current_category'] = self.request.GET.get('category', '')
        return context
```

#### Step 3：検索フォームをテンプレートに追加する

`templates/tasks/task_list.html` のカテゴリフィルタを、検索+フィルタの統合フォームに置き換える：

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

#### Step 4：Django シェルで Q オブジェクトを試す

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

# 複合フィルタ（ステータス + 期限あり）
from datetime import date
Task.objects.filter(
    created_by=user,
    status__in=['todo', 'in_progress'],
    due_date__isnull=False
)

exit()
```

---

## 午後：ページネーション・フラッシュメッセージ

### 4. ページネーションの全体像

タスクが増えると一覧が長くなる。ページネーションで分割する：

```
/tasks/       → 1〜10件目
/tasks/?page=2 → 11〜20件目
/tasks/?page=3 → 21〜30件目
```

### 5. Paginator の仕組み

ListView では `paginate_by` を設定するだけで自動的にページ分割される：

```python
class TaskListView(LoginRequiredMixin, ListView):
    ...
    paginate_by = 10  # 1ページ10件に分割
```

テンプレートでは `page_obj` という変数が自動的に使えるようになる：

```python
page_obj.number         # 現在のページ番号
page_obj.has_previous() # 前のページがあるか
page_obj.has_next()     # 次のページがあるか
page_obj.previous_page_number  # 前のページ番号
page_obj.next_page_number      # 次のページ番号
page_obj.paginator.num_pages   # 総ページ数
```

### 6. フラッシュメッセージとは

「タスクを作成しました」「削除しました」のような**1回だけ表示される通知**。

```
タスクを作成 → リダイレクト → 一覧ページに「作成しました」メッセージ表示 → 次のリロードでは消える
```

ビューでメッセージをセット：

```python
from django.contrib import messages

messages.success(request, 'タスクを作成しました。')
messages.error(request, '削除に失敗しました。')
messages.warning(request, '期限が過ぎています。')
```

テンプレートで表示：

```html
{% for message in messages %}
<div class="alert alert-{{ message.tags }}">{{ message }}</div>
{% endfor %}
```

**メッセージレベルと Bootstrap の色**

| メソッド | tags | Bootstrap クラス | 色 |
|---------|------|----------------|-----|
| `messages.success` | `success` | `alert-success` | 緑 |
| `messages.info` | `info` | `alert-info` | 青 |
| `messages.warning` | `warning` | `alert-warning` | 黄 |
| `messages.error` | `danger` | `alert-danger` | 赤 |

> `messages.error` の tags は `error` だが Bootstrap は `alert-danger` を使う。`MESSAGE_TAGS` で変換設定が必要（後述）。

### ハンズオン（午後）

#### Step 5：ListView にページネーションを追加する

`tasks/views.py` の `TaskListView` に `paginate_by` を追加する：

```python
class TaskListView(LoginRequiredMixin, ListView):
    ...
    paginate_by = 10
```

`templates/tasks/task_list.html` のテーブルの下に追加する：

```html
{% if is_paginated %}
<nav>
    <ul class="pagination justify-content-center mt-3">
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

#### Step 6：フラッシュメッセージを追加する

`config/settings.py` の末尾に追加する（`error` → `danger` の変換）：

```python
from django.contrib.messages import constants as messages

MESSAGE_TAGS = {
    messages.ERROR: 'danger',
}
```

`tasks/views.py` の各ビューにメッセージを追加する：

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

#### Step 7：base.html にメッセージ表示エリアを追加する

`templates/base.html` のコンテナ内、`{% block content %}` の上に追加する：

```html
<div class="container mt-4">
    {% for message in messages %}
    <div class="alert alert-{{ message.tags }} alert-dismissible fade show" role="alert">
        {{ message }}
        <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
    </div>
    {% endfor %}

    {% block content %}{% endblock %}
</div>
```

Bootstrap の JS も追加（閉じるボタンを動かすために必要）。`</body>` の直前に追加：

```html
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
```

#### Step 8：動作確認

1. タスクを作成 → 一覧ページに「作成しました」の緑メッセージが表示される
2. タスクを10件以上作成 → `/tasks/` でページネーションが表示される
3. 検索フォームにキーワードを入力 → 該当タスクだけが表示される
4. ステータスフィルタを選択 → 該当ステータスのタスクだけが表示される
5. 検索後に「次へ」を押す → 検索条件が維持されたままページが変わる

#### Step 9：PR を作成してマージする

```bash
git add tasks/ templates/ config/settings.py
git commit -m "feat: キーワード検索・ステータスフィルタ・ページネーション・フラッシュメッセージを実装"
git push origin feature/search-filter
```

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| 検索後に「次へ」で検索条件がリセットされる | ページリンクにクエリパラメータが含まれていない | ページリンクの `href` に `q={{ current_q }}&` を追加する |
| メッセージが表示されない | `MESSAGE_TAGS` 設定漏れ | `settings.py` に `MESSAGE_TAGS` を追加する |
| メッセージの閉じるボタンが動かない | Bootstrap の JS が読み込まれていない | `base.html` に `bootstrap.bundle.min.js` を追加する |

---

## この日の GitHub チェックポイント

> 詳細なルールは [github_flow.md](github_flow.md) を参照。

### いつブランチを切るか

Day 8 のマージ後の `main` から `feature/search-filter` を切る。

**理由**：検索・ページネーション・フラッシュメッセージは「UX 改善」という独立した機能単位。カテゴリフィルタが確定した `main` をベースにしないと、フィルタとの組み合わせ動作確認がやりにくい。

### いつコミットするか

| タイミング | コミットの意味 | 理由 |
|-----------|--------------|------|
| キーワード検索が動いたとき | 検索完成 | 検索ロジック（views.py）とフォーム（template）がセットで動いた状態 |
| ページネーションが表示されたとき | ページ分割完成 | `paginate_by` とテンプレートのページリンクが揃った状態 |
| フラッシュメッセージが表示されたとき | UX フィードバック完成 | 各 CRUD 操作後にメッセージが出る状態 |

### コミットメッセージ例

```bash
git commit -m "feat: キーワード検索とステータスフィルタリングを実装"
git commit -m "feat: タスク一覧にページネーションを追加"
git commit -m "feat: CRUD操作後のフラッシュメッセージを追加"
```

### いつ PR をマージするか

**条件**：検索・ステータスフィルタ・ページネーション・フラッシュメッセージが動作確認済みで、検索後のページネーションで検索条件がリセットされないことを確認済みであること。

**理由**：Day 10 は最終確認とゼロ起動テスト。この PR がマージされた `main` が提出物の最終形になる。
