# Qiita スケジュール公開（A案：GitHub Actions）

「サルでもわかるバイブコーディング！実践編」全5回を、指定日時に自動で Qiita へ公開する仕組み。
Qiita には予約投稿機能が無いため、GitHub Actions の cron で Qiita CLI を実行して代替する。

対象アカウント: https://qiita.com/horiken1977

## 公開スケジュール

| 記事 | ファイル | 公開日時 (JST) | cron (UTC) |
|---|---|---|---|
| #1 たった一文から始まった | `public/jissen-01.md` | 2026-08-28(金) 23:00 | 08-28 14:00 |
| #2 AIと二人三脚で"壁"を越える | `public/jissen-02.md` | 2026-09-04(金) 23:00 | 09-04 14:00 |
| #3 "ちゃんと使える"に育てる | `public/jissen-03.md` | 2026-09-11(金) 23:00 | 09-11 14:00 |
| #4 自分用ツールを"商品"にする | `public/jissen-04.md` | 2026-09-18(金) 23:00 | 09-18 14:00 |
| #5 定年後の小さな稼ぎにする | `public/jissen-05.md` | 2026-09-25(金) 23:00 | 09-25 14:00 |

**🔁 2026-08-22 再設定:** 初回シリーズ（08-07〜）の #1・#2 が設定ミスで失敗し、その後 #1 から手動でまとめて公開されてしまったため、公開済み記事を Qiita から削除。上表のとおり 2026-08-28 起点で #1 から週次で公開し直す。

> ⚠️ GitHub Actions の cron は UTC 基準で、混雑時は数分〜十数分**遅れる**ことがある（前倒しはされない）。
> 「23:00 ちょうど」は保証されない。厳密な定刻が必要なら B案（Mac の launchd）を検討。

## セットアップ手順（初回のみ）

### 1. GitHub リポジトリを作る
新規リポジトリを作成（**Private でよい**）。このフォルダ (`qiita-scheduler/`) の中身を**リポジトリのルート**に置く。
```
<repo>/
  public/jissen-01.md ... jissen-05.md
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

1. `public/jissen-01.md` の `private: false` を一時的に **`private: true`** に変更して push
   （`private: true` = **限定共有**。リンクを知っている人だけが見られる＝一般公開されない）
2. GitHub の **Actions タブ → "Publish Qiita articles on schedule" → Run workflow**
   - `article` に `jissen-01` を入力して実行
3. Qiita のマイページで、限定共有記事として正しく表示されるか確認
4. 確認できたらその記事を**削除**し、`jissen-01.md` を **`private: false` に戻して** push
   （本番の cron 実行時に改めて一般公開される。※「二重投稿ガード」は同じタイトルの
   記事が残っていると公開をスキップするので、ドライランで作った記事は必ず削除しておく）

## 記事の直し方・スクショの入れ方

- 記事本文は `public/jissen-0N.md` の frontmatter より下を編集して push すればよい。
- **スクショ差し込み**：#1〜#3 の本文末尾に `<!-- 差し込みスクショ候補 ... -->` というコメントがある。
  画像を Qiita エディタにドラッグ → 生成された `![...](https://qiita-image-store...)` URL を、
  該当箇所へ貼って push する。**画像は公開予定日時までに入れておくこと**（自動では入らない）。
- タイトルを変えたら、frontmatter の `title:` も必ず更新する（`#` を含むので**ダブルクオート必須**）。

## 仕組み（workflow の中身）

`.github/workflows/publish-scheduled.yml`
1. 毎週金曜 14:00 UTC に起動（＋手動実行 `workflow_dispatch` も可）
2. 実行日(UTC)を見て、スケジュール表から公開対象を1本決める（該当なしなら何もしない）
3. **二重投稿ガード**：Qiita API で同じタイトルの記事が既にあれば公開をスキップ
4. `npx qiita publish jissen-0N` で公開（frontmatter の `private: false` で一般公開）

## 全記事の公開が終わったら

9/4 の #5 公開後は、この cron は毎週金曜に空振り実行を続ける（害はないが無駄）。
不要になったら **`.github/workflows/publish-scheduled.yml` を削除**するか、
Actions タブでこのワークフローを **Disable** する。
