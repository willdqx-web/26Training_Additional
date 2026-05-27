# Day 10 — README・テスト・コード整理・最終確認・提出準備

## この日のゴール

- README の手順通りにゼロから環境を起動できる
- `python manage.py test` が通る
- 最終チェックリストをすべて満たし、提出できる状態になる

---

## 午前：README・テスト・コード整理

### 1. 良い README の構成

README はプロジェクトの「玄関」。技術審査では **README を最初に読む**。以下の構成を基本とする。

```
# プロジェクト名
概要（1〜2行で何をするアプリかを説明）

## 機能一覧
## 使用技術
## 開発環境の起動方法
## 環境変数
## テストの実行方法
（スクリーンショット）
```

**審査者が「使える」と判断するポイント**：`git clone` から `http://localhost:8000` にアクセスできるまで、README の手順通りにたどり着けること。手順に不備があると「ドキュメントを書けない人」という印象になる。

---

### 2. Django TestCase の書き方

```python
from django.test import TestCase
from django.contrib.auth import get_user_model
from .models import Task

User = get_user_model()


class TaskModelTest(TestCase):

    def setUp(self):
        # テストの前に毎回実行される。テスト用データを用意する
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )

    def test_task_creation(self):
        # タスクが正しく作成されるか
        task = Task.objects.create(
            title='テストタスク',
            created_by=self.user
        )
        self.assertEqual(task.title, 'テストタスク')
        self.assertEqual(task.status, 'todo')   # デフォルト値の確認

    def test_task_str(self):
        # __str__ が正しく動作するか
        task = Task.objects.create(title='会議準備', created_by=self.user)
        self.assertEqual(str(task), '会議準備')

    def test_status_choices(self):
        # ステータス変更が正しく動作するか
        task = Task.objects.create(title='テスト', created_by=self.user)
        task.status = 'in_progress'
        task.save()
        task.refresh_from_db()
        self.assertEqual(task.status, 'in_progress')
```

**`setUp` と `tearDown`**

| メソッド | タイミング | 用途 |
|---------|-----------|------|
| `setUp` | 各テストメソッドの前 | テスト用データの作成 |
| `tearDown` | 各テストメソッドの後 | クリーンアップ（通常は不要） |
| `setUpTestData` | クラス全体で1回だけ | 変更しないデータの作成（高速化） |

テストデータは各テスト終了後に自動でロールバックされる（DB に残らない）。

---

### 3. flake8 によるコード品質チェック

```bash
docker compose exec web flake8 .
```

よくある警告と対処：

| 警告 | 意味 | 対処 |
|------|------|------|
| `E501` | 1行が79文字を超えている | 行を分割する |
| `F401` | import したが使っていない | 不要な import を削除する |
| `W293` | 空白行に余分なスペース | 末尾スペースを削除する |
| `E302` | 関数・クラスの前に2行空けていない | 空行を追加する |

`setup.cfg` または `pyproject.toml` で除外設定できる：

```ini
# setup.cfg
[flake8]
max-line-length = 119
exclude = migrations
```

---

### ハンズオン（午前）

#### Step 1：ブランチ作成

```bash
git checkout -b feature/readme-tests
```

#### Step 2：requirements.txt に flake8 を追加

```
Django==5.0.6
psycopg2-binary==2.9.9
python-dotenv==1.0.1
flake8==7.0.0
```

```bash
docker compose build  # イメージを再ビルド
```

#### Step 3：テストを追加

`tasks/tests.py`：

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
        self.assertEqual(response.status_code, 302)  # リダイレクト
        self.assertTrue(Task.objects.filter(title='新しいタスク').exists())
```

テストを実行：

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

#### Step 4：flake8 でコード品質チェック

```bash
docker compose exec web flake8 . --exclude=migrations
```

警告が出た場合は修正する。

#### Step 5：README.md を作成

以下のテンプレートを参考に記述する。

```markdown
# TaskBoard

Django + PostgreSQL + Docker で構築したタスク管理アプリです。

## 機能一覧

