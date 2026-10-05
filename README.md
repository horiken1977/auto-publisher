# Qiita スケジュール公開（A案：GitHub Actions）

Qiita の連載記事を、指定日時に自動で Qiita へ公開する仕組み。現在の対象は「非エンジニアのAI業務自動化（新幹線の領収書ツール編）」全5回。
Qiita には予約投稿機能が無いため、GitHub Actions の cron で Qiita CLI を実行して代替する。

対象アカウント: https://qiita.com/horiken1977

## 公開スケジュール

| 記事 | ファイル | 公開日時 (JST) | cron (UTC) |
|---|---|---|---|
| #1 URLだけでは止まる。画面を見せればAIは作れる | `public/ex-receipt-01.md` | 2026-10-03(土) 18:00 | 10-03 09:00 |
| #2 ログインだけは、人間がやる | `public/ex-receipt-02.md` | 2026-10-08(木) 23:00 | 10-08 14:00 |
| #3 人がクリックする順番を、そのままAIに書かせる | `public/ex-receipt-03.md` | 2026-10-12(月) 23:00 | 10-12 14:00 |
| #4 止まった画面は、撮ってAIに見せればいい | `public/ex-receipt-04.md` | 2026-10-15(木) 23:00 | 10-15 14:00 |
| #5 レビューとは、動かした結果を見て判断すること | `public/ex-receipt-05.md` | 2026-10-19(月) 23:00 | 10-19 14:00 |

ワークフローは毎週月曜・木曜の 14:00 UTC（23:00 JST）に起動し（#1 だけは 10-03 09:00 UTC の1回限りの cron）、`.github/workflows/publish-scheduled.yml` の日付の表にある記事を1本だけ公開する。同じタイトルの記事が既にあれば公開しない（二重投稿ガード）。

**経緯:** 旧連載「サルでもわかるバイブコーディング！実践編」#1〜#5（2026-08-28〜09-25 公開）は、2026-10 に Qiita から削除し、`public/jissen-0N.md` も外した。新しい連載で書き直している（記事の正本は `03.Business/side/blog/drafts/ex-receipt/`）。

> ⚠️ GitHub Actions の cron は UTC 基準で、混雑時は数分〜十数分**遅れる**ことがある（前倒しはされない）。
> 「23:00 ちょうど」は保証されない。厳密な定刻が必要なら B案（Mac の launchd）を検討。

## セットアップ手順（初回のみ）

### 1. GitHub リポジトリを作る
新規リポジトリを作成（**Private でよい**）。このフォルダ (`qiita-scheduler/`) の中身を**リポジトリのルート**に置く。
```
<repo>/
  public/ex-receipt-01.md ... ex-receipt-05.md
  .github/workflows/publish-scheduled.yml
  package.json
  .gitignore
  README.md
```

### 2. Qiita トークンを発行する
https://qiita.com/settings/tokens/new で、**`read_qiita` と `write_qiita`** にチェックして発行。
（トークンは一度しか表示されない。控えておく。**このトークンは誰にも渡さない／チャットに貼らない**。）

### 3. GitHub Secrets に登録する
リポジトリの **Settings → Secrets and variables → Actions → New repository secret**
- Name: `QIITA_TOKEN`
- Secret: 手順2で発行したトークン

### 4. push する
デフォルトブランチ（`main`）に push。**scheduled ワークフローは push 後に有効化される**ため、
**8/7 より最低でも1日前までに** push しておくこと。
```bash
git init
git add .
git commit -m "Add Qiita scheduled publishing"
git branch -M main
git remote add origin git@github.com:horiken1977/<repo>.git
git push -u origin main
```

### 5. 動作確認（本番前のドライラン・任意だが推奨）
本番公開の前に、パイプライン（トークン・CLI・公開経路）が通るか確認する安全な方法：

1. `public/ex-receipt-01.md` の `private: false` を一時的に **`private: true`** に変更して push
   （`private: true` = **限定共有**。リンクを知っている人だけが見られる＝一般公開されない）
2. GitHub の **Actions タブ → "Publish Qiita articles on schedule" → Run workflow**
   - `article` に `ex-receipt-01` を入力して実行
3. Qiita のマイページで、限定共有記事として正しく表示されるか確認
4. 確認できたらその記事を**削除**し、`ex-receipt-01.md` を **`private: false` に戻して** push
   （本番の cron 実行時に改めて一般公開される。※「二重投稿ガード」は同じタイトルの
   記事が残っていると公開をスキップするので、ドライランで作った記事は必ず削除しておく）

## 記事の直し方・スクショの入れ方

- 記事本文は `public/ex-receipt-0N.md` の frontmatter より下を編集して push すればよい。
- **スクショ差し込み**：#1〜#3 の本文末尾に `<!-- 差し込みスクショ候補 ... -->` というコメントがある。
  画像を Qiita エディタにドラッグ → 生成された `![...](https://qiita-image-store...)` URL を、
  該当箇所へ貼って push する。**画像は公開予定日時までに入れておくこと**（自動では入らない）。
- タイトルを変えたら、frontmatter の `title:` も必ず更新する（`#` を含むので**ダブルクオート必須**）。

## 仕組み（workflow の中身）

`.github/workflows/publish-scheduled.yml`
1. 毎週金曜 14:00 UTC に起動（＋手動実行 `workflow_dispatch` も可）
2. 実行日(UTC)を見て、スケジュール表から公開対象を1本決める（該当なしなら何もしない）
3. **二重投稿ガード**：Qiita API で同じタイトルの記事が既にあれば公開をスキップ
4. `npx qiita publish ex-receipt-0N` で公開（frontmatter の `private: false` で一般公開）

## 全記事の公開が終わったら

9/4 の #5 公開後は、この cron は毎週金曜に空振り実行を続ける（害はないが無駄）。
不要になったら **`.github/workflows/publish-scheduled.yml` を削除**するか、
Actions タブでこのワークフローを **Disable** する。
