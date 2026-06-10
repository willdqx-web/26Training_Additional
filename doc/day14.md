# Day 14 — 全文検索実装（PostgreSQL固有機能）

## このプロジェクト全体の流れ

```
Day 1〜10  → Day 11〜13 → [Day 14] → Day 15
基礎完成     psql・JOIN   全文検索    インデックス
             集計画面     ★今日      最終整備
```

集計画面が完成しました。今日は **PostgreSQL 固有の全文検索（Full-Text Search）** を実装します。これは SQLite では使えない機能で、「PostgreSQL を選んだ理由」として具体的に説明できる実装になります。

---

## この日のゴール

- ILIKE検索と全文検索の違いを説明できる
- PostgreSQL の tsvector / tsquery の概念を理解できる
- Django の SearchVector / SearchQuery を使って全文検索を実装できる

---

## この日の前提

- Day 13 の `feature/report-screen` が main にマージ済み

---

## 午前：全文検索の概念とpsqlで試す

### 1. ILIKE 検索の限界

Day 9 で実装した検索は `ILIKE` を使った「部分一致検索」です：

```sql
-- 「会議」を含むタスクを検索（ILIKE）
SELECT * FROM tasks_task WHERE title ILIKE '%会議%';
```

**ILIKE の問題点**

| 問題 | 説明 |
|------|------|
| インデックスが効かない | `LIKE '%xxx%'` はフルスキャンが必要（遅い） |
| 単語の概念がない | 「meeting」で「meetings」はヒットしない |
| 重みづけができない | タイトルと説明文を同じ重みで扱う |

### 2. 全文検索（FTS）とは

全文検索はテキストを「トークン（単語）」に分解し、インデックスを作って高速に検索する仕組みです。

```
「Meeting notes for project review」
        ↓ トークン化（tsvector）
'meeting':1 'note':2 'project':4 'review':5

検索語「review」（tsquery）
        ↓ マッチング
'review' が tsvector に含まれる → TRUE
```

### 3. tsvector と tsquery

**tsvector**：文書をトークンの集合に変換したもの

```sql
SELECT to_tsvector('simple', 'Meeting notes for project review');
-- → 'for':3 'meeting':1 'notes':2 'project':4 'review':5
```

**tsquery**：検索クエリ

```sql
SELECT to_tsquery('simple', 'review');
-- → 'review'

-- AND 検索
SELECT to_tsquery('simple', 'meeting & review');

-- OR 検索
SELECT to_tsquery('simple', 'meeting | notes');
```

**マッチング（@@演算子）**

```sql
SELECT 'meeting notes for review'::text @@ to_tsquery('simple', 'review');
-- → true

SELECT to_tsvector('simple', 'meeting notes') @@ to_tsquery('simple', 'review');
-- → false
```

**`config='simple'` について**

PostgreSQL の全文検索はテキストを言語別に解析します。日本語を正しく分解するには `pg_bigm` などの拡張が必要ですが、インストールが複雑です。今回は `simple`（最小限の変換のみ）を使います：

| config | 動作 | 対応言語 |
|--------|------|---------|
| `simple` | 小文字化のみ | 言語非依存（英語のみ実用的） |
| `english` | ストップワード除去 + 語幹化 | 英語 |
| `japanese`（要拡張） | 形態素解析 | 日本語 |

### ハンズオン（午前）

#### Step 1：psqlに接続する

```bash
docker compose exec db psql -U postgres -d taskboard
```

#### Step 2：tsvector と tsquery を試す

> **事前準備**：全文検索は英単語単位で動作する。psql の実験前に、ブラウザからタイトルが英語のタスクを最低2件作成しておく（例：`meeting notes`、`project review`、`deployment task`）。日本語タイトルのタスクだけだと0件になり動作確認できない。

```sql
-- tsvector の確認（文字列をトークンに分解）
SELECT to_tsvector('simple', 'Meeting notes for project review');

-- tsquery の確認
SELECT to_tsquery('simple', 'review');

-- マッチング（@@ 演算子）
SELECT to_tsvector('simple', 'Meeting notes for project review')
       @@ to_tsquery('simple', 'review');
-- → t

-- 実際のタスクデータで試す（英語タイトルのタスクが必要）
SELECT title
FROM tasks_task
WHERE to_tsvector('simple', title || ' ' || COALESCE(description, ''))
      @@ to_tsquery('simple', 'meeting');
```

#### Step 3：ILIKE との違いを比較する

```sql
-- ILIKE（部分一致）
SELECT title FROM tasks_task WHERE title ILIKE '%task%';

-- 全文検索
SELECT title FROM tasks_task
WHERE to_tsvector('simple', title) @@ to_tsquery('simple', 'task');
```

> 日本語タイトルの場合、`simple` config では形態素解析ができないため全文検索は機能しません。これは限界として認識し、実務では `pg_bigm` などを使うことをコメントで記録します。

---

