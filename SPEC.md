# 一棟収益物件 日次サーチ 仕様書

毎朝のスケジュールタスク「テラさん収益物件サーチ」のプロンプト（＝本作業の仕様）の控え。
タスクが削除・破損した場合は、この内容をそのままプロンプトとして再作成すれば復元できる。

実際の運用手順（PRのマージまでが完了、公開の確認方法）と実行環境の制約は `CLAUDE.md` を参照。

---

中国・四国の一棟収益物件を毎朝全件確認し、(A) 社内用PDFレポートと (B) お客様用Webページ（GitHub Pages・固定URL）を作成する。

## ■ 出力先（最重要）

リポジトリ theearthres-cmd/shueki-daily を使う。

- お客様用ページ … リポジトリ直下の index.html を上書き
- PDF … リポジトリ直下の report.pdf として置く（同じ内容を 一棟収益物件_日次レポート_YYYYMMDD.pdf でも残す）
- main ブランチに直接 push する（GitHub Pages が main/root から配信されているため）
- 公開URL https://theearthres-cmd.github.io/shueki-daily/
- push 後に必ず curl -s -o /dev/null -w "%{http_code}" で 200 を確認し、さらにページ本文に当日の日付が入っていることを確認する。反映は数十秒かかるので、最大3分まで10秒間隔でリトライする。
- index.html には `<meta name="robots" content="noindex,nofollow">` を入れる（検索避け）。

## ■ 前日との比較（NEW／継続の判定と物件IDの引き継ぎ）

push する前に、リポジトリにある既存の index.html を読む。これが前日の掲載内容なので、

- 前日の各カードの data-pid と住所・物件名を取り出す
- 今日の物件と住所（丁目などの表記ゆれを正規化して比較）で突き合わせ、一致したものは前日の data-pid・物件名・住所表記をそのまま引き継ぎ、印は「継続」にする
- 前日に無かった物件だけ「NEW」にし、md5（住所|築年月|土地面積）の先頭10桁を新しい data-pid にする

data-pid を引き継がないと閲覧者の非表示設定がリセットされるので必ず守ること。

## ■ 検索の範囲と判定

健美家の中国・四国（/pp2/o/ 一棟アパート、/pp3/o/ 一棟マンション、/pp4/o/ 一棟ビル）を、ページネーション（/n-2/ 以降）まで含めて全件取得する（800件前後）。1カテゴリの1ページ目だけで判断しない。

- **枠A** … 中国・四国全域（広島・岡山・山口・鳥取・島根・徳島・香川・愛媛・高知）／RC造・SRC造の一棟もの（木造・軽量鉄骨は対象外）／築33年以下／価格4,000万〜8,000万円／表面利回り11%以上
- **枠B** … 価格が実勢価格（実際の取引相場）の75%以下（木造・軽量鉄骨を除く。築年・利回り・価格は不問）

築年数は健美家の表示にあわせて整数（経過年数の切り捨て）で判定する。

**実勢価格の求め方**：国土数値情報 L01（https://nlftp.mlit.go.jp/ksj/gml/data/L01/L01-26/L01-26_GML.zip ）の geojson から中国・四国9県（市区町村コード先頭31〜39）の 用途区分 L01_002="000"（住宅地）を抽出。物件住所を国土地理院ジオコーダ（https://msearch.gsi.go.jp/address-search/AddressSearch?q=… ）で緯度経度に変換して最寄りの住宅地標準地を採り、標準地単価×土地面積＝土地値。公示価格は実勢価格のおおむね9割とされるため ÷0.9 で割り戻したものを実勢価格とする。標準地までの距離が2.0kmを超えるものは採用しない。

ジオコード結果は都道府県名が一致することを必ず検証する。キャッシュのファイル名にはmd5等のハッシュを使う（日本語をそのまま使わない）。

積算（枠A）＝土地（標準地価格×土地面積）＋建物（延床×20万円/㎡×(47−築年数)÷47）

同一物件が複数業者から重複掲載されるため、土地面積と築年月・住所で名寄せして1件にまとめる。価格が複数あるときは最安値を採用し、講評で「価格は要確認」と書く。

利回りの記載がない物件、研修所・合宿所など収益物件として実体のないものは、機械計算で比率が突出しても掲載しない。

## ■ 物件写真（毎回必須）

健美家の物件詳細ページを curl で取得し、HTML内の /upload/pXXXX/XXXXXXX/....jpg を抽出。/images/・/news_img/・column_list_image はバナーなので除外。

取得：`curl -sL -A "Mozilla/5.0 ..." -e "https://www.kenbiya.com/" "https://www.kenbiya.com/upload/..." -o photos/xxx.jpg`

`convert 元.jpg -resize 900x900^ -gravity center -extent 900x600 -quality 88 出力.jpg` で3:2に統一し、base64で埋め込む。

1枚目が間取り図や室内写真のことがある。外観写真が2枚目以降にあればそちらを使う。全ページ分のコンタクトシートを作って目視で選ぶこと。

