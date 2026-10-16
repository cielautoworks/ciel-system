# ciel-system リポジトリ 作業ルール

## 最初に必ず読むこと
- 作業を始める前に `shared_contract.md`(アプリ間の共通仕様書)を必ず読むこと。
- `shared_contract.md` は car-app リポジトリにも同じ内容のものがある。更新した場合は両方を同じ内容にそろえること。

## このリポジトリのアプリ
| ファイル | 内容 |
|---|---|
| `index.html` | ciel-systemハブページ |
| `ads.html` | 広告掲載データ管理 |
| `ju-history.html` | オークション出品・落札履歴 |
| `pricing.html` | プライシング |
| `pricetag.html` | 価格表作成 |

- 各HTMLは互いのコードを参照しない独立ファイル。つながりはFirebaseとURL遷移(`?q=`)だけ。
- `juhistory.html`(ハイフンなし)は過去のリネームミスの残骸。参照も編集もしないこと。削除はユーザーの指示があるまで行わない。

## 作業の分担(worktree運用)
- 1つのセッション(worktree)では、担当する1つのアプリのファイルだけを編集する。担当外のファイルは読むだけにする。
- `shared_contract.md` の1〜4章(Firebase構造・ノード対応表・normVin・URL連携)に関わる変更が必要になった場合は、自分で実装せず、管理セッション(またはユーザー)に報告して判断を仰ぐ。
- 書き込んでよいFirebaseノードは、そのアプリ専用のノードだけ(`shared_contract.md` 2章・5章を参照)。

## 修正時のルール
- 修正のたびに、ファイル内のバージョン表記を+0.01する(新規ツールはv1.00から始める)。
- 修正後はJSの構文チェックを行う(例: `<script>` 部分を抜き出して `node --check`)。
- コミットメッセージは日本語で「ファイル名 vX.XX: 変更内容」の形式にする(例: `pricing.html v1.18: 検索欄の不具合を修正`)。

## GitHub
- リモート: `https://github.com/cielautoworks/ciel-system`(mainブランチ)
- mainへのマージとpushは、差分をユーザーに確認してもらってから行う。
