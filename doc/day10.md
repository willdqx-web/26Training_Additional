# Day 10 — README・テスト・コード整理・最終確認・提出準備

## このプロジェクト全体の流れ

```
Day 1  → Day 2  → Day 3  → Day 4  → Day 5〜7 → Day 8 → Day 9 → [Day 10]
Docker   paiza    Django   ログイン  タスク      カテゴリ 検索    提出
環境構築  学習     初期設定  認証      CRUD        UI改善  ページ  ★今日
                                                        ネーション
```

アプリの全機能が完成しました。今日は「他の人が見ても分かる・動かせる状態」に仕上げる作業です。

---

## この日のゴール

- README の手順通りにゼロから環境を起動できる
- `python manage.py test` が全件グリーンになる
- 最終チェックリストを全項目クリアし、提出できる状態になる

---

## この日の前提

- Day 9 の `feature/search-filter` が main にマージ済みであること
- すべての機能が動作する状態であること

---

## 午前：README・テスト・コード整理

### 1. README とは何か、なぜ重要か

README は GitHub リポジトリを開いたときに最初に表示されるドキュメント。技術審査では **README を最初に読んで、書かれている手順通りに起動を試みる**。

起動できなければそこで審査終了。README の質が審査結果を大きく左右する。

**良い README の条件**：
- 前提条件（Docker Desktop が必要、など）が明記されている
- 「このコマンドを実行する」という具体的な手順が番号付きで書かれている
- 手順通りにやれば必ず動く（自分でゼロ起動テストして確認済み）

### 2. Django TestCase の仕組み

テストとは「コードが期待通りに動くか自動確認する仕組み」。

```python
from django.test import TestCase
from django.contrib.auth import get_user_model
from .models import Task

User = get_user_model()


class TaskModelTest(TestCase):

    def setUp(self):
        # 各テストの前に実行される。テスト用データを準備する
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )

    def test_task_creation(self):
        task = Task.objects.create(title='テスト', created_by=self.user)
        self.assertEqual(task.status, 'todo')   # デフォルト値が'todo'であることを確認

    def test_task_str(self):
        task = Task.objects.create(title='会議準備', created_by=self.user)
        self.assertEqual(str(task), '会議準備')  # __str__ の動作を確認
```

**重要なポイント**：
- テストは毎回クリーンな DB で実行される（テストデータは本物の DB に影響しない）
- `setUp()` で作成したデータは各テストの後に自動でロールバックされる
- テストメソッドは `test_` で始める名前にする

### 3. flake8 によるコード品質チェック

flake8 は「コードが Python の規約（PEP8）に沿っているか」を自動チェックするツール。

```bash
docker compose exec web flake8 . --exclude=migrations
```

よくある警告：

| 警告コード | 意味 | 対処 |
|-----------|------|------|
| `E501` | 1行が79文字を超えている | 行を分割する、または `max-line-length = 119` に設定 |
| `F401` | import したが使っていない | 不要な import を削除する |
| `W293` | 空白行に余分なスペースがある | 末尾のスペースを削除する |
| `E302` | 関数・クラスの前に2行空けていない | 空行を追加する |

### ハンズオン（午前）

#### Step 1：ブランチを作成する

```bash
git checkout main
git pull origin main
git checkout -b feature/readme-tests
```

#### Step 2：requirements.txt に flake8 を追加する

```
Django==5.0.6
psycopg2-binary==2.9.9
python-dotenv==1.0.1
flake8==7.0.0
```

変更を反映するためにイメージを再ビルドする：

```bash
docker compose build
```

#### Step 3：setup.cfg を作成する

flake8 の設定ファイルをプロジェクトルートに作成する：

```ini
[flake8]
max-line-length = 119
exclude = migrations,__pycache__
```

#### Step 4：テストを追加する

`tasks/tests.py` を以下の内容に書き換える：

```python
from django.test import TestCase, Client
from django.urls import reverse
from django.contrib.auth import get_user_model
from .models import Category, Task

User = get_user_model()


class TaskModelTest(TestCase):

    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )

    def test_task_creation_with_defaults(self):
        task = Task.objects.create(title='テスト', created_by=self.user)
        self.assertEqual(task.status, 'todo')
        self.assertIsNone(task.due_date)

    def test_task_str(self):
        task = Task.objects.create(title='会議準備', created_by=self.user)
        self.assertEqual(str(task), '会議準備')

    def test_status_display(self):
        task = Task.objects.create(title='テスト', created_by=self.user, status='in_progress')
        self.assertEqual(task.get_status_display(), '進行中')

    def test_task_update_status(self):
        task = Task.objects.create(title='テスト', created_by=self.user)
        task.status = 'done'
        task.save()
        task.refresh_from_db()
        self.assertEqual(task.status, 'done')


class TaskViewTest(TestCase):

    def setUp(self):
        self.client = Client()
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        self.client.login(username='testuser', password='testpass123')

    def test_task_list_requires_login(self):
        self.client.logout()
        response = self.client.get(reverse('tasks:task_list'))
        self.assertRedirects(response, '/accounts/login/?next=/tasks/')

    def test_task_list_shows_own_tasks(self):
        Task.objects.create(title='自分のタスク', created_by=self.user)
        other_user = User.objects.create_user(username='other', password='pass')
        Task.objects.create(title='他人のタスク', created_by=other_user)

        response = self.client.get(reverse('tasks:task_list'))
        self.assertContains(response, '自分のタスク')
        self.assertNotContains(response, '他人のタスク')

    def test_task_create(self):
        response = self.client.post(reverse('tasks:task_create'), {
            'title': '新しいタスク',
            'status': 'todo',
        })
        self.assertEqual(response.status_code, 302)   # リダイレクトされる
        self.assertTrue(Task.objects.filter(title='新しいタスク').exists())
```

