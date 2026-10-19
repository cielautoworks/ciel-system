# ciel-system 共通仕様書（アプリ間の「契約」部分のみ）

最終更新: 2026-10-20

## この文書の位置づけ

car-app・ciel-systemハブ(`cielsystem_index.html`)・ads.html・ju-history.html・pricing.html・pricetag.htmlは、互いのコード(JS/HTML)を一切import・参照しない、完全に独立した静的HTMLファイルである。つながっているのは以下の2点だけ:

1. 共通のFirebase Realtime Database(`ciel-autoworks`)を読み書きしていること
2. 一部のツール間で、URL遷移＋クエリパラメータによる画面連携があること

**この文書に書かれている内容(1〜4章)を変更する場合は、関係する全アプリに影響するため、単独アプリのセッション/リポジトリだけで進めず、必要性を確認してから行うこと。** これ以外の部分(見た目・そのアプリ内だけで完結する機能)は、各アプリ専用のセッション/リポジトリで自由に進めてよい。

**各アプリ専用のセッション(別チャット・別リポジトリ・別worktreeなど)で作業を始める前に、必ずこの文書を先に読むこと。**

## 1. Firebase（共通データベース）

- プロジェクト: `ciel-autoworks`
- Realtime Database (asia-southeast1)
- 設定値(全アプリ共通。クライアント側に埋め込まれる公開情報であり、各ファイルに既に記載済み):

```js
const firebaseConfig = {
  apiKey: "AIzaSyAFJKdWkTa9WvX9W3ZA95P0BoDDugQUkuM",
  authDomain: "ciel-autoworks.firebaseapp.com",
  databaseURL: "https://ciel-autoworks-default-rtdb.asia-southeast1.firebasedatabase.app",
  projectId: "ciel-autoworks",
  storageBucket: "ciel-autoworks.firebasestorage.app",
  messagingSenderId: "266528519369",
  appId: "1:266528519369:web:d32686387aa9b0362cc918"
};
```

- Firebase Storageは2026年2月よりSparkプランで使用不可のため、画像もRealtime Database内にBase64で保存する方式に統一されている(ads.htmlのAA出品票画像を参照)。新しいツールで画像を保存する場合もこの方式を踏襲すること。

## 2. ルートノード対応表

2026-10-16に各担当セッションがコードを調査し、実際の読み書きに合わせて更新した。「その他の書き込み(実態)」列は本来の担当外からの書き込みで、整理するかどうかは未決定(下の「既知の課題」参照)。

