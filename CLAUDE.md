# CLAUDE.md

このリポジトリ (`ryutaroyano.github.io`) で作業する際の前提・制約・進め方をまとめる。

## リポジトリの構成

- ビルドステップなしの静的サイト（`package.json` なし）。素のHTML/CSS/JavaScriptのみ。
- GitHub Pagesでホスティング、カスタムドメインは `ryutaroyano.jp`。
- 意図的にパブリックリポジトリ。ソースをそのままpublicに置く構成のため、private/public分離のメリットがないと判断済み（この議論は再燃させなくてよい）。
- 主なコンテンツは2系統: デジタル名刺（`index.html` / `contact/staytoyoma/`）と、ゲーミフィケーション自己報酬アプリ（`skills/gamification.html`）。

## デジタル名刺（ホームページ／STAY TOYOMA）

NFC名刺・QRコードから開くことを想定した名刺ページ群。用途別に2ページあり、意図的に別デザイン・別リンク構成・別vCardにしている。**統合しない。**

- **`index.html`**（トップページ）: 個人としてのプロフィール名刺。SNS（Facebook / Instagram / YouTube）とメールへのリンクのみ。
- **`contact/staytoyoma/index.html`**: STAY TOYOMA宿守としての業務用名刺。電話・メール・各種SNS・note・地図・vCardダウンロードボタンを掲載。`<meta name="robots" content="noindex">` を付与し検索エンジンには出さない（今後名刺ページを増やす場合も同様の運用とする）。

### ファイル構成

```
index.html
images/profile.jpg
images/favicon.png
contact/staytoyoma/index.html
contact/staytoyoma/profile.jpg
contact/staytoyoma/Ryutaro_Yano_staytoyoma.vcf
```

### 設計上の決定事項

- **画像はリポジトリ内に置く**: 当初Google Drive上の画像をURL参照する構成だったが、公開設定の不安定さを避けるため `images/` 配下にコミットする方式へ移行済み。この方針を維持し、Google Drive URL参照には戻さない。
- **vCardはページごとに個別ファイル**: ページのリンク構成（SNSアカウント等）を変更したら、対応するvCardの内容も必ず合わせて更新すること。

### インフラ構成

| 項目 | 内容 |
|---|---|
| ホスティング | GitHub Pages（無料） |
| ドメイン取得 | お名前.com |
| ドメイン | ryutaroyano.jp |
| DNS管理 | お名前.com（NSは01〜04.dnsv.jp） |
| メール | Google Workspace（MXレコード: smtp.google.com） |
| HTTPS | GitHub Pages の「Enforce HTTPS」設定を使用 |

お名前.com DNSレコード（設定済み）:

| TYPE | ホスト名 | VALUE |
|---|---|---|
| NS | ryutaroyano.jp | 01〜04.dnsv.jp |
| MX | ryutaroyano.jp | smtp.google.com（優先度1） |
| TXT | ryutaroyano.jp | google-site-verification=... |
| CNAME | www | GitHubユーザー名.github.io |
| A | （空欄） | 185.199.108.153 / .109.153 / .110.153 / .111.153 |

### 未確認・要フォローアップ事項

- GitHub Pages の Enforce HTTPS が実際にオンになっているか
- `contact/staytoyoma/` の地図埋め込みiframeがGitHub Pages上で問題なく表示されるか（APIキーなしの表示制限の有無）
- QRコード生成・NFCタグへのvCard書き込みの進捗

## ゲーミフィケーション自己報酬アプリ (`skills/gamification.html`)

個人用の自己管理アプリ。詳細な設計思想は [README.md](README.md) と、ハンドオフ用Googleドキュメント（id: `1d7KgL8ng8eCPcd-En1W6EBUqAsq1qbhm9c5lglCuCXk`）を参照。

### 変更してはいけない設計上の制約

- **罰則なし設計**: やらなかったことによる減点機能は追加しない。
- **サイコロの目の線対称性**: `BOARD[a][b] === BOARD[b][a]` を維持する。2つのサイコロに順序の区別がないという前提のため。
- **オフラインファースト**: `localStorage` を正とし、GASへは pending queue → flush方式で同期する。
- **列構成はHTML側が主導**: GAS側のスプレッドシート列はHTML側のカテゴリ定義に追従させる設計。

### バックエンド (Google Apps Script)

- GASコード本体はこのgitリポジトリの外（Apps Scriptエディタ）で管理されている。コード変更時は**必ず全文を出力する**こと。部分的なスニペットでの差分提示はしない（過去にコピペミス・マージ漏れが起きたため）。
- **全文出力は必ずチャット本文（べた貼り）で行い、このリポジトリ内にファイルとして書き出さない。** GASコードには `SECRET`（合言葉）が平文で入っており、このリポジトリはpublicなので、`git pull`される場所に置くと合言葉が漏洩する（実際に一度誤って書き出してしまった経緯があるため、繰り返さないこと）。
- GASの関数名に末尾 `_` を付けると、エディタの実行対象ドロップダウンから隠れる（private関数の慣習）。OAuthスコープの認可トリガーなど、UIから直接実行したい処理は末尾`_`を付けない一時関数を用意する必要がある。
- コード変更を `/exec` の本番URLに反映させるには、Apps Scriptエディタで「デプロイを管理」から新しいバージョンを選択する必要がある（保存だけでは反映されない）。
- GASの実行ログはUIから詳細なスタックトレースを追いにくいことがある。診断が必要な場合は、Driveにエラー内容をJSONで書き出すログ機構（`gamification_last_error.json` 相当）を仕込み、Google Drive MCPツールで直接読む方が早い。

### フロントエンドの設計判断

- 日付跨ぎ等の「時間経過で状態が変わる」処理は、`setInterval` によるポーリングではなく、`visibilitychange` やユーザー操作（ボタン押下など）をトリガーにしたイベント駆動でチェックする。ポーリングは無駄が多いという理由でユーザーが明示的に却下した経緯がある。
- GASへ複数リクエストがほぼ同時に飛ぶと `LockService` 競合で片方が失敗することがある。並行実行させず、Promiseチェーンで直列化すること。

## 作業の進め方（このユーザーとの合意事項）

- コミットメッセージ・README・ドキュメントは日本語で書く。
- ローカルの実データが既に本番環境と同期済みである場合、ユーザーの明示的な同意があれば本番のGoogle Sheets/Driveに対して直接検証してよい（MCPツール経由の読み取りなど）。
- PRの作成は `gh` CLI経由で進めてよいが、**マージは必ず明示的な指示を得てから**実行する。
- リポジトリのvisibility変更やアカウント認可（OAuth同意）など、アクセス制御に関わる操作はユーザー自身に行ってもらう。