## 午後：Django での全文検索実装

### 4. django.contrib.postgres.search の仕組み

Django には PostgreSQL の全文検索をラップした API が組み込まれています（追加インストール不要）。

```python
from django.contrib.postgres.search import SearchVector, SearchQuery, SearchRank

# SearchVector：検索対象フィールドを指定
# SearchQuery：検索語
# SearchRank：関連度スコア

Task.objects.annotate(
    search=SearchVector('title', 'description', config='simple'),
).filter(search=SearchQuery('review', config='simple'))
```

**SearchVector / SearchQuery vs ILIKE の比較**

| 機能 | ILIKE | SearchVector |
|------|-------|-------------|
| 部分一致 | ✓ | △（トークン単位） |
| インデックス利用 | ✗ | ✓（GINインデックス） |
| 重みづけ | ✗ | ✓（`weight='A'` など） |
| 関連度スコア | ✗ | ✓（SearchRank） |
| 日本語対応 | ✓（文字一致） | △（simple configのみ） |

### ハンズオン（午後）

#### Step 5：ブランチを作成する

```bash
git checkout main
git pull origin main
git checkout -b feature/full-text-search
```

#### Step 6：settings.py を更新する

`config/settings.py` の `INSTALLED_APPS` に追加する：

```python
INSTALLED_APPS = [
    # ...既存のアプリ...
    'django.contrib.postgres',  # PostgreSQL固有機能を有効化
]
```

#### Step 7：tasks/views.py の検索ロジックを差し替える

`TaskListView.get_queryset()` 内の ILIKE 検索部分を SearchVector に置き換える：

```python
from django.contrib.postgres.search import SearchVector, SearchQuery


class TaskListView(LoginRequiredMixin, ListView):
    # ...

    def get_queryset(self):
        queryset = Task.objects.filter(
            created_by=self.request.user
        ).select_related('created_by', 'assigned_to', 'category')

        keyword = self.request.GET.get('q', '').strip()
        if keyword:
            # PostgreSQL 全文検索
            # search_type='plain' を指定しないと & | ! などの文字でクラッシュする
            search_query = SearchQuery(keyword, config='simple', search_type='plain')
            queryset = queryset.annotate(
                search=SearchVector('title', 'description', config='simple'),
            ).filter(search=search_query)

        # ステータスフィルタ（変更なし）
        status = self.request.GET.get('status', '')
        if status:
            queryset = queryset.filter(status=status)

        # カテゴリフィルタ（変更なし）
        category_id = self.request.GET.get('category', '')
        if category_id:
            queryset = queryset.filter(category_id=category_id)

        return queryset
```

#### Step 8：動作確認

1. 英語タイトルのタスクをいくつか作成する（例：「meeting notes」「project review」「deployment task」）
2. 検索フォームに「review」と入力して検索する
3. 該当タスクがヒットすることを確認する
4. DjDT で生成SQLに `to_tsvector` が含まれることを確認する

#### Step 9：日本語の限界を記録する

`tasks/views.py` のコメントに記録する：

```python
# 全文検索（PostgreSQL SearchVector）
# search_type='plain': & | ! などの特殊文字をキーワードとして安全に扱う。
# config='simple': 小文字化のみ行うため英語テキストで有効。
# 日本語を正確に分かち書きするには pg_bigm 拡張が必要だが、
# 本プロジェクトのスコープ外のため ILIKE フォールバックは削除済み。
# 日本語タイトルへの対応は本番環境では pg_bigm を検討すること。
search_query = SearchQuery(keyword, config='simple', search_type='plain')
```

#### Step 10：PR を作成してマージする

```bash
git add tasks/views.py config/settings.py
git commit -m "feat: PostgreSQL全文検索（SearchVector）を実装"
git push origin feature/full-text-search
```

PRのコメントに「生成SQL（to_tsvectorを含む）のスクリーンショット」を添付してマージする。

---

## よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `ProgrammingError: operator does not exist` | `django.contrib.postgres` が未設定 | INSTALLED_APPS に追加する |
| 日本語で検索してもヒットしない | `simple` configは日本語非対応 | 仕様として認識する（コメントに記録） |
| 検索後にページネーションが崩れる | `annotate` 後のクエリセットの扱い | `distinct()` を追加して重複を除去する |
| `SearchQuery` で特殊文字エラー | 記号が tsquery 構文として解析される | `search_type='plain'` を指定する：`SearchQuery(keyword, config='simple', search_type='plain')` |

---

## この日の GitHub チェックポイント

### コミットメッセージ例

```bash
git commit -m "feat: PostgreSQL全文検索（SearchVector）を実装"
```

### いつ PR をマージするか

**条件**：英語キーワードで全文検索が動作し、DjDTで `to_tsvector` を含むSQLが確認できること。

**理由**：Day 15 はインデックス設計。全文検索が完成した状態でインデックスの効果を確認するほうがわかりやすい。
