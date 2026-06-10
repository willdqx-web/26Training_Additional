# TASKS.md — 10日間タスクブレークダウン

## 凡例

- **学習目標**：その日終了時点で「できるようになっていること」
- **ハンズオン課題**：実際に手を動かすタスク
- **完了条件**：PR が main にマージされ、動作確認済み
- 見積は6〜8時間/日のフルコミットを前提

---

## Day 1 — Docker 開発環境構築

**学習目標**
- `docker compose up` で Django + PostgreSQL が起動できる
- Dockerfile と docker-compose.yml の役割を説明できる

**ハンズオン課題**

- [ ] GitHub リポジトリ `taskboard` を作成（README.md 付き）
- [ ] `develop` ブランチを作成し、以降の開発ベースにする（運用上 main に直接 PR でも可）
- [ ] ブランチ `feature/docker-setup` を切る
- [ ] `Dockerfile` を作成（Python 3.12, 必要パッケージのインストール）
- [ ] `docker-compose.yml` を作成（`web` + `db` の2サービス）
- [ ] `docker compose up` で両サービスが起動することを確認
- [ ] `docker compose exec web python --version` でコンテナ内 Python が動くことを確認
- [ ] PR を作成し、自己レビューコメントを1件以上残してマージ

**完了条件**
- `docker compose up` が成功し、`db` サービスが PostgreSQL として応答する

---

## Day 2 — paiza 学習：Django入門編①②（終日インプット）

**学習目標**
- Django の MVT アーキテクチャ・URLconf・テンプレートを説明できる
- Django シェルで ORM クエリを実行できる
- マイグレーションの仕組みと手順を説明できる
- DB への書き込み・削除の流れを理解できる

### 午前（約3時間）— 入門編①：Djangoの基本を理解しよう

| チャプター | 内容 |
|-----------|------|
| Ch1 | Django とは |
| Ch2 | 最初のプロジェクトを作ろう（演習×2） |
| Ch3 | Hello World（演習×3） |
| Ch4 | テンプレートを追加しよう（演習×2） |
| Ch5 | テンプレートにデータを渡そう（演習×2） |
| Ch6 | モデルを作ろう（演習×1） |
| Ch7 | 管理サイトを使ってみよう（演習×1） |
| Ch8 | モデルのデータを一覧表示しよう（演習×2） |
| Ch9 | データの詳細を表示しよう その1 |
| Ch10 | データの詳細を表示しよう その2 |

- [ ] Ch1〜Ch5 を受講する（URL・ビュー・テンプレートの基本）
- [ ] Ch6〜Ch8 を受講する（モデル・管理画面・一覧表示）
- [ ] Ch9〜Ch10 を受講する（詳細ビュー）

### 午後（約3〜4時間）— 入門編②：Djangoの動作を理解しよう

| チャプター | 内容 |
|-----------|------|
| Ch1 | Django の機能構成を理解しよう |
| Ch2 | Django シェルでデータベースを確認しよう（演習×1） |
| Ch3 | データベースへのマイグレーションを理解しよう（演習×1） |
| Ch4 | テンプレートにカラムを追加しよう（演習×1） |
| Ch5 | Web アプリのデータの流れを理解しよう（演習×1） |
| Ch6 | データベースに書き込んでみよう（演習×1） |
| Ch7 | データベースから記事を削除しよう（演習×1） |

- [ ] Ch1〜Ch3 を受講する（機能構成・Django シェル・マイグレーション）
- [ ] Ch4〜Ch7 を受講する（テンプレート拡張・データフロー・書き込み・削除）
- [ ] 入門編①②で学んだ概念（MVT・ORM・migration・Django シェル）をノートにまとめる

**完了条件**
- 入門編①②の全チャプターを受講し終える

---

## Day 3 — Django プロジェクト初期設定 ＋ ユーザー認証（モデル・登録）