写真がない物件はプレースホルダーを作る：
`convert -size 900x600 xc:'#eef1f6' -fill '#8a94a6' -gravity center -pointsize 46 -font /usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc -annotate 0 '写真掲載なし' nophoto.jpg`

写真から構造・階数・店舗併設の有無・外観の劣化が読み取れる場合は講評に必ず反映する。

掲載が終了して同一物件が新IDで再登録されることがある。旧IDが404／掲載終了なら、住所と価格で同一性を確認して「継続（再掲載）」とする。

## ■ (A) PDF（社内用）

A4縦。Chromium のヘッドレスで生成する：

```
CHROME=$(ls -d /opt/pw-browsers/chromium_headless_shell-*/chrome-linux/headless_shell 2>/dev/null | head -1)
[ -z "$CHROME" ] && CHROME=$(which chromium chromium-browser google-chrome 2>/dev/null | head -1)
[ -z "$CHROME" ] && { npx -y playwright install --with-deps chromium; CHROME=$(ls -d ~/.cache/ms-playwright/chromium*/chrome-linux/headless_shell 2>/dev/null | head -1); }
"$CHROME" --headless --disable-gpu --no-sandbox --no-pdf-header-footer --print-to-pdf="report.pdf" file:///絶対パス/report.html
```

### 構成

1. 条件合致物件（枠A） … 一覧表（印・写真・物件名/所在地・価格・利回り・築年月・構造・戸数・土地面積・建物面積・積算概算・積算比率）＋物件ごとのカード（写真＋講評）
2. 実勢価格より安い物件（枠B） … 同じ項目＋「実勢価格」「実勢価格比」。カードに接道・再建築の可否と、算出根拠を1行（使った住宅地標準地の所在・単価・対象地からの距離・土地面積・÷0.9の割り戻し・価格が実勢の何%か）
3. 惜しい物件 … 枠Aに近いもの2〜3件＋「外れた条件」列。ここから改ページ
4. ご提案 … 4-1 優先順位（順位・物件・ねらい・理由）／4-2 物件ごとの具体アクション（確認事項・交渉ライン・判断基準・リスク／妙味）／4-3 全体メモ（融資・出口）
5. 積算の前提（概算・要注意）

### PDFの体裁ルール

- 数値セルは white-space:nowrap
- 「外れた条件」列は幅14%・nowrap・6.9pt、`<br>` で必ず2行（例「価格が上限を<br>500万円超過」）。3行以上に折り返させない
- 「印」列は幅5.5%以上。NEW／継続バッジがセルからはみ出さないこと（バッジ6.2pt・padding 0.25mm 0.8mm）
- 構造の列は nowrap にせず折り返す（隣のセルにはみ出すため）
- 検索条件の枠「枠A」「枠B」のラベルは white-space:nowrap
- PDFの一覧表サムネイルは width:100%;height:14mm;object-fit:cover、カードは 46mm×30.6mm
- 末尾のThe Earth問い合わせ枠（帯）は入れない
- 「調査済みソース」「本日の動き方」の章は入れない。「価格交渉の余地」「直近1年の稼働推移」という項目名は使わない。物件資料PDF作成の案内文も入れない
- 送付前に `pdftoppm -png -r 72 report.pdf chk` で画像化し、Read ツールで全ページを目視確認する

## ■ (B) お客様用Webページ（index.html）

表は使わず、物件ごとのカード（写真→価格・利回り→スペックのグリッド→講評）で1カラム。max-width 760px。

Google Fonts の Zen Kaku Gothic New（見出し）＋Noto Sans JP（本文）。

配色は白で統一。ダークモードには対応させない。`:root` に色トークンを1組だけ置き、`color-scheme:light` を指定。`@media (prefers-color-scheme: dark)` と `:root[data-theme="dark"]` のブロックは作らない。`--bg` と `--card` はどちらも #ffffff、カード・サマリー・検索条件枠は白地＋1px の枠線（#dde3ec）で区切る。body には必ず `background:var(--bg)` を指定。

積算比率はバッジで色分け（85%以上=緑、60〜84%=黄、60%未満=グレー、算定不可はグレーで「算定不可（倍率地域）」）。枠Bの「実勢価格比」バッジは常に緑。

冒頭のサマリーは「枠A 条件合致 N件」「枠B 実勢価格より安い N件」の2つだけ。サマリーのタイル自体はリンクにしない。

### 枠の切り替えはフローティングボタン

画面下中央に固定表示の `<nav class="fab" id="fab">` を置き、「枠A ｜件数」「枠B ｜件数」の2つのアンカー（`<a href="#sec-a" data-jump="A">` / `<a href="#sec-b" data-jump="B">`）を入れる。position:fixed;left:50%;transform:translateX(-50%);bottom:calc(16px + env(safe-area-inset-bottom,0px));z-index:50。白地・1px枠線・border-radius:999px・box-shadow 0 4px 16px rgba(22,35,61,.14)。現在地のほうに `.on` を付けて反転表示（背景 --navy・文字白）し aria-current も切り替える。判定は `#sec-b` の getBoundingClientRect().top <= 80 で、scroll と resize に passive で追随させる。見出しは `<h2 id="sec-a">` / `<h2 id="sec-b">` とし scroll-margin-top:12px。`.wrap` に padding-bottom:96px。件数は非表示機能と連動（span に data-count="A"/"B"）。`@media print` では `.fab` を display:none。