- ユーザー登録・ログイン・ログアウト
- タスクの作成・編集・削除
- ステータス管理（未着手 / 進行中 / 完了）
- カテゴリによる分類・フィルタリング
- キーワード検索・ステータス絞り込み
- ページネーション（10件/ページ）

## 使用技術

| 技術 | バージョン |
|------|-----------|
| Python | 3.12 |
| Django | 5.0.6 |
| PostgreSQL | 16 |
| Docker | - |
| Bootstrap | 5.3 |

## 開発環境の起動方法

### 前提条件

- Docker Desktop がインストールされていること

### 手順

1. リポジトリをクローン

   git clone https://github.com/<username>/taskboard.git
   cd taskboard

2. 環境変数ファイルを作成

   .env.example をコピーして .env を作成し、必要に応じて値を変更してください。

3. コンテナを起動

   docker compose up --build

4. マイグレーションを実行

   docker compose exec web python manage.py migrate

5. スーパーユーザーを作成（管理画面用）

   docker compose exec web python manage.py createsuperuser

6. ブラウザで開く

   http://localhost:8000

## 環境変数

| 変数名 | 説明 | 例 |
|--------|------|-----|
| SECRET_KEY | Django の秘密鍵 | - |
| DEBUG | デバッグモード | True |
| DB_NAME | DB 名 | taskboard |
| DB_USER | DB ユーザー | taskboard_user |
| DB_PASSWORD | DB パスワード | password |
| DB_HOST | DB ホスト | db |
| DB_PORT | DB ポート | 5432 |

## テストの実行方法

   docker compose exec web python manage.py test

## 開発フロー

このプロジェクトは GitHub Flow で開発しています。
各機能は feature ブランチで開発し、PR を経由して main にマージしています。
```

#### Step 6：.env.example を最新化

現在の設定に合わせて更新する：

```
SECRET_KEY=your-secret-key-here
DEBUG=True
DB_NAME=taskboard
DB_USER=taskboard_user
DB_PASSWORD=password
DB_HOST=db
DB_PORT=5432
```

#### Step 7：PR 作成・マージ

```bash
git add .
git commit -m "docs: README追加・テスト追加・コード品質チェック"
git push origin feature/readme-tests
```

---

## 午後：最終確認・提出準備

### 4. ゼロ起動テストとは

「他人が自分のリポジトリを初めてクローンしたとき、README の手順だけで起動できるか」を自分で確認する。

**なぜ重要か**：技術審査担当者は README を見ながら実際に起動しようとする。起動できなければ審査終了。

---

### ハンズオン（午後）

#### Step 8：不要ブランチを削除

```bash
# ローカルのマージ済みブランチを確認
git branch --merged main

# 削除（main と現在のブランチ以外）
git branch -d feature/docker-setup
git branch -d feature/django-init
# ... 必要に応じて繰り返す
```

GitHub のリモートブランチも削除（PR マージ時に自動削除の設定をしている場合は不要）。

#### Step 9：ゼロ起動テスト

```bash
# 別ディレクトリにクローン
cd /tmp
git clone https://github.com/<username>/taskboard.git taskboard-test
cd taskboard-test

# README の手順通りに実行
cp .env.example .env
docker compose up --build -d
docker compose exec web python manage.py migrate
docker compose exec web python manage.py createsuperuser
```

ブラウザで `http://localhost:8000` を開き、動作を確認する。確認後はクリーンアップ：

```bash
docker compose down -v
cd ..
rm -rf taskboard-test
```

#### Step 10：最終チェックリスト

すべての項目を確認してから提出する。

- [ ] `docker compose up` で起動できる
- [ ] `http://localhost:8000` でアプリにアクセスできる
- [ ] ユーザー登録 → ログイン → ログアウトのフローが動作する
- [ ] タスクの作成・編集・削除・ステータス変更が動作する
- [ ] キーワード検索・ステータスフィルタが動作する
- [ ] カテゴリ作成・フィルタリングが動作する
- [ ] GitHub に機能ごとのブランチ・PR が残っている
- [ ] README に起動手順が書かれている
- [ ] `.env` が `.gitignore` されており、`.env.example` がある
- [ ] `python manage.py test` が全件通る
- [ ] PR のコメント・説明が適切に記載されている