テストを実行する：

```bash
docker compose exec web python manage.py test tasks
```

成功すると以下のように表示される：

```
Found 6 test(s).
Creating test database for alias 'default'...
......
----------------------------------------------------------------------
Ran 6 tests in 0.123s

OK
```

#### Step 5：flake8 でコード品質チェックをする

```bash
docker compose exec web flake8 . --exclude=migrations
```

警告が出た場合は修正する。警告ゼロになったことを確認してからコミットする。

#### Step 6：README.md を作成する

`README.md` を以下の構成で書く（`taskboard/README.md` を更新）：

````markdown
# TaskBoard

Django + PostgreSQL + Docker で構築したタスク管理アプリです。

## 機能一覧

- ユーザー登録・ログイン・ログアウト
- タスクの作成・編集・削除・ステータス管理
- カテゴリによる分類・フィルタリング
- キーワード検索・ステータス絞り込み
- ページネーション（10件/ページ）
- 操作完了後のフラッシュメッセージ

## 使用技術

| 技術 | バージョン |
|------|-----------|
| Python | 3.12 |
| Django | 5.0.6 |
| PostgreSQL | 16 |
| Docker | - |
| Bootstrap | 5.3 |

## 前提条件

以下がインストール済みであること：
- Docker Desktop

## 起動方法

**1. リポジトリをクローン**

```bash
git clone git@github.com:<username>/taskboard.git
cd taskboard
```

**2. 環境変数ファイルを作成**

```bash
cp .env.example .env
```

**3. コンテナを起動**

```bash
docker compose up --build
```

**4. マイグレーションを実行**

```bash
docker compose exec web python manage.py migrate
```

**5. 管理画面用スーパーユーザーを作成**

```bash
docker compose exec web python manage.py createsuperuser
```

**6. ブラウザで開く**

http://localhost:8000

## テストの実行

```bash
docker compose exec web python manage.py test
```

## 環境変数

`.env.example` を参照してください。

## 開発フロー

GitHub Flow を採用しています。機能ごとに `feature/xxx` ブランチを作成し、PR を経由して main にマージしています。
````

#### Step 7：.env.example を最新化する

```
SECRET_KEY=your-secret-key-here
DEBUG=True
DB_NAME=taskboard
DB_USER=taskboard_user
DB_PASSWORD=password
DB_HOST=db
DB_PORT=5432
```

#### Step 8：config/urls.py にルートURLを追加する

`http://localhost:8000`（ルート `/`）にアクセスしたとき 404 にならないよう、タスク一覧へのリダイレクトを追加する。

`config/urls.py` を以下の内容に更新する：

```python
from django.contrib import admin
from django.urls import path, include
from django.views.generic import RedirectView

urlpatterns = [
    path('', RedirectView.as_view(url='/tasks/', permanent=False)),
    path('admin/', admin.site.urls),
    path('accounts/', include('accounts.urls')),
    path('tasks/', include('tasks.urls')),
]
```

ブラウザで `http://localhost:8000` を開き、ログインページまたはタスク一覧にリダイレクトされることを確認する。

#### Step 9：コミットしてマージする

```bash
git add .
git commit -m "docs: READMEとテストを追加、flake8警告を解消、ルートURLを追加"
git push origin feature/readme-tests
```

---

## 午後：最終確認・提出準備

### 4. ゼロ起動テストとは

「他の人が自分のリポジトリを初めてクローンして、README 通りに起動できるか」を自分で確認する作業。

**なぜ必要か**：自分の環境には既に Docker イメージや DB データが残っているため、自分の環境では動いても「初めての人」の環境では動かないことがある。別ディレクトリにクローンすることで「初めての人」の状態を再現する。

### ハンズオン（午後）

#### Step 10：不要ブランチを削除する

```bash
# マージ済みのローカルブランチを確認
git branch --merged main

# 削除（main と現在のブランチ以外）
git branch -d feature/docker-setup
git branch -d feature/django-init
git branch -d feature/user-auth
git branch -d feature/task-crud
git branch -d feature/category
git branch -d feature/search-filter
git branch -d feature/readme-tests
```