| ノード | 主な書き込み元 | その他の書き込み(実態) | 読取専用で参照するツール | 内容 |
|---|---|---|---|---|
| `stock/{id}` | car-app | ハブ・pricing.html(棚卸CSVで未登録車のみ新規登録) | ads.html(在庫一覧。v1.33〜)、ju-history.html、pricetag.html、flyer.html(掲載車両の選択) | 在庫車両。フィールド例: `id, vin, name, color, year, shaken, sup, date, price, total, regNo, status1, status2, memo`。car-appはこのほか `cleaned, photographed, maintained, needsParts, isLoaner, addedAt, displayNote, deliveryClean, deliveryMaint, docDone` も持つ |
| `stockArchive/{id}` | car-app | - | ハブ、ju-history.html、pricing.html | 納車済みアーカイブ(stockと同形式＋`archivedAt`) |
| `cars/{id}` | car-app | - | - | 代車・レンタカー。`stockId`経由でstockを都度参照(vin・colorは持たない) |
| `juHistory/{normVin}/{source_kai_shuppinNo}` | ju-history.html | pricing.html(精算書CSV取込) | ハブ、pricing.html | オークション出品・落札履歴。フィールド例: `source, kai, date, shuppinNo, carName, grade, nendo, category('shuppin'|'rakusatsu'), status, pref, sharyDai, recycle, shuppinRyo, seiyakuRyo, rakusatsuRyo, jizei, maxBid` |
| `adsData/{carsensor,goonet,quicknetwork,autoserver,jutentori}` | ads.html | - | pricetag.html(carsensor, goonet)、flyer.html(carsensor, goonet) | 各広告サイトの掲載データ |
| `adsData/tanaoroshi/{normVin}` | ads.html | car-app・ハブ・pricing.html(CSV取込時に未登録分を追加) | ju-history.html、pricing.html | 棚卸CSV由来データ(`car_name, color, shaken, shiire_date`) |
| `adsData/lastUpdated, noGyohanVins, manualShufuku` | ads.html | - | pricetag.html(manualShufuku)、flyer.html(manualShufuku) | 更新日時・業販除外フラグ・修復歴手動設定 |
| `adsData/aaPhotoCounts` | ads.html | car-app・ハブ・pricing.html・pricetag.html(いずれも全体を上書き) | ju-history.html | AA画像枚数 |
| `aaPhotos/{VIN}` | ads.html | car-app・ハブ・pricing.html・pricetag.html | ju-history.html | AA出品票画像(`img1, img2`)。通信量対策で一覧データとは別パスに保存し、モーダルを開いた時だけ取得する |
| `pricingData/{normVin}` | pricing.html | ハブ(`manualCost`) | ads.html(`retailTarget`のみ。v1.33〜)、pricetag.html(`body`)、flyer.html(`body`。支払総額=body+10万円)、ハブ | 車体価格(`body`)・`extraCost, as, marginTotal, marginWholesale, retailTarget, manualCost, manualRecycle, updatedAt` |
| `priceTagTemplates/{templateId}` | pricetag.html | - | - | 価格表テンプレート設定 |
| `priceTagData/{vin}/manual/{field}`, `printTarget`, `templateId`, `lastPrintedAt`, `lastPrintedPrice` | pricetag.html | - | - | 一覧の個別編集値・印刷対象・使用テンプレート・最終印刷日時と価格 |
| `flyerLayouts/{layoutId}` | flyer.html | - | - | チラシのレイアウト(配置・サイズ・番号・コマデザイン等の小さなJSON。画像なし)。キーは各レイアウトのID(英数字)で、数は可変。削除したレイアウトは `{deleted:true, ua}` の目印を残す(他のパソコンへ削除を伝えるため)。各レイアウトJSONに、選択した車台番号(normVin)の配列 `cars`(掲載順)を含む。変更の0.5秒後に自動保存 |
| `flyerCells/{A\|B\|C\|D}` | flyer.html | - | - | コマテンプレ(コマデザインA〜D)。`{json:文字列, ua:更新時刻}`。jsonに縦横比・項目配置・外枠設定・飾り画像のIDを含む(画像本体は `flyerImages`)。変更の1.5秒後に自動保存 |
| `flyerImages/{imgId}` | flyer.html | - | - | チラシのレイアウトに置く画像(WebP/Base64、長辺縮小済み)。「保存」ボタンを押した時だけ書き込む |
| `flyerCarPhotos/{normVin}` | flyer.html | - | - | チラシ用の背景透過車両画像(1台1枚)。`{name, data:Base64(WebP/PNG・縮小済み), ua}`。AA画像(`aaPhotos`)とは別物 |
| `flyerCarPhotoIndex/{normVin}` | flyer.html | - | - | `flyerCarPhotos` に画像があるかの目印 `{ua}`(一覧表示用に軽く読むため) |
| `newsLog` | car-app | ハブ・pricing.html(新規登録の通知) | - | 変更履歴(car-appが3ヶ月より古いものを自動削除) |
| `inspections`, `inspectionPhotos`, `loanHistories` | car-app | - | - | car-app専用(点検シート・貸出履歴) |
| `listings`, `listingPhotos` | (廃止) | - | car-app(読取・削除のみ) | v1.90で廃止した旧方式の残り |

**既知の課題(2026-10-16調査。対応方針は未決定)**
- `adsData/aaPhotoCounts` を複数ツールが全体上書きしており、同時に使うと他ツールの更新分が消えるおそれがある。ads.htmlの「データ更新」は `adsData` 全体を丸ごと上書きする。
- `aaPhotos` のキーがツールによって違う(car-app: 生のVIN、ads.html: canonicalVinMapの正式番号、その他: normVin)。
- ハブとpricing.htmlは、棚卸CSVから `stock` に新規登録する時にVINを正規化して保存している(3章の「生の表記のまま」と異なる)。

**注意**: `stock.price`・`stock.total`は棚卸CSV由来だが用途が未確認。推測で原価・粗利計算等に使わないこと(使う場合は必ずユーザーに確認する)。

## 3. 車台番号(VIN)の正規化・突合ルール

```js
function normVin(v){ return (v||'').replace(/\s+/g,'').toUpperCase(); }
```

- 全ツール共通。車両データを他ツールのデータと突合する時のキーは必ずこの関数を通す。
- `stock/{id}/vin`自体は正規化前の生の表記(ハイフン区切りの型式-シリアル形式が基本)のまま保存される。car-app側の既存動作を変えないため、正規化は参照側(突合する側)で行う。
- **ads.htmlのみ**、型式の表記ゆれ(例: `ZWR80-XXXXXXX`と`ZWR80G-XXXXXXX`)を追加で吸収する`canonicalVinMap`という上位ロジックを持つ(同一製造番号で型式が前方一致する場合は同一車両とみなす)。
- **pricetag.html等、新しいツールはこの「あいまい一致」は踏襲せず、normVinの完全一致のみを使う方針**(2026-10-02決定)。表記ゆれによる取りこぼしは既知の制約として残る。新しいツールを作る際、どちらの方式を踏襲するかは実装前に確認すること。

## 4. 画面間リンク(URL連携)の規約

