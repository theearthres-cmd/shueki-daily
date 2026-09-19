# shueki-daily 運用メモ

中国・四国の一棟収益物件を毎朝全件確認し、(A) 社内用PDF (`report.pdf`) と (B) お客様用ページ (`index.html`) を更新するリポジトリ。

## 公開の仕組み（毎日ここまでやって完了）

GitHub Pages は **main ブランチの root** から配信されている。公開URLは
https://theearthres-cmd.github.io/shueki-daily/

日次更新は次の4ステップを終えて初めて「完了」とする。作業ブランチに push した時点では**公開ページは一切変わらない**。

1. 作業ブランチにコミット＆push
2. main 宛の PR を作成する
3. **その PR を main にマージする**（`rebase` マージで履歴を一直線に保つ）
4. Actions の `pages build and deployment` が `success` になるまで確認する

このリポジトリでは main への直接 push を避け、必ず PR 経由でマージすること。PR をマージし忘れると、ページは前日のまま止まる。

### 反映確認について

実行環境のネットワークポリシーにより `*.github.io` へは直接アクセスできない（403 `host_not_allowed`）。`curl` での公開URL確認はできないので、代わりに **Actions の `pages build and deployment` の実行結果**（対象コミットの head_sha が一致し conclusion が success）をもって反映確認とする。あわせて `git show origin/main:index.html` で当日の日付と件数が入っていることを確認する。

## 前日との突き合わせ

`index.html` の各カードの `data-pid` は閲覧者の非表示設定のキーなので、**同一物件では絶対に変えない**。前日の `index.html` を読み、住所（表記ゆれを正規化）で突き合わせて、一致したものは `data-pid`・物件名・住所表記をそのまま引き継ぎ「継続」とする。

掲載継続ルール：一度掲載した物件は、**掲載終了が確認できるまで毎日掲載を続ける**。その日の条件判定を再度満たさなくても外さない。当日取得した全件カタログに住所が見当たらない場合のみ掲載終了とみなす（物件詳細URLが404／「掲載終了」表示なら確定）。前日 `NEW` だったカードは翌日 `継続` に直すこと。

## 過去にハマった箇所

**価格のパース**：一覧ページの価格は3パターンある。`<span>6,600</span>万円`／`<span>2</span>億<span>7,000</span>万円`（＝27,000万円）／`<span>1</span>億円`（万円の span が無い）。億を取りこぼすと枠A・枠Bの判定が丸ごと狂う。

**物件URL**：`/pp2/o/{県}/{市}/re_xxx/` だけでなく `/pp2/o/{県}/{市}/{区}/re_xxx/` の3段もある。

**構造の判定**：除外するのは「木造」「軽量鉄骨」と明記されているものだけ。階数だけ書かれた無印の `S造◯階建` は軽量鉄骨扱いにせず、枠Bの対象に含める。枠Aは `RC` / `SRC` のみ。

**名寄せ**：土地面積（小数2桁）＋市区町村＋築年でグルーピングし、価格は最安値を採用（差異があれば講評に「価格は要確認」）。住所には `徳島県小松島市徳島県小松島市…` `岡山県倉敷市倉敷市…` のように語句が重複した表記があるので、連続する重複部分を畳んでから比較する。同一住所に別物件が複数ある場合は土地面積で判別する。

**比率の向き**：積算比率 = 積算額 ÷ 価格 × 100（85%以上=緑／60〜84%=黄／60%未満=グレー）。実勢価格比 = 価格 ÷ 実勢価格 × 100（バッジは常に緑）。逆にすると前日と数字が合わなくなる。

**掲載しない物件**：利回りの記載が無いもの、研修所・合宿所など収益物件の実体が無いものは、比率が突出していても掲載しない。

## 実行環境で使えるもの・使えないもの

- **使えない**：`pip install`（pypi.org が egress 許可外）、`apt-get install`（ubuntu ミラーが許可外）、`pdftoppm` / `pdftotext`、ImageMagick の `convert`
- **使える**：`/opt/pw-browsers/chromium_headless_shell-*/chrome-linux/headless_shell`（`--headless --no-pdf-header-footer --print-to-pdf` でPDF生成）、グローバル導入済みの Playwright（`NODE_PATH=/opt/node22/lib/node_modules`、実行ファイルは `/opt/pw-browsers/chromium-*/chrome-linux/chrome`）
- **PDFの目視確認**：`pdftoppm` が無いため、PDF化前の `report.html` を Playwright で `emulateMedia({media:'print'})`・viewport 794x1123 で開き、スクロールしながらスクリーンショットを撮って Read ツールで全ページ確認する
- 健美家は短時間に叩きすぎると 429 を返す。同時実行6程度＋指数バックオフで全806件取得できる（所要3〜4分）
