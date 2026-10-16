# ciel-system 共通仕様書（アプリ間の「契約」部分のみ）

最終更新: 2026-10-16

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

| ノード | 主な書き込み元 | 他ツールからの参照 | 内容 |
|---|---|---|---|
| `stock/{id}` | car-app | ciel-systemハブ(棚卸CSVで未登録車のみ新規登録)、pricing.html/pricetag.html(読取専用) | 在庫車両。フィールド例: `id, vin, name, color, year, shaken, sup, date, price, total, regNo, status1, status2, memo` |
| `stockArchive/{id}` | car-app | pricing.html/pricetag.html(読取専用) | 納車済みアーカイブ(stockと同形式) |
| `cars/{id}` | car-app | - | 代車・レンタカー。`stockId`経由でstockを都度参照(vin・colorは持たない) |
| `juHistory/{normVin}/{source_kai_shuppinNo}` | ju-history.html | pricing.html/pricetag.html(読取専用) | オークション出品・落札履歴。フィールド例: `source, kai, date, shuppinNo, carName, grade, nendo, category('shuppin'|'rakusatsu'), status, pref, sharyDai, recycle, shuppinRyo, seiyakuRyo, rakusatsuRyo, jizei, maxBid` |
| `adsData/{carsensor,goonet,quicknetwork,autoserver,jutentori,tanaoroshi}` | ads.html | pricetag.html等(読取専用) | 各広告サイトの掲載データ・棚卸CSV由来データ |
| `adsData/lastUpdated, noGyohanVins, manualShufuku, aaPhotoCounts` | ads.html | 各ツール(読取専用) | 更新日時・業販除外フラグ・修復歴手動設定・AA画像枚数 |
| `aaPhotos/{VIN}` | ads.html | ju-history.html/car-app(閲覧専用) | AA出品票画像。通信量対策で一覧データとは別パスに保存し、モーダルを開いた時だけ取得する |
| `pricingData/{vin}` | pricing.html | pricetag.html(読取専用) | 車体価格(`body`)等 |
| `priceTagTemplates/{templateId}` | pricetag.html | - | 価格表テンプレート設定 |
| `priceTagData/{vin}/manual/{field}`, `priceTagData/{vin}/printTarget`, `priceTagData/{vin}/templateId` | pricetag.html | - | 一覧の個別編集値・印刷対象・使用テンプレート |

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
- 実装済みのリンク: ads.html⇔pricing.html(相互)。ciel-systemハブ→各ツール(「🏠 一覧へ」で戻るリンクは各ツール側に維持)。
- **新しいツール間リンクを追加する場合、または`?q=`以外の新しいクエリパラメータを導入する場合は、両端のツールに影響するため、どちらか片方のセッションだけで進めず、両方を確認してから実装すること。**

## 5. 新しいツールを作る/既存ノードを参照する時の原則

- 既存ノード(`stock`, `stockArchive`, `cars`, `juHistory`, `adsData`等)は基本「読み取り専用」として参照し、自分専用の新しいノード(例: `pricingData`, `priceTagData`)にだけ書き込む。
- 新しいツールを作る場合、ads.html・ju-history.htmlと同じ見た目・設計(ダークモード対応、カード/テーブルのCSSパターン、Firebase共有)を踏襲する。
- バージョン表記は修正のたびに+0.01する(新規ツールはv1.00開始)。

## 6. アプリ↔GitHubリポジトリ対応表

| ローカルファイル名 | 保存先リポジトリ | リポジトリ内パス |
|---|---|---|
| `carapp_index.html` | `cielautoworks/car-app` | `index.html` |
| `cielsystem_index.html` | `cielautoworks/ciel-system` | `index.html`(ハブページ) |
| `ads.html` | `cielautoworks/ciel-system` | `ads.html` |
| `ju-history.html` | `cielautoworks/ciel-system` | `ju-history.html` |
| `pricing.html` | `cielautoworks/ciel-system` | `pricing.html` |
| `pricetag.html` | `cielautoworks/ciel-system` | `pricetag.html`(2026-10-16にGitHub上の存在を確認済み) |

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