### 閲覧者による非表示機能（Webページのみ・PDFには入れない）

非表示ボタンは物件名の横。`<div class="namerow">` で h3.name と `<button class="hidebtn">非表示</button>` を横並びにし、display:flex;align-items:flex-start;justify-content:space-between;gap:10px、`.namerow>.name` は flex:1 1 auto;min-width:0、`.hidebtn` は flex:0 0 auto;margin-top:2px;white-space:nowrap。ボタン内の文言は「非表示」だけにし、aria-label に「◯◯を非表示にする」と物件名を入れる。カード末尾には置かない。

押すとそのカードを hidden にし、data-pid を localStorage のキー `te-shueki-hidden-v1` に配列で保存する。読み書きは必ず try/catch で囲み、値が壊れていても空配列で動くようにする。

サマリー・セクション見出し・フローティングボタンの件数は表示中の件数に連動させる。ある枠が全件非表示になったら「この枠の物件はすべて非表示にしています。」を出す。

サマリー直下に復帰バーを置き、非表示が1件以上あるときだけ表示：「非表示にしている物件が N 件あります（この端末のみ）。」＋「すべて表示に戻す」ボタン。

**【重要】CSSに `[hidden]{display:none !important}` を必ず入れる。** `.restore` に display:flex を当てているため、これが無いと「0件」の復帰バーが出っぱなしになる。

ボタンは白地＋1px枠線・font-size .72rem、`:focus-visible` にアウトラインを付ける。`@media print` では `.hidebtn` と `.restore` を display:none。

CSSの子孫セレクタに注意（`.tally div` は入れ子のdivにも当たる。`.tally>div` と書く）。

### その他

ページ末尾に「PDF版をダウンロード」のリンク（report.pdf への相対リンク）を1つだけ置く。それ以外のボタン（印刷等）は一切置かない。印刷用CSS（@media print）だけ残す。

末尾にThe Earth問い合わせ枠：「The Earth株式会社 -ジアース-」、〒730-0822 広島市中区吉島東一丁目20番20号、TEL 082-248-9393／FAX 082-248-9384、広島県知事（1）第11423号、公益社団法人 全日本不動産協会 会員。あわせて「掲載時点の情報／価格・条件は予告なく変更／積算は概算」の注意書き。

公開後は Playwright（viewport 390x780 のスマホ相当と 820x900 の2通り）で、初期は復帰バーが非表示→1件非表示→件数連動→リロード後も維持→「すべて表示に戻す」で復帰→フローティングで枠A/枠Bを往復→最下部で問い合わせ枠が隠れていないこと、まで動作確認する。カード単体のスクリーンショットも撮り、物件名とボタンが1行に収まっているか目視する。

## ■ お客様向けの書き方

元付業者名・掲載ポータル名（健美家・LIFULL・連合隊など）は一切載せない。「元付へのヒアリング」「当社にて確認します」という表現は使わない。確認が必要な事項は「〜は要確認です。」と書き、掲載のない項目は「掲載なし（要確認）」とする。写真がない物件のキャプションは「写真は準備中です」。

用語：枠Bの比較対象は「実勢価格（実際の取引相場）」と呼ぶ。「土地評価」「土地値」「土地の相場」「売買価格」という語は使わない（物件の値段は単に「価格」と書く）。比率は「◯割安い」ではなく「実勢価格の約◯%の価格」と%で表示する。「土地値/価格」ではなく「実勢価格比」。

## ■ 掲載ルール

掲載するのは枠A・枠Bに条件合致した物件だけ。参考枠は設けない。

一度条件合致した物件は、掲載終了（売り止め・成約）が確認できるまで毎日掲載を続ける。掲載IDが変わる再掲載は住所と価格で同一性を確認して掲載を継続する。売り止めを確認した物件はページから外す。

合致物件が1件もない日は、サマリー（枠A 0件／枠B 0件）と「現在、条件に合致する物件はありません。毎朝の検索で条件に合致した物件は、この場所に売り止めとなるまで掲載を続けます。」の空状態表示だけのページにする。

「検索基準を改定しました」等のお知らせ枠は置かない。検索条件の枠A・枠Bはそれぞれ改行して表示する。

枠Aは0件の日が続いてよい。無理に条件を緩めない。

## ■ 実行後の報告

セッションの最後に、枠A・枠Bの件数、NEW／継続の内訳、惜しい物件、公開URLの確認結果（HTTPコードと日付の一致）を簡潔に報告する。

枠A・枠Bとも新規0件で掲載継続中の物件にも変化がなければ「本日 0件」と短く報告する（PDFは作らず、Webは日付だけ更新する）。