**学習目標**
- Django プロジェクト・アプリの構成を理解できる（paiza ②の復習として実践）
- PostgreSQL を Django の DB として接続できる
- `AbstractUser` を拡張したカスタムユーザーモデルを作れる
- Django の認証フォームとビューの仕組みを説明できる

### 午前（約3時間）— プロジェクト初期設定

- [ ] ブランチ `feature/django-init` を切る
- [ ] `django-admin startproject config .` でプロジェクト作成
- [ ] `python manage.py startapp tasks` でアプリ作成
- [ ] `settings.py` の `DATABASES` を PostgreSQL に切り替える
- [ ] `.env` + `python-dotenv` でシークレット管理を設定する
- [ ] `.env.example` を作成し、`.env` を `.gitignore` に追加する
- [ ] `docker compose exec web python manage.py migrate` で初期マイグレーション成功を確認
- [ ] `http://localhost:8000` で Django のウェルカムページが表示されることを確認
- [ ] PR 作成・マージ

### 午後（約3〜4時間）— ユーザー認証（登録機能）

- [ ] ブランチ `feature/user-auth` を切る
- [ ] `accounts` アプリを作成する
- [ ] `AbstractUser` を継承した `CustomUser` モデルを作成する
- [ ] `settings.py` に `AUTH_USER_MODEL = 'accounts.CustomUser'` を設定する
- [ ] ユーザー登録フォーム（`UserCreationForm` ベース）を作成する
- [ ] 登録ビュー・URL を実装する（クラスベースビュー `CreateView` を使う）
- [ ] 登録フォームのテンプレートを作成する（Bootstrap 5 CDN を `base.html` に追加）
- [ ] ユーザー登録が動作することをブラウザで確認する

**完了条件**
- `migrate` 成功・ウェルカムページ表示、かつ登録フォームから新規ユーザーが DB に保存される

---

## Day 4 — ユーザー認証（ログイン・ログアウト・認可）

**学習目標**
- Django 標準の `LoginView` / `LogoutView` を使いこなせる
- `LoginRequiredMixin` でアクセス制御できる

**ハンズオン課題**

- [ ] ログインビュー・ログアウトビューを URL に接続する（Django 標準ビュー使用）
- [ ] ログインページのテンプレートを作成する
- [ ] `base.html` にナビゲーションバー（ログイン/ログアウトリンク）を追加する
- [ ] `settings.py` に `LOGIN_REDIRECT_URL` / `LOGOUT_REDIRECT_URL` を設定する
- [ ] ログイン・ログアウトの動作をブラウザで確認する
- [ ] PR に「認証フローの動作確認」スクリーンショットをコメントで追加してマージ

**完了条件**
- 登録 → ログイン → ログアウトの全フローが正常動作する

---

## Day 5 — タスクモデル・マイグレーション

**学習目標**
- Django ORM でリレーション（ForeignKey）を設計できる
- `makemigrations` / `migrate` のサイクルを説明できる

**ハンズオン課題**

- [ ] ブランチ `feature/task-crud` を切る
- [ ] `Task` モデルを SPEC.md の定義に従って実装する
- [ ] `Category` モデルを実装する
- [ ] `makemigrations` → `migrate` を実行する
- [ ] `django admin` にモデルを登録し、管理画面からデータを作成できることを確認する
- [ ] Django shell でクエリ（`Task.objects.all()` / `filter()` / `create()`）を試す

**完了条件**
- migrate 成功、管理画面からタスクの CRUD が動作する

---

## Day 6 — タスク一覧・詳細ビュー

**学習目標**
- クラスベースビュー `ListView` / `DetailView` を実装できる
- Django テンプレートの `for` ループ・`if` 分岐を使えるようになる

**ハンズオン課題**

- [ ] タスク一覧ビュー（`ListView`）を実装する
  - 自分が作成したタスクのみ表示するようフィルタリング
  - `LoginRequiredMixin` でアクセス制御