- 各ツールは別ファイルのまま、ヘッダーのボタンでURL遷移する方式(1ファイルへの統合はしない方針。理由: ID・変数名の重複が多く書き換え範囲が大きいため。2026-09-28決定)。
- 検索ボックスの入力中文字列は、リンクをタップした瞬間にURLの`?q=`パラメータへ反映される。遷移先のページは読み込み時にこのパラメータを見て検索欄に自動入力し、絞り込みも自動実行する。
- 実装済みのリンク: ads.html⇔pricing.html(相互)、ads.html⇔pricetag.html(相互)、pricing.html⇔pricetag.html(相互)。ciel-systemハブ→各ツール(「🏠 一覧へ」で戻るリンクは各ツール側に維持。car-appのみ外部URLで新しいタブを開く)。
- `?q=` の対応状況(2026-10-20更新): ads.html・pricing.html・pricetag.html(v1.28〜)は受け取りと送り出しの両方に対応。flyer.html(v1.35〜)は受け取り(③画面の検索欄に入れて絞り込み)と、ads・pricing・pricetagへの送り出しに対応。ju-history.html・ハブ・car-appは `?q=` を使わない。
- flyer.html のヘッダーに ads.html / pricing.html / pricetag.html へのリンクがある(クリック時に検索欄の文字を `?q=` で渡す)。逆方向(ads・pricing・pricetag → flyer.html)のリンクは未実装(追加する場合は各担当で。flyer側は `?q=` を受け取れる)。
- **新しいツール間リンクを追加する場合、または`?q=`以外の新しいクエリパラメータを導入する場合は、両端のツールに影響するため、どちらか片方のセッションだけで進めず、両方を確認してから実装すること。**

## 5. 新しいツールを作る/既存ノードを参照する時の原則

- 既存ノード(`stock`, `stockArchive`, `cars`, `juHistory`, `adsData`等)は基本「読み取り専用」として参照し、自分専用の新しいノード(例: `pricingData`, `priceTagData`)にだけ書き込む。
- 新しいツールを作る場合、ads.html・ju-history.htmlと同じ見た目・設計(ダークモード対応、カード/テーブルのCSSパターン、Firebase共有)を踏襲する。
- バージョン表記は修正のたびに+0.01する(新規ツールはv1.00開始)。
- 仕入先名(`stock.sup`)を画面に表示するときは `displaySupplier()` で変換する(表示だけの変換で、保存値は変えない): 「新潟県中古車販売商工組合」を含む→`JU新潟`、「トヨタユーゼック」または「TAA東北」を含む→`TAA東北`、それ以外はそのまま。実装済み: pricing.html・pricetag.html・flyer.html。car-appは未対応(2026-10-20依頼)。ハブ・ads・ju-historyは仕入先名を表示しない。

## 6. アプリ↔GitHubリポジトリ対応表

| ローカルファイル名 | 保存先リポジトリ | リポジトリ内パス |
|---|---|---|
| `carapp_index.html` | `cielautoworks/car-app` | `index.html` |
| `cielsystem_index.html` | `cielautoworks/ciel-system` | `index.html`(ハブページ) |
| `ads.html` | `cielautoworks/ciel-system` | `ads.html` |
| `ju-history.html` | `cielautoworks/ciel-system` | `ju-history.html` |
| `pricing.html` | `cielautoworks/ciel-system` | `pricing.html` |
| `pricetag.html` | `cielautoworks/ciel-system` | `pricetag.html`(2026-10-16にGitHub上の存在を確認済み) |
| `flyer.html` | `cielautoworks/ciel-system` | `flyer.html`(チラシ作成。2026-10-20新規。ハブにリンクあり) |

既知の未対応事項: `ciel-system`リポジトリ内に、正しい`ju-history.html`とは別にハイフンなしの`juhistory.html`という重複ファイルが存在する(過去のリネームミスの残骸)。削除は破壊的操作のため未実施。

## 7. GitHub連携(ローカルgit運用)

- 2026-10-16より、ユーザーのPC上のローカルリポジトリ + Claude Code(デスクトップアプリ)で編集・commit・pushする運用に移行。
  - `C:\cielautoworks\claude\document\app\repos\carapp` → `cielautoworks/car-app`
  - `C:\cielautoworks\claude\document\app\repos\cielsystem` → `cielautoworks/ciel-system`
- 認証はWindows版gitのGit Credential Manager(ブラウザでGitHubにログイン)を使う。トークンをチャットに貼る必要はない。
- 各アプリの修正はアプリごとに別のworktree(別ブランチ)で行い、mainへのマージとpushはユーザーの確認後に行う。
- 参考(旧運用): Claudeのクラウドサンドボックス環境からの直接pushは、セッション単位の「認可済みリポジトリ一覧」により拒否されていた(2026-09-16確認)。そのため「チャットで編集→ダウンロード→手動UP」で運用していた。

## 8. この文書の更新ルール

- 1〜4章(Firebase構造・ノード対応表・normVin規約・URL連携規約)の内容を変更する場合は、関係する全アプリのセッションに影響するため、**変更前に必ずこの文書を更新**し、関連する全アプリ側の実装にも反映されているか確認すること。
- 5〜7章は運用ルール・参考情報であり、随時更新してよい。