#### Step 11：GitHub リポジトリの最終確認

GitHub を開いて以下を確認する：

1. **Insights > Network**：ブランチの分岐・マージが確認できる
2. **Pull requests（Closed）**：各機能の PR 一覧が残っている
3. **Actions**（設定している場合）：CI が通っている

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| テストで `AssertionError: 302 != 200` | ログインが必要なページにログインせずアクセス | `setUp` で `self.client.login()` を呼ぶ |
| `django.db.utils.ProgrammingError` | テスト DB のマイグレーション失敗 | `python manage.py migrate` を実行してから再テスト |
| ゼロ起動テストで `migrate` エラー | `.env` の DB 設定が間違っている | `.env.example` の内容が正確か確認 |
| flake8 で `E501` が大量に出る | 行が長い | `setup.cfg` で `max-line-length = 119` に設定する |

---

## この日の GitHub チェックポイント

> 詳細なルールは [github_flow.md](github_flow.md) を参照。

### いつブランチを切るか

**タイミング**：Day 9 の `feature/search-filter` がマージされた後、`main` から `feature/readme-tests` を切る。

**理由**：README・テスト・コード整理は「機能追加」ではなく「品質担保とドキュメント化」。機能実装のブランチと分けることで、「この PR は機能追加ゼロ、品質改善のみ」という意図が PR の差分から一目で分かる。

### いつコミットするか

| タイミング | コミットの意味 | 理由 |
|-----------|--------------|------|
| テストが通ったとき | テストコード追加完了 | `python manage.py test` が通った状態でコミット。テストは「通った状態」でコミットするのが原則 |
| flake8 の警告をすべて解消したとき | コード品質整理完了 | 警告が残ったままコミットすると「修正途中」の状態が履歴に残る |
| README を書き終えたとき | ドキュメント完成 | README は1つのドキュメントとして完成したときにコミット |

### コミットメッセージ例

```bash
git commit -m "test: TaskモデルとビューのユニットテストをTestCaseで追加"
git commit -m "style: flake8警告をすべて解消"
git commit -m "docs: READMEに起動手順と機能一覧を追加"
```

### いつ PR をマージするか

**条件**：以下をすべて満たすこと。
1. `python manage.py test` が全件グリーン
2. README の手順通りに別ディレクトリでゼロ起動テストが通る
3. 最終チェックリストが全項目クリア

**理由**：これが最後の PR。ここでマージされた `main` が提出物になる。ゼロ起動テストで「README の手順が正しい」ことを自分で確認してからマージしないと、審査担当者が起動できない提出物になってしまう。マージ前の最終確認が最も重要なタイミング。

---

## GitHub 履歴の最終チェック

提出前に GitHub の以下を確認する。

### Closed PR の一覧（期待される構成）

| PR タイトル | ブランチ | 対応日 |
|-----------|---------|--------|
| `feat: Docker開発環境を構築` | `feature/docker-setup` | Day 1 |
| `feat: DjangoプロジェクトをPostgreSQL接続で初期化` | `feature/django-init` | Day 3 午前 |
| `feat: 認証フロー（登録・ログイン・ログアウト）を完成` | `feature/user-auth` | Day 3〜4 |
| `feat: タスク管理機能（CRUD・ステータス管理）を実装` | `feature/task-crud` | Day 5〜7 |
| `feat: カテゴリ機能とUIコンポーネント分割を実装` | `feature/category` | Day 8 |
| `feat: 検索・フィルタリング・ページネーションを実装` | `feature/search-filter` | Day 9 |
| `docs: READMEとテストを追加` | `feature/readme-tests` | Day 10 |

この構成が揃っていると「機能単位でブランチを切り、完成したらマージする」GitHub Flow が実践できていることを審査担当者にアピールできる。