- [ ] タスク一覧テンプレートを作成する（Bootstrap のカード/テーブルレイアウト）
- [ ] タスク詳細ビュー（`DetailView`）を実装する
- [ ] URL 設計：`/tasks/`、`/tasks/<pk>/`
- [ ] ブラウザで一覧・詳細ページの動作を確認する

**完了条件**
- ログイン後、タスク一覧・詳細ページが表示される

---

## Day 7 — タスク作成・編集・削除・ステータス変更

**学習目標**
- `CreateView` / `UpdateView` / `DeleteView` を実装できる
- `ModelForm` を使ってフォームバリデーションができる

**ハンズオン課題**

- [ ] `TaskForm`（`ModelForm`）を作成する
- [ ] タスク作成ビュー（`CreateView`）を実装する
- [ ] タスク編集ビュー（`UpdateView`）を実装する
- [ ] タスク削除ビュー（`DeleteView`）を実装する（確認画面あり）
- [ ] ステータス変更機能を実装する（POST リクエストでステータスを切り替え）
- [ ] 各操作後に一覧にリダイレクトされることを確認する
- [ ] PR 作成・マージ（`feature/task-crud` → main）

**完了条件**
- タスクの作成・編集・削除・ステータス変更がすべて動作する

---

## Day 8 — カテゴリ機能・UI 改善

**学習目標**
- 関連モデルを持つフォームを扱える
- テンプレートをコンポーネント分割（`include`）できる

**ハンズオン課題**

- [ ] ブランチ `feature/category` を切る
- [ ] カテゴリ一覧・作成・削除を実装する
- [ ] タスク作成・編集フォームにカテゴリ選択を追加する
- [ ] タスク一覧でカテゴリフィルタリングを実装する
- [ ] `base.html` を整理し、`_navbar.html` を `include` で分割する
- [ ] PR 作成・マージ

**完了条件**
- カテゴリ機能が動作し、タスク一覧でカテゴリ絞り込みができる

---

## Day 9 — 検索・フィルタリング・ページネーション・フラッシュメッセージ

**学習目標**
- Django ORM の `Q` オブジェクトを使ったキーワード検索を実装できる
- クエリパラメータを使ったステータス・カテゴリフィルタリングを実装できる
- `Paginator` でページネーションを実装できる
- Django メッセージフレームワークで操作フィードバックを表示できる

### 午前（約3時間）— 検索・フィルタリング

- [ ] ブランチ `feature/search-filter` を切る
- [ ] タスク一覧にキーワード検索を追加する
  - `Q` オブジェクトでタイトル・説明文の部分一致検索
  - URL: `/tasks/?q=キーワード`
- [ ] ステータスフィルタリングを追加する
  - URL: `/tasks/?status=in_progress` などクエリパラメータで絞り込み
- [ ] 検索フォームをテンプレートに追加する（GET メソッド）
- [ ] Django シェルで `Q` オブジェクトを使ったクエリを試す
  - 例：`Task.objects.filter(Q(title__icontains='会議') | Q(description__icontains='会議'))`

### 午後（約3〜4時間）— ページネーション・フラッシュメッセージ

- [ ] `Paginator` を使ってタスク一覧を10件単位でページ分割する
  - テンプレートに前へ/次へリンクを追加する
- [ ] Django メッセージフレームワーク（`django.contrib.messages`）を設定する
- [ ] タスク作成・編集・削除の完了後に成功メッセージを表示する
- [ ] `base.html` にメッセージ表示エリアを追加する（Bootstrap の `alert` コンポーネント）
- [ ] PR 作成・マージ（`feature/search-filter` → main）

**完了条件**
- キーワード検索・ステータスフィルタ・ページネーション・フラッシュメッセージがすべて動作する

---

## Day 10 — README・テスト・コード整理・最終確認・提出準備

**学習目標**
- README に起動手順を明確に書ける
- Django の `TestCase` でモデルテストを書ける
- ゼロから環境構築できることを自分で確認できる
- GitHub の PR 履歴を見て開発フローを説明できる

### 午前（約3時間）— README・テスト・コード整理

