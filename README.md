# Qiita スケジュール公開（A案：GitHub Actions）

Qiita の連載記事を、指定日時に自動で Qiita へ公開する仕組み。現在の対象は「非エンジニアのAI業務自動化（新幹線の領収書ツール編）」全5回。
Qiita には予約投稿機能が無いため、GitHub Actions の cron で Qiita CLI を実行して代替する。

対象アカウント: https://qiita.com/horiken1977

## 公開スケジュール

| 記事 | ファイル | 公開予定 (JST) |
|---|---|---|
| #1 業務自動化（経費清算用領収書大量出力） | `public/ex-receipt-01.md` | 2026-10-03(土) 公開済み（予約の起動が動かず 18:53 に手動実行。タイトルは公開後に Qiita で変更） |
| ログインだけは、人間がやる | `public/ex-receipt-02.md` | 2026-10-08(木) 23:00 |
| 人がクリックする順番を、そのままAIに書かせる | `public/ex-receipt-03.md` | 2026-10-12(月) 23:00 |
| 止まった画面は、撮ってAIに見せればいい | `public/ex-receipt-04.md` | 2026-10-15(木) 23:00 |
| レビューとは、動かした結果を見て判断すること | `public/ex-receipt-05.md` | 2026-10-19(月) 23:00 |
| （Orca編）1人の天才より複数の秀才に対話させる | `public/orca-multi-agent-01.md` | 2026-10-22(木) 23:00 |
| （Orca編）対話ループをOrcaで簡単に実装 | `public/orca-multi-agent-02.md` | 2026-10-26(月) 23:00 |
| （Orca編）マネージャーの仕事は「基準を決めて、チェックする」 | `public/orca-multi-agent-03.md` | 2026-10-29(木) 23:00 |
| （Orca編）仕事単位で測る | `public/orca-multi-agent-04.md` | 2026-11-02(月) 23:00 |

**2026-10-08 に、残りの8本（ex-receipt-02〜05・orca-multi-agent-01〜04）を予定を待たずに手で公開した**（workflow_dispatch で1本ずつ）。公開の前に、画像の見る所に赤枠を付けた（ex-receipt の赤枠を付け直した画像は `images/ex-receipt/` に置き、記事からは raw.githubusercontent.com の URL で参照する）。下の表の日時は、もとの予定。

#2〜#5 のタイトルは「非エンジニアのAI業務自動化（EX領収書自動出力）/…」、Orca編は「非エンジニアのAI業務自動化（Orcaマルチエージェント）/…」。Orca編の画像は `images/orca-multi-agent/` に置き、記事からは raw.githubusercontent.com の URL で参照する（このリポジトリが公開なので見える。記事の正本は `drafts/orca-multi-agent/`、公開用は下書きメモを消し画像の URL を差し替えたもの）。

ワークフローは月・木 14:00 UTC（23:00 JST）に起動し、遅れや起動の抜けに備えて予備の起動を2回（20:30 UTC・翌 03:30 UTC）入れている。起動した時刻が、`.github/workflows/publish-scheduled.yml` の表の公開予定から24時間以内なら、その記事を1本だけ公開する。

**経緯:** 旧連載「サルでもわかるバイブコーディング！実践編」#1〜#5（2026-08-28〜09-25 公開）は、2026-10 に Qiita から削除し、`public/jissen-0N.md` も外した。新しい連載で書き直している（記事の正本は `03.Business/side/publishment/Qiita/drafts/ex-receipt/`）。

> ⚠️ GitHub Actions の cron は UTC 基準で、混雑時は数分〜十数分**遅れる**ことがある（前倒しはされない）。
> 「23:00 ちょうど」は保証されない。実績では3〜9時間遅れることが多く、10/3 の1回限りの起動は動かなかった。厳密な定刻が必要なら B案（Mac の launchd）を検討。

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
1. 月・木 14:00 UTC に起動（予備は 20:30 UTC と翌 03:30 UTC。手動実行 `workflow_dispatch` も可）
2. 今の時刻と表の公開予定を比べ、予定から24時間以内の記事を1本決める（該当なしなら何もしない）
3. **二重投稿ガード**：Qiita API で ①同じタイトルの記事がある、または ②公開予定の時刻のあとに記事が出ている、なら公開しない（②は公開後に Qiita でタイトルを変えても二重に出さないため）。API がエラーを返したら（トークン切れなど）**失敗として止める**
4. `npx qiita publish <記事名>` で公開（frontmatter の `private: false` で一般公開）

### 予行演習（公開せずに確かめる）

Actions タブ → "Publish Qiita articles on schedule" → Run workflow で、`dry_run` にチェック、`now` に試したい時刻（例 `2026-10-08T14:00:00Z`）を入れて実行する。記事の選び方・トークン・ガードまで通り、公開はしない。

## 全記事の公開が終わったら

最後の記事の公開後は、この cron は月・木・火・金に空振り実行を続ける（害はないが無駄）。
不要になったら **`.github/workflows/publish-scheduled.yml` を削除**するか、
Actions タブでこのワークフローを **Disable** する。
