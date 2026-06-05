# Day 7 — タスク作成・編集・削除・ステータス変更

## このプロジェクト全体の流れ

```
Day 1  → Day 2  → Day 3  → Day 4  → Day 5 → Day 6 → [Day 7] → Day 8 → Day 9 → Day 10
Docker   paiza    Django   ログイン  タスク   一覧     CRUD      カテゴリ 検索    提出
環境構築  学習     初期設定  認証      モデル   詳細     ★今日
```

Day 5〜6 でデータ設計と一覧・詳細画面を作りました。今日は「作成・編集・削除・ステータス変更」を実装し、タスク管理の中核機能を完成させます。

Day 7 の終わりに、Day 5〜7 で作ったタスク機能全体を PR にまとめてマージします。

---

## この日のゴール

- タスクを新規作成できる（フォームから入力して保存）
- タスクを編集できる（内容を変更して保存）
- タスクを削除できる（確認画面を経由して削除）
- ステータスを「未着手 → 進行中 → 完了」に切り替えられる

---

## この日の前提

- Day 5〜6 から継続の `feature/task-crud` ブランチで作業する
- タスクの一覧・詳細が表示される状態であること

---

## 1. CRUD とは何か

CRUD は Web アプリの基本操作の頭文字：

| 操作 | 英語 | Django ビュー | HTTP メソッド |
|------|------|-------------|-------------|
| 作成 | Create | `CreateView` | POST |
| 読み取り | Read | `ListView` / `DetailView` | GET |
| 更新 | Update | `UpdateView` | POST |
| 削除 | Delete | `DeleteView` | POST |

読み取りは Day 6 で完成済み。今日は残りの「C・U・D」を実装する。

---

## 2. ModelForm とは

モデルの定義からフォームを自動生成する仕組み。

```python
class TaskForm(ModelForm):
    class Meta:
        model = Task
        fields = ['title', 'description', 'status', 'due_date', 'category']
```

- `fields` に書いたフィールドだけがフォームに表示される
- `created_by` はフォームに含めない（ビュー側でログインユーザーを自動セット）
- バリデーション（必須チェック、最大文字数チェックなど）は Model の定義に基づいて自動で行われる

---

## 3. CreateView / UpdateView / DeleteView の仕組み

### CreateView

```python
class TaskCreateView(LoginRequiredMixin, CreateView):
    model = Task
    form_class = TaskForm
    template_name = 'tasks/task_form.html'
    success_url = reverse_lazy('tasks:task_list')

    def form_valid(self, form):
        # フォームが valid（バリデーション通過）のとき、保存前に created_by をセット
        form.instance.created_by = self.request.user
        return super().form_valid(form)
```

**処理の流れ**：
1. GET リクエスト → 空のフォームを表示
2. POST リクエスト → フォームをバリデーション
3. バリデーション OK → `form_valid()` で DB に保存 → `success_url` にリダイレクト
4. バリデーション NG → エラーメッセージ付きでフォームを再表示

### UpdateView

```python
class TaskUpdateView(LoginRequiredMixin, UpdateView):
    model = Task
    form_class = TaskForm
    template_name = 'tasks/task_form.html'  # CreateView と同じテンプレートを使い回せる

    def get_success_url(self):
        return reverse_lazy('tasks:task_detail', kwargs={'pk': self.object.pk})

    def get_queryset(self):
        return Task.objects.filter(created_by=self.request.user)  # 自分のタスクだけ編集可
```

### DeleteView

```python
class TaskDeleteView(LoginRequiredMixin, DeleteView):
    model = Task
    template_name = 'tasks/task_confirm_delete.html'  # 確認画面用のテンプレート
    success_url = reverse_lazy('tasks:task_list')

    def get_queryset(self):
        return Task.objects.filter(created_by=self.request.user)  # 自分のタスクだけ削除可
```

---

## 4. ステータス変更：独自ビューの実装

「ステータスをセレクトボックスで変更する」は CreateView / UpdateView などの汎用ビューに当てはまらない。`View` を直接継承して書く。

```python
from django.views import View
from django.shortcuts import get_object_or_404, redirect

class TaskStatusUpdateView(LoginRequiredMixin, View):
    def post(self, request, pk):
        task = get_object_or_404(Task, pk=pk, created_by=request.user)
        new_status = request.POST.get('status')
        if new_status in dict(Task.STATUS_CHOICES):
            task.status = new_status
            task.save()
        return redirect('tasks:task_detail', pk=pk)
```

**なぜ POST だけ受け付けるのか**：GET（リンクを踏むだけ）でステータスを変更できると、悪意のあるサイトに `<img src="http://localhost:8000/tasks/1/status/?status=done">` のようなタグを置くだけでステータスを書き換えられてしまう（CSRF 攻撃）。