- [ ] ブランチ `feature/readme-tests` を切る
- [ ] `README.md` を作成する
  - プロジェクト概要（日本語）
  - 起動手順（`git clone` → `docker compose up` まで）
  - 機能一覧
  - スクリーンショット（任意）
- [ ] `tasks/tests.py` にモデルテストを追加する
  - `Task` の作成・ステータス変更のテスト
  - `python manage.py test` が通ることを確認
- [ ] `flake8` または `ruff` でコード品質チェック、警告を解消する
- [ ] `.env.example` を最新化する
- [ ] PR 作成・マージ

### 午後（約3〜4時間）— 最終確認・提出準備

- [ ] リポジトリをクリーンな状態に整理する（不要ブランチ削除）
- [ ] **ゼロ起動テスト**：別ディレクトリに `git clone` して README 通りに起動できることを確認
- [ ] GitHub の PR 一覧を確認し、各 PR に適切な説明があることを確認する

**最終チェックリスト**
- [ ] `docker compose up` で起動できる
- [ ] ユーザー登録・ログイン・ログアウトが動作する
- [ ] タスクの CRUD が動作する
- [ ] キーワード検索・フィルタリング・ページネーションが動作する
- [ ] GitHub に機能ごとのブランチ・PR が残っている
- [ ] README に起動手順が書かれている
- [ ] `.env` が `.gitignore` されており、`.env.example` がある
- [ ] `python manage.py test` が通る

**完了条件**
- ゼロ起動テスト合格・最終チェックリストをすべて満たす

---

---

## Day 11〜15 学習の全体像

### なぜ Day 11〜15 が必要か

Day 10 までで Django アプリは完成しています。しかし現状の知識ベースでは「PostgreSQL に接続した Django を動かした」にとどまります。客先が求める **「PostgreSQLなどのRDBMSを用いた実務経験」** に答えるには、「SQLを書ける・DB設計の意図を語れる」レベルが必要です。

Day 11〜15 はその差を埋めるフェーズです。

---

### 5日間で何を学ぶか

```
Day 11          Day 12          Day 13          Day 14          Day 15
─────────────   ─────────────   ─────────────   ─────────────   ─────────────
PostgreSQL      SQL実践          集計機能         全文検索         インデックス
基礎理解         JOINと           レポート画面     実装             設計
                N+1解消
─────────────   ─────────────   ─────────────   ─────────────   ─────────────
「DBとは        「複数の         「GROUP BY で    「PostgreSQL     「なぜここに
何か」を        テーブルを       集計できる」     でしかできない   インデックスが
手で触って      SQLで            レポート画面     ことをやる」     必要か」を
理解する        つなぐ」         を作る」                         説明できる
```

---

### 各日のテーマと成果物

| Day | テーマ | psqlで手打ちする内容 | Djangoに追加する機能 |
|-----|--------|--------------------|--------------------|
| 11 | RDBMSとpsql入門 | SELECT / WHERE / ORDER BY | django-debug-toolbar（SQL可視化） |
| 12 | JOIN・N+1解消 | INNER JOIN / LEFT JOIN | select_related（N+1問題の解消） |
| 13 | 集計・GROUP BY | COUNT / GROUP BY / HAVING | レポート画面（/tasks/report/） |
| 14 | 全文検索 | to_tsvector / @@ | SearchVector/SearchQuery（全文検索） |
| 15 | インデックス設計 | EXPLAIN ANALYZE / CREATE INDEX | Meta.indexes（migration管理） |

---

### 学習の流れ（各日共通）

```
午前：理論 + psql で SQL を手打ち
        ↓
午後：Django に実装（ORMでも内部で同じSQLが動いていると理解しながら）
        ↓
PR 作成・マージ（GitHub 履歴として証跡を残す）
```

---

### 完了後に「証拠として見せられるもの」

