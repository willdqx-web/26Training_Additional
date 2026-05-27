# Day 7 — タスク作成・編集・削除・ステータス変更

## この日のゴール

- タスクの作成・編集・削除が動作する
- ステータスを「未着手 → 進行中 → 完了」と切り替えられる

---

## 1. ModelForm とは

Model の定義からフォームを自動生成する仕組み。バリデーションも Model の定義に基づいて自動で行われる。

```python
class TaskForm(ModelForm):
    class Meta:
        model = Task
        fields = ['title', 'description', 'status', 'due_date', 'category']
```

`fields` に書いたフィールドだけがフォームに表示される。`created_by` のような「ユーザーが操作すべきでないフィールド」は含めない。

---

## 2. CreateView / UpdateView / DeleteView の仕組み

```python
class TaskCreateView(LoginRequiredMixin, CreateView):
    model = Task
    form_class = TaskForm
    template_name = 'tasks/task_form.html'
    success_url = reverse_lazy('tasks:task_list')

    def form_valid(self, form):
        # フォームが valid なとき、保存前に created_by をセット
        form.instance.created_by = self.request.user
        return super().form_valid(form)
```

**`form_valid()` をオーバーライドする理由**：`created_by` はフォームに含めていないので、ビュー側で自動セットする必要がある。

```python
class TaskUpdateView(LoginRequiredMixin, UpdateView):
    model = Task
    form_class = TaskForm
    template_name = 'tasks/task_form.html'

    def get_success_url(self):
        return reverse_lazy('tasks:task_detail', kwargs={'pk': self.object.pk})

    def get_queryset(self):
        # 自分のタスクだけ編集可能
        return Task.objects.filter(created_by=self.request.user)
```

```python
class TaskDeleteView(LoginRequiredMixin, DeleteView):
    model = Task
    template_name = 'tasks/task_confirm_delete.html'
    success_url = reverse_lazy('tasks:task_list')

    def get_queryset(self):
        return Task.objects.filter(created_by=self.request.user)
```

---

## 3. ステータス変更：カスタムビュー

CBV の汎用ビューに当てはまらない操作は `View` を直接継承して書く。

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

**POST のみ受け付ける理由**：GET でステータスを変更できると、リンクを埋め込んだメールやページを踏むだけで変更されるリスクがある（CSRF 対策の観点）。

---

## 4. ハンズオン

### Step 1：フォームを作成

`tasks/forms.py`（新規作成）：

```python
from django import forms
from .models import Task


class TaskForm(forms.ModelForm):
    class Meta:
        model = Task
        fields = ['title', 'description', 'status', 'due_date', 'category', 'assigned_to']
        widgets = {
            'due_date': forms.DateInput(attrs={'type': 'date'}),
            'description': forms.Textarea(attrs={'rows': 4}),
        }
```

`widgets` でフォームの HTML 属性を制御できる。`type='date'` でブラウザのネイティブな日付ピッカーが使えるようになる。

### Step 2：ビューを追加

`tasks/views.py` に追加：

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

### Step 3：URL を追加

`tasks/urls.py` を更新：

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

### Step 4：テンプレートを作成

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

### Step 5：詳細ページにステータス変更フォームを追加

`templates/tasks/task_detail.html` の `card-body` 内に追加：

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

1. 一覧ページの「+ 新規タスク」から作成 → 一覧に戻り新しいタスクが表示される
2. 詳細ページの「編集」からタイトル変更 → 詳細ページに戻り変更が反映される
3. 「削除」から確認画面 → 削除実行 → 一覧から消える
4. ステータスをセレクトボックスで変更 → バッジの色が変わる

### Step 7：PR 作成・マージ

```bash
git add tasks/
git commit -m "feat: タスクCRUD・ステータス変更を実装"
git push origin feature/task-crud
```

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `ImproperlyConfigured: No URL to redirect to` | `success_url` が未設定 | `success_url` または `get_success_url()` を実装する |
| フォーム送信後に同じページに戻る | バリデーションエラー | テンプレートに `{{ form.errors }}` を追加してエラー内容を確認 |
| カテゴリのセレクトボックスに全ユーザーのカテゴリが出る | `get_form()` の絞り込みが未実装 | `form.fields['category'].queryset` を設定する |
| ステータス変更が効かない | form の `name` 属性が `status` でない | HTML の `name="status"` を確認 |

---

## この日の GitHub チェックポイント

> 詳細なルールは [github_flow.md](github_flow.md) を参照。

### いつブランチを切るか

**タイミング**：Day 5 から継続の `feature/task-crud` ブランチを使う。新しいブランチは切らない。

**理由**：タスクの CRUD は一覧・詳細・作成・編集・削除・ステータス変更がすべて揃って初めて「タスク管理機能」として完成する。途中でブランチを分割すると「機能が半完成の状態の PR」が複数生まれてしまう。

### いつコミットするか

| タイミング | コミットの意味 | 理由 |
|-----------|--------------|------|
| 作成フォームから保存できたとき | CreateView 完成 | フォーム → バリデーション → DB 保存 → リダイレクトの一連が動いた状態 |
| 編集・削除が動いたとき | UpdateView・DeleteView 完成 | CRUD の「U・D」が完成した状態。まとめてコミットしてもよい |
| ステータス変更が動いたとき | ステータス管理完成 | カスタムビューの動作確認済み。CRUD 完成の最後のピース |

### コミットメッセージ例

```bash
git commit -m "feat: タスク作成フォームとCreateViewを実装"
git commit -m "feat: タスク編集・削除ビューを実装"
git commit -m "feat: タスクステータス変更機能を追加"

# Day 6 〜 7 のすべてをまとめてマージする場合の最終コミット後の PR タイトル例
# → "feat: タスク管理機能（CRUD・ステータス管理）を実装"
```

### いつ PR をマージするか

**条件**：タスクの作成・編集・削除・ステータス変更がすべてブラウザで動作確認済みであること。

**理由**：Day 5〜7 の3日分がこの PR に集まっている。ここでマージすると「タスク管理機能」が main に入り、Day 8 のカテゴリ機能の実装ベースとして安定した状態が得られる。中途半端な状態でマージすると「カテゴリ追加しようとしたらタスク作成が壊れていた」という状況が起きる。