---

## 5. ハンズオン

### Step 1：フォームを作成する

`tasks/forms.py` を新規作成する：

```python
from django import forms
from .models import Task


class TaskForm(forms.ModelForm):
    class Meta:
        model = Task
        fields = ['title', 'description', 'status', 'due_date', 'category', 'assigned_to']
        widgets = {
            'due_date': forms.DateInput(attrs={'type': 'date'}),   # ブラウザの日付ピッカーを使う
            'description': forms.Textarea(attrs={'rows': 4}),
        }
```

### Step 2：views.py にビューを追加する

`tasks/views.py` の末尾に追加する：

```python
from django.urls import reverse_lazy
from django.views import View
from django.shortcuts import get_object_or_404, redirect
from django.views.generic import CreateView, UpdateView, DeleteView
from .forms import TaskForm


class TaskCreateView(LoginRequiredMixin, CreateView):
    model = Task
    form_class = TaskForm
    template_name = 'tasks/task_form.html'
    success_url = reverse_lazy('tasks:task_list')

    def form_valid(self, form):
        form.instance.created_by = self.request.user
        return super().form_valid(form)

    def get_form(self, form_class=None):
        form = super().get_form(form_class)
        # カテゴリの選択肢をログインユーザーのものだけに絞る
        form.fields['category'].queryset = self.request.user.categories.all()
        return form


class TaskUpdateView(LoginRequiredMixin, UpdateView):
    model = Task
    form_class = TaskForm
    template_name = 'tasks/task_form.html'

    def get_success_url(self):
        return reverse_lazy('tasks:task_detail', kwargs={'pk': self.object.pk})

    def get_queryset(self):
        return Task.objects.filter(created_by=self.request.user)

    def get_form(self, form_class=None):
        form = super().get_form(form_class)
        form.fields['category'].queryset = self.request.user.categories.all()
        return form


class TaskDeleteView(LoginRequiredMixin, DeleteView):
    model = Task
    template_name = 'tasks/task_confirm_delete.html'
    success_url = reverse_lazy('tasks:task_list')

    def get_queryset(self):
        return Task.objects.filter(created_by=self.request.user)


class TaskStatusUpdateView(LoginRequiredMixin, View):
    def post(self, request, pk):
        task = get_object_or_404(Task, pk=pk, created_by=request.user)
        new_status = request.POST.get('status')
        if new_status in dict(Task.STATUS_CHOICES):
            task.status = new_status
            task.save()
        return redirect('tasks:task_detail', pk=pk)
```

### Step 3：URL を追加する

`tasks/urls.py` を更新する：

```python
from django.urls import path
from .views import (
    TaskListView, TaskDetailView,
    TaskCreateView, TaskUpdateView, TaskDeleteView,
    TaskStatusUpdateView,
)

app_name = 'tasks'

urlpatterns = [
    path('', TaskListView.as_view(), name='task_list'),
    path('<int:pk>/', TaskDetailView.as_view(), name='task_detail'),
    path('create/', TaskCreateView.as_view(), name='task_create'),
    path('<int:pk>/update/', TaskUpdateView.as_view(), name='task_update'),
    path('<int:pk>/delete/', TaskDeleteView.as_view(), name='task_delete'),
    path('<int:pk>/status/', TaskStatusUpdateView.as_view(), name='task_status_update'),
]
```

### Step 4：テンプレートを作成する

`templates/tasks/task_form.html`（作成・編集で共用）：

```html
{% extends 'base.html' %}

{% block content %}
<h2>{% if object %}タスクを編集{% else %}新規タスク{% endif %}</h2>

<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit" class="btn btn-primary">保存</button>
    <a href="{% url 'tasks:task_list' %}" class="btn btn-secondary">キャンセル</a>
</form>
{% endblock %}
```

`templates/tasks/task_confirm_delete.html`：

```html
{% extends 'base.html' %}

{% block content %}
<div class="card border-danger">
    <div class="card-header bg-danger text-white">タスクの削除</div>
    <div class="card-body">
        <p>「{{ object.title }}」を削除してもよいですか？この操作は取り消せません。</p>
        <form method="post">
            {% csrf_token %}
            <button type="submit" class="btn btn-danger">削除する</button>
            <a href="{% url 'tasks:task_detail' object.pk %}" class="btn btn-secondary">キャンセル</a>
        </form>
    </div>
</div>
{% endblock %}
```

また、`templates/tasks/task_list.html` のヘッダ部分を以下に書き換えて「+ 新規タスク」ボタンを追加する：