| 何を見せるか | どこにあるか |
|------------|------------|
| 集計レポート画面（ステータス別・カテゴリ別タスク数） | `http://localhost:8000/tasks/report/` |
| 全文検索が動くタスク一覧 | `http://localhost:8000/tasks/?q=keyword` |
| インデックスが定義された migration ファイル | `tasks/migrations/` |
| SQL発行数を改善したコミット履歴 | GitHub PR `feature/select-related` |
| psqlでの手打ちSQL例（コメントや doc/）| `tasks/views.py` コメント |

---

### 用語集（Day 11〜15 で登場する言葉）

**データベース基礎**

| 用語 | 読み方 | 意味 |
|------|--------|------|
| RDBMS | アールディービーエムエス | Relational DataBase Management System。表（テーブル）の形でデータを管理するソフトウェアの総称。PostgreSQL・MySQL・SQLite などが該当する |
| SQL | エスキューエル | Structured Query Language。RDBMSへの命令を書くための言語。「このデータをください」「このデータを保存して」を伝える手段 |
| テーブル | — | Excelのシートに相当。Djangoのモデル1つが1つのテーブルになる |
| カラム（列） | — | テーブルの縦方向の項目。Djangoのフィールドがカラムになる |
| レコード（行） | — | テーブルの1件分のデータ |
| PK（主キー） | プライマリーキー | 各レコードを一意に識別するID。DjangoのAutoFieldがこれ |
| FK（外部キー） | フォーリンキー | 別テーブルのPKを参照するカラム。DjangoのForeignKeyがこれ |
| psql | ピーエスキューエル | PostgreSQLに付属するコマンドラインのクライアントツール。ターミナルからSQLを直接実行できる |

**SQLの構文**

| 用語 | 意味・使い方 |
|------|------------|
| `SELECT` | 取得するカラムを指定する（`SELECT title, status`） |
| `FROM` | どのテーブルから取得するかを指定する（`FROM tasks_task`） |
| `WHERE` | 絞り込み条件を指定する（`WHERE status = 'todo'`） |
| `ORDER BY` | 並び順を指定する（`ORDER BY created_at DESC`） |
| `LIMIT` | 取得件数の上限を指定する（`LIMIT 10`） |
| `JOIN` | 複数のテーブルを結合する。ForeignKeyの先にあるデータを一緒に取得するときに使う |
| `INNER JOIN` | 両方のテーブルに一致するレコードだけを結合する |
| `LEFT JOIN` | 左テーブルの全レコードを返し、右テーブルに一致しない場合はNULLで補完する。NULLが入りうるForeignKeyに使う |
| `GROUP BY` | 同じ値を持つレコードをグループ化してまとめる（ステータス別に集計、など） |
| `COUNT(*)` | レコードの件数を数える集計関数 |
| `HAVING` | GROUP BY でグループ化した後の絞り込み条件（WHEREはグループ化前） |

**Django ORM 関連**

| 用語 | 意味・使い方 |
|------|------------|
| ORM | Object-Relational Mapping。PythonのコードをSQLに自動変換する仕組み。`Task.objects.filter(status='todo')` が内部では `SELECT ... WHERE status = 'todo'` になる |
| `select_related` | ForeignKeyで関連するテーブルをJOINで一括取得するORMのオプション。N+1問題を解消する |
| `annotate` | クエリの各行に集計値を付加する（`GROUP BY` の結果を行ごとに持つ）。例：ステータスごとの件数をQuerySetの各行に追加 |
| `aggregate` | QuerySet全体から1つの集計値を取り出す。例：タスクの総数を1つの数値として取得 |
| N+1問題 | タスクが10件あるとき、タスク一覧取得1回 + 担当者取得10回 = 合計11回のSQLが発行される問題。`select_related` で解消する |

**PostgreSQL固有の機能**