#### Step 11：ゼロ起動テストを実行する

```bash
# 別ディレクトリにクローン（/tmp は Windows だと C:\temp など）
cd C:\temp
git clone git@github.com:<username>/taskboard.git taskboard-test
cd taskboard-test

# README の手順通りに実行する
copy .env.example .env
docker compose up --build -d
docker compose exec web python manage.py migrate
docker compose exec web python manage.py createsuperuser
```

ブラウザで `http://localhost:8000` を開いて動作確認する。
（ログインページまたはタスク一覧が表示されれば成功。）

完了後はクリーンアップする：

```bash
docker compose down -v
cd ..
rm -rf taskboard-test   # Windowsは del /s /q taskboard-test
```

#### Step 12：最終チェックリスト

提出前にすべての項目を確認する。

- [ ] `docker compose up` で起動できる
- [ ] `http://localhost:8000` でアプリにアクセスできる
- [ ] ユーザー登録 → ログイン → ログアウトのフローが動作する
- [ ] タスクの作成・編集・削除・ステータス変更が動作する
- [ ] キーワード検索・ステータスフィルタが動作する
- [ ] カテゴリ作成・フィルタリングが動作する
- [ ] タスクが10件以上あるときページネーションが表示される
- [ ] GitHub に機能ごとのブランチ・PR が残っている（Closed PR 一覧で確認）
- [ ] README に起動手順が書かれている（ゼロ起動テストで確認済み）
- [ ] `.env` が `.gitignore` されており、`.env.example` がある
- [ ] `python manage.py test` が全件通る

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| テストで `AssertionError: 302 != 200` | ログインが必要なページにログインせずアクセス | `setUp` で `self.client.login()` を呼んでいるか確認 |
| `django.db.utils.ProgrammingError` | テスト DB のマイグレーション失敗 | `python manage.py migrate` を実行してから再テスト |
| ゼロ起動テストで migrate エラー | `.env` の DB 設定が間違っている | `.env.example` の内容を `README` と一致させる |
| flake8 で `E501` が大量に出る | 行が長い | `setup.cfg` で `max-line-length = 119` に設定する |

---

## この日の GitHub チェックポイント

> 詳細なルールは [github_flow.md](github_flow.md) を参照。

### いつブランチを切るか

Day 9 のマージ後の `main` から `feature/readme-tests` を切る。

**理由**：README・テスト・コード整理は「機能追加」ではなく「品質担保とドキュメント化」。機能実装のブランチと分けることで、「この PR は機能追加ゼロ、品質改善のみ」という意図が PR の差分から一目で分かる。

### いつコミットするか

| タイミング | コミットの意味 | 理由 |
|-----------|--------------|------|
| テストが全件通ったとき | テストコード追加完了 | テストは「通った状態」でコミットするのが原則。失敗している状態のテストをコミットしない |
| flake8 の警告をゼロにしたとき | コード品質整理完了 | 警告が残ったままコミットすると「修正途中」の状態が履歴に残る |
| README が完成したとき | ドキュメント完成 | ゼロ起動テストで動作を確認してからコミットする |

### コミットメッセージ例

```bash
git commit -m "test: TaskモデルとビューのユニットテストをTestCaseで追加"
git commit -m "style: flake8警告をすべて解消"
git commit -m "docs: READMEに起動手順と機能一覧を追加"
```

### いつ PR をマージするか

**条件**：ゼロ起動テストが通り、最終チェックリストが全項目クリアであること。

**理由**：これが最後の PR。ここでマージされた `main` が提出物になる。ゼロ起動テストで「README の手順が正しい」ことを自分で確認してからマージする。

---

## GitHub 履歴の最終確認

GitHub の Closed PR 一覧を開いて、以下の構成が揃っていることを確認する。

| PR タイトル | ブランチ | 対応日 |
|-----------|---------|--------|
| `feat: Docker開発環境を構築` | `feature/docker-setup` | Day 1 |
| `feat: DjangoプロジェクトをPostgreSQL接続で初期化` | `feature/django-init` | Day 3 午前 |
| `feat: 認証フロー（登録・ログイン・ログアウト）を完成` | `feature/user-auth` | Day 3〜4 |
| `feat: タスク管理機能（CRUD・ステータス管理）を実装` | `feature/task-crud` | Day 5〜7 |
| `feat: カテゴリ機能とUIコンポーネント分割を実装` | `feature/category` | Day 8 |
| `feat: 検索・フィルタリング・ページネーションを実装` | `feature/search-filter` | Day 9 |
| `docs: READMEとテストを追加` | `feature/readme-tests` | Day 10 |

この PR 一覧が「機能単位でブランチを切り、完成したらマージした証跡」として審査担当者に示す GitHub Flow の実践例になる。