```html
<div class="d-flex justify-content-between align-items-center mb-3">
    <h2>タスク一覧</h2>
    <a href="{% url 'tasks:task_create' %}" class="btn btn-primary">+ 新規タスク</a>
</div>
```

### Step 5：詳細ページに編集・削除ボタンとステータス変更フォームを追加する

`templates/tasks/task_detail.html` の `card-header` を以下に書き換えて編集・削除ボタンを追加する：

```html
<div class="card-header d-flex justify-content-between">
    <h3>{{ task.title }}</h3>
    <div>
        <a href="{% url 'tasks:task_update' task.pk %}" class="btn btn-sm btn-outline-secondary">編集</a>
        <a href="{% url 'tasks:task_delete' task.pk %}" class="btn btn-sm btn-outline-danger">削除</a>
    </div>
</div>
```

続いて、同テンプレートの `card-body` の `<dl>` 内に追加する：

```html
<dt class="col-sm-3">ステータス変更</dt>
<dd class="col-sm-9">
    <form method="post" action="{% url 'tasks:task_status_update' task.pk %}" class="d-inline">
        {% csrf_token %}
        <select name="status" class="form-select form-select-sm d-inline w-auto">
            {% for value, label in task.STATUS_CHOICES %}
            <option value="{{ value }}" {% if task.status == value %}selected{% endif %}>
                {{ label }}
            </option>
            {% endfor %}
        </select>
        <button type="submit" class="btn btn-sm btn-outline-primary">変更</button>
    </form>
</dd>
```

### Step 6：動作確認

1. `http://localhost:8000/tasks/create/` → フォームが表示されタスクを作成できる
2. 作成後に一覧ページにリダイレクトされ、新しいタスクが表示される
3. 詳細ページの「編集」からタイトルを変更 → 詳細ページに戻り変更が反映される
4. 「削除」から確認画面 → 削除実行 → 一覧からタスクが消える
5. ステータスをセレクトボックスで変更 → バッジの色が変わる

### Step 7：PR を作成してマージする

```bash
git add tasks/ templates/
git commit -m "feat: タスクCRUDとステータス変更機能を実装"
git push origin feature/task-crud
```

PR の説明に Day 5〜7 の変更内容をまとめて記載する：

```markdown
## 概要
タスク管理機能（CRUD・ステータス管理）を実装しました。

## 変更内容
- Task・Category モデルを追加（Day 5）
- タスク一覧・詳細ビューを実装（Day 6）
- タスク作成・編集・削除・ステータス変更を実装（Day 7）

## 確認方法
1. ログイン後に /tasks/create/ からタスクを作成
2. 一覧に表示されることを確認
3. 編集・削除・ステータス変更が動作することを確認
```

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `ImproperlyConfigured: No URL to redirect to` | `success_url` が未設定 | `success_url` または `get_success_url()` を実装する |
| フォーム送信後に同じページに戻る | バリデーションエラー | テンプレートに `{{ form.errors }}` を追加してエラー内容を確認する |
| カテゴリのセレクトボックスに全ユーザーのカテゴリが表示される | `get_form()` の絞り込みが未実装 | `form.fields['category'].queryset` を設定する |
| ステータス変更が効かない | `select` の `name` 属性が `status` でない | HTML の `name="status"` と `request.POST.get('status')` が一致しているか確認する |

---

## この日の GitHub チェックポイント

> 詳細なルールは [github_flow.md](github_flow.md) を参照。

### いつブランチを切るか

Day 5 から継続の `feature/task-crud` ブランチを使う。

### いつコミットするか

| タイミング | コミットの意味 | 理由 |
|-----------|--------------|------|
| 作成フォームから保存できたとき | CreateView 完成 | フォーム表示→バリデーション→保存→リダイレクトの一連が動いた状態 |
| 編集・削除が動いたとき | UpdateView・DeleteView 完成 | CRUD の「U・D」が完成した状態 |
| ステータス変更が動いたとき | ステータス管理完成 | Day 5〜7 の全機能が揃った状態 |

### コミットメッセージ例

```bash
git commit -m "feat: タスク作成フォームとCreateViewを実装"
git commit -m "feat: タスク編集・削除ビューを実装"
git commit -m "feat: タスクステータス変更機能を追加"
```

### いつ PR をマージするか

**条件**：タスクの作成・編集・削除・ステータス変更がすべてブラウザで動作確認済みであること。

**理由**：Day 8 はカテゴリ機能を追加するが、タスクフォームにカテゴリ選択を組み込む必要がある。CRUD が完成した安定した `main` をベースにしないと、カテゴリの実装中に CRUD の問題と混在してしまう。