| 用語 | 意味・使い方 |
|------|------------|
| 全文検索（FTS） | Full-Text Search。テキストを単語単位に分解してインデックスを作り、高速に検索する仕組み。LIKEより高速で、単語の正規化もできる |
| tsvector | テキストを全文検索用のトークン（単語）に変換した形式。`to_tsvector('simple', 'Meeting notes')` → `'meeting':1 'notes':2` |
| tsquery | 全文検索の検索クエリ形式。`to_tsquery('simple', 'meeting')` → `'meeting'` |
| `@@` 演算子 | tsvector と tsquery がマッチするか判定する。`tsvector @@ tsquery` で `true` / `false` を返す |
| `SearchVector` | Django での tsvector 相当。`SearchVector('title', 'description')` で複数フィールドを検索対象にできる |
| `SearchQuery` | Django での tsquery 相当。検索語を渡す |
| インデックス | 本の索引と同じ。「status = 'todo' のレコードは1・3・7行目」という対応表を事前に作っておくことで検索を高速化する |
| `EXPLAIN ANALYZE` | SQLの実行計画と実際の実行時間を表示するコマンド。`Seq Scan`（全件スキャン）と `Index Scan`（インデックス利用）の違いを確認できる |
| `Seq Scan` | Sequential Scan。テーブルの全行を先頭から順に読むスキャン方式。インデックスがない場合に使われる |
| `Index Scan` | インデックスを使って必要な行だけを読むスキャン方式。大量データの絞り込みで高速 |
| `Meta.indexes` | Djangoのモデルにインデックスを定義する書き方。`makemigrations` でSQLとして管理される |

**ツール**

| 用語 | 意味・使い方 |
|------|------------|
| django-debug-toolbar | ブラウザ画面の右端にデバッグパネルを表示するDjangoの開発ツール。実行されたSQLの本数・内容・時間を確認できる |

---

## Day 11 — PostgreSQL基礎理解：RDBMSとpsql入門

**学習目標**
- RDBMSとPostgreSQLの役割を自分の言葉で説明できる
- psqlでTaskBoardのテーブル構造を確認できる
- SELECT / WHERE / ORDER BY をpsqlで手打ちして結果を確認できる
- Django ORM が内部でどんなSQLを生成しているか確認できる（django-debug-toolbar）

**ハンズオン課題**

### 午前（約3時間）— RDBMSの概念とpsql操作

- [ ] ブランチ `feature/postgres-basics` を切る
- [ ] psqlでTaskBoardのDBに接続する（`docker compose exec db psql -U postgres -d taskboard`）
- [ ] `\dt` でテーブル一覧、`\d tasks_task` でテーブル定義を確認する
- [ ] SELECT / WHERE / ORDER BY / LIMIT を手打ちで実行する
  - 例：`SELECT id, title, status FROM tasks_task ORDER BY created_at DESC LIMIT 5;`
- [ ] タスクデータを数件手動で確認し、ForeignKeyが外部テーブルのIDとして格納されていることを確認する

### 午後（約3〜4時間）— django-debug-toolbar でORM生成SQLを可視化

- [ ] `django-debug-toolbar` を `requirements.txt` に追加してインストールする
- [ ] `settings.py` に debug-toolbar の設定を追加する（開発環境のみ有効化）
- [ ] `urls.py` に debug-toolbar の URL を追加する
- [ ] ブラウザでタスク一覧ページを開き、実行されたSQLと件数を確認する
- [ ] Django shell で `Task.objects.all().query` を実行し生成SQLを確認する
- [ ] PR作成・マージ（`feature/postgres-basics` → main）

**完了条件**
- psqlで手打ちSELECTが成功し、debug-toolbar でSQL一覧が確認できる

---

## Day 12 — SQL実践：JOIN（複数テーブル結合）とN+1解消

**学習目標**
- INNER JOIN / LEFT JOIN の違いを説明できる
- ForeignKey がSQL上でどう見えるかを理解できる
- psqlで複数テーブルを結合したSELECTを手打ちできる
- N+1問題を発見し、`select_related` で解消できる

**ハンズオン課題**

### 午前（約3時間）— JOIN をpsqlで実践

- [ ] psqlでINNER JOINを使い、タスクと作成者（CustomUser）を結合したクエリを実行する
  - 例：`SELECT t.title, u.username FROM tasks_task t INNER JOIN accounts_customuser u ON t.created_by_id = u.id;`
- [ ] LEFT JOINでカテゴリなし（NULL）のタスクも含めたクエリを実行する
- [ ] 3テーブル結合（task + user + category）を実行する

### 午後（約3〜4時間）— N+1問題とselect_related

- [ ] debug-toolbar でタスク詳細ページのSQL件数を確認し、N+1問題を特定する
- [ ] ブランチ `feature/select-related` を切る
- [ ] `tasks/views.py` の ListView・DetailView に `select_related('created_by', 'assigned_to', 'category')` を追加する
- [ ] debug-toolbar でSQL件数が減ったことを確認する
- [ ] PR作成・マージ（`feature/select-related` → main）

**完了条件**
- psqlでJOINクエリが成功し、select_related 適用前後のSQL数の違いを説明できる

---

## Day 13 — 集計機能実装：レポート画面（GROUP BY・ORM annotate）

**学習目標**
- COUNT / GROUP BY / HAVING をpsqlで手打ちできる
- Django ORM の `annotate` / `aggregate` でDB集計ができる
- 集計データをテンプレートに渡してレポート画面を実装できる

**ハンズオン課題**

### 午前（約3時間）— GROUP BY をpsqlで実践

- [ ] psqlでステータス別タスク数を集計する
  - 例：`SELECT status, COUNT(*) FROM tasks_task GROUP BY status;`
- [ ] カテゴリ別タスク数、担当者別タスク数を同様に集計する
- [ ] HAVINGで「2件以上あるカテゴリのみ」に絞る
- [ ] Django shell で `Task.objects.values('status').annotate(count=Count('id'))` を実行し、psqlと同じ結果になることを確認する

### 午後（約3〜4時間）— レポート画面の実装

- [ ] ブランチ `feature/report-screen` を切る
- [ ] `tasks/views.py` に `TaskReportView` を追加する（LoginRequiredMixin + TemplateView）
- [ ] ビューにステータス別・カテゴリ別・担当者別の集計データをコンテキストで渡す
- [ ] 期限超過タスク数を集計してコンテキストに追加する（`due_date__lt=date.today()`）
- [ ] `templates/tasks/task_report.html` を作成する（Bootstrapのカード・テーブルレイアウト）
- [ ] `tasks/urls.py` に `/tasks/report/` を追加する
- [ ] ナビゲーションバーにレポートページのリンクを追加する
- [ ] PR作成・マージ（`feature/report-screen` → main）

**完了条件**
- `/tasks/report/` でレポート画面が表示され、集計データが正しく表示される

---

## Day 14 — 全文検索実装（PostgreSQL固有機能）

**学習目標**
- ILIKEと全文検索の違いを説明できる
- PostgreSQLのtsvector / tsqueryの概念を理解できる
- DjangoのSearchVector / SearchQueryを実装できる

**ハンズオン課題**

### 午前（約3時間）— 全文検索の概念とpsqlで試す

- [ ] psqlで`to_tsvector`と`to_tsquery`を試す
  - 例：`SELECT to_tsvector('simple', 'Meeting notes review') @@ to_tsquery('simple', 'review');`
- [ ] `tsvector_update_trigger` や GIN インデックスの概念をドキュメントで確認する（実装は不要）
- [ ] ILIKEと全文検索の動作の違い（部分一致 vs トークン単位）をpsqlで比較する

### 午後（約3〜4時間）— Django での全文検索実装

- [ ] ブランチ `feature/full-text-search` を切る
- [ ] `settings.py` の `INSTALLED_APPS` に `'django.contrib.postgres'` を追加する
- [ ] `tasks/views.py` の検索ロジックを `SearchVector` / `SearchQuery` に置き換える
  - `SearchVector('title', 'description', config='simple')`
- [ ] 既存のILIKE検索との差し替えを動作確認する
- [ ] 日本語テキストでの挙動の限界を確認し、コメントとして記録する
- [ ] PR作成・マージ（`feature/full-text-search` → main）

**完了条件**
- SearchVector を使った全文検索が動作し、ILIKEとの違いを説明できる

---

## Day 15 — インデックス設計・最終整備

**学習目標**
- インデックスがなぜ必要かを説明できる
- EXPLAIN ANALYZEでクエリの実行計画を（基本的に）読める
- Django の `Meta.indexes` でインデックスをmigration管理できる
- Day 11〜15 の成果をREADMEとGitHub履歴で説明できる状態にする

**ハンズオン課題**

### 午前（約3時間）— インデックスとEXPLAIN

- [ ] psqlで `EXPLAIN ANALYZE SELECT * FROM tasks_task WHERE status = 'todo';` を実行し、Seq Scan を確認する
- [ ] `CREATE INDEX` コマンドでインデックスを手動作成して再度 EXPLAIN を実行し、Index Scan になることを確認する
- [ ] `DROP INDEX` で手動インデックスを削除する（Django migration で管理するため）

### 午後（約3〜4時間）— Django migrationでインデックスを管理

- [ ] ブランチ `feature/db-index` を切る
- [ ] `tasks/models.py` の `Task` モデルに `Meta.indexes` を追加する（status・due_date・複合インデックス）
- [ ] `makemigrations` → `migrate` を実行してインデックスが作成されることを確認する
- [ ] EXPLAIN ANALYZE で実行計画の変化を確認する
- [ ] `README.md` に PostgreSQL に関するセクションを追加する
  - なぜ SQLite ではなく PostgreSQL を選んだか
  - 全文検索・インデックス設計の実装概要
- [ ] PR作成・マージ（`feature/db-index` → main）

**最終チェックリスト（Day 11〜15）**

- [ ] psql で SELECT / JOIN / GROUP BY を手打ち実行できる
- [ ] debug-toolbar でSQL発行状況を確認できる
- [ ] N+1問題を select_related で解消している
- [ ] `/tasks/report/` でレポート画面が表示される
- [ ] SearchVector を使った全文検索が動作する
- [ ] インデックスが migration ファイルで管理されている
- [ ] EXPLAIN ANALYZE の出力を読んで説明できる
- [ ] README に PostgreSQL の採用理由と実装概要が記載されている

**完了条件**
- 最終チェックリストをすべて満たす

---

## マイルストーン一覧

| マイルストーン | 日程 | 内容 |
|--------------|------|------|
| M1 | Day 3 午前 | Docker + Django + PostgreSQL が動作する |
| M2 | Day 4 終了 | 認証フロー（登録・ログイン・ログアウト）が完成 |
| M3 | Day 7 終了 | タスク CRUD + ステータス管理が完成 |
| M4 | Day 9 終了 | 検索・フィルタリング・ページネーションが完成 |
| M5 | Day 10 終了 | テスト・README 整備・ゼロ起動テスト合格 |
| M6 | Day 12 終了 | psql操作・JOIN・N+1解消が完成 |
| M7 | Day 13 終了 | レポート画面（集計）が完成 |
| M8 | Day 14 終了 | 全文検索が完成 |
| M9 | Day 15 終了 | インデックス設計・最終整備・最終提出 |

---

## スキルマッピング

| MUST スキル | 習得するタスク |
|------------|--------------|
| Git / GitHub チーム開発 | 全日程（ブランチ・PR・コミット） |
| Django 実務開発 | Day 2〜14 |
| Docker 開発環境構築 | Day 1 |

| BETTER スキル | 習得するタスク |
|--------------|--------------|
| PostgreSQL / RDBMS（**重点強化**） | Day 11（psql基礎）・Day 12（JOIN）・Day 13（集計）・Day 14（全文検索）・Day 15（インデックス） |
| Django 深度 | Day 3〜14（認証・CBV・ModelForm・検索・ページネーション・テスト・annotate） |
