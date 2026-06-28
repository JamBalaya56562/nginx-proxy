# nginx-proxy/nginx-proxy — オープンIssue & PR 調査カタログ

> このドキュメントは [nginx-proxy/nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) のオープンなIssueとPRを棚卸しし、**自分たちで実装・解決していく候補**を整理した作業メモです。
> upstream へPRを出すためのものではなく、調査結果を忘れないための内部カタログです。

| 項目 | 値 |
|---|---|
| 調査日 | 2026-06-18 |
| 対象リポジトリ | [nginx-proxy/nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) |
| オープンIssue件数 | 292 件 |
| オープンPR件数 | 38 件 |
| 調査方法 | `gh` でメタデータ取得 → 並列エージェント(PR 6グループ深掘り + Issue 8チャンク分類 + 統合分析)でトリアージ |
| 生データ | `research/data/triage-result.json` (本カタログの元データ) |

**注意点 / 前提:**

- TLS/Let's Encrypt 周りの実体は別プロジェクト [acme-companion](https://github.com/nginx-proxy/acme-companion) の管轄。本リポジトリのTLS系Issueはテンプレ出力部分に限られる。
- 古いPR(2015〜2019)は当時の `nginx.tmpl` 構造から大きく変わっており、ほぼ全件リベース/再実装が前提。
- 各PRの `mergeState` は調査時点の `gh` 取得値。`UNKNOWN` はGitHubがマージ可否を再計算中だったもの。
- 分類(type / category / 実装容易度 / 推奨アクション)はエージェントによる一次トリアージであり、着手前に個別の再確認を推奨。

---

## 目次

1. [エグゼクティブサマリ](#エグゼクティブサマリ)
2. [実装ロードマップ(優先順)](#実装ロードマップ優先順)
3. [クイックウィン](#クイックウィン)
4. [横断テーマ](#横断テーマ)
5. [オープンPR カタログ(全38件)](#オープンpr-カタログ全38件)
6. [Issue トリアージ](#issue-トリアージ)
7. [統計・分布](#統計分布)

---

## エグゼクティブサマリ

- **オープンIssueは292件**だが、その大半は2015〜2018年に作成された古いもの。一次トリアージでは **約178件が陳腐化(likely-obsolete)** と判定。現行版で解決済み・仕様変更済みの可能性が高い。
- 実際に着手しやすいのは **feasible 52件 / good-first 4件**。設計合意が要るものが needs-discussion 46件。
- Issueの内訳は **質問・サポート 112件 / バグ 99件 / 機能要望 75件**。質問系は実装対象というよりドキュメント改善・FAQ化の候補。
- **オープンPRは38件**。既にコードがある分、Issueより着手効率が高い。推奨アクション別: needs-discussion 12 / reimplement-fresh 9 / revive-rebase 7 / likely-obsolete 6 / low-priority 4。
- PRのうち **価値high 8件**。特に `revive-rebase`(リベースすれば活かせる)7件 と、テスト/ドキュメント不足だけが障害の `status/pr-needs-tests` 系が狙い目。
- 最頻出テーマは **VIRTUAL_HOST / ルーティング、TLS/HTTPSリダイレクト、カスタム設定(nginx.tmpl)、Docker Compose/Swarm連携、プロキシ調整(body size/timeout/websocket)**。

→ **着手戦略**: まず下記[実装ロードマップ](#実装ロードマップ優先順)の高優先項目と[クイックウィン](#クイックウィン)から。コードが既にあるPRを軸に、それが解決する複数Issueをまとめてクローズするのが効率的。

---

## 実装ロードマップ(優先順)

統合分析が抽出した、**実際に着手すべき具体的な作業項目**。既存PRのコードがあるものを優先的に紐付けている。

### 1. VIRTUAL_UPSTREAM: コンテナ外サービスへのプロキシ対応

`優先度: high` ・ `工数: medium`

socatダミーコンテナという回避策が定着するほど需要が高い(#1474=6コメント, #998=11コメント, #663は2025年まで現役)。PR #1477に既存コードがあり、ラベルはstatus/pr-needs-tests + needs-docs。upstreamブロック生成に環境変数サポートを追加するスコープは明確。メンテナが進めるhost networking(#2222)と重複しないよう設計をすり合わせ、インデント崩れ修正・pytestテスト・ドキュメントを追加すれば着地可能。#1100は古くテスト全滅報告のため#1477ベースを推奨。

- 関連PR: [#1477](https://github.com/nginx-proxy/nginx-proxy/pull/1477), [#1100](https://github.com/nginx-proxy/nginx-proxy/pull/1100)
- 関連Issue: [#1474](https://github.com/nginx-proxy/nginx-proxy/issues/1474), [#1350](https://github.com/nginx-proxy/nginx-proxy/issues/1350), [#998](https://github.com/nginx-proxy/nginx-proxy/issues/998), [#663](https://github.com/nginx-proxy/nginx-proxy/issues/663)

### 2. upstream パラメータの汎用環境変数化(weight / backup / fail_timeout / ip_hash)

`優先度: high` ・ `工数: medium`

weight(#255), backup(#2453), ip_hash(#299/#871)と個別PRが乱立しているが、レビュアーが提案する汎用 SERVER_PARAMETERS 方式で一本化すれば保守性が高い。現行mainは既にkeepalive対応済みなので、同じupstream定義(nginx.tmpl L352付近)への拡張として実装範囲が明確。#255はGo text/templateの変数スコープ問題が指摘済みで参考になる。test_loadbalancingに統合テストを追加。

- 関連PR: [#255](https://github.com/nginx-proxy/nginx-proxy/pull/255), [#2453](https://github.com/nginx-proxy/nginx-proxy/pull/2453), [#299](https://github.com/nginx-proxy/nginx-proxy/pull/299), [#886](https://github.com/nginx-proxy/nginx-proxy/pull/886)
- 関連Issue: [#221](https://github.com/nginx-proxy/nginx-proxy/issues/221), [#419](https://github.com/nginx-proxy/nginx-proxy/issues/419), [#909](https://github.com/nginx-proxy/nginx-proxy/issues/909), [#886](https://github.com/nginx-proxy/nginx-proxy/issues/886), [#871](https://github.com/nginx-proxy/nginx-proxy/issues/871), [#306](https://github.com/nginx-proxy/nginx-proxy/issues/306)

### 3. client_max_body_size / proxy_buffer_size 等プロキシチューニングのドキュメント整備と環境変数化

`優先度: high` ・ `工数: medium`

#981(44コメント=全Issue最高)はアップロードサイズ上限の設定方法質問、#694はAWS大ヘッダ起因の502。多くはvhost.d/proxy.confで対応可能だが手順が知られていないためサポートが繰り返される。まずmkdocsに明確なレシピを追加(低コスト高効果)、その上で一部をvhost単位の環境変数として実装する。NGINX_*一括反映PR #1326は無効変数(NGINX_VERSION)混入の安全性問題があるため、許可リスト方式で限定的に再設計する。

- 関連PR: [#1326](https://github.com/nginx-proxy/nginx-proxy/pull/1326)
- 関連Issue: [#981](https://github.com/nginx-proxy/nginx-proxy/issues/981), [#694](https://github.com/nginx-proxy/nginx-proxy/issues/694), [#1331](https://github.com/nginx-proxy/nginx-proxy/issues/1331), [#1208](https://github.com/nginx-proxy/nginx-proxy/issues/1208), [#1229](https://github.com/nginx-proxy/nginx-proxy/issues/1229), [#1190](https://github.com/nginx-proxy/nginx-proxy/issues/1190), [#1247](https://github.com/nginx-proxy/nginx-proxy/issues/1247)

### 4. www⇔非www / ホストエイリアス リダイレクト機能の統一実装

`優先度: high` ・ `工数: high`

全Issue中で最大級の需要(#180=28コメント, #1204=24コメント, #958/#960も高エンゲージメント)。既存PRに#1563・#1369・#1779とコードが複数存在するが互いに競合・テンプレ全面改修に追従できていない。これらを単一の設計(例: 環境変数でエイリアス群と正規ホストを定義し301/308を生成)に集約し、ACME証明書取得との相互作用(MarkErik報告)を考慮した上でテストとmkdocsドキュメントを付ける。メンテナ(buchdag)は2023年に『ドキュメント+テストがあればマージしたい』と明言済み。

- 関連PR: [#1563](https://github.com/nginx-proxy/nginx-proxy/pull/1563), [#1369](https://github.com/nginx-proxy/nginx-proxy/pull/1369), [#1779](https://github.com/nginx-proxy/nginx-proxy/pull/1779)
- 関連Issue: [#180](https://github.com/nginx-proxy/nginx-proxy/issues/180), [#1204](https://github.com/nginx-proxy/nginx-proxy/issues/1204), [#960](https://github.com/nginx-proxy/nginx-proxy/issues/960), [#958](https://github.com/nginx-proxy/nginx-proxy/issues/958), [#1285](https://github.com/nginx-proxy/nginx-proxy/issues/1285), [#1254](https://github.com/nginx-proxy/nginx-proxy/issues/1254), [#444](https://github.com/nginx-proxy/nginx-proxy/issues/444), [#1095](https://github.com/nginx-proxy/nginx-proxy/issues/1095)

### 5. 間欠的502エラーの調査と再起動時reload堅牢化

`優先度: high` ・ `工数: high`

間欠502(#1383=8, #516=21, #992=15コメント)はサポート負荷の最大要因。PR #569の『reload前config test』はメンテナにより設計欠陥(再起動時に結局起動失敗、新規VIRTUAL_HOSTが無視される)を指摘されているため、その方式は採らず、upstream解決タイミングとdocker-genのデバウンス/イベント処理の観点から根本調査する。再現手順の確立とドキュメント(既知の緩和策)整備を先行させる。

- 関連PR: [#569](https://github.com/nginx-proxy/nginx-proxy/pull/569)
- 関連Issue: [#1383](https://github.com/nginx-proxy/nginx-proxy/issues/1383), [#516](https://github.com/nginx-proxy/nginx-proxy/issues/516), [#490](https://github.com/nginx-proxy/nginx-proxy/issues/490), [#451](https://github.com/nginx-proxy/nginx-proxy/issues/451), [#420](https://github.com/nginx-proxy/nginx-proxy/issues/420), [#992](https://github.com/nginx-proxy/nginx-proxy/issues/992)

### 6. OCSPステープリング用resolverと RESOLVERS 上書きバグ修正

`優先度: medium` ・ `工数: low`

#1256(11コメント)はOCSPステープリングにresolver未定義で機能しない問題、#2699はRESOLVERSが上書きされドキュメントと矛盾するバグ。現行mainはRESOLVERSをグローバルで一度だけ読み込む(nginx.tmpl L39, L577)構造で、典型的なgood-firstスコープ。TLS実用性が高く小変更で解決できる。

- 関連PR: —
- 関連Issue: [#1256](https://github.com/nginx-proxy/nginx-proxy/issues/1256), [#2699](https://github.com/nginx-proxy/nginx-proxy/issues/2699)

### 7. HTTPSリダイレクト server ブロックへの vhost.d include 追加

`優先度: medium` ・ `工数: medium`

PR #1618に既存コードあり(type/fix・type/feat・type/testラベル付きで品質要件を満たす)。現行nginx.tmplを確認するとHTTPリダイレクトの location /(L988付近)にはvhost.d includeが無く、SSL serverブロックにのみ存在する非対称が確認できる。#1136(HSTSがlocationで無効化)等とも関連。コンフリクト解消のみで着地できる。

- 関連PR: [#1618](https://github.com/nginx-proxy/nginx-proxy/pull/1618)
- 関連Issue: [#1613](https://github.com/nginx-proxy/nginx-proxy/issues/1613), [#1136](https://github.com/nginx-proxy/nginx-proxy/issues/1136)

### 8. vhost.d / htpasswd の closest マッチング精度改善

`優先度: medium` ・ `工数: medium`

#1558(Draft)はclosest関数の部分一致バグ(b.cがa.b.c.dに誤マッチ)を解消するもの。docker-gen側にclosestSuffix関数を追加する前提でメンテナ(buchdag)が2025-01に前向き回答済み。docker-gen実装とセットで進める必要があるため中エフォート。#1309のフォールバックロジック改善も同時に解決。

- 関連PR: [#1558](https://github.com/nginx-proxy/nginx-proxy/pull/1558), [#1146](https://github.com/nginx-proxy/nginx-proxy/pull/1146)
- 関連Issue: [#1309](https://github.com/nginx-proxy/nginx-proxy/issues/1309), [#830](https://github.com/nginx-proxy/nginx-proxy/issues/830)

### 9. CORS + Basic Auth 併用の opt-in 実装統一

`優先度: medium` ・ `工数: medium`

#1176と#1779は同一課題(Basic Auth時にCORSプリフライトOPTIONSを通す)への別アプローチ。メンテナ(tkw1536)が無条件OPTIONS除外の情報漏洩リスクを指摘し、環境変数によるopt-in+README明記を要望。両PRを統合し安全なopt-inフラグ方式で再設計、テストを追加する。

- 関連PR: [#1176](https://github.com/nginx-proxy/nginx-proxy/pull/1176), [#1779](https://github.com/nginx-proxy/nginx-proxy/pull/1779)
- 関連Issue: [#1150](https://github.com/nginx-proxy/nginx-proxy/issues/1150), [#1125](https://github.com/nginx-proxy/nginx-proxy/issues/1125), [#1087](https://github.com/nginx-proxy/nginx-proxy/issues/1087), [#1070](https://github.com/nginx-proxy/nginx-proxy/issues/1070)

### 10. Dockerヘルスチェック連携によるupstream除外(opt-in)

`優先度: medium` ・ `工数: high`

#987は2025年11月まで更新の現役要求、#419も実運用で重要。Dockerのhealthcheck状態をdocker-genで参照し、unhealthyなコンテナをupstreamから除外するopt-in機能。現代のコンテナ運用に合致し間欠502(#1383系)の緩和にもつながる。docker-gen側のラベル/状態連携が必要なため中〜高エフォート。

- 関連PR: —
- 関連Issue: [#987](https://github.com/nginx-proxy/nginx-proxy/issues/987), [#419](https://github.com/nginx-proxy/nginx-proxy/issues/419), [#968](https://github.com/nginx-proxy/nginx-proxy/issues/968), [#979](https://github.com/nginx-proxy/nginx-proxy/issues/979)

### 11. Docker Swarm 制限事項と推奨構成のドキュメント化

`優先度: low` ・ `工数: low`

#97は55コメントで全Issue最高エンゲージメント。完全なSwarm対応は大規模で別アーキテクチャを要するが、現状の制限・既知の回避策・推奨構成をmkdocsにまとめるだけでも繰り返されるサポート質問を大幅に減らせる。コード変更なしの高ROI。

- 関連PR: —
- 関連Issue: [#97](https://github.com/nginx-proxy/nginx-proxy/issues/97), [#814](https://github.com/nginx-proxy/nginx-proxy/issues/814), [#727](https://github.com/nginx-proxy/nginx-proxy/issues/727), [#650](https://github.com/nginx-proxy/nginx-proxy/issues/650), [#648](https://github.com/nginx-proxy/nginx-proxy/issues/648), [#664](https://github.com/nginx-proxy/nginx-proxy/issues/664)

### 12. ログローテーション / IP匿名化 / per-vhost access_log の現行構造への再実装

`優先度: low` ・ `工数: medium`

現行mainは既にLOG_JSON・log_format・access_logテンプレートを備える。これに対し#1455(ログローテーション, 2025年まで現役)、#1135(IP匿名化ログ), #273(per-vhost access_log, PR #273に旧コードあり)を現行のaccess_logテンプレート(nginx.tmpl L548付近)に合わせ再実装する。古いPRはそのままでは競合するため再実装扱い。

- 関連PR: [#273](https://github.com/nginx-proxy/nginx-proxy/pull/273), [#1135](https://github.com/nginx-proxy/nginx-proxy/pull/1135)
- 関連Issue: [#1455](https://github.com/nginx-proxy/nginx-proxy/issues/1455), [#654](https://github.com/nginx-proxy/nginx-proxy/issues/654), [#820](https://github.com/nginx-proxy/nginx-proxy/issues/820)

### 13. TCP/UDP stream モジュール対応の再設計

`優先度: low` ・ `工数: high`

#792は2026年まで更新の最も活発な機能要求、#2637/#929も継続的関心。PR #2608のアプローチ(単一テンプレを条件分岐で2コンテキスト動作)はメンテナに保守不能と否定済み。複数テンプレートを単一docker-genプロセスで描画する方向で新規設計が必要。長期的価値は高いが大型。

- 関連PR: [#2608](https://github.com/nginx-proxy/nginx-proxy/pull/2608)
- 関連Issue: [#792](https://github.com/nginx-proxy/nginx-proxy/issues/792), [#2637](https://github.com/nginx-proxy/nginx-proxy/issues/2637), [#929](https://github.com/nginx-proxy/nginx-proxy/issues/929), [#651](https://github.com/nginx-proxy/nginx-proxy/issues/651), [#699](https://github.com/nginx-proxy/nginx-proxy/issues/699)

---

## クイックウィン

小さく確度が高い、低コストで着手できる項目。

- **PR #1618** — HTTPSリダイレクトserverブロックへのvhost.d include追加。type/fix・feat・testラベル付きで品質要件を満たしコードも揃っている。現行nginx.tmplの非対称(HTTP redirect側にincludeが無い)を解消するだけ。コンフリクト解消のみで着地可能。
- **Issue #2699** — RESOLVERSが上書きされドキュメントと矛盾するバグ。nginx.tmpl L39/L577周辺の小修正で対応できる典型的good-first。
- **Issue #1256** — OCSPステープリング用resolver未定義の修正。TLS実用性が高くnginx.tmplに数行追加するだけ。
- **PR #2697** — LETSENCRYPT_CERTNAMEでホスト管理certbot証明書を使う機能。コンフリクト無し・差分小(eff low)だがブランチ保護でBLOCKED中。メンテナのCERTBOT_MANAGED_CERTIFICATE案と設計をすり合わせれば着地可能。
- **PR #2567** — 複数ネットワークタグ対応。差分が小さく(eff low)、テスト/ドキュメントを足すだけ。現行のネットワーク選択ロジックへの素直な拡張。
- **PR #2453** — BACKUP_SERVERでupstreamをbackup指定する機能。MERGEABLE(Draftによりblocked)。汎用SERVER_PARAMETERS化(roadmap項目)の一部として取り込めば低コスト。
- **PR #1430** — IPv6サポートと正しいクライアントIPに関するドキュメント追記。現行mainは既にIPv6/trust-downstream-proxy対応済みなので、配置をdocs/のmkdocs構成に合わせ直して再投入するだけ。
- **Issue #1431** — HTTPSリダイレクトのステータスコード(301 vs 307)問題。現行mainには既にNON_GET_REDIRECT(nginx.tmpl L37)が存在するため、ドキュメント案内のみで解決する可能性が高い。要切り分け。
- **Issue #891** — X-Request-ID(UUID)付与。デバッグ/トレーシングに有用でnginx.tmplへ数行追加するgood-firstレベル。JSONログformatには既にrequest_idフィールドがある。
- **Issue #1148** — 複数ポート公開コンテナでのHTTPポート自動選択(7コメント, 2024年更新)。VIRTUAL_PORT省略を可能にする実用改善でスコープが明確。

---

## 横断テーマ

Issue と PR をまたぐ主要テーマ。同種の要望が複数年にわたり繰り返されているものを束ねた。

### External / non-container upstream support (VIRTUAL_UPSTREAM)

Dockerコンテナ外のサービス（ホスト上のサービスや外部サーバー）へプロキシしたいという需要が最も構造的に大きいテーマ。環境変数でupstreamを直接指定する機能要求がPRとIssueの両方で長年繰り返されており、socatダミーコンテナ等の回避策が定着している。メンテナはhost networking対応(#2222)を別途進めており方向性のすり合わせが必要。

- 関連PR: [#1477](https://github.com/nginx-proxy/nginx-proxy/pull/1477), [#1100](https://github.com/nginx-proxy/nginx-proxy/pull/1100)
- 関連Issue: [#1474](https://github.com/nginx-proxy/nginx-proxy/issues/1474), [#1350](https://github.com/nginx-proxy/nginx-proxy/issues/1350), [#998](https://github.com/nginx-proxy/nginx-proxy/issues/998), [#663](https://github.com/nginx-proxy/nginx-proxy/issues/663), [#1485](https://github.com/nginx-proxy/nginx-proxy/issues/1485)

### ホストエイリアス / www⇔非www リダイレクト

www→非www（およびその逆）や複数ホスト名を1つのvhostに集約しつつ正規ホストへ301する機能が、全Issue中で突出して高いエンゲージメント(#180は28コメント、#1204は24コメント)を集める最重要テーマ。VIRTUAL_HOST_ALIASやredirect系PRが複数あるがテンプレ全面改修との競合とテスト/ドキュメント不足で停滞。ACME証明書取得との相互作用バグ報告もあり設計を一本化する必要がある。

- 関連PR: [#1563](https://github.com/nginx-proxy/nginx-proxy/pull/1563), [#1369](https://github.com/nginx-proxy/nginx-proxy/pull/1369), [#1779](https://github.com/nginx-proxy/nginx-proxy/pull/1779)
- 関連Issue: [#180](https://github.com/nginx-proxy/nginx-proxy/issues/180), [#1204](https://github.com/nginx-proxy/nginx-proxy/issues/1204), [#960](https://github.com/nginx-proxy/nginx-proxy/issues/960), [#958](https://github.com/nginx-proxy/nginx-proxy/issues/958), [#1285](https://github.com/nginx-proxy/nginx-proxy/issues/1285), [#1254](https://github.com/nginx-proxy/nginx-proxy/issues/1254), [#1264](https://github.com/nginx-proxy/nginx-proxy/issues/1264), [#444](https://github.com/nginx-proxy/nginx-proxy/issues/444), [#1095](https://github.com/nginx-proxy/nginx-proxy/issues/1095)

### TLS/HTTPS設定（リダイレクト挙動・resolver・OCSP・HSTS）

証明書管理自体はacme-companionの責務だが、nginx-proxy本体側のTLS設定（HTTPSリダイレクトのステータスコード、OCSPステープリング用resolver未定義、HSTSがlocation設定で無効化される、上流ダウン時の誤リダイレクト）に関する報告が多い。多くはnginx.tmplの小規模修正で対応可能。現行mainではHSTSは既にserver context、NON_GET_REDIRECTも実装済みのため、未解決分の切り分けが必要。

- 関連PR: [#1618](https://github.com/nginx-proxy/nginx-proxy/pull/1618), [#2697](https://github.com/nginx-proxy/nginx-proxy/pull/2697), [#943](https://github.com/nginx-proxy/nginx-proxy/pull/943), [#454](https://github.com/nginx-proxy/nginx-proxy/pull/454), [#245](https://github.com/nginx-proxy/nginx-proxy/pull/245)
- 関連Issue: [#1256](https://github.com/nginx-proxy/nginx-proxy/issues/1256), [#1136](https://github.com/nginx-proxy/nginx-proxy/issues/1136), [#1431](https://github.com/nginx-proxy/nginx-proxy/issues/1431), [#992](https://github.com/nginx-proxy/nginx-proxy/issues/992), [#1613](https://github.com/nginx-proxy/nginx-proxy/issues/1613), [#2709](https://github.com/nginx-proxy/nginx-proxy/issues/2709), [#2707](https://github.com/nginx-proxy/nginx-proxy/issues/2707), [#2615](https://github.com/nginx-proxy/nginx-proxy/issues/2615)

### vhost.d / htpasswd のマッチングとincludeの一貫性

カスタム設定ファイル(vhost.d)とhtpasswdのフォールバック・closestマッチングの正確性、およびHTTPSリダイレクトserverブロックでもvhost.d includeを効かせたいという要求群。closest関数の部分一致バグ(b.cがa.b.c.dにマッチ)はdocker-gen側のclosestSuffix実装待ち。VIRTUAL_HTPASSWDによる環境変数経由の認証設定要求もある。

- 関連PR: [#1558](https://github.com/nginx-proxy/nginx-proxy/pull/1558), [#1618](https://github.com/nginx-proxy/nginx-proxy/pull/1618), [#1308](https://github.com/nginx-proxy/nginx-proxy/pull/1308), [#1146](https://github.com/nginx-proxy/nginx-proxy/pull/1146)
- 関連Issue: [#1309](https://github.com/nginx-proxy/nginx-proxy/issues/1309), [#1136](https://github.com/nginx-proxy/nginx-proxy/issues/1136), [#830](https://github.com/nginx-proxy/nginx-proxy/issues/830), [#1136](https://github.com/nginx-proxy/nginx-proxy/issues/1136)

### load balancing / upstream チューニング（weight, backup, ip_hash, keepalive, healthcheck）

upstreamブロックにweight・backup・fail_timeout・ip_hashなどのnginxオプションを環境変数で付与したい要求と、Dockerのhealthcheck状態に基づくupstream除外の要求。現行mainは既にkeepalive対応済み。汎用的なSERVER_PARAMETERS化で複数PR(#255 weight, #2453 backup, #299/#871 ip_hash)を一本化できる可能性が高い。

- 関連PR: [#255](https://github.com/nginx-proxy/nginx-proxy/pull/255), [#2453](https://github.com/nginx-proxy/nginx-proxy/pull/2453), [#299](https://github.com/nginx-proxy/nginx-proxy/pull/299), [#886](https://github.com/nginx-proxy/nginx-proxy/pull/886)
- 関連Issue: [#221](https://github.com/nginx-proxy/nginx-proxy/issues/221), [#419](https://github.com/nginx-proxy/nginx-proxy/issues/419), [#987](https://github.com/nginx-proxy/nginx-proxy/issues/987), [#871](https://github.com/nginx-proxy/nginx-proxy/issues/871), [#909](https://github.com/nginx-proxy/nginx-proxy/issues/909), [#886](https://github.com/nginx-proxy/nginx-proxy/issues/886), [#306](https://github.com/nginx-proxy/nginx-proxy/issues/306)

### 間欠的502/504エラーと再起動時のreload堅牢性

コンテナ再起動時のupstream解決タイミングやreloadの堅牢性に起因する散発的502エラーが全期間で最も多いサポート負荷源(#1383は8コメント、#516は21コメント、#992は15コメント)。reload前のconfigテスト(#569)はメンテナに設計欠陥を指摘され却下方向だが、根本のタイミング問題は依然未解決。

- 関連PR: [#569](https://github.com/nginx-proxy/nginx-proxy/pull/569), [#379](https://github.com/nginx-proxy/nginx-proxy/pull/379)
- 関連Issue: [#1383](https://github.com/nginx-proxy/nginx-proxy/issues/1383), [#516](https://github.com/nginx-proxy/nginx-proxy/issues/516), [#490](https://github.com/nginx-proxy/nginx-proxy/issues/490), [#451](https://github.com/nginx-proxy/nginx-proxy/issues/451), [#420](https://github.com/nginx-proxy/nginx-proxy/issues/420), [#2378](https://github.com/nginx-proxy/nginx-proxy/issues/2378), [#2302](https://github.com/nginx-proxy/nginx-proxy/issues/2302), [#992](https://github.com/nginx-proxy/nginx-proxy/issues/992)

### プロキシチューニングの環境変数化（body_size, buffer, timeout, nginx_* 変数）

client_max_body_size(#981は44コメントで全Issue最高)・proxy_buffer_size(#694)・各種timeoutをvhost単位や環境変数で設定したいという最頻出の実用要求。NGINX_*環境変数を一括でvhostに反映するPR(#1326)があるが、nginx:alpineが出すNGINX_VERSION等の無効変数混入という安全性問題が未解決。

- 関連PR: [#1326](https://github.com/nginx-proxy/nginx-proxy/pull/1326)
- 関連Issue: [#981](https://github.com/nginx-proxy/nginx-proxy/issues/981), [#694](https://github.com/nginx-proxy/nginx-proxy/issues/694), [#1331](https://github.com/nginx-proxy/nginx-proxy/issues/1331), [#1208](https://github.com/nginx-proxy/nginx-proxy/issues/1208), [#1229](https://github.com/nginx-proxy/nginx-proxy/issues/1229), [#1190](https://github.com/nginx-proxy/nginx-proxy/issues/1190), [#1247](https://github.com/nginx-proxy/nginx-proxy/issues/1247), [#918](https://github.com/nginx-proxy/nginx-proxy/issues/918)

### TCP/UDP / stream モジュール（非HTTPプロトコル）対応

nginx streamモジュールによるTCP/UDPプロキシとSSLパススルーの要求(#792は2026年まで更新の活発Issue)。PR #2608は単一nginx.tmplを条件分岐で2コンテキスト動作させるアプローチをメンテナに長期保守不能と否定された。複数テンプレを単一docker-genで描画する方向での再設計が必要な大型テーマ。

- 関連PR: [#2608](https://github.com/nginx-proxy/nginx-proxy/pull/2608)
- 関連Issue: [#792](https://github.com/nginx-proxy/nginx-proxy/issues/792), [#2637](https://github.com/nginx-proxy/nginx-proxy/issues/2637), [#929](https://github.com/nginx-proxy/nginx-proxy/issues/929), [#651](https://github.com/nginx-proxy/nginx-proxy/issues/651), [#699](https://github.com/nginx-proxy/nginx-proxy/issues/699)

### ログ・デバッグ・可観測性

per-vhostのaccess_logカスタマイズ、IP匿名化ログ、生成configへのデバッグコメント挿入、X-Request-ID付与、ログローテーション等。現行mainは既にLOG_JSON/log_formatとdebug endpointを備えるため、未対応分(ローテーション#1455、匿名化#1135、per-vhost access_log #273)を現行構造に合わせ再実装する余地がある。

- 関連PR: [#273](https://github.com/nginx-proxy/nginx-proxy/pull/273), [#1135](https://github.com/nginx-proxy/nginx-proxy/pull/1135), [#745](https://github.com/nginx-proxy/nginx-proxy/pull/745)
- 関連Issue: [#1455](https://github.com/nginx-proxy/nginx-proxy/issues/1455), [#654](https://github.com/nginx-proxy/nginx-proxy/issues/654), [#647](https://github.com/nginx-proxy/nginx-proxy/issues/647), [#820](https://github.com/nginx-proxy/nginx-proxy/issues/820), [#891](https://github.com/nginx-proxy/nginx-proxy/issues/891)

### カスタム設定の注入ポイント拡張（http context / load_module / static files）

http context へのinclude(#2377)、conf.dに置けないload_moduleディレクティブ用のインクルードポイント(#1377)、静的ファイル配信バックエンド(VIRTUAL_ROOT #1037, php-fpm #2069)など、テンプレの拡張ポイントを増やす要求群。静的ファイル系はボリュームを本体に手動マウントする構造的制約があり、メンテナは別nginxコンテナ併設に対する優位性を疑問視。

- 関連PR: [#2377](https://github.com/nginx-proxy/nginx-proxy/pull/2377), [#561](https://github.com/nginx-proxy/nginx-proxy/pull/561), [#1037](https://github.com/nginx-proxy/nginx-proxy/pull/1037), [#2069](https://github.com/nginx-proxy/nginx-proxy/pull/2069), [#1342](https://github.com/nginx-proxy/nginx-proxy/pull/1342)
- 関連Issue: [#811](https://github.com/nginx-proxy/nginx-proxy/issues/811), [#794](https://github.com/nginx-proxy/nginx-proxy/issues/794), [#773](https://github.com/nginx-proxy/nginx-proxy/issues/773), [#688](https://github.com/nginx-proxy/nginx-proxy/issues/688), [#663](https://github.com/nginx-proxy/nginx-proxy/issues/663), [#1377](https://github.com/nginx-proxy/nginx-proxy/issues/1377), [#1415](https://github.com/nginx-proxy/nginx-proxy/issues/1415)

### Docker Swarm / 複数ネットワーク / 複数ポート

Swarmサポート(#97は55コメントで全Issue最高エンゲージメント)、複数ネットワーク/ポートを公開するコンテナでのHTTPポート自動選択(#1148)、ネットワークタグ複数指定(#2567)、NETWORK_ACCESSのグローバルデフォルト(#1334)など。完全なSwarm対応は大規模だが、現状の制限と推奨構成のドキュメント化だけでも価値が高い。

- 関連PR: [#2567](https://github.com/nginx-proxy/nginx-proxy/pull/2567), [#2091](https://github.com/nginx-proxy/nginx-proxy/pull/2091), [#1477](https://github.com/nginx-proxy/nginx-proxy/pull/1477), [#750](https://github.com/nginx-proxy/nginx-proxy/pull/750)
- 関連Issue: [#97](https://github.com/nginx-proxy/nginx-proxy/issues/97), [#1148](https://github.com/nginx-proxy/nginx-proxy/issues/1148), [#1138](https://github.com/nginx-proxy/nginx-proxy/issues/1138), [#1050](https://github.com/nginx-proxy/nginx-proxy/issues/1050), [#1334](https://github.com/nginx-proxy/nginx-proxy/issues/1334), [#1199](https://github.com/nginx-proxy/nginx-proxy/issues/1199), [#1380](https://github.com/nginx-proxy/nginx-proxy/issues/1380)

### CORS と Basic Auth の併用

Basic Auth有効時にCORSプリフライト(OPTIONS)を通すための要求。無条件にOPTIONSを認証除外すると情報漏洩リスクがあるとメンテナが指摘しており、環境変数によるopt-in方式に設計を統一する必要がある。#1176と#1779は同一問題への別アプローチで統合すべき。

- 関連PR: [#1176](https://github.com/nginx-proxy/nginx-proxy/pull/1176), [#1779](https://github.com/nginx-proxy/nginx-proxy/pull/1779)
- 関連Issue: [#1150](https://github.com/nginx-proxy/nginx-proxy/issues/1150), [#1125](https://github.com/nginx-proxy/nginx-proxy/issues/1125), [#1087](https://github.com/nginx-proxy/nginx-proxy/issues/1087), [#1070](https://github.com/nginx-proxy/nginx-proxy/issues/1070)

---

## オープンPR カタログ(全38件)

### PR一覧(価値・工数順)

| PR | タイトル | カテゴリ | 価値 | 復活工数 | 推奨 | マージ状態 | 規模 | 作成 |
|---|---|---|---|---|---|---|---|---|
| [#2697](https://github.com/nginx-proxy/nginx-proxy/pull/2697) | Use LETSENCRYPT_CERTNAME for host-managed certbot certs | ssl/certs | high | low | needs-discussion | MERGEABLE / BLOCKED (base main) | S | 2026-05-01 |
| [#1477](https://github.com/nginx-proxy/nginx-proxy/pull/1477) | Support no Docker proxy by env VIRTUAL_UPSTREAM | backend/configurable-upstream | high | medium | needs-discussion | CONFLICTING (base main) | S | 2020-07-22 |
| [#1558](https://github.com/nginx-proxy/nginx-proxy/pull/1558) | nginx.tmpl: use closest vhost.d and htpasswd files (#1309) | vhost.d-matching / nginx.tmpl | high | medium | needs-discussion | CONFLICTING (base main) | M | 2021-02-09 |
| [#1563](https://github.com/nginx-proxy/nginx-proxy/pull/1563) | Add support for virtual host redirection | redirect/host-aliases | high | medium | revive-rebase | UNKNOWN (base main) — 5年前 + テンプレ更新済みのため CONFLICTING(2024年に利用者が手動で追従修正したと報告) | M | 2021-02-25 |
| [#1618](https://github.com/nginx-proxy/nginx-proxy/pull/1618) | Add vhost.d includes to nginx.tmpl's https redirect server block | ssl-redirect / custom-config | high | medium | revive-rebase | CONFLICTING (base main) | M | 2021-05-07 |
| [#1100](https://github.com/nginx-proxy/nginx-proxy/pull/1100) | Support external (non-container) upstream hosts via UPSTREAM_NAME | external-upstream / nginx.tmpl | high | high | reimplement-fresh | CONFLICTING (base main) | M | 2017-12-01 |
| [#1326](https://github.com/nginx-proxy/nginx-proxy/pull/1326) | support nginx_* env variables on nginx container and vhosts | vhost-config/proxy-tuning | high | high | needs-discussion | UNKNOWN (base main) — 作者が2022年に master へリベース済みだが、その後さらに変更があり再CONFLICTINGの可能性 | S | 2019-09-05 |
| [#1369](https://github.com/nginx-proxy/nginx-proxy/pull/1369) | Add support for VIRTUAL_HOST_ALIAS | routing/redirects | high | high | reimplement-fresh | CONFLICTING (base main) | L | 2019-12-19 |
| [#255](https://github.com/nginx-proxy/nginx-proxy/pull/255) | Add VIRTUAL_WEIGHT for Load Balancing | load-balancing / nginx.tmpl | medium | low | revive-rebase | CONFLICTING (base main) | S | 2015-10-09 |
| [#1135](https://github.com/nginx-proxy/nginx-proxy/pull/1135) | Adding the possibility to use anonymized logging | logging / privacy | medium | low | reimplement-fresh | CONFLICTING (base main) | S | 2018-05-24 |
| [#1430](https://github.com/nginx-proxy/nginx-proxy/pull/1430) | docs: IPv6 support + correct client ip addresses | docs | medium | low | revive-rebase | UNKNOWN (base main) | S | 2020-05-01 |
| [#2453](https://github.com/nginx-proxy/nginx-proxy/pull/2453) | Draft: feat: var BACKUP_SERVER to tag server as backup | load-balancing / upstream | medium | low | needs-discussion | MERGEABLE (base main; BLOCKED) | S | 2024-05-17 |
| [#2567](https://github.com/nginx-proxy/nginx-proxy/pull/2567) | Add possibility to handle multiple network tags | networking/access-control | medium | low | revive-rebase | UNKNOWN (base main) | S | 2024-12-20 |
| [#273](https://github.com/nginx-proxy/nginx-proxy/pull/273) | Enable access_log customisation per virtual host | logging | medium | medium | reimplement-fresh | UNKNOWN (base main) — 10年前のためほぼ確実に CONFLICTING。当時の nginx.tmpl 構造は現在と大きく異なる | S | 2015-10-20 |
| [#299](https://github.com/nginx-proxy/nginx-proxy/pull/299) | Allow per-container ip_hash via USE_IP_HASH env var | load-balancing / upstream | medium | medium | reimplement-fresh | CONFLICTING (base main) | S | 2015-11-21 |
| [#454](https://github.com/nginx-proxy/nginx-proxy/pull/454) | Optional passphrase support | tls/ssl | medium | medium | revive-rebase | UNKNOWN (base main) | S | 2016-05-11 |
| [#1146](https://github.com/nginx-proxy/nginx-proxy/pull/1146) | nginx.tmpl: Adding possibility for default htpasswd file | auth/basic-auth | medium | medium | reimplement-fresh | CONFLICTING (base main) | S | 2018-06-27 |
| [#1176](https://github.com/nginx-proxy/nginx-proxy/pull/1176) | disable basic authentication for HTTP OPTIONS for CORS | auth/security | medium | medium | needs-discussion | UNKNOWN (base main) | S | 2018-10-17 |
| [#1294](https://github.com/nginx-proxy/nginx-proxy/pull/1294) | Container env SERVER_PASS -> proxy_pass line | template/proxy-pass | medium | medium | reimplement-fresh | CONFLICTING (base main) | S | 2019-06-23 |
| [#1779](https://github.com/nginx-proxy/nginx-proxy/pull/1779) | Add an environment variable to enable CORS requests with basic authentication | auth/cors | medium | medium | revive-rebase | CONFLICTING (base main) | S | 2021-09-13 |
| [#745](https://github.com/nginx-proxy/nginx-proxy/pull/745) | Add useful debugging comments into the generated nginx config file | debugging/observability | medium | high | reimplement-fresh | UNKNOWN (base main) — 9年前のため CONFLICTING ほぼ確実 | M | 2017-02-26 |
| [#750](https://github.com/nginx-proxy/nginx-proxy/pull/750) | Introduce NGINX_CONTAINER env var (separate nginx/docker-gen containers) | separate-containers / docker-gen | medium | high | needs-discussion | CONFLICTING (base main) | L | 2017-02-28 |
| [#1037](https://github.com/nginx-proxy/nginx-proxy/pull/1037) | Add static file backend (VIRTUAL_ROOT) and include stopped containers | backend/static-files | medium | high | needs-discussion | CONFLICTING (base main) | M | 2017-12 |
| [#1308](https://github.com/nginx-proxy/nginx-proxy/pull/1308) | Add htpasswd from reading containers VIRTUAL_HTPASSWD env var | per-vhost-config / auth / nginx.tmpl | medium | high | needs-discussion | CONFLICTING (base main) | L | 2019-07-29 |
| [#1938](https://github.com/nginx-proxy/nginx-proxy/pull/1938) | feat: Use container names for upstream names to avoid repeats | template/upstream | medium | high | needs-discussion | UNKNOWN (base main) | L | 2022-04-06 |
| [#2608](https://github.com/nginx-proxy/nginx-proxy/pull/2608) | split nginx.tmpl to support stream module | feature/stream | medium | high | reimplement-fresh | UNKNOWN (base main) | M | 2025-04-10 |
| [#161](https://github.com/nginx-proxy/nginx-proxy/pull/161) | added info how to modify nginx config in startup command | docs | low | low | likely-obsolete | UNKNOWN (base main) | S | 2015-05-08 |
| [#245](https://github.com/nginx-proxy/nginx-proxy/pull/245) | feat(ssl): Allow using .pem for public key file | ssl/certs | low | low | needs-discussion | CONFLICTING (base main) | S | 2015-09-29 |
| [#2377](https://github.com/nginx-proxy/nginx-proxy/pull/2377) | add http include | custom-config | low | low | likely-obsolete | UNKNOWN (base main) — 比較的新しく diff も極小だが、近年テンプレ更新があり軽微な CONFLICTING の可能性 | S | 2024-01-22 |
| [#1118](https://github.com/nginx-proxy/nginx-proxy/pull/1118) | added VIRTUAL_INDEX to allow for specifying an index for the location to use | vhost-config/routing | low | medium | low-priority | UNKNOWN (base main) — 8年前のため CONFLICTING の可能性高 | S | 2018-04-03 |
| [#2091](https://github.com/nginx-proxy/nginx-proxy/pull/2091) | Load Network Config depending on NETWORK_ACCESS env var | network-config / nginx.tmpl | low | medium | low-priority | CONFLICTING (base main) | S | 2022-11-08 |
| [#379](https://github.com/nginx-proxy/nginx-proxy/pull/379) | Only start nginx after dockergen is done | startup/process-ordering | low | high | likely-obsolete | UNKNOWN (base main) | S | 2016-03-02 |
| [#561](https://github.com/nginx-proxy/nginx-proxy/pull/561) | Dedicated nginx configuration file | config/build | low | high | likely-obsolete | CONFLICTING (base main) | S | 2016-09-06 |
| [#569](https://github.com/nginx-proxy/nginx-proxy/pull/569) | test config before reloading [untested] | reload / robustness | low | high | needs-discussion | CONFLICTING (base main) | S | 2016-09-14 |
| [#784](https://github.com/nginx-proxy/nginx-proxy/pull/784) | Added DEFAULT_IP env variable | upstream/networking | low | high | likely-obsolete | CONFLICTING (base main) | M | 2017-04-06 |
| [#943](https://github.com/nginx-proxy/nginx-proxy/pull/943) | Add in vhost https bypass | tls/acme | low | high | likely-obsolete | UNKNOWN (base main) | S | 2017-10-07 |
| [#1342](https://github.com/nginx-proxy/nginx-proxy/pull/1342) | Make it possible to serve static files locally with fastcgi upstream | fastcgi / static-files | low | high | low-priority | CONFLICTING (base main) | M | 2019-10-08 |
| [#2069](https://github.com/nginx-proxy/nginx-proxy/pull/2069) | Added php-fpm backend | backend/php-fpm | low | high | low-priority | CONFLICTING (base main) | M | 2022-10-15 |

### PR詳細(推奨アクション別)

#### リベースして復活 (revive-rebase) — 7件

##### [#1563](https://github.com/nginx-proxy/nginx-proxy/pull/1563) Add support for virtual host redirection

**価値:** high ・ **復活工数:** medium ・ **規模:** M ・ **マージ状態:** UNKNOWN (base main) — 5年前 + テンプレ更新済みのため CONFLICTING(2024年に利用者が手動で追従修正したと報告) ・ **作成:** 2021-02-25 (約5.3年前) ・ **作者:** propheth

- **内容:** 新しい REDIRECT 環境変数を追加し、VIRTUAL_HOST に列挙した複数ホスト間で正規ホストへの 301 リダイレクトを生成(例: non-www → www、www → non-www)。HTTP/HTTPS 両対応。nginx.tmpl と README を変更。
- **停滞理由:** ラベル status/pr-needs-tests + type/feat + scope/host-aliases。メンテナ(buchdag)は2023年に『ドキュメント+テストがあればマージしたい』と明言。コントリビュータ R0Wi が conflict 解消とテスト追加を申し出たが続報なし。テスト/ドキュメント未整備で停滞
- **メモ:** 需要が極めて高い(issue 6件をまとめて解決、関連PR #1836/#1927 とともにマージ要望あり)。メンテナがマージ歓迎を明言済みで方向性は確定。残作業は現行テンプレへのリベース+テスト+ドキュメント。alekna 提案のリダイレクトコード設定可能化(デフォルト302)も取り込むと良い。最有力の復活候補。
- **関連Issue:** [#180](https://github.com/nginx-proxy/nginx-proxy/issues/180), [#960](https://github.com/nginx-proxy/nginx-proxy/issues/960), [#958](https://github.com/nginx-proxy/nginx-proxy/issues/958), [#444](https://github.com/nginx-proxy/nginx-proxy/issues/444), [#1095](https://github.com/nginx-proxy/nginx-proxy/issues/1095), [#1369](https://github.com/nginx-proxy/nginx-proxy/issues/1369)

##### [#1618](https://github.com/nginx-proxy/nginx-proxy/pull/1618) Add vhost.d includes to nginx.tmpl's https redirect server block

**価値:** high ・ **復活工数:** medium ・ **規模:** M ・ **マージ状態:** CONFLICTING (base main) ・ **作成:** 2021-05-07 (約5.1年前) ・ **作者:** dakotahawkins

- **内容:** HTTPSリダイレクトのserverブロックにもvhost.d include(per-vhost / defaultのカスタム設定)を差し込めるようにし、301返却前にレスポンスヘッダ追加等のカスタマイズを可能にする。acme-companionのacme-challengeロケーションと衝突しないよう vhost.d にファイルが無い場合のみ含める。
- **停滞理由:** コメントゼロでメンテナ未レビューのまま放置。現行 main と競合。type/fix・type/feat・type/test ラベル付きで品質要件は満たしている。
- **メモ:** 6PR中で最も状態良好: #1613 を fixes、テスト同梱(test_ssl配下に複数ケース追加)、acme-companion連携も考慮済み。テンプレ変更も14行と小さく、リダイレクトserverブロック周辺のrebaseのみで復活可能。最優先で revive 候補。
- **関連Issue:** [#1613](https://github.com/nginx-proxy/nginx-proxy/issues/1613)

##### [#255](https://github.com/nginx-proxy/nginx-proxy/pull/255) Add VIRTUAL_WEIGHT for Load Balancing

**価値:** medium ・ **復活工数:** low ・ **規模:** S ・ **マージ状態:** CONFLICTING (base main) ・ **作成:** 2015-10-09 (約10.7年前) ・ **作者:** kelunik

- **内容:** upstream ブロックの server 行に weight= を出力し、コンテナの VIRTUAL_WEIGHT 環境変数で重み付きロードバランシングを可能にする。nginx.tmpl のみ変更 (+5/-4)。
- **停滞理由:** no-maintainer-response。レビューで Go text/template の変数スコープ問題が指摘され author が修正 (or 関数化) したが、実動作確認 (VIRTUAL_WEIGHT が反映されるか) 未完了のまま放置。複数ユーザーが merge を要望したがメンテナ無反応。
- **メモ:** 10年超で stale。仕組みは単純で現行 nginx.tmpl にも upstream ブロックが存在し再実装容易。ただし複数レプリカ間の重み付けはユースケースが限定的 (Swarm/scale 利用者向け)。要: rebase + テスト + ドキュメント。md5 提案の {{ $weight := or $container.Env.VIRTUAL_WEIGHT 1 }} 方式で再実装するのが安全。

##### [#454](https://github.com/nginx-proxy/nginx-proxy/pull/454) Optional passphrase support

**価値:** medium ・ **復活工数:** medium ・ **規模:** S ・ **マージ状態:** UNKNOWN (base main) ・ **作成:** 2016-05-11 (約10.1年前) ・ **作者:** IvanDev

- **内容:** certs フォルダに `<host>.pw` ファイルを置くと、その内容を nginx の `ssl_password_file` ディレクティブとして vhost に出力し、パスフレーズ付き秘密鍵を扱えるようにする (nginx.tmpl +10行, README +5行)。
- **停滞理由:** ラベル status/pr-needs-tests。テスト未整備のまま10年メンテナ沈黙。複数ユーザーが「マージして」と要望するもレビューされず。
- **メモ:** 機能自体は今も有効でニッチだが実需あり(暗号化鍵の利用)。10年でnginx.tmplが大幅変更されているため、SSL周りのテンプレ箇所への再配置 + test/ にpytestケース追加が必須。ロジックは小さいので revive-rebase 推奨だが、TLS実体は acme-companion 管轄である点と整合確認が要る。

##### [#1430](https://github.com/nginx-proxy/nginx-proxy/pull/1430) docs: IPv6 support + correct client ip addresses

**価値:** medium ・ **復活工数:** low ・ **規模:** S ・ **マージ状態:** UNKNOWN (base main) ・ **作成:** 2020-05-01 (約6.1年前) ・ **作者:** tuxmainy

- **内容:** IPv6 利用時に正しいクライアントIPを得るための設定(IPv6有効化・docker daemon設定等)を README に追記するドキュメント変更 (README +26)。issue #1419 のドキュメント要望に対応。
- **停滞理由:** ラベル type/docs のみ。コメントゼロでレビューされず放置。docs が docs/ ディレクトリ + mkdocs 構成へ移行したため当時の README への追記は配置がずれている。
- **メモ:** IPv6 + 正しいクライアントIPは現在も頻出の質問でドキュメント価値あり。内容は概ね今も有効。現行の docs/ 構成(mkdocs)へ転記する形での revive-rebase が妥当。コードリスクなし。
- **関連Issue:** [#1419](https://github.com/nginx-proxy/nginx-proxy/issues/1419)

##### [#1779](https://github.com/nginx-proxy/nginx-proxy/pull/1779) Add an environment variable to enable CORS requests with basic authentication

**価値:** medium ・ **復活工数:** medium ・ **規模:** S ・ **マージ状態:** CONFLICTING (base main) ・ **作成:** 2021-09-13 (約4.8年前) ・ **作者:** vemonet

- **内容:** ENABLE_CORS_AUTH環境変数を追加し、Basic認証下でもHTTP OPTIONS(CORSプリフライト)に対しては認証をスキップできるようにする。デフォルト挙動は不変。
- **停滞理由:** needs-tests; ラベル status/pr-needs-tests + type/feat。メンテナjunderwが『Needs tests』とコメント(👍4)。テスト未提供で停滞。
- **メモ:** 現行テンプレにCORS/OPTIONSの認証バイパスは存在せず機能は今も有用。関連issue #1150/#1176ともOPEN。変更は8行と小さく、現行のauth_basicブロックに当てやすい。要件はテスト追加のみで明確=リバイブ向き。env名(ENABLE_CORS_AUTH)は誤解を招くと作者自身が指摘しており命名は要議論。先行PR #1176あり。
- **関連Issue:** [#1150](https://github.com/nginx-proxy/nginx-proxy/issues/1150), [#1176](https://github.com/nginx-proxy/nginx-proxy/issues/1176)

##### [#2567](https://github.com/nginx-proxy/nginx-proxy/pull/2567) Add possibility to handle multiple network tags

**価値:** medium ・ **復活工数:** low ・ **規模:** S ・ **マージ状態:** UNKNOWN (base main) ・ **作成:** 2024-12-20 (約1.5年前) ・ **作者:** Murazaki

- **内容:** 既存の network_internal.conf(internalタグ固定)を一般化し、NETWORK_ACCESSタグに応じて /etc/nginx/network_<tag>.conf を動的にincludeできるようにする(group0/1/2…等、サービス毎に異なるアクセス制御)。
- **停滞理由:** needs-tests/needs-docs; ラベルなし・コメント0件でメンテナ未レビューのまま放置。テスト/ドキュメント未整備。
- **メモ:** 陳腐化していない。現行テンプレの network_tag/$vpath/NETWORK_ACCESS 機構の上に構築されており、5行の最小diffで internal 固定を任意タグへ一般化する自然な拡張。既存testに test_internal 等あり同パターンでテスト追加容易=最も低工数。比較的新しく現行コードと整合。要: testとREADME/docsへのNETWORK_ACCESS拡張説明追記。

#### 方針合意が必要 (needs-discussion) — 12件

##### [#1326](https://github.com/nginx-proxy/nginx-proxy/pull/1326) support nginx_* env variables on nginx container and vhosts

**価値:** high ・ **復活工数:** high ・ **規模:** S ・ **マージ状態:** UNKNOWN (base main) — 作者が2022年に master へリベース済みだが、その後さらに変更があり再CONFLICTINGの可能性 ・ **作成:** 2019-09-05 (約6.8年前) ・ **作者:** poma

- **内容:** `nginx_*`(および後に `NGINX_*`)プレフィックスの環境変数を nginx 設定ディレクティブとして自動展開する。proxy コンテナの変数はグローバル設定へ、各コンテナの変数は対応する vhost へ注入(例: nginx_client_max_body_size=30M)。nginx.tmpl のみ18行追加。
- **停滞理由:** ラベル status/pr-needs-tests + status/pr-needs-docs + type/feat。テストとドキュメント未整備。さらに作者自身が指摘する重大な破壊的問題: nginx:alpine 等が出力する NGINX_VERSION のような無効な変数が混入し nginx 設定生成を壊しうる。設計上の安全性議論が未決着
- **メモ:** client_max_body_size 等の任意ディレクティブを env で渡せるのは需要が非常に高い(頻出トピック)。ただし任意env→nginxディレクティブの無条件展開は設定破壊・インジェクションのリスクがあり、プレフィックスやホワイトリスト設計の合意が必要。価値は高いが安全な設計の議論が前提。テスト/ドキュメント必須。
- **関連Issue:** [#1324](https://github.com/nginx-proxy/nginx-proxy/issues/1324)

##### [#1477](https://github.com/nginx-proxy/nginx-proxy/pull/1477) Support no Docker proxy by env VIRTUAL_UPSTREAM

**価値:** high ・ **復活工数:** medium ・ **規模:** S ・ **マージ状態:** CONFLICTING (base main) ・ **作成:** 2020-07-22 (約5.9年前) ・ **作者:** ccpaging

- **内容:** VIRTUAL_UPSTREAM 環境変数で非Docker/外部アドレス(例 192.168.1.8:8080)を upstream に指定可能にする。host networking やDockerized外サービスのプロキシ用途。
- **停滞理由:** ラベル status/pr-needs-tests と status/pr-needs-docs。インデント崩れの指摘あり。メンテナはhost networking対応を別PR #2222 で進行中と回答。
- **メモ:** scope/configurable-upstream ラベル付き。需要は高い(host network/外部upstream)が #2222 と一部競合・置き換えの可能性。#2222 の現状を確認した上で取り込み可否を判断すべき。テスト+ドキュメント+インデント修正が必要。
- **関連Issue:** [#2222](https://github.com/nginx-proxy/nginx-proxy/issues/2222)

##### [#1558](https://github.com/nginx-proxy/nginx-proxy/pull/1558) nginx.tmpl: use closest vhost.d and htpasswd files (#1309)

**価値:** high ・ **復活工数:** medium ・ **規模:** M ・ **マージ状態:** CONFLICTING (base main) ・ **作成:** 2021-02-09 (約5.4年前) ・ **作者:** jjakob

- **内容:** vhost.d と htpasswd ファイルの選択に証明書と同じ closest マッチングロジックを使い、ワイルドカード/サブドメインの VIRTUAL_HOST でも親ドメインの設定ファイルがマッチするようにする。nginx.tmpl のみ (+45/-6)。Fixes #1309。
- **停滞理由:** Draft 状態 (type/feat ラベル)。author 自身が現行 closest 関数は部分一致で不正確 (b.c が a.b.c.d にマッチ) と認識し、docker-gen 側に末尾一致の closestSuffix 関数を追加してから完成させたいと表明。メンテナ buchdag が 2025-01 に前向き回答したが docker-gen 実装待ちで停止。
- **メモ:** 6PR中最も生きている案件: メンテナが2025年に関与、issue #1309 と明確に紐付き需要も高い。ブロッカーは upstream docker-gen への arrayClosestSuffix (HasSuffix ベース) 関数追加。まず docker-gen に関数を入れ、その後このPRを rebase+完成させる流れ。needs-discussion (docker-gen 側の調整) だが優先度は高い。
- **関連Issue:** [#1309](https://github.com/nginx-proxy/nginx-proxy/issues/1309)

##### [#2697](https://github.com/nginx-proxy/nginx-proxy/pull/2697) Use LETSENCRYPT_CERTNAME for host-managed certbot certs

**価値:** high ・ **復活工数:** low ・ **規模:** S ・ **マージ状態:** MERGEABLE / BLOCKED (base main) ・ **作成:** 2026-05-01 (約0.1年前) ・ **作者:** jekewa

- **内容:** 環境変数 LETSENCRYPT_CERTNAME を指定すると、ホスト側certbotが管理する /etc/letsencrypt/live/<name> の証明書をvhostのserverブロックで使う。acme-companion不要で外部certbotの証明書を流用できる。
- **停滞理由:** コンフリクトは無いがブランチ保護でBLOCKED。メンテナ(buchdag)はより汎用的な実装(コンテナごとに証明書パスを設定できる仕組み、CERTBOT_MANAGED_CERTIFICATE 案)を提案中で、設計をすり合わせ中。
- **メモ:** 唯一マージ可能かつ現在進行中で議論が活発。実装は小さくリベース容易。メンテナが命名/汎用化(CERTBOT_MANAGED_CERTIFICATE)を望んでおり、その方向に合わせれば取り込み現実的。最有望候補。

##### [#750](https://github.com/nginx-proxy/nginx-proxy/pull/750) Introduce NGINX_CONTAINER env var (separate nginx/docker-gen containers)

**価値:** medium ・ **復活工数:** high ・ **規模:** L ・ **マージ状態:** CONFLICTING (base main) ・ **作成:** 2017-02-28 (約9.3年前) ・ **作者:** thomasleveil

- **内容:** docker-gen と nginx を別コンテナで動かす構成で、どのコンテナが実際に nginx を動かしているかを NGINX_CONTAINER 環境変数で明示できるようにする。658e20f以降の『docker-genコンテナ=nginxコンテナ』前提を緩和。
- **停滞理由:** 設計議論が未決着(container_name指定 vs ネットワーク/共有ボリュームによる自動検出)。CONTRIBUTORコメントで『docker-gen側テンプレートで対応すべき』との指摘あり。テストが旧test_dockergen構成の大規模リネームを含み現行と大きく競合。
- **メモ:** 17ファイル変更とテスト構成の大改造を含むため rebase は重い。アプローチ自体に異論(自動検出が望ましい)。reachability判定の現行実装を確認のうえ設計合意が先。separate-containers構成は今もサポート対象なので価値はあるが要再設計。
- **関連Issue:** [#709](https://github.com/nginx-proxy/nginx-proxy/issues/709)

##### [#1037](https://github.com/nginx-proxy/nginx-proxy/pull/1037) Add static file backend (VIRTUAL_ROOT) and include stopped containers

**価値:** medium ・ **復活工数:** high ・ **規模:** M ・ **マージ状態:** CONFLICTING (base main) ・ **作成:** 2017-12 (約9年前) ・ **作者:** Mactory

- **内容:** マウントしたボリュームから静的ファイルを直接配信する新バックエンドを追加。停止中コンテナも環境変数で含められるようにする。
- **停滞理由:** メンテナ(buchdag)が「背後に別nginxコンテナを置くのと比べた利点は?」と価値を疑問視。ボリュームを手動で nginx-proxy 本体にマウントせねばならず設定が2コンテナに分散するハードな制約(JonathonReinhart指摘)。
- **メモ:** 👍は付くが設計上のブロッカー(実行中コンテナへ動的にボリューム追加不可)が根本的。需要はあるので再設計(別アプローチ)を前提に議論が必要。

##### [#1176](https://github.com/nginx-proxy/nginx-proxy/pull/1176) disable basic authentication for HTTP OPTIONS for CORS

**価値:** medium ・ **復活工数:** medium ・ **規模:** S ・ **マージ状態:** UNKNOWN (base main) ・ **作成:** 2018-10-17 (約7.7年前) ・ **作者:** rparree

- **内容:** Basic認証が有効な vhost で HTTP OPTIONS (CORSプリフライト) リクエストを認証対象から外す (nginx.tmpl +4/-2)。ブラウザはプリフライトに Authorization ヘッダを送らないため、CORS+Basic認証併用を可能にする。
- **停滞理由:** メンテナ(tkw1536)がセキュリティ懸念を表明: デフォルトで全リクエストを認証後に転送すべきで、無条件にOPTIONSを除外すると情報漏洩リスク。「環境変数/フラグで opt-in にし README に明記すべき」との要望に対し著者が対応せず放置。
- **メモ:** 需要は今も継続(CORS+Basic認証は定番のつまずき)が、無条件除外はセキュリティ上マージ不可というメンテナ方針が明確。環境変数で opt-in 化 + ドキュメント + テスト を加えた再実装が前提。方針合意が先決のため needs-discussion。

##### [#1308](https://github.com/nginx-proxy/nginx-proxy/pull/1308) Add htpasswd from reading containers VIRTUAL_HTPASSWD env var

**価値:** medium ・ **復活工数:** high ・ **規模:** L ・ **マージ状態:** CONFLICTING (base main) ・ **作成:** 2019-07-29 (約6.9年前) ・ **作者:** hasnat

- **内容:** コンテナの環境変数から per-vhost 設定を注入: VHOST_CONF (server レベル vhost.d)、VHOST_LOCATION_CONF (location レベル)、VHOST_HTPASSWD (Basic 認証)。htpasswd_generator.tmpl を新規追加し nginx.tmpl を書き換え (10ファイル, +83/-83)。
- **停滞理由:** コメント0でメンテナレビュー一切なし (no-maintainer-response)。10ファイルに渡り docker-compose サンプル削除や Dockerfile 変更を含む大きめの差分で、セキュリティ面 (環境変数経由で htpasswd/任意 nginx config を注入) のレビュー負荷が高く放置されたと推測。
- **メモ:** 6.9年 stale。環境変数で任意の nginx config を流し込む設計はセキュリティ・スコープ的に賛否あり、メンテナ方針 (専用 config ファイル方式) と衝突しうるため要議論。VHOST_HTPASSWD/VHOST_CONF/VHOST_LOCATION_CONF の3機能を分割し認証だけ切り出すなど縮小して再提案が現実的。差分が古く clean rebase 不可。

##### [#1938](https://github.com/nginx-proxy/nginx-proxy/pull/1938) feat: Use container names for upstream names to avoid repeats

**価値:** medium ・ **復活工数:** high ・ **規模:** L ・ **マージ状態:** UNKNOWN (base main) ・ **作成:** 2022-04-06 (約4.2年前) ・ **作者:** rhansen

- **内容:** 1コンテナが複数 VIRTUAL_HOST を持つ場合に同一 upstream ブロックが重複生成される問題を解消し、コンテナ名ベースの upstream 名で単一化する (nginx.tmpl +89/-13, docker-entrypoint.sh, README, テスト追加)。デフォルト挙動は据え置きで opt-in。
- **停滞理由:** PR #2127 依存でドラフト化(著者自身が宣言)。メンテナ(buchdag)が「動的にコンテナ名/レプリカが変わる環境では upstream 名が予測不能になりデフォルト化は非現実的」「追加複雑性に見合う利点が不明瞭」と指摘。著者も汎用解を再検討中のまま停滞。
- **メモ:** 著者は COLLABORATOR(rhansen)でテスト付き・質は高いが、#2127依存かつ設計論点(upstream命名の安定性・予測可能性)が未決。4年でnginx.tmplが変わりリベース重め。マージには命名戦略の設計合意が必須なので needs-discussion。#1504 のための布石。
- **関連Issue:** [#1504](https://github.com/nginx-proxy/nginx-proxy/issues/1504), [#2127](https://github.com/nginx-proxy/nginx-proxy/issues/2127)

##### [#2453](https://github.com/nginx-proxy/nginx-proxy/pull/2453) Draft: feat: var BACKUP_SERVER to tag server as backup

**価値:** medium ・ **復活工数:** low ・ **規模:** S ・ **マージ状態:** MERGEABLE (base main; BLOCKED) ・ **作成:** 2024-05-17 (約2.1年前) ・ **作者:** pini-gh

- **内容:** BACKUP_SERVER 環境変数でupstreamのserverエントリに backup フラグを付け、当該コンテナをバックアップ(フェイルオーバ)サーバとして扱えるようにする。テスト同梱。
- **停滞理由:** Draft状態のまま。mergeable自体はMERGEABLEだがmergeStateStatus=BLOCKED(draft/チェック未通過)。レビュアー(CONTRIBUTOR SchoNie)が backup単体でなく weight, fail_timeout 等も指定できる汎用 SERVER_PARAMETERS 化を提案しており方針未決。
- **メモ:** 6PR中で唯一コンフリクト無し・最も新しく・テスト付きで技術的には最も近い。関連: acme-companion#1062。ただし #299 と同じ『汎用 upstream server パラメータ』議論の一部。BACKUP_SERVER だけ入れるか SERVER_PARAMETERS として一般化するかを決めれば即マージ可能。#299 と束ねて設計合意するのが望ましい。

##### [#245](https://github.com/nginx-proxy/nginx-proxy/pull/245) feat(ssl): Allow using .pem for public key file

**価値:** low ・ **復活工数:** low ・ **規模:** S ・ **マージ状態:** CONFLICTING (base main) ・ **作成:** 2015-09-29 (約10.7年前) ・ **作者:** randombk

- **内容:** 証明書ファイルとして .crt にフォールバックする前にまず $cert.pem を使うようにする。PEM拡張子を優先したいユーザー向け。
- **停滞理由:** メンテナ(jwilder)が「.crt が標準では?」と疑問を呈し方向性が定まらず放置。.pem/.crt 両方ある場合のリグレッション懸念もあり、空白の扱いの指摘も未対応。
- **メモ:** 10年以上放置。両ファイル共存時の優先順位がハードな設計論点。実装は数行で容易だが、メンテナの合意が前提。現状のnginx.tmpl(証明書探索ロジック)は大きく変わっており要リベース。

##### [#569](https://github.com/nginx-proxy/nginx-proxy/pull/569) test config before reloading [untested]

**価値:** low ・ **復活工数:** high ・ **規模:** S ・ **マージ状態:** CONFLICTING (base main) ・ **作成:** 2016-09-14 (約9.8年前) ・ **作者:** schmunk42

- **内容:** Procfile の nginx reload コマンドに nginx -t (config test) を追加し、設定が壊れている場合に reload せず既存の動作中 config を維持する狙い。Procfile 1行のみ変更 (+1/-1)。
- **停滞理由:** author 自身がタイトルで [untested] と明記。コントリビュータ thomasleveil が『コンテナ再起動時には結局 nginx 起動失敗』『新規 VIRTUAL_HOST コンテナが無視されサポート問い合わせが増える』と設計上の欠陥を指摘。議論が収束せず放置 (rejected-direction 孄り)。
- **メモ:** 問題提起としては有効だが、この実装 (単純な nginx -t 追加) は根本解決にならないとメンテナが明言。現在の nginx-proxy は docker-gen 側で reload 戦略が大きく変わり Procfile ベースの前提自体が古い。『壊れた config 検証→ロールバック』を起動スクリプト側で設計し直す必要があり別案件。低優先。
- **関連Issue:** [#502](https://github.com/nginx-proxy/nginx-proxy/issues/502), [#438](https://github.com/nginx-proxy/nginx-proxy/issues/438), [#585](https://github.com/nginx-proxy/nginx-proxy/issues/585)

#### 作り直し推奨 (reimplement-fresh) — 9件

##### [#1100](https://github.com/nginx-proxy/nginx-proxy/pull/1100) Support external (non-container) upstream hosts via UPSTREAM_NAME

**価値:** high ・ **復活工数:** high ・ **規模:** M ・ **マージ状態:** CONFLICTING (base main) ・ **作成:** 2017-12-01 (約8.5年前) ・ **作者:** CWempe

- **内容:** コンテナでない物理マシン/外部ホストへプロキシできるよう、ダミーコンテナに UPSTREAM_NAME 環境変数を付けその値を upstream の server アドレスとして nginx.tmpl に出力する (+31、nginx.tmpl + 1ファイル)。
- **停滞理由:** no-maintainer-response。コミュニティ需要は非常に高い (長年 merge 要望が続く) が author 自身がテスト全滅と報告。『ダミーコンテナが必要』という設計を複数ユーザーが疑問視し、socat コンテナや別プロジェクト (nginx-proxy-pass-docker) など回避策が定着。
- **メモ:** 8年以上 stale で nginx.tmpl は大幅に変化し単純 rebase 不可。需要は今も高く機能価値大だが、ダミーコンテナ方式ではなく ENV/ファイルベースの外部 upstream 定義として再設計すべき (コメントでも要望)。acme-companion 連携も要検討。reimplement 推奨。

##### [#1369](https://github.com/nginx-proxy/nginx-proxy/pull/1369) Add support for VIRTUAL_HOST_ALIAS

**価値:** high ・ **復活工数:** high ・ **規模:** L ・ **マージ状態:** CONFLICTING (base main) ・ **作成:** 2019-12-19 (約6.5年前) ・ **作者:** dannycarrera

- **内容:** VIRTUAL_HOST_ALIAS環境変数を追加し、別名ホスト(www.やold.等)をVIRTUAL_HOSTの最初のエントリへ301リダイレクトする。www↔non-www正規化のニーズに対応。
- **停滞理由:** needs-rebase + unresolved-review; テンプレ全面書き換えで300コミット以上遅れ、競合多数。aiomasterのレビュー指摘未解決、ACME証明書取得との相互作用バグ報告あり(MarkErik)。一方shuhanmirza-bkがAPPROVED、コミュニティ要望は非常に高い(👍15+、複数フォーク派生)。
- **メモ:** 需要が最も高い機能。関連issue #180/#958/#1204は全てOPENで未解決。現行nginx.tmplはVIRTUAL_PATH/HTTPS_METHOD/non_get_redirect等が刷新されており2019年のdiffは流用不可。alias×LETSENCRYPTのACMEチャレンジ問題を設計段階で解決する必要あり→新規実装推奨。www正規化ドキュメントも整備を。
- **関連Issue:** [#180](https://github.com/nginx-proxy/nginx-proxy/issues/180), [#958](https://github.com/nginx-proxy/nginx-proxy/issues/958), [#1204](https://github.com/nginx-proxy/nginx-proxy/issues/1204)

##### [#273](https://github.com/nginx-proxy/nginx-proxy/pull/273) Enable access_log customisation per virtual host

**価値:** medium ・ **復活工数:** medium ・ **規模:** S ・ **マージ状態:** UNKNOWN (base main) — 10年前のためほぼ確実に CONFLICTING。当時の nginx.tmpl 構造は現在と大きく異なる ・ **作成:** 2015-10-20 (約10.7年前) ・ **作者:** 27Bslash6

- **内容:** デフォルトの nginx.conf から access_log ディレクティブを取り除き、vhost.d/* で各バーチャルホストごとに独自の access_log を定義できるようにする。Dockerfile と nginx.tmpl を変更。
- **停滞理由:** メンテナ(md5)は2015年に「reasonable」と同意したが、nginx.conf 全体を置き換える方がよいという議論で止まり、その後放置。ラベル無し。2018年に subdavis が「repo is dead」とコメント。maintainer-silence + scope-discussion未決着
- **メモ:** 10年超の超老朽PR。現在の nginx.tmpl は構造が一新されており当時の diff はそのまま当たらない。アイデア自体(per-vhost access_log)は有効だが、現行テンプレートに対して新規実装すべき。実質5行の小変更なので再実装は容易。
- **関連Issue:** [#216](https://github.com/nginx-proxy/nginx-proxy/issues/216), [#230](https://github.com/nginx-proxy/nginx-proxy/issues/230)

##### [#299](https://github.com/nginx-proxy/nginx-proxy/pull/299) Allow per-container ip_hash via USE_IP_HASH env var

**価値:** medium ・ **復活工数:** medium ・ **規模:** S ・ **マージ状態:** CONFLICTING (base main) ・ **作成:** 2015-11-21 (約10.6年前) ・ **作者:** tpcwang

- **内容:** コンテナ単位で USE_IP_HASH 環境変数を設定すると、そのupstreamブロックに ip_hash; ディレクティブを出力し、セッションスティッキー(IPアフィニティ)を有効化する。docker-compose scale 時のファイルアップロード等で需要が高い。
- **停滞理由:** メンテナ無反応(maintainer silence)で長期放置。多数の+1コメントがあるが現行 main と競合。テストは旧bats形式で現行pytestスイートに非対応。
- **メモ:** 10年超で最古。コメント#2453の議論にあるように、ip_hash単体ではなく汎用的な upstream server/parameters 指定(weight, fail_timeout, backup等)へ発展させる方が筋が良い。小さい変更なので現行nginx.tmpl + pytestで作り直す方が早い。実質 #2453 系の汎用化提案と重複領域。
- **関連Issue:** [#221](https://github.com/nginx-proxy/nginx-proxy/issues/221)

##### [#745](https://github.com/nginx-proxy/nginx-proxy/pull/745) Add useful debugging comments into the generated nginx config file

**価値:** medium ・ **復活工数:** high ・ **規模:** M ・ **マージ状態:** UNKNOWN (base main) — 9年前のため CONFLICTING ほぼ確実 ・ **作成:** 2017-02-26 (約9.3年前) ・ **作者:** thomasleveil

- **内容:** 生成される nginx config(default.conf 等)に、各 vhost/upstream の環境変数・ネットワーク・コンテナ情報をコメントとして埋め込み、issue でのデバッグを容易にする。nginx.tmpl のみ変更。
- **停滞理由:** 2021年にメンテナ(buchdag)が『有用なので現行 nginx.tmpl に更新してほしい』と依頼(4年経過と自認)したが、作者が応答せず放置。author-unresponsive
- **メモ:** デバッグ支援として依然有用でメンテナも歓迎姿勢。ただし当時の nginx.tmpl から大幅に変わっており、コメント挿入箇所を全面的に再配置する必要がある。元作者は無反応のため、現行テンプレートへ新規実装するのが現実的。

##### [#1135](https://github.com/nginx-proxy/nginx-proxy/pull/1135) Adding the possibility to use anonymized logging

**価値:** medium ・ **復活工数:** low ・ **規模:** S ・ **マージ状態:** CONFLICTING (base main) ・ **作成:** 2018-05-24 (約8.1年前) ・ **作者:** Mactory

- **内容:** ANONYMIZE_LOGGING 環境変数を設定すると、nginxのログ出力でIPアドレスを匿名化し、referer/user-agentを除去するログフォーマットを使う(GDPR等のプライバシー配慮)。
- **停滞理由:** メンテナ無反応で放置。テスト無し、ラベル無し。現行 main と競合。実装はStackOverflow由来の単純なログフォーマット変更。
- **メモ:** 変更はnginx.tmpl+READMEのみで小さい。グローバル設定なので現行テンプレートのlog_format定義に合わせて作り直しが容易。GDPR需要はあるがニッチ。LOG_FORMAT/カスタムログ設定の有無を現行コードで確認すると重複の可能性あり。

##### [#1146](https://github.com/nginx-proxy/nginx-proxy/pull/1146) nginx.tmpl: Adding possibility for default htpasswd file

**価値:** medium ・ **復活工数:** medium ・ **規模:** S ・ **マージ状態:** CONFLICTING (base main) ・ **作成:** 2018-06-27 (約8年前) ・ **作者:** baxerus

- **内容:** $VIRTUAL_HOST名のhtpasswdが無い場合に /etc/nginx/htpasswd/default をフォールバックとして使うBasic認証のデフォルトファイル機能を追加。全vhost共通認証や、設定漏れによる無防備な公開の防止に有用。
- **停滞理由:** maintainer-silence; テスト/ドキュメント不足のまま放置(コメント0件)。
- **メモ:** 現行テンプレのhtpasswd判定は /%s_%s(Host_sha1Path) と /%s(Host) のみで default フォールバックは未実装=機能は今も有用。ただし該当箇所(L333付近)はVIRTUAL_PATH対応で大幅に書き換わっており、9行diffはそのまま当たらない。現行構造に合わせて再実装し、testとREADME追記が必要。セキュリティ面の付加価値あり。

##### [#1294](https://github.com/nginx-proxy/nginx-proxy/pull/1294) Container env SERVER_PASS -> proxy_pass line

**価値:** medium ・ **復活工数:** medium ・ **規模:** S ・ **マージ状態:** CONFLICTING (base main) ・ **作成:** 2019-06-23 (約7年前) ・ **作者:** hasnat

- **内容:** コンテナ環境変数 SERVER_PASS で proxy_pass 行を任意の指定(fastcgi_pass / grpc_pass 等)に差し替えられるようにする。upstream名は使わない。
- **停滞理由:** レビュー・コメントが一切付かずメンテナ沈黙のまま放置。テスト/ドキュメントも無し。
- **メモ:** gRPC/FastCGIへの直接proxyという需要自体は今も有効。ただし任意文字列をテンプレに注入する設計はセキュリティ/堅牢性面で再設計が望ましい。現行nginx.tmplと衝突。

##### [#2608](https://github.com/nginx-proxy/nginx-proxy/pull/2608) split nginx.tmpl to support stream module

**価値:** medium ・ **復活工数:** high ・ **規模:** M ・ **マージ状態:** UNKNOWN (base main) ・ **作成:** 2025-04-10 (約1.2年前) ・ **作者:** lloydzhou

- **内容:** nginx の stream モジュール(TCP/UDPプロキシ)対応。環境変数 TARGET=streamupstream で同じ nginx.tmpl を別コンテキストで描画し stream.conf.d/10-upstream.conf を生成、stream ブロックから include 可能にする (11ファイル, +65/-23, テスト追加)。
- **停滞理由:** メンテナ(buchdag)がアプローチ自体を否定: 「1つの nginx.tmpl を条件分岐で2コンテキスト動作させるのは長期保守不能」「複数 docker-gen プロセスは避けるべき(単一プロセスで複数テンプレ描画可能)」。方向性に難色。
- **メモ:** stream(TCP/UDP)対応の需要は実在し機能価値はあるが、メンテナが実装方式を明確に拒否。別テンプレファイル + 単一 docker-gen で複数テンプレ描画する形での作り直しが必要。比較的新しい(1.2年)が方式から再設計のため reimplement-fresh。関連 issue #2606。
- **関連Issue:** [#2606](https://github.com/nginx-proxy/nginx-proxy/issues/2606)

#### 優先度低 (low-priority) — 4件

##### [#1118](https://github.com/nginx-proxy/nginx-proxy/pull/1118) added VIRTUAL_INDEX to allow for specifying an index for the location to use

**価値:** low ・ **復活工数:** medium ・ **規模:** S ・ **マージ状態:** UNKNOWN (base main) — 8年前のため CONFLICTING の可能性高 ・ **作成:** 2018-04-03 (約8.2年前) ・ **作者:** dialupdisaster

- **内容:** 新しい VIRTUAL_INDEX 環境変数を追加し、生成される location ブロックに nginx の index ディレクティブを設定できるようにする(例: index.php)。nginx.tmpl に7行追加のみ。
- **停滞理由:** 2018年に一度 approve されたとされるが(『approved back in May』)マージされず放置。その後メンテナの動きなし。ラベル無し。maintainer-silence
- **メモ:** issue #1117 を解決。機能は小さく無害だが、index ディレクティブは vhost.d/* のカスタム設定でも実現可能なため必要性は低め。同種ニーズは custom config で吸収できる。優先度低。再実装するなら docs とテスト追加が必要。
- **関連Issue:** [#1117](https://github.com/nginx-proxy/nginx-proxy/issues/1117)

##### [#1342](https://github.com/nginx-proxy/nginx-proxy/pull/1342) Make it possible to serve static files locally with fastcgi upstream

**価値:** low ・ **復活工数:** high ・ **規模:** M ・ **マージ状態:** CONFLICTING (base main) ・ **作成:** 2019-10-08 (約6.7年前) ・ **作者:** TheLastProject

- **内容:** 選んだ拡張子のみをupstream(PHP-FPM等のfastcgi)へ転送し、それ以外の静的ファイルは nginx-proxy 自身が直接配信できるようにする。fastcgiバックエンド利用時の実用性を上げる。
- **停滞理由:** ラベル status/pr-needs-tests が付与済み(テスト不足)。作者によるfastcgi+PHP-FPM限定の検証のみ。現行 main と競合。
- **メモ:** 78行のテンプレート変更でfastcgi経路という比較的ニッチな領域。静的配信のボリュームマウント前提も必要で要件が複雑。テスト整備+rebaseで工数大。需要は限定的なため優先度低。

##### [#2069](https://github.com/nginx-proxy/nginx-proxy/pull/2069) Added php-fpm backend

**価値:** low ・ **復活工数:** high ・ **規模:** M ・ **マージ状態:** CONFLICTING (base main) ・ **作成:** 2022-10-15 (約3.7年前) ・ **作者:** g-alfieri

- **内容:** /usr/share/html/$virtual_host の静的ファイルをnginxで配信し、PHPは php-fpm に渡す php-fpm バックエンドを追加。
- **停滞理由:** レビュー・コメント皆無、メンテナ沈黙。テスト/ドキュメント無し。静的ファイル配信のためのボリューム手動マウント問題は #1037 と同じ構造的制約を抱える。
- **メモ:** type/feat ラベルのみ。#1037 と同じく nginx-proxy 本体へのボリュームマウント前提という難点。PHP配信はユーザーが別phpコンテナを背後に置く方が筋が良く、優先度低い。

##### [#2091](https://github.com/nginx-proxy/nginx-proxy/pull/2091) Load Network Config depending on NETWORK_ACCESS env var

**価値:** low ・ **復活工数:** medium ・ **規模:** S ・ **マージ状態:** CONFLICTING (base main) ・ **作成:** 2022-11-08 (約3.6年前) ・ **作者:** solrac200

- **内容:** 従来 NETWORK_ACCESS=internal の時だけ network_internal.conf を読む挙動を、NETWORK_ACCESS の値をそのまま config ファイル名に使って network_<value>.conf を読むよう一般化。Dockerfile と nginx.tmpl を変更 (+4/-4)。
- **停滞理由:** メンテナ buchdag が即座に『テストなし・ドキュメントなし・network_internal.conf 利用者の後方互換を壊す。これらが対処されない限り merge しない』と明確にブロック (needs-tests/needs-docs)。author が応答せず3年半放置。
- **メモ:** needs-tests + needs-docs + type/feat ラベル付き。ブロッカーはメンテナにより明文化済み。機能はニッチ (任意名の network config 読み込み) でユースケースが弱く value 低。後方互換を保つ実装 (internal は従来通り + 追加で任意名対応) + テスト + docs が必須。author 不在のため revive なら別コントリビュータが再実装する形。低優先。

#### 陳腐化の可能性大 (likely-obsolete) — 6件

##### [#161](https://github.com/nginx-proxy/nginx-proxy/pull/161) added info how to modify nginx config in startup command

**価値:** low ・ **復活工数:** low ・ **規模:** S ・ **マージ状態:** UNKNOWN (base main) ・ **作成:** 2015-05-08 (約11.1年前) ・ **作者:** schmunk42

- **内容:** README に、コンテナ起動コマンド内で sed 等を使って nginx.tmpl / 生成済み設定を一時的に書き換えるデバッグ用 Tips を追記するドキュメント変更。著者自身も「temporary / hacky なら閉じてよい」と述べている。
- **停滞理由:** メンテナ(md5)が「wikiに書くのが適切」とコメントし放置。著者も自虐的に hacky と認めており本体マージの意欲が薄い。11年間メンテナ沈黙。
- **メモ:** 現在は per-vhost カスタム設定ファイル(/etc/nginx/vhost.d/)や proxy.conf 上書きという正規手段があるため、この sed ハック手順はほぼ不要。docs/ 構成も当時と変わっており再利用価値は低い。likely-obsolete。

##### [#379](https://github.com/nginx-proxy/nginx-proxy/pull/379) Only start nginx after dockergen is done

**価値:** low ・ **復活工数:** high ・ **規模:** S ・ **マージ状態:** UNKNOWN (base main) ・ **作成:** 2016-03-02 (約10.3年前) ・ **作者:** msabramo

- **内容:** Procfileを2行変更し、forego/dockergen完了後にnginxを起動するよう順序を変える(カスタムnginx設定が生成済み設定に依存して起動失敗する問題への対処)。
- **停滞理由:** maintainer-silence; 2016年の超古いPRで放置。ラベルなし、コメントなし。
- **メモ:** 対象のルート直下Procfileは現在存在せず、プロセス構成はapp/docker-entrypoint.sh + app/Procfileへ再編済み。アーキテクチャが変わっており現状のdocker-genは設定生成後にreloadする方式。関連issue #378は今もOPENだが、この変更自体は陳腐化。再実装するなら現行entrypointで対応すべき。
- **関連Issue:** [#378](https://github.com/nginx-proxy/nginx-proxy/issues/378)

##### [#561](https://github.com/nginx-proxy/nginx-proxy/pull/561) Dedicated nginx configuration file

**価値:** low ・ **復活工数:** high ・ **規模:** S ・ **マージ状態:** CONFLICTING (base main) ・ **作成:** 2016-09-06 (約9.8年前) ・ **作者:** rgerovski

- **内容:** Dockerfile内のsed操作をやめ、専用の nginx.conf ファイルをリポジトリに追加してそこへ設定を集約する。
- **停滞理由:** メンテナ陣が上流nginxのデフォルト設定を追従する設計意図を説明し、静的configを持つ是非で議論が止まった。COPYで上書きするだけで代替可能との指摘も。
- **メモ:** 約10年前。現在のリポジトリは既にビルド/設定構成が刷新されており、当時のDockerfile前提が陳腐化。再実装するなら現行構成に合わせ一から設計すべき。

##### [#784](https://github.com/nginx-proxy/nginx-proxy/pull/784) Added DEFAULT_IP env variable

**価値:** low ・ **復活工数:** high ・ **規模:** M ・ **マージ状態:** CONFLICTING (base main) ・ **作成:** 2017-04-06 (約9.2年前) ・ **作者:** senzacionale

- **内容:** コンテナごとにDEFAULT_IP環境変数でupstreamのIPを上書き可能にする(未指定時はコンテナIP)。同時にupstream生成ロジックを簡略化。
- **停滞理由:** maintainer-silence; フォーク個人カスタマイズ的内容で放置。コメント0件。
- **メモ:** diffにメンテナLABELやDocker Hub参照をsenzacionale/に書き換える等の不適切な変更を含む。nginx:1.14.1固定、ネットワーク到達不可時のserver...downフォールバックロジックを削除しており現行テンプレと非互換。実質フォーク用改変でマージ不可。ユースケース(任意IP指定)はニッチ。再実装するならVIRTUAL_DEST等既存機能で代替検討。

##### [#943](https://github.com/nginx-proxy/nginx-proxy/pull/943) Add in vhost https bypass

**価値:** low ・ **復活工数:** high ・ **規模:** S ・ **マージ状態:** UNKNOWN (base main) ・ **作成:** 2017-10-07 (約8.7年前) ・ **作者:** cryxia

- **内容:** vhost include で提供される location を HTTPSリダイレクトの対象から除外できるようにする (nginx.tmpl +12/-1)。Let's Encrypt の HTTP-01 チャレンジを証明書発行前でも200で返せるようにするのが目的。
- **停滞理由:** コメントゼロ・ラベルなしで完全放置。8.7年メンテナ無反応。
- **メモ:** 目的(ACMEチャレンジのHTTP通過)は、現在 acme-companion + HTTPS_METHOD の仕組みおよびテンプレ側のACMEチャレンジ用 location 既定処理で既に解決済み。当時のnginx.tmplから構造が激変しており、そのままのパッチは適用不能で再設計が必要。価値も既存機能と重複。likely-obsolete。

##### [#2377](https://github.com/nginx-proxy/nginx-proxy/pull/2377) add http include

**価値:** low ・ **復活工数:** low ・ **規模:** S ・ **マージ状態:** UNKNOWN (base main) — 比較的新しく diff も極小だが、近年テンプレ更新があり軽微な CONFLICTING の可能性 ・ **作成:** 2024-01-22 (約2.4年前) ・ **作者:** azlux

- **内容:** 生成される nginx 設定の http ブロックに任意の include を差し込めるようにする(http コンテキストレベルのカスタム設定読み込み)。nginx.tmpl 3行 + docs/README.md 2行。
- **停滞理由:** discussion #2376 起点。2025年にメンテナ(buchdag)が『まだ必要か?』と確認したところ、作者(azlux)が『自分にはもう不要』と回答。requester-no-longer-interested
- **メモ:** 提案者本人が不要と明言したため当PRとしては実質終了。ただし http コンテキストレベルの include 機能自体は他ユースケースで有用な可能性あり(buchdag も他用途に言及)。diff は極小で復活コスト自体は低いが、需要・ドキュメント確認なしに進める価値は低い。クローズ候補。
- **関連Issue:** [#2376](https://github.com/nginx-proxy/nginx-proxy/issues/2376)

---

## Issue トリアージ

### カテゴリ別サマリ

| カテゴリ | 件数 |
|---|---|
| VIRTUAL_HOST / routing / vhost matching | 59 |
| TLS/HTTPS/redirect | 49 |
| Custom config / templating (nginx.tmpl) | 31 |
| Docker Compose / Swarm / labels / docker-gen | 29 |
| Proxy tuning (body size, timeouts, buffering, websockets, HTTP2/3) | 25 |
| Headers / CORS / forwarded-for | 20 |
| Multi-port / multi-network / IPv6 | 19 |
| Load balancing / upstream / health checks | 18 |
| DNS / hostnames / aliases / default host | 16 |
| Build / image / base / dependencies | 12 |
| Documentation | 7 |
| Logging / monitoring / debugging | 4 |
| Other | 3 |

### 実装候補トップ(48件)

各チャンクのトリアージが「実際に着手する価値が高い」と判定したIssue。

| Issue | タイトル | カテゴリ | 容易度 | コメント | 着目理由 |
|---|---|---|---|---|---|
| [#97](https://github.com/nginx-proxy/nginx-proxy/issues/97) | Docker Swarm Support or Multiple Node Support | Docker Compose / Swarm / labels / docker-gen | needs-discussion | 55 | Docker Swarmサポートはコメント数55と全チャンク最多のエンゲージメント。完全実装は複雑だが、現状の制限と推奨構成をドキュメント化するだけでも大きな価値があり、2025年まで更新されていることから依然として関心が高い。 |
| [#139](https://github.com/nginx-proxy/nginx-proxy/issues/139) | gzip_types has no effect without setting gzip and gzip_proxied | Proxy tuning (body size, timeouts, buffering, websockets, HTTP2/3) | likely-obsolete | 23 | gzip設定のデフォルト有効化はコメント数23と高エンゲージメント。nginx.tmplにgzip on、gzip_proxied any、gzip_typesを正しく設定するだけの小さな変更で多くのユーザーのパフォーマンスが向上する。 |
| [#180](https://github.com/nginx-proxy/nginx-proxy/issues/180) | page redirection from www to non-www | VIRTUAL_HOST / routing / vhost matching | needs-discussion | 28 | wwwから非wwwリダイレクトはコメント数28と非常に高いエンゲージメントを持ち、多くのユーザーが必要としている機能。環境変数（例：REDIRECT_WWW）でnginx.tmplにリダイレクトブロックを追加する形で実装可能であり、スコープも明確。 |
| [#221](https://github.com/nginx-proxy/nginx-proxy/issues/221) | session affinity | Load balancing / upstream / health checks | complex | 3 | セッションアフィニティはロードバランシング環境では必須の機能で、コメント数3と比較的新しいエンゲージメントがある。nginxのip_hashやhash $remote_addrをテンプレートで選択できるようにする実装が考えられる。 |
| [#419](https://github.com/nginx-proxy/nginx-proxy/issues/419) | Waiting for healthy containers | Load balancing / upstream / health checks | needs-discussion | 0 | コンテナのヘルスチェック連携は実際の運用で重要な機能。Dockerのhealthcheck状態をdocker-genで参照してupstreamから除外するロジックをnginx.tmplに追加する形で実装可能であり、現代のコンテナ運用に合致した要望。 |
| [#516](https://github.com/nginx-proxy/nginx-proxy/issues/516) | Occasional Error 502 Bad Gateway | Proxy tuning (body size, timeouts, buffering, websockets, HTTP2/3) | likely-obsolete | 21 | 散発的な502エラーはコメント数21と高エンゲージメント。コンテナ再起動時のnginxリロードとupstream解決のタイミング問題を修正することで多くのユーザーに恩恵があり、再現手順の調査価値がある。 |
| [#634](https://github.com/nginx-proxy/nginx-proxy/issues/634) | Send default 404 page when no vhost is matched | VIRTUAL_HOST / routing / vhost matching | feasible | 1 | vhostにマッチしないリクエストに対してデフォルト404ページを返す機能は小さく明確なスコープで実装可能。2025年まで更新されており現役のissueとして需要がある。 |
| [#663](https://github.com/nginx-proxy/nginx-proxy/issues/663) | Any way to route to something besides a docker container? | Custom config / templating (nginx.tmpl) | feasible | 4 | 「scope/configurable-upstream」ラベルが付いており、Dockerコンテナ以外へのルーティング（ホストサービスや外部サーバー）のサポートは実用的なユースケースが多く、2025年まで更新されている現役のissue。実装範囲が明確で取り組みやすい。 |
| [#694](https://github.com/nginx-proxy/nginx-proxy/issues/694) | AWS upstream sent too big header while reading response header from upstream | Headers / CORS / forwarded-for | feasible | 4 | AWSバックエンドからの大きなヘッダーによるエラーは多くのユーザーが遭遇する実際の問題であり、proxy_buffer_sizeのデフォルト値引き上げまたは環境変数での設定可能化という明確な解決策がある。 |
| [#708](https://github.com/nginx-proxy/nginx-proxy/issues/708) | Support for Zero downtime deployment - asking for guidance for PR | Load balancing / upstream / health checks | complex | 3 | ゼロダウンタイムデプロイのサポートはCI/CDワークフローで頻繁に求められる機能。PRガイダンスを求めているissueであり、実装方針の議論ベースが既にある。 |
| [#725](https://github.com/nginx-proxy/nginx-proxy/issues/725) | Suggestion: Add an index page | VIRTUAL_HOST / routing / vhost matching | feasible | 3 | プロキシ先コンテナ一覧を示すインデックスページの追加は小規模で自己完結した機能追加。デバッグやサービス発見の観点から実用的で、nginx.tmplへの変更のみで実装可能。 |
| [#792](https://github.com/nginx-proxy/nginx-proxy/issues/792) | Support for stream module | Custom config / templating (nginx.tmpl) | complex | 6 | nginxのstreamモジュールによるTCP/UDPプロキシサポートはnginx-proxyの大きな機能拡張となり、コメント数6かつ2026年まで更新されている非常に活発なissue。需要が高くコミュニティの関心も継続している。 |
| [#830](https://github.com/nginx-proxy/nginx-proxy/issues/830) | htpasswd when using wildcard virtual host name | Custom config / templating (nginx.tmpl) | feasible | 0 | ワイルドカードVIRTUAL_HOSTでhtpasswd認証が機能しない問題はテンプレートのマッチングロジックの修正で対応可能で、範囲が明確なバグ修正。 |
| [#871](https://github.com/nginx-proxy/nginx-proxy/issues/871) | USE_IP_HASH seems to have zero impact to the default.conf | Load balancing / upstream / health checks | likely-obsolete | 2 | USE_IP_HASHが設定に反映されないバグはロードバランシングのセッション固定に影響する問題で、テンプレートのデバッグと修正で対応できる範囲が明確な課題。 |
| [#886](https://github.com/nginx-proxy/nginx-proxy/issues/886) | Allow upstream options | Load balancing / upstream / health checks | needs-discussion | 0 | upstreamブロックへのオプション追加（keepalive等）はパフォーマンス向上に直結し、環境変数経由でのテンプレート拡張として実装できる有用な機能拡張。 |
| [#891](https://github.com/nginx-proxy/nginx-proxy/issues/891) | Add an UUID to every incoming request | Headers / CORS / forwarded-for | feasible | 0 | リクエストごとのUUID付与（X-Request-ID）はデバッグ・トレーシングに有用な機能で、nginx.tmplへの数行追加で実装可能なgood-firstレベルの改善。 |
| [#918](https://github.com/nginx-proxy/nginx-proxy/issues/918) | CloudFlare Flexible SSL breaks redirections because of X-Forwarded-Port | Headers / CORS / forwarded-for | feasible | 4 | CloudFlare Flexible SSL使用時のX-Forwarded-Portによるリダイレクト問題は2024年まで更新が続いており依然関連性が高く、テンプレート修正で対応できる明確な範囲のバグ。 |
| [#929](https://github.com/nginx-proxy/nginx-proxy/issues/929) | Forward SSL to container (not handle ssl by proxy) | TLS/HTTPS/redirect | needs-discussion | 4 | SSLパススルー機能はコメント数や2020年まで更新された事実から需要が高く、将来的なstream{}モジュール対応の基盤となる重要な機能要求。 |
| [#960](https://github.com/nginx-proxy/nginx-proxy/issues/960) | Add an option to redirect a domain to www.domain | VIRTUAL_HOST / routing / vhost matching | feasible | 2 | ドメインをwwwサブドメインに自動リダイレクトするオプション追加。よく要求される機能で、テンプレート修正またはフラグ追加で実装可能。関連issue #958も20コメントあり需要が高い。 |
| [#981](https://github.com/nginx-proxy/nginx-proxy/issues/981) | jwilder/nginx-proxy upload limits? | Proxy tuning (body size, timeouts, buffering, websockets, HTTP2/3) | feasible | 44 | アップロードサイズ制限（client_max_body_size）の設定方法についての質問がコメント44件と最高エンゲージメント。ドキュメントやREADMEへの明記、または環境変数対応で多くのユーザーの問題を解消できる。 |
| [#987](https://github.com/nginx-proxy/nginx-proxy/issues/987) | Feature Proposal: Opt-in support for HEALTHCHECK option | Load balancing / upstream / health checks | feasible | 3 | コンテナのHEALTHCHECKステータスに基づきプロキシ対象を制御するオプトイン機能で、2025年11月まで更新があり継続的な需要が確認できる。docker-genのラベル連携で実装可能。 |
| [#992](https://github.com/nginx-proxy/nginx-proxy/issues/992) | https website redirect to another website if upstream is down | TLS/HTTPS/redirect | feasible | 15 | 上流ダウン時に誤ったHTTPSリダイレクトが発生するバグで、2025年1月まで更新があり現在も再現している可能性が高い。コメント数15と高エンゲージメントで、テンプレート修正で対応できる範囲。 |
| [#998](https://github.com/nginx-proxy/nginx-proxy/issues/998) | Howto proxy an app running without docker | Documentation | feasible | 11 | 非Dockerアプリをプロキシする方法についての質問でコメント11件と高い関心。2024年まで更新あり。ドキュメントに設定例を追加するだけで多くのユーザーに価値を提供できる。 |
| [#1013](https://github.com/nginx-proxy/nginx-proxy/issues/1013) | Empty upstream definition leads to nginx-proxy failing to start/restart | Custom config / templating (nginx.tmpl) | likely-obsolete | 1 | 空のupstream定義時にnginx-proxyが起動失敗するバグで、テンプレート側での空チェック追加という明確な解決策がある。スコープが小さく実装しやすい。 |
| [#1091](https://github.com/nginx-proxy/nginx-proxy/issues/1091) | The template can (and sometimes does) generate an empty upstream section | Custom config / templating (nginx.tmpl) | feasible | 2 | 空のupstreamセクションが生成されるバグはnginxの設定ファイルエラーを引き起こすため影響が大きく、テンプレートの条件分岐修正という明確な解決策がある。 |
| [#1121](https://github.com/nginx-proxy/nginx-proxy/issues/1121) | SNI support with HTTPS backends | TLS/HTTPS/redirect | feasible | 0 | HTTPSバックエンドへのSNI送信機能で、proxy_ssl_server_name onを追加するだけという小規模で明確な実装が可能。現代のマイクロサービス環境では需要がある。 |
| [#1136](https://github.com/nginx-proxy/nginx-proxy/issues/1136) | vhost.d location config disables HSTS | TLS/HTTPS/redirect | feasible | 1 | vhost.dのlocation設定を使うとHSTSが無効化されるセキュリティに関わるバグ。HSTSをserver contextに移動するという修正方針が明確で影響が大きい。 |
| [#1148](https://github.com/nginx-proxy/nginx-proxy/issues/1148) | Proxying container exposing multiple ports (but only one HTTP) | Multi-port / multi-network / IPv6 | feasible | 7 | 複数ポートを公開するコンテナでHTTPポートを自動選択する機能で、コメント数7と需要が高く、2024年にも更新されており現在も有効な課題。VIRTUAL_PORTを指定しなくて良くなる実用的な改善。 |
| [#1150](https://github.com/nginx-proxy/nginx-proxy/issues/1150) | Make CORS with Basic Auth possible by including _location files after basic_auth | Headers / CORS / forwarded-for | feasible | 1 | CORS + Basic Auth の組み合わせを有効にするためのテンプレート順序変更で、実装範囲が小さく実用的なユースケースに対応できる。 |
| [#1161](https://github.com/nginx-proxy/nginx-proxy/issues/1161) | generate-dhparam becomes a zombie process | Build / image / base / dependencies | feasible | 6 | generate-dhparamのゾンビプロセス問題でコメント数6、2020年まで追跡されており実際の運用に影響がある。プロセス管理の修正として範囲が明確で実装しやすい。 |
| [#1204](https://github.com/nginx-proxy/nginx-proxy/issues/1204) | Redirect from http non-www to https www (single redirect) | TLS/HTTPS/redirect | feasible | 24 | コメント数24と最多で需要が非常に高い。http非wwwからhttps wwwへのシングルリダイレクトは多くのユーザーが直面する実用的な要求で、エイリアス処理の改善として実装スコープも明確。 |
| [#1208](https://github.com/nginx-proxy/nginx-proxy/issues/1208) | pwritev() "/var/cache/nginx/client_temp/0000000014" failed (28: No space left on device) | Proxy tuning (body size, timeouts, buffering, websockets, HTTP2/3) | feasible | 6 | ディスクフルエラーはコメント6件あり実環境での影響が大きい。client_body_temp_pathのtmpfsマウントや設定ドキュメントの整備で対応でき実装角度が明確。 |
| [#1256](https://github.com/nginx-proxy/nginx-proxy/issues/1256) | no resolver defined to resolve ocsp.int-x3.letsencrypt.org while requesting certificate status | TLS/HTTPS/redirect | feasible | 11 | コメント数11と多く、OCSPステープリング用のresolver未定義はnginx.tmplに数行追加するだけで解決できる明確なバグ修正。TLS関連の実用性も高い。 |
| [#1285](https://github.com/nginx-proxy/nginx-proxy/issues/1285) | Improve / alternate aliases handling | DNS / hostnames / aliases / default host | feasible | 4 | エイリアスホスト処理の改善はコメント4件あり、1204や1254など関連する複数issueの根本解決にもつながる。実装範囲が明確でテンプレート修正で対応可能。 |
| [#1309](https://github.com/nginx-proxy/nginx-proxy/issues/1309) | Check closest host match when looking for vhost.d/config files before checking default | Custom config / templating (nginx.tmpl) | feasible | 0 | vhost.d設定ファイルのフォールバックロジック改善は、nginx.tmplのテンプレートロジック変更のみで対応可能なスコープが明確な機能改善。 |
| [#1334](https://github.com/nginx-proxy/nginx-proxy/issues/1334) | Ability to set the NETWORK_ACCESS default to prevent accidental service exposure | Multi-port / multi-network / IPv6 | feasible | 0 | NETWORK_ACCESSのグローバルデフォルト設定は、セキュリティ観点から需要があり環境変数一つの追加で実装できる比較的小規模な機能追加。 |
| [#1377](https://github.com/nginx-proxy/nginx-proxy/issues/1377) | "load_module" directive is not allowed here in /etc/nginx/conf.d/nginx.conf:1 | Custom config / templating (nginx.tmpl) | feasible | 2 | load_moduleディレクティブがconf.dに置けない問題は設定構造上の制約であり、nginx.tmplまたはメイン設定にload_module用のインクルードポイントを追加することで解決できる明確な改善。 |
| [#1380](https://github.com/nginx-proxy/nginx-proxy/issues/1380) | Dockerfile EXPOSE overrides --expose argument | Multi-port / multi-network / IPv6 | feasible | 1 | DockerfileのEXPOSEが--expose引数より優先される問題はポート選択ロジックのバグとして明確で、docker-genのポート選択コードの修正で対応できる。 |
| [#1383](https://github.com/nginx-proxy/nginx-proxy/issues/1383) | Intermittent 502 bad gateway error | Load balancing / upstream / health checks | feasible | 8 | 断続的な502エラーはコメント数8で最も議論が活発。コンテナ再起動時の設定再生成タイミング問題として具体的な調査と改善が可能で、影響ユーザーが多い。 |
| [#1431](https://github.com/nginx-proxy/nginx-proxy/issues/1431) | Incorrect Redirect Response Code | TLS/HTTPS/redirect | feasible | 5 | HTTPSリダイレクト時のステータスコード（301 vs 307）の問題はコメント数5で再現性が高く、nginx.tmplのreturnディレクティブ修正という小さな変更で解決できる可能性がある。 |
| [#1455](https://github.com/nginx-proxy/nginx-proxy/issues/1455) | Logrotate | Logging / monitoring / debugging | feasible | 4 | ログローテーション機能はコメント数4で需要があり、2025年まで更新が続いている現役イシュー。logrotate設定の同梱またはUSR1シグナル対応という実装方法が明確。 |
| [#1474](https://github.com/nginx-proxy/nginx-proxy/issues/1474) | Required VIRTUAL_UPSTREAM: support the web service not in docker. | VIRTUAL_HOST / routing / vhost matching | feasible | 6 | Dockerコンテナ外のサービスをVIRTUAL_UPSTREAMで指定する機能はコメント数6で需要が高く、#1350とも類似。nginx.tmplのupstreamブロック生成に環境変数サポートを追加する形で実装範囲が明確。 |
| [#1555](https://github.com/nginx-proxy/nginx-proxy/issues/1555) | nginx-proxy uses default host without DEFAULT_HOST set | DNS / hostnames / aliases / default host | feasible | 7 | DEFAULT_HOST未設定時の意図せぬデフォルトホスト動作は7コメントと継続的な関心あり。動作定義が明確でnginx.tmpl修正で対応可能。 |
| [#1613](https://github.com/nginx-proxy/nginx-proxy/issues/1613) | Should nginx.tmpl's https_method=redirect block try to include vhost.d files? | TLS/HTTPS/redirect | good-first | 3 | https_method=redirectブロックでvhost.dをincludeするかどうかという明確な改善提案。nginx.tmplへの小さな変更で対応可能。 |
| [#1974](https://github.com/nginx-proxy/nginx-proxy/issues/1974) | Prevent X-Forwareded-For header passing from untrusted upstreams | Headers / CORS / forwarded-for | feasible | 4 | 信頼できないアップストリームからのX-Forwarded-Forヘッダー除去はセキュリティ上重要で、4コメントのエンゲージメントあり。nginx.tmpl修正で対応可能。 |
| [#2577](https://github.com/nginx-proxy/nginx-proxy/issues/2577) | Exposing gRPC service through SSL results in protocol error | Proxy tuning (body size, timeouts, buffering, websockets, HTTP2/3) | feasible | 5 | gRPC/SSL経由のプロトコルエラーは5コメントの関心があり、HTTP/2設定の明確な改善で対応できる可能性が高い。 |
| [#2637](https://github.com/nginx-proxy/nginx-proxy/issues/2637) | extend multiports to finally have a proper raw / non-http ports API | Multi-port / multi-network / IPv6 | complex | 4 | non-HTTP（TCP/UDP）ポートへの対応は4コメントで活発な議論あり。streamモジュール統合という明確な方向性があり長期的価値が高い。 |
| [#2699](https://github.com/nginx-proxy/nginx-proxy/issues/2699) | [Bug/Docs] Environment variable RESOLVERS is overwritten by entrypoint script, contradicting documentation | DNS / hostnames / aliases / default host | good-first | 0 | RESOLVERSが上書きされるバグでスコープが明確。ドキュメントとの矛盾も修正できる典型的なgood-firstイシュー。 |

### 全Issueカタログ(カテゴリ別・全292件)

凡例 — **種別:** bug=バグ / feature=機能要望 / question-support=質問・サポート / docs=ドキュメント / meta=運用。 **容易度:** good-first=着手容易 / feasible=実装可能 / complex=大規模 / needs-discussion=要議論 / likely-obsolete=陳腐化の可能性。

<details>
<summary><b>VIRTUAL_HOST / routing / vhost matching</b> — 59件</summary>

| Issue | タイトル | 種別 | 容易度 | コメント | 作成 | メモ |
|---|---|---|---|---|---|---|
| [#1474](https://github.com/nginx-proxy/nginx-proxy/issues/1474) | Required VIRTUAL_UPSTREAM: support the web service not in docker. | feature | feasible | 6 | 2020-07-21 | VIRTUAL_UPSTREAM環境変数でDocker外部のサービス（例: ホストマシンや外部サーバ）を上流として指定できるようにする機能要求。nginx.tmplのupstreamブロック生成に変数サポートを追加する形で実装可能。 |
| [#725](https://github.com/nginx-proxy/nginx-proxy/issues/725) | Suggestion: Add an index page | feature | feasible | 3 | 2017-02-17 | デフォルトのvhostにプロキシ先一覧を表示するインデックスページを追加してほしい。nginx.tmplの変更またはデフォルトroot設定で実装可能。 |
| [#960](https://github.com/nginx-proxy/nginx-proxy/issues/960) | Add an option to redirect a domain to www.domain | feature | feasible | 2 | 2017-10-31 | ドメインをwwwサブドメインに自動リダイレクトするオプションの追加要求。 |
| [#634](https://github.com/nginx-proxy/nginx-proxy/issues/634) | Send default 404 page when no vhost is matched | feature | feasible | 1 | 2016-11-20 | vhostにマッチしない場合にデフォルト404ページを返す機能のリクエスト。デフォルトサーバーブロックのカスタマイズで実装可能。 |
| [#1024](https://github.com/nginx-proxy/nginx-proxy/issues/1024) | add Virtual path support with virtual_host | feature | feasible | 0 | 2018-01-14 | VIRTUAL_HOSTとともにVIRTUAL_PATH（パスベースルーティング）をサポートする機能追加要求。 |
| [#1350](https://github.com/nginx-proxy/nginx-proxy/issues/1350) | Configure vhost to proxy to remote server via ENV var | feature | feasible | 0 | 2019-10-31 | 環境変数でDocker外部のリモートサーバーへプロキシするvhostを設定する機能要求。VIRTUAL_UPSTREAMやVIRTUAL_PROTOとの組み合わせで実装可能で、#1474と類似の要求。 |
| [#1249](https://github.com/nginx-proxy/nginx-proxy/issues/1249) | Cannot use subdomain with dot if 'parent' is already defined. | bug | feasible | 1 | 2019-03-20 | 親ドメインが既に定義されている場合にドット付きサブドメインが正しくルーティングされないバグ。nginx.tmplのvhostマッチングロジックの修正が必要。 |
| [#2689](https://github.com/nginx-proxy/nginx-proxy/issues/2689) | [1.9.0] docker-compose v1 - FAILED test/test_virtual-path/test_virtual-paths.py::test_container_hotplug - assert 502 == 404 | bug | feasible | 0 | 2025-12-20 | v1.9.0でVIRTUAL_PATHのホットプラグテストが502を返し404を期待する失敗。テストスイートまたはテンプレートのロジック修正が必要。 |
| [#1528](https://github.com/nginx-proxy/nginx-proxy/issues/1528) | Containers with overloaded virtual hosts do not work as expected | bug | feasible | 0 | 2020-11-09 | 同一バーチャルホスト名に複数コンテナを割り当てた際の予期せぬ動作。ロードバランシング設定のバグの可能性。 |
| [#1486](https://github.com/nginx-proxy/nginx-proxy/issues/1486) | Using a VIRTUAL_HOST with a FQDN without a hostname randomly causes default nginx webpage to be displayed | bug | feasible | 0 | 2020-08-07 | FQDNにホスト名なし（例: example.com.）を使うとデフォルトページが表示される問題。nginx.tmplのvhostマッチングロジックの末尾ドット処理を修正する必要がある。 |
| [#180](https://github.com/nginx-proxy/nginx-proxy/issues/180) | page redirection from www to non-www | feature | needs-discussion | 28 | 2015-06-12 | wwwから非wwwへの自動リダイレクト機能の要望。コメント数28と非常に多く、ニーズは高いが実装方針（環境変数で制御など）の議論が必要。 |
| [#1099](https://github.com/nginx-proxy/nginx-proxy/issues/1099) | proxy_pass to IP instead of VIRTUAL_HOST | feature | needs-discussion | 5 | 2018-03-15 | VIRTUAL_HOSTのホスト名ではなくIPアドレスでproxy_passしたい要求。コメント数が多く需要あるが設計変更が伴う。 |
| [#772](https://github.com/nginx-proxy/nginx-proxy/issues/772) | Preventing VIRTUAL_HOST mixup | feature | needs-discussion | 1 | 2017-03-21 | 複数のコンテナが同じVIRTUAL_HOSTを設定したときの意図しない混在を防ぐ手段を求めている。ラベルの重複検出ロジックが必要。 |
| [#1529](https://github.com/nginx-proxy/nginx-proxy/issues/1529) | 502 Bad Gateway on Centos 8 | question-support | needs-discussion | 15 | 2020-11-09 | CentOS 8環境での502エラー。15コメントと高い関心を集めているがOS固有の設定問題の可能性。 |
| [#2302](https://github.com/nginx-proxy/nginx-proxy/issues/2302) | 500 Internal Server Error | question-support | needs-discussion | 9 | 2023-09-17 | 500エラーの汎用的な問題レポート。コメント数が多く様々な原因が議論されているサポート系。 |
| [#2378](https://github.com/nginx-proxy/nginx-proxy/issues/2378) | 502 Bad Gateway & Upstream connection refused | question-support | needs-discussion | 8 | 2024-01-24 | 502エラーとアップストリーム接続拒否の問題。エンゲージメントが高くネットワーク設定やコンテナ検出の問題が多いサポート系。 |
| [#1513](https://github.com/nginx-proxy/nginx-proxy/issues/1513) | How can I fix this 502 Bad Gateway? | question-support | needs-discussion | 8 | 2020-10-12 | 汎用的な502エラーの質問。8コメントと関心は高いが一般的なサポート系。ドキュメント改善で対応可能。 |
| [#1485](https://github.com/nginx-proxy/nginx-proxy/issues/1485) | Proxy to host service | question-support | needs-discussion | 2 | 2020-08-06 | Dockerコンテナからホストマシン上のサービスへプロキシしたい要望。host.docker.internalやVIRTUAL_UPSTREAMで対応可能かどうかの議論が必要。 |
| [#1051](https://github.com/nginx-proxy/nginx-proxy/issues/1051) | Add virtual host without recreating container | feature | likely-obsolete | 5 | 2018-02-02 | コンテナを再作成せずにvirtualホストを追加したい機能要求。docker-genがDockerイベントを監視して自動更新するため現在は解決済みの可能性。 |
| [#710](https://github.com/nginx-proxy/nginx-proxy/issues/710) | Redirect default vhost to simple index.html | feature | likely-obsolete | 2 | 2017-02-09 | デフォルトvhostをカスタムindex.htmlにリダイレクトまたは表示させる方法を求めている。カスタムファイルのボリュームマウントで対応可能。 |
| [#576](https://github.com/nginx-proxy/nginx-proxy/issues/576) | Virtual host mask | feature | likely-obsolete | 1 | 2016-09-20 | ワイルドカード形式のVIRTUAL_HOSTマスク（例：*.example.com）のサポート要望。現在はワイルドカード対応済みの可能性があり、古いイシュー。 |
| [#1031](https://github.com/nginx-proxy/nginx-proxy/issues/1031) | Nginx-Proxy not forwarding | bug | likely-obsolete | 6 | 2018-01-17 | nginx-proxyがリクエストを正しくフォワードしない問題。 |
| [#434](https://github.com/nginx-proxy/nginx-proxy/issues/434) | wrong regExp matching | bug | likely-obsolete | 3 | 2016-04-28 | 正規表現ベースのVIRTUAL_HOSTマッチングが誤動作するバグ。テンプレートのロジック修正が必要だが古いイシュー。 |
| [#1212](https://github.com/nginx-proxy/nginx-proxy/issues/1212) | Error only with Subdomain 502 Bad gateway | bug | likely-obsolete | 3 | 2018-12-15 | サブドメインのみ502エラーになる問題。SSL証明書やDNS設定の問題が原因の可能性が高い。 |
| [#1143](https://github.com/nginx-proxy/nginx-proxy/issues/1143) | Alternative ports producing 503 error | bug | likely-obsolete | 3 | 2018-06-18 | 代替ポート使用時に503エラーが発生するバグ。VIRTUAL_PORT設定に関連する古い問題。 |
| [#1186](https://github.com/nginx-proxy/nginx-proxy/issues/1186) | VIRTUAL_PORT variable is not working | bug | likely-obsolete | 2 | 2018-10-25 | VIRTUAL_PORT環境変数が機能しないというバグ報告。2018年と古く現在は修正済みの可能性が高い。 |
| [#686](https://github.com/nginx-proxy/nginx-proxy/issues/686) | VIRTUAL_PORT doesnt work with php:5.6-apache without expose | bug | likely-obsolete | 2 | 2017-01-22 | EXPOSEなしでVIRTUAL_PORTを指定してもphp:5.6-apacheで機能しない問題。docker-genがEXPOSEポートのみを認識する仕様の問題。 |
| [#1316](https://github.com/nginx-proxy/nginx-proxy/issues/1316) | Really weird domain mixup | bug | likely-obsolete | 2 | 2019-08-14 | 異なるドメイン間でVIRTUAL_HOSTのルーティングが混在する奇妙な挙動の報告。設定ミスかdocker-genキャッシュが原因の可能性。 |
| [#2003](https://github.com/nginx-proxy/nginx-proxy/issues/2003) | VIRTUAL_PATH Error | bug | likely-obsolete | 2 | 2022-06-21 | VIRTUAL_PATH設定時のエラー。古いバージョンの問題で現バージョンでは修正済みの可能性。 |
| [#1421](https://github.com/nginx-proxy/nginx-proxy/issues/1421) | Proxying two virtual hosts on the same machine routes to only one of them | bug | likely-obsolete | 1 | 2020-04-15 | 同一マシン上の2つのvhostが一方にしかルーティングされない問題。コンテナネットワーク設定またはVIRTUAL_HOST重複の可能性。 |
| [#1233](https://github.com/nginx-proxy/nginx-proxy/issues/1233) | Failed communicate using VIRTUAL HOST after docker update | bug | likely-obsolete | 1 | 2019-02-15 | Dockerアップデート後にVIRTUAL_HOSTが機能しなくなった問題。Docker APIの変更に対するdocker-genの互換性が原因。 |
| [#625](https://github.com/nginx-proxy/nginx-proxy/issues/625) | container cannot start when any empty VIRTUAL_HOST is supplied | bug | likely-obsolete | 0 | 2016-11-06 | VIRTUAL_HOSTが空文字の場合にコンテナが起動しないバグ。nginx.tmplでの空値バリデーション追加で対応可能だが、古いイシューで現在は修正済みの可能性が高い。 |
| [#779](https://github.com/nginx-proxy/nginx-proxy/issues/779) | Have to use port to access certain vhosts | bug | likely-obsolete | 0 | 2017-03-29 | 特定のvhostへのアクセスにポート番号が必要になる問題。設定ミスや複数コンテナのポート競合が原因と思われる。 |
| [#701](https://github.com/nginx-proxy/nginx-proxy/issues/701) | Reverse Proxy not working | question-support | likely-obsolete | 30 | 2017-01-30 | リバースプロキシが全く機能しない汎用的なトラブル報告。コメント数30と多く、多くの原因パターンを含むFAQ的なissue。 |
| [#1222](https://github.com/nginx-proxy/nginx-proxy/issues/1222) | Error 404 Not Found on .php files only | question-support | likely-obsolete | 18 | 2019-01-26 | PHPファイルのみ404エラーになる問題。nginx-proxyはPHPを直接処理しないため、バックエンドコンテナ（PHP-FPM等）の設定問題。コメント数が多く根強い混乱がある。 |
| [#905](https://github.com/nginx-proxy/nginx-proxy/issues/905) | 502 error with wordpress:fpm | question-support | likely-obsolete | 7 | 2017-08-14 | WordPress FPM構成での502エラー。FPMはHTTPではなくFastCGIプロトコルを使うためnginx-proxyとは直接組み合わせられないケースで、設定ミスの質問。 |
| [#894](https://github.com/nginx-proxy/nginx-proxy/issues/894) | Setup to forward non-www to www subdomain | question-support | likely-obsolete | 4 | 2017-08-02 | non-wwwからwwwへのリダイレクト設定についての質問。カスタム設定ファイルで対応可能な一般的なユースケース。 |
| [#925](https://github.com/nginx-proxy/nginx-proxy/issues/925) | Subdomain 502 Bad gateway | question-support | likely-obsolete | 4 | 2017-09-13 | サブドメイン使用時の502エラー。ネットワーク設定ミスが原因のサポート質問と考えられ、現在は無関係。 |
| [#946](https://github.com/nginx-proxy/nginx-proxy/issues/946) | Multiple virtual hosts, including sub domains | question-support | likely-obsolete | 4 | 2017-10-12 | サブドメインを含む複数のVIRTUAL_HOSTの設定方法についての質問。 |
| [#920](https://github.com/nginx-proxy/nginx-proxy/issues/920) | listening on subdomain | question-support | likely-obsolete | 4 | 2017-09-04 | サブドメインでのリスニング設定に関する使い方の質問。ドキュメントで対応可能な内容。 |
| [#815](https://github.com/nginx-proxy/nginx-proxy/issues/815) | virtual host env did not work when using static ip? | question-support | likely-obsolete | 3 | 2017-05-09 | 静的IPを使用するコンテナでVIRTUAL_HOSTが機能しないという問題。ネットワーク設定の誤りが原因と思われる。 |
| [#1343](https://github.com/nginx-proxy/nginx-proxy/issues/1343) | Unpredictable behaviour | question-support | likely-obsolete | 2 | 2019-10-09 | 予期しない動作の報告。タイトルが曖昧で具体的な問題の特定が困難なサポート案件。 |
| [#1576](https://github.com/nginx-proxy/nginx-proxy/issues/1576) | nginx return file not found with nginx-proxy | question-support | likely-obsolete | 2 | 2021-03-30 | nginx-proxyでファイルが見つからない404エラー。古い問題でサポート系、現バージョンでは解決済みの可能性。 |
| [#1248](https://github.com/nginx-proxy/nginx-proxy/issues/1248) | Curl from container A to B erros out as 400 | question-support | likely-obsolete | 2 | 2019-03-19 | コンテナAからコンテナBへのcurlが400エラーになる問題。Hostヘッダーの扱いやネットワーク設定ミスが原因の典型例。 |
| [#1174](https://github.com/nginx-proxy/nginx-proxy/issues/1174) | +1 Error 503 when access from host/outside to container nginx-proxy | question-support | likely-obsolete | 2 | 2018-10-12 | ホスト外からコンテナへアクセス時に503エラーが発生するサポート質問。設定ミスの可能性が高く古い。 |
| [#1139](https://github.com/nginx-proxy/nginx-proxy/issues/1139) | load http content error 503 | question-support | likely-obsolete | 2 | 2018-06-05 | HTTPコンテンツ読み込み時に503エラーが発生するサポート質問。設定不備の可能性が高い。 |
| [#1295](https://github.com/nginx-proxy/nginx-proxy/issues/1295) | Question: Setting up additional apps on top of (nextcloud+nginx reverse-proxy+letsencrypt + mariadb)) | question-support | likely-obsolete | 1 | 2019-06-24 | 既存のNextcloud＋nginx-proxy構成に追加アプリを乗せる方法の質問。ドキュメントで解決可能な基本的な使い方。 |
| [#1408](https://github.com/nginx-proxy/nginx-proxy/issues/1408) | nginx-proxy not forwarding to ghost blog | question-support | likely-obsolete | 1 | 2020-03-17 | GhostブログコンテナへのプロキシがVIRTUAL_HOST設定後も機能しない問題。コンテナネットワーク設定ミスの可能性。 |
| [#978](https://github.com/nginx-proxy/nginx-proxy/issues/978) | Issues proxying locally | question-support | likely-obsolete | 1 | 2017-11-14 | ローカル環境でプロキシがうまく動かないという問題報告。 |
| [#988](https://github.com/nginx-proxy/nginx-proxy/issues/988) | TypeError: Network request failed | question-support | likely-obsolete | 1 | 2017-11-25 | ネットワークリクエストに失敗するTypeErrorの報告。詳細情報不足。 |
| [#1491](https://github.com/nginx-proxy/nginx-proxy/issues/1491) | 502 bad gateway | question-support | likely-obsolete | 1 | 2020-08-19 | 汎用的な502エラー問題。情報不足の古いサポート系。 |
| [#1320](https://github.com/nginx-proxy/nginx-proxy/issues/1320) | [Question]  How do I use this docker with an internal Website? | question-support | likely-obsolete | 1 | 2019-08-21 | 内部Webサイトをnginx-proxy配下で動かす方法についての基本的な使い方質問。ドキュメントで解決可能。 |
| [#1501](https://github.com/nginx-proxy/nginx-proxy/issues/1501) | proxy_pass in always return 502 bad gateway error | question-support | likely-obsolete | 1 | 2020-09-19 | proxy_passで常に502エラーが発生する問題。古いサポート系で情報不足。 |
| [#887](https://github.com/nginx-proxy/nginx-proxy/issues/887) | Visible redirection. | question-support | likely-obsolete | 0 | 2017-07-26 | リダイレクトが外部から見えてしまう問題についての質問。詳細不明でコメント0件。 |
| [#1035](https://github.com/nginx-proxy/nginx-proxy/issues/1035) | How to redirect all traffic to only ONE host containter? | question-support | likely-obsolete | 0 | 2018-01-20 | すべてのトラフィックを1つのホストコンテナにリダイレクトする方法についての質問。 |
| [#896](https://github.com/nginx-proxy/nginx-proxy/issues/896) | Nginx routing | question-support | likely-obsolete | 0 | 2017-08-08 | nginxルーティングに関する質問。コメント0件で詳細不明、古い未回答の質問。 |
| [#1442](https://github.com/nginx-proxy/nginx-proxy/issues/1442) | Multiple Sites and redirection issue | question-support | likely-obsolete | 0 | 2020-05-29 | 複数サイトのリダイレクト設定に関する質問。設定ミスの可能性が高く、サポート案件。 |
| [#917](https://github.com/nginx-proxy/nginx-proxy/issues/917) | routing issues | question-support | likely-obsolete | 0 | 2017-08-31 | ルーティング問題の報告だがコメント0件で詳細不明。古い未解決の質問。 |
| [#1507](https://github.com/nginx-proxy/nginx-proxy/issues/1507) | Proxy pass 404 error | question-support | likely-obsolete | 0 | 2020-10-06 | プロキシパスで404エラーが発生する問題。情報不足の古いサポート系。 |

</details>

<details>
<summary><b>TLS/HTTPS/redirect</b> — 49件</summary>

| Issue | タイトル | 種別 | 容易度 | コメント | 作成 | メモ |
|---|---|---|---|---|---|---|
| [#1613](https://github.com/nginx-proxy/nginx-proxy/issues/1613) | Should nginx.tmpl's https_method=redirect block try to include vhost.d files? | feature | good-first | 3 | 2021-05-05 | https_method=redirectのサーバーブロックでvhost.dファイルをincludeするかどうかの改善提案。nginx.tmplの小さな修正で対応可能。 |
| [#1204](https://github.com/nginx-proxy/nginx-proxy/issues/1204) | Redirect from http non-www to https www (single redirect) | feature | feasible | 24 | 2018-11-29 | http://exmaple.com から https://www.example.com へシングルリダイレクトを実現する機能。コメント数24と非常に多く需要が高い。エイリアス処理とHTTPS_METHODの組み合わせで実装検討可能。 |
| [#2709](https://github.com/nginx-proxy/nginx-proxy/issues/2709) | [Feature request] Add per-container option to generate self-signed certificate | feature | feasible | 0 | 2026-03-02 | コンテナごとに自己署名証明書を自動生成するオプションを追加してほしいという要望。nginx.tmplとエントリポイントスクリプトの両方改修が必要。 |
| [#1121](https://github.com/nginx-proxy/nginx-proxy/issues/1121) | SNI support with HTTPS backends | feature | feasible | 0 | 2018-04-12 | HTTPSバックエンドへプロキシ時にSNIを送信する機能要求。proxy_ssl_server_name onの設定追加で実装可能。 |
| [#992](https://github.com/nginx-proxy/nginx-proxy/issues/992) | https website redirect to another website if upstream is down | bug | feasible | 15 | 2017-12-06 | 上流コンテナがダウンしているとHTTPSサイトが別サイトにリダイレクトされるバグ。2025年まで更新あり、まだ再現している可能性が高い。 |
| [#1256](https://github.com/nginx-proxy/nginx-proxy/issues/1256) | no resolver defined to resolve ocsp.int-x3.letsencrypt.org while requesting certificate status | bug | feasible | 11 | 2019-03-27 | OCSPステープリング用resolverが未定義でLet's EncryptのOCSP URLが解決できないエラー。nginx.tmplにresolver設定を追加することで解消可能。 |
| [#2023](https://github.com/nginx-proxy/nginx-proxy/issues/2023) | Multiple hosts defined, second one does not get https config in /etc/nginx/conf.d/default.conf | bug | feasible | 8 | 2022-07-18 | 複数ホスト定義時に2番目以降のホストにHTTPS設定が生成されないバグ。nginx.tmplのロジックに問題がある可能性。 |
| [#1431](https://github.com/nginx-proxy/nginx-proxy/issues/1431) | Incorrect Redirect Response Code | bug | feasible | 5 | 2020-05-04 | HTTPからHTTPSへのリダイレクト時に返るステータスコードが期待値と異なる（301ではなく307など）問題。nginx.tmplのreturnディレクティブを修正することで対応可能。 |
| [#1136](https://github.com/nginx-proxy/nginx-proxy/issues/1136) | vhost.d location config disables HSTS | bug | feasible | 1 | 2018-05-26 | vhost.dのlocationカスタム設定を使うとHSTSヘッダーが無効化される問題。HSTSをlocation外のserver contextに移動することで修正可能。 |
| [#1164](https://github.com/nginx-proxy/nginx-proxy/issues/1164) | nginx.tmpl finds certificate of top-level domain for sub-domain. | bug | feasible | 0 | 2018-08-23 | サブドメインに対して親ドメインの証明書が誤って選択される問題。nginx.tmplの証明書マッチングロジックの修正が必要。 |
| [#929](https://github.com/nginx-proxy/nginx-proxy/issues/929) | Forward SSL to container (not handle ssl by proxy) | feature | needs-discussion | 4 | 2017-09-15 | SSLをプロキシで終端せずバックエンドコンテナにSSLをパススルーする機能要求。TCPストリームモードの追加が必要で設計変更を伴う。 |
| [#1122](https://github.com/nginx-proxy/nginx-proxy/issues/1122) | SSL network speed is slow | bug | needs-discussion | 6 | 2018-04-20 | SSL通信の速度が低下する問題。コメント数が多く影響が広いが、原因がnginx設定かDockerネットワークか不明で調査が必要。 |
| [#2615](https://github.com/nginx-proxy/nginx-proxy/issues/2615) | nginx-proxy adds if ($https) return 500; randomly | bug | needs-discussion | 1 | 2025-05-08 | 特定条件で`if ($https) return 500;`が生成コンフィグに混入するバグ。needs-more-infoラベルあり、再現条件の特定が必要。 |
| [#1546](https://github.com/nginx-proxy/nginx-proxy/issues/1546) | SSL received a record that exceeded the maximum permissible length. | question-support | needs-discussion | 28 | 2020-12-19 | SSL最大レコード長超過エラー。28コメントと非常に高い関心を集めており、バックエンドへのHTTPS転送設定の問題が多い。 |
| [#1340](https://github.com/nginx-proxy/nginx-proxy/issues/1340) | How can proxy_pass work with nginx_proxy without 301s on HTTP addresses? | question-support | needs-discussion | 7 | 2019-10-05 | HTTPでアクセスしているのにnginx-proxyが301リダイレクトを返す問題への質問。HTTPS_METHOD=noredirect の利用案内が実装角度。 |
| [#2572](https://github.com/nginx-proxy/nginx-proxy/issues/2572) | unrecognized name:../ssl/record/rec_layer_s3.c:1552:SSL alert number 112 | question-support | needs-discussion | 1 | 2025-01-08 | SSL SNI未認識エラーが発生する問題。バックエンドとのSSLハンドシェイク設定やSNI設定の確認が必要なサポート系。 |
| [#2707](https://github.com/nginx-proxy/nginx-proxy/issues/2707) | Native ACME Support | feature | complex | 1 | 2026-02-19 | acme-companionに依存せずnginx-proxy本体でACMEプロトコルをネイティブサポートしてほしいという要望。設計変更が大きく議論が必要。 |
| [#776](https://github.com/nginx-proxy/nginx-proxy/issues/776) | HTTPS_METHOD HSTS vs 301 only | feature | likely-obsolete | 1 | 2017-03-25 | HTTPS_METHODでHSTSと301リダイレクトを独立して制御できるようにしてほしい。現在は両方同時に有効になる仕様。 |
| [#263](https://github.com/nginx-proxy/nginx-proxy/issues/263) | Optional forcing of https redirect, even without cert | feature | likely-obsolete | 1 | 2015-10-13 | 証明書がない場合でも強制HTTPSリダイレクトを有効にするオプションが欲しい要望。HTTPS_METHODで現在は対応済みの可能性が高い。 |
| [#621](https://github.com/nginx-proxy/nginx-proxy/issues/621) | Disable https redirect for some domains in the same container. | feature | likely-obsolete | 0 | 2016-11-03 | 同一コンテナで一部ドメインのHTTPSリダイレクトを無効化したい要望。HTTPS_METHODによる制御が後に実装されており、現在は解決済みの可能性が高い。 |
| [#1476](https://github.com/nginx-proxy/nginx-proxy/issues/1476) | 502 Bad gateway only with SSL, working connecting with http | bug | likely-obsolete | 6 | 2020-07-21 | HTTP接続では正常だがSSLのみ502が発生する。バックエンドコンテナへのVIRTUAL_PROTO設定やSSL終端の設定ミスが原因の可能性。 |
| [#1018](https://github.com/nginx-proxy/nginx-proxy/issues/1018) | Invalid redirection for non-existent https host | bug | likely-obsolete | 6 | 2018-01-05 | 存在しないHTTPSホストへのリクエスト時に誤ったリダイレクトが発生するバグ。 |
| [#937](https://github.com/nginx-proxy/nginx-proxy/issues/937) | Not creating https server and 301 for one subdomain container | bug | likely-obsolete | 5 | 2017-09-24 | 特定のサブドメインコンテナでHTTPSサーバーと301リダイレクトが生成されないバグ。 |
| [#952](https://github.com/nginx-proxy/nginx-proxy/issues/952) | iOS 11 devices can't connect to HTTPS using this image | bug | likely-obsolete | 5 | 2017-10-21 | iOS 11デバイスがHTTPS接続できないバグ（TLS設定/暗号スイートの問題）。現在は該当iOSバージョンが古くなっており解決済みと思われる。 |
| [#1465](https://github.com/nginx-proxy/nginx-proxy/issues/1465) | 502 Bad Gateway when connecting to backend with VIRTUAL_PROTO=https | bug | likely-obsolete | 4 | 2020-07-05 | VIRTUAL_PROTO=httpsで上流に接続する際に502が発生する。自己署名証明書の検証やSNIの設定に関係するnginx.tmpl側の問題の可能性。 |
| [#879](https://github.com/nginx-proxy/nginx-proxy/issues/879) | Wrong dhparam.pem path when using nginx official | bug | likely-obsolete | 3 | 2017-07-20 | 公式nginxイメージ使用時にdhparam.pemのパスが誤っているバグ。2017年の問題で修正済みの可能性が高い。 |
| [#813](https://github.com/nginx-proxy/nginx-proxy/issues/813) | SSL port 443 problems on latest update | bug | likely-obsolete | 1 | 2017-05-05 | アップデート後にSSL 443番ポートで問題が発生。古いバージョン固有の問題で現在は解消済みと思われる。 |
| [#1561](https://github.com/nginx-proxy/nginx-proxy/issues/1561) | nginx proxy v0.8.0 stopped 301 to https | bug | likely-obsolete | 1 | 2021-02-21 | v0.8.0でHTTPSへのリダイレクトが停止したバグ。古いバージョン固有の問題で現在は解決済みの可能性。 |
| [#805](https://github.com/nginx-proxy/nginx-proxy/issues/805) | Infinite Redirects | bug | likely-obsolete | 1 | 2017-04-28 | HTTPSリダイレクト設定時に無限リダイレクトが発生する問題。X-Forwarded-Protoヘッダーの扱いが原因の典型的な問題。 |
| [#737](https://github.com/nginx-proxy/nginx-proxy/issues/737) | Wildcard subdomains on `noredirect` sites only work for https. | bug | likely-obsolete | 0 | 2017-02-21 | noredirectサイトのワイルドカードサブドメインがHTTPSでのみ動作しHTTPで機能しないバグ。テンプレートのHTTP server block生成ロジックの問題。 |
| [#416](https://github.com/nginx-proxy/nginx-proxy/issues/416) | mount point /etc/nginx/certs seems lost sometimes | bug | likely-obsolete | 0 | 2016-04-12 | /etc/nginx/certsのマウントポイントが時折失われるという報告。Dockerのボリューム管理の問題と思われ、古いイシュー。 |
| [#1410](https://github.com/nginx-proxy/nginx-proxy/issues/1410) | Let's Encrypt 301 redirect prevention breaks working setup | bug | likely-obsolete | 0 | 2020-03-22 | Let's Encrypt用の301リダイレクト防止設定が既存の動作中のセットアップを破壊する問題。acme-companionとの設定競合と思われる。 |
| [#1038](https://github.com/nginx-proxy/nginx-proxy/issues/1038) | Incorrect certificate served to external connections, correct one to internal connections | bug | likely-obsolete | 0 | 2018-01-23 | 外部接続に誤った証明書が提供される一方、内部接続では正しい証明書が使われるバグ。 |
| [#958](https://github.com/nginx-proxy/nginx-proxy/issues/958) | How do I make nginx-proxy redirect to www. and enforce https/ssl? | question-support | likely-obsolete | 20 | 2017-10-28 | wwwリダイレクトとHTTPS強制を同時に行う方法についての質問。20コメントで高いエンゲージメント。 |
| [#702](https://github.com/nginx-proxy/nginx-proxy/issues/702) | Not listening to 443 (https) | question-support | likely-obsolete | 13 | 2017-01-31 | 443番ポートがリッスンされていない問題。証明書が存在しない・ポートマッピング設定ミスが主な原因で、ドキュメント充実で対応済みと思われる。 |
| [#897](https://github.com/nginx-proxy/nginx-proxy/issues/897) | HTTPS Problem using self signed certs | question-support | likely-obsolete | 7 | 2017-08-09 | 自己署名証明書使用時のHTTPS問題。設定ミスによるサポート質問で、現在のドキュメントで対応済みの可能性が高い。 |
| [#1002](https://github.com/nginx-proxy/nginx-proxy/issues/1002) | Https Connection refused but http works fine | question-support | likely-obsolete | 6 | 2017-12-15 | HTTPは正常に動くがHTTPS接続が拒否されるという問題の質問。 |
| [#844](https://github.com/nginx-proxy/nginx-proxy/issues/844) | Docker Cloud and SSL Certs | question-support | likely-obsolete | 6 | 2017-06-07 | Docker Cloud（現在廃止）環境でのSSL証明書設定についての質問。Docker Cloudが廃止されたため完全に obsolete。 |
| [#799](https://github.com/nginx-proxy/nginx-proxy/issues/799) | How to harden / customize SSL config? | question-support | likely-obsolete | 2 | 2017-04-22 | SSL設定をセキュリティ強化・カスタマイズする方法の質問。DEFAULT_HOST設定やカスタムnginx.tmplで対応する。 |
| [#1391](https://github.com/nginx-proxy/nginx-proxy/issues/1391) | multiple ssl subdomains per container | question-support | likely-obsolete | 1 | 2020-02-07 | 1つのコンテナに対して複数のSSLサブドメインを設定する方法の質問。VIRTUAL_HOSTにカンマ区切りで複数ドメインを指定する既存機能で対応可能と思われる。 |
| [#1520](https://github.com/nginx-proxy/nginx-proxy/issues/1520) | Unable to set up https-connections (Error 502) with self-signed certificates | question-support | likely-obsolete | 1 | 2020-10-30 | 自己署名証明書でHTTPS接続時に502エラーが発生する問題。古いバージョンの設定問題で現在は解決済みの可能性。 |
| [#1241](https://github.com/nginx-proxy/nginx-proxy/issues/1241) | renew letsencrypt certificate with nginx client-certificate turning on | question-support | likely-obsolete | 1 | 2019-03-04 | クライアント証明書認証有効時にLet's Encryptの証明書更新が失敗する問題。acme-companionとの連携設定が原因。 |
| [#985](https://github.com/nginx-proxy/nginx-proxy/issues/985) | How can I redirect all possibilities to https://example.com? | question-support | likely-obsolete | 1 | 2017-11-21 | すべてのリクエストをhttps://example.comにリダイレクトする方法についての質問。 |
| [#1010](https://github.com/nginx-proxy/nginx-proxy/issues/1010) | Can't automatic creat certificates | question-support | likely-obsolete | 1 | 2017-12-23 | 証明書の自動生成ができないという質問（acme-companionの範疇）。 |
| [#324](https://github.com/nginx-proxy/nginx-proxy/issues/324) | Why do I see the certificate for the domain 3 levels, if I connected the certificate only for domain 2 level? | question-support | likely-obsolete | 1 | 2015-12-25 | サブドメイン用の証明書がより深いサブドメインでも提示される挙動の質問。nginxの証明書選択ロジックに関するサポート質問。 |
| [#884](https://github.com/nginx-proxy/nginx-proxy/issues/884) | nginx-proxy + docker-letsencrypt-nginx-proxy-companion -> SSL errors | question-support | likely-obsolete | 1 | 2017-07-25 | Let's Encrypt連携時のSSLエラー。旧companion projectとの組み合わせ問題で現在はacme-companionに移行済み。 |
| [#1445](https://github.com/nginx-proxy/nginx-proxy/issues/1445) | Self-signed cert in child container | question-support | likely-obsolete | 0 | 2020-06-02 | 子コンテナが自己署名証明書を使用している場合のVIRTUAL_PROTO=https設定に関する質問。 |
| [#1435](https://github.com/nginx-proxy/nginx-proxy/issues/1435) | weird behavior and no enforced https | question-support | likely-obsolete | 0 | 2020-05-14 | HTTPSの強制リダイレクトが機能しない問題。HTTPS_METHOD環境変数の設定ミスが疑われる。 |
| [#1074](https://github.com/nginx-proxy/nginx-proxy/issues/1074) | Unable to use port 443 | question-support | likely-obsolete | 0 | 2018-02-20 | ポート443が使えないというサポート質問。設定ミスや権限問題が原因と思われる古い質問。 |

</details>

<details>
<summary><b>Custom config / templating (nginx.tmpl)</b> — 31件</summary>

| Issue | タイトル | 種別 | 容易度 | コメント | 作成 | メモ |
|---|---|---|---|---|---|---|
| [#1117](https://github.com/nginx-proxy/nginx-proxy/issues/1117) | Add the ability to set the index | feature | good-first | 4 | 2018-04-03 | nginxのindexディレクティブを環境変数で設定できるようにしてほしい機能要求。テンプレートへの小規模追加で実装可能。 |
| [#663](https://github.com/nginx-proxy/nginx-proxy/issues/663) | Any way to route to something besides a docker container? | feature | feasible | 4 | 2016-12-20 | Dockerコンテナ以外（ホストサービス、外部サーバー）へのルーティングをサポートしてほしい。ラベル「scope/configurable-upstream」付きで、カスタムupstream定義の仕組みが必要。 |
| [#1309](https://github.com/nginx-proxy/nginx-proxy/issues/1309) | Check closest host match when looking for vhost.d/config files before checking default | feature | feasible | 0 | 2019-07-31 | vhost.d設定ファイルを参照する際、defaultにフォールバックする前に最も近いホスト名マッチを試みる仕様の改善要求。 |
| [#1377](https://github.com/nginx-proxy/nginx-proxy/issues/1377) | "load_module" directive is not allowed here in /etc/nginx/conf.d/nginx.conf:1 | bug | feasible | 2 | 2020-01-07 | カスタム設定ファイルでload_moduleディレクティブを使おうとするとconf.d配置が原因でエラーになる問題。load_moduleはmainコンテキストにのみ許可されているため、nginx.tmplかメイン設定の構造変更が必要。 |
| [#1091](https://github.com/nginx-proxy/nginx-proxy/issues/1091) | The template can (and sometimes does) generate an empty upstream section | bug | feasible | 2 | 2018-03-09 | nginx.tmplが空のupstreamセクションを生成することがあるバグ。コンテナが存在しない状態での条件分岐修正が必要。 |
| [#830](https://github.com/nginx-proxy/nginx-proxy/issues/830) | htpasswd when using wildcard virtual host name | bug | feasible | 0 | 2017-05-23 | ワイルドカードVIRTUAL_HOST使用時にhtpasswd認証が動作しない問題。テンプレートのファイル名マッチングロジックの修正で対応可能。 |
| [#688](https://github.com/nginx-proxy/nginx-proxy/issues/688) | Nginx configuration by environment variables | feature | needs-discussion | 4 | 2017-01-23 | 環境変数でnginx設定を動的に変更できるようにしてほしい。現在は環境変数の一部をサポートしているが、より包括的な設定変数のサポートを求めている。 |
| [#1324](https://github.com/nginx-proxy/nginx-proxy/issues/1324) | Config using environment variables | feature | needs-discussion | 3 | 2019-09-04 | 環境変数を使ってnginxの設定値（タイムアウト等）を動的に注入できるようにする機能要求。docker-gen templateへの変数追加が実装角度。 |
| [#829](https://github.com/nginx-proxy/nginx-proxy/issues/829) | Using unix socket instead of tcp | feature | needs-discussion | 1 | 2017-05-22 | TCPの代わりにUnixソケット経由でバックエンドコンテナに接続する機能要求。テンプレートとdocker-genの設定変更が必要な設計上の変更。 |
| [#890](https://github.com/nginx-proxy/nginx-proxy/issues/890) | Blacklist User Agent | feature | needs-discussion | 0 | 2017-07-30 | 特定のUser-Agentをブラックリスト化する機能の要求。カスタム設定ファイルで対応可能だがテンプレートへの組み込みは設計判断が必要。 |
| [#1052](https://github.com/nginx-proxy/nginx-proxy/issues/1052) | Multiple sites scenario with only one php-fpm container and a crazy idea... | question-support | needs-discussion | 13 | 2018-02-02 | 1つのphp-fpmコンテナで複数サイトを運用する方法についての議論。13コメントと活発だが根本的なアーキテクチャ上の限界がある。 |
| [#2424](https://github.com/nginx-proxy/nginx-proxy/issues/2424) | Can't get .conf.template files to work with env variables/file. | question-support | needs-discussion | 2 | 2024-04-23 | .conf.templateファイルで環境変数の展開が機能しないという問題。docker-genのテンプレート処理との干渉の可能性。 |
| [#792](https://github.com/nginx-proxy/nginx-proxy/issues/792) | Support for stream module | feature | complex | 6 | 2017-04-15 | nginxのstreamモジュールを使ったTCP/UDPプロキシのサポートを要求。HTTP以外のプロトコルへの対応でアーキテクチャ変更が必要。 |
| [#1415](https://github.com/nginx-proxy/nginx-proxy/issues/1415) | fetaure request - Custom nginx templates and select them with NGINX_TEMPLATE var on client side | feature | complex | 2 | 2020-04-01 | NGINX_TEMPLATE環境変数でコンテナごとに異なるnginxテンプレートを選択できるようにする機能要求。docker-gen側の対応が必要で実装範囲が広い。 |
| [#2606](https://github.com/nginx-proxy/nginx-proxy/issues/2606) | split nginx.tmpl to support stream module | feature | complex | 1 | 2025-04-08 | nginxのstreamモジュール（TCP/UDPプロキシ）をサポートするためにnginx.tmplを分割・拡張してほしいという要望。設計変更が大きい。 |
| [#1168](https://github.com/nginx-proxy/nginx-proxy/issues/1168) | Potential race condition when generating upstream vs server | bug | complex | 1 | 2018-09-05 | upstream設定とserver設定の生成順序に競合状態が生じる可能性があるバグ。テンプレート生成ロジックの修正が必要。 |
| [#597](https://github.com/nginx-proxy/nginx-proxy/issues/597) | Wish/Enhancement: Per-VIRTUAL_HOST configuration via environment | feature | likely-obsolete | 0 | 2016-10-06 | 環境変数でVIRTUAL_HOSTごとの設定を上書きしたい要望。カスタム設定ファイルのマウントで対応可能なため、現在は不要な可能性が高い。 |
| [#378](https://github.com/nginx-proxy/nginx-proxy/issues/378) | nginx error when using proxy-wide configuration | bug | likely-obsolete | 4 | 2016-03-02 | プロキシ全体のカスタム設定ファイルを使用した際にnginxエラーが発生するバグ。設定の配置場所や構文の問題と思われる。 |
| [#1365](https://github.com/nginx-proxy/nginx-proxy/issues/1365) | "map_hash_max_size" directive is duplicate | bug | likely-obsolete | 3 | 2019-12-04 | map_hash_max_sizeディレクティブが重複して生成されnginxの設定検証が失敗する問題。nginx.tmplの重複排除ロジックの修正で対応可能。既に修正済みの可能性あり。 |
| [#1013](https://github.com/nginx-proxy/nginx-proxy/issues/1013) | Empty upstream definition leads to nginx-proxy failing to start/restart | bug | likely-obsolete | 1 | 2017-12-28 | 上流定義が空の場合にnginx-proxyの起動/再起動が失敗するバグ。テンプレート側での空チェックが必要。 |
| [#1300](https://github.com/nginx-proxy/nginx-proxy/issues/1300) | nginx.tmpl config is wrong: proxy_pass {{ trim $proto }} results in 502 Bad Gateway between containers | bug | likely-obsolete | 1 | 2019-07-04 | nginx.tmplのproxy_passでprotoのtrimが誤動作し502エラーになるバグ報告。テンプレート修正で対応可能だが古い。 |
| [#513](https://github.com/nginx-proxy/nginx-proxy/issues/513) | Template error: readdirent: no such file or directory on a start of extra container | bug | likely-obsolete | 0 | 2016-07-21 | 追加コンテナ起動時にdocker-genのテンプレートエラーが発生するバグ。ディレクトリ存在チェックの欠如が原因と思われ、古いdocker-genバージョンのバグの可能性が高い。 |
| [#664](https://github.com/nginx-proxy/nginx-proxy/issues/664) | Proxying to a second nginx container | question-support | likely-obsolete | 13 | 2016-12-26 | nginx-proxyから別のnginxコンテナへのプロキシ設定方法の質問。ネットワーク設定とVIRTUAL_HOSTの組み合わせで対応可能。 |
| [#651](https://github.com/nginx-proxy/nginx-proxy/issues/651) | Best solution for TCP Ports? haproxy? | question-support | likely-obsolete | 9 | 2016-12-06 | TCPポートのプロキシに最善のアプローチを尋ねている。nginxのstreamモジュール使用やhaproxyへの切り替えを議論している。 |
| [#864](https://github.com/nginx-proxy/nginx-proxy/issues/864) | Reverse Proxy for Plex | question-support | likely-obsolete | 2 | 2017-06-24 | Plexメディアサーバーのリバースプロキシ設定についての質問。アプリ固有の設定はカスタム設定ファイルで対応可能。 |
| [#821](https://github.com/nginx-proxy/nginx-proxy/issues/821) | nginx-proxy with netdata (not dockered) | question-support | likely-obsolete | 2 | 2017-05-15 | Dockerコンテナ外のnetdataをnginx-proxy経由でプロキシする方法についての質問。非Dockerバックエンドへの手動設定が必要。 |
| [#794](https://github.com/nginx-proxy/nginx-proxy/issues/794) | Specific settings for each domain | question-support | likely-obsolete | 1 | 2017-04-19 | ドメインごとに個別のnginx設定を適用したい。/etc/nginx/vhost.d/<domain>ファイルで対応済み。 |
| [#941](https://github.com/nginx-proxy/nginx-proxy/issues/941) | Using this with prerender.io | question-support | likely-obsolete | 1 | 2017-10-03 | prerender.ioとnginx-proxyを組み合わせる方法についての質問。 |
| [#322](https://github.com/nginx-proxy/nginx-proxy/issues/322) | include a proxy redirect into generated /etc/nginx/conf.d/default.conf ? | question-support | likely-obsolete | 1 | 2015-12-21 | 生成されるdefault.confにproxy_redirectを追加してほしいという質問兼要望。カスタム設定ファイルで対応可能。 |
| [#811](https://github.com/nginx-proxy/nginx-proxy/issues/811) | How to provide per-vhost conf files automagically in addition to env vars? | question-support | likely-obsolete | 0 | 2017-05-03 | 環境変数に加えてvhost別の設定ファイルを自動的に適用する方法の質問。現在は/etc/nginx/vhost.d/の仕組みで対応済み。 |
| [#773](https://github.com/nginx-proxy/nginx-proxy/issues/773) | using map module per vhost custom config | question-support | likely-obsolete | 0 | 2017-03-22 | vhost別にnginxのmapモジュールを使うカスタム設定の方法を尋ねている。nginx.tmplの修正か設定ファイルの追加で対応可能。 |

</details>

<details>
<summary><b>Docker Compose / Swarm / labels / docker-gen</b> — 29件</summary>

| Issue | タイトル | 種別 | 容易度 | コメント | 作成 | メモ |
|---|---|---|---|---|---|---|
| [#97](https://github.com/nginx-proxy/nginx-proxy/issues/97) | Docker Swarm Support or Multiple Node Support | feature | needs-discussion | 55 | 2015-02-06 | Docker Swarmや複数ノード環境でのサポート要望。コメント数55と最多エンゲージメントだが、Swarm対応は部分的に進んでいる。現状の制限をドキュメント化することが現実的。 |
| [#1158](https://github.com/nginx-proxy/nginx-proxy/issues/1158) | dockerns-remap support | feature | needs-discussion | 0 | 2018-08-05 | Docker user namespace remapping (userns-remap) への対応要求。セキュリティ機能だが設計検討が必要。 |
| [#2655](https://github.com/nginx-proxy/nginx-proxy/issues/2655) | nginx proxy exiting in docker compose when we get hello from palo alto networks | bug | needs-discussion | 0 | 2025-09-05 | Palo Altoネットワーク機器からの予期せぬ接続でnginx-proxyがDocker Compose内で終了するという問題。外部要因の可能性が高く原因特定が難しい。 |
| [#2148](https://github.com/nginx-proxy/nginx-proxy/issues/2148) | Migrate from environment variables to container labels | feature | complex | 3 | 2023-01-23 | VIRTUAL_HOST等の設定をenv varからDocker labelsへ移行してほしいという大きな機能要求。docker-gen側の変更も必要で設計議論が必要。 |
| [#431](https://github.com/nginx-proxy/nginx-proxy/issues/431) | Question/proposal: Watching network change events only | feature | likely-obsolete | 1 | 2016-04-26 | docker-genのイベント監視をnetworkイベントのみに絞ることでパフォーマンスを改善したいという提案。docker-gen側の機能に依存するため検討が必要。 |
| [#622](https://github.com/nginx-proxy/nginx-proxy/issues/622) | upstreams are empty when using docker compose with separate containers for nginx and dockergen | bug | likely-obsolete | 9 | 2016-11-03 | docker-genをnginxと別コンテナで動かすdocker compose構成でupstreamが空になるバグ。ネットワーク設定（shared network）が必要なケースで、ドキュメント整備または設定例の追加で対処可能。 |
| [#88](https://github.com/nginx-proxy/nginx-proxy/issues/88) | Doesn't work with docker machine | bug | likely-obsolete | 5 | 2015-01-16 | Docker Machine環境で動作しないバグ。Docker Machineは廃止済みのため完全に時代遅れのイシュー。 |
| [#650](https://github.com/nginx-proxy/nginx-proxy/issues/650) | Two servers : same docker-compose, different behaviour | bug | likely-obsolete | 4 | 2016-12-06 | 同じdocker-compose設定で2台のサーバーで異なる動作をする問題。環境依存の設定差異やDockerバージョン差異が原因と思われる。 |
| [#997](https://github.com/nginx-proxy/nginx-proxy/issues/997) | Nginx-proxy in docker-compose and `$ docker run -e VIRTUAL_HOST=foo.bar.com  ...` gives empty upstream | bug | likely-obsolete | 3 | 2017-12-11 | docker-compose内のnginx-proxyと`docker run`で起動したコンテナが接続できずempty upstreamになるバグ。 |
| [#219](https://github.com/nginx-proxy/nginx-proxy/issues/219) | configuration not updating | bug | likely-obsolete | 2 | 2015-08-19 | コンテナ追加後にnginx設定が更新されないバグ。docker-genのイベント監視の問題と思われるが非常に古いイシュー。 |
| [#1534](https://github.com/nginx-proxy/nginx-proxy/issues/1534) | [BUG] Nginx proxy after reboot pointing to wrong container | bug | likely-obsolete | 2 | 2020-11-18 | 再起動後にnginx-proxyが別コンテナを指してしまうバグ。docker-genのコンテナ再検出タイミングの問題。古く解決済みの可能性。 |
| [#814](https://github.com/nginx-proxy/nginx-proxy/issues/814) | Unstable behavior when connecting to a Docker network with preexisting containers. | bug | likely-obsolete | 1 | 2017-05-08 | 既存コンテナがいるDockerネットワークへの接続時に不安定な動作が発生する。docker-genのイベント処理タイミングの問題。 |
| [#727](https://github.com/nginx-proxy/nginx-proxy/issues/727) | docker-compose separate container fail at launch if no template file is present | bug | likely-obsolete | 0 | 2017-02-18 | docker-composeで別コンテナとして起動する際にテンプレートファイルが無いと起動失敗する問題。エラーハンドリングの改善が必要。 |
| [#648](https://github.com/nginx-proxy/nginx-proxy/issues/648) | nginx-proxy with Rancher 1.2 | question-support | likely-obsolete | 14 | 2016-12-02 | Rancher 1.2環境でのnginx-proxy動作についての質問。Rancherは現在廃止されており完全に時代遅れのissue。 |
| [#1039](https://github.com/nginx-proxy/nginx-proxy/issues/1039) | Confused with setup nginx-proxy - phpFpm - multiple sites... (not issue) | question-support | likely-obsolete | 4 | 2018-01-24 | phpFpmと複数サイトのセットアップに関する質問（issue自体が「issue非対象」と明記）。 |
| [#854](https://github.com/nginx-proxy/nginx-proxy/issues/854) | Running nginx-proxy outside of docker-compose; inter-container communication from compose | question-support | likely-obsolete | 2 | 2017-06-16 | docker-compose外でnginx-proxyを実行した際のコンテナ間通信の設定についての質問。ネットワーク設定の理解に関する内容。 |
| [#1471](https://github.com/nginx-proxy/nginx-proxy/issues/1471) | Windows invalid volume specification | question-support | likely-obsolete | 2 | 2020-07-13 | Windowsでのボリュームマウントパス指定が無効になる問題。Windows Docker特有の構文問題であり、ドキュメント改善で対応すべき。 |
| [#919](https://github.com/nginx-proxy/nginx-proxy/issues/919) | Nginx-proxy on docker-compose v1 (old docker without docker network) | question-support | likely-obsolete | 2 | 2017-09-04 | Docker Compose v1の古い環境でのネットワーク設定に関する質問。Docker Compose v1はすでに廃止されており完全に obsolete。 |
| [#822](https://github.com/nginx-proxy/nginx-proxy/issues/822) | How to get Container to Container communication in Mac | question-support | likely-obsolete | 1 | 2017-05-16 | Mac環境でのコンテナ間通信の設定についての質問。Docker for Macの古い問題でOS固有のネットワーク設定の質問。 |
| [#1077](https://github.com/nginx-proxy/nginx-proxy/issues/1077) | nginx-proxy behavior at host startup | question-support | likely-obsolete | 1 | 2018-02-22 | ホスト起動時のnginx-proxyの動作に関する質問。depends_onやrestart設定で解決できる一般的なサポート質問。 |
| [#908](https://github.com/nginx-proxy/nginx-proxy/issues/908) | nginx-proxy with regular app | question-support | likely-obsolete | 1 | 2017-08-22 | 通常アプリとの連携に関する使い方の質問。詳細が少なくドキュメントで対応可能な内容。 |
| [#1015](https://github.com/nginx-proxy/nginx-proxy/issues/1015) | No support for newer docker compose ? | question-support | likely-obsolete | 1 | 2017-12-29 | 新しいバージョンのDocker Composeでの動作サポートについての質問（現在はおそらく解決済み）。 |
| [#1044](https://github.com/nginx-proxy/nginx-proxy/issues/1044) | How to ensure nginx-proxy container up's first after rebooting the server | question-support | likely-obsolete | 1 | 2018-01-25 | サーバー再起動後にnginx-proxyコンテナを最初に起動させる方法の質問。depends_onやrestart policyで解決できる一般的な問題。 |
| [#1063](https://github.com/nginx-proxy/nginx-proxy/issues/1063) | Problem runing example docker-compose | question-support | likely-obsolete | 1 | 2018-02-08 | サンプルのdocker-composeが動かないというサポート質問。設定や構文の問題と思われる古い報告。 |
| [#853](https://github.com/nginx-proxy/nginx-proxy/issues/853) | Proxying not working in "Separate Containers" configuration | question-support | likely-obsolete | 1 | 2017-06-15 | 分離コンテナ構成でプロキシが動作しない問題。Dockerネットワーク設定の問題のサポート質問。 |
| [#1395](https://github.com/nginx-proxy/nginx-proxy/issues/1395) | Issues with services (in swarm) | question-support | likely-obsolete | 0 | 2020-02-11 | Docker Swarmモードでサービスを使用する際の問題。docker-genのSwarmサービス対応に関する設定の質問。 |
| [#1449](https://github.com/nginx-proxy/nginx-proxy/issues/1449) | Why need add "not-watch" volume to `nginx-gen` container? | question-support | likely-obsolete | 0 | 2020-06-09 | nginx-genコンテナに「not-watch」ボリュームが必要な理由についての質問。ドキュメントの説明不足が原因。 |
| [#1045](https://github.com/nginx-proxy/nginx-proxy/issues/1045) | Errors Right After Reboot | question-support | likely-obsolete | 0 | 2018-01-27 | 再起動直後にエラーが発生するというサポート質問。コンテナ起動順序の問題と思われる古い報告。 |
| [#991](https://github.com/nginx-proxy/nginx-proxy/issues/991) | Running New Container Get Error | question-support | likely-obsolete | 0 | 2017-12-02 | 新しいコンテナ起動時にエラーが発生するという漠然とした問題報告。 |

</details>

<details>
<summary><b>Proxy tuning (body size, timeouts, buffering, websockets, HTTP2/3)</b> — 25件</summary>

| Issue | タイトル | 種別 | 容易度 | コメント | 作成 | メモ |
|---|---|---|---|---|---|---|
| [#1497](https://github.com/nginx-proxy/nginx-proxy/issues/1497) | Feature request: big time ranges for rate-limiter | feature | feasible | 3 | 2020-09-03 | レートリミッターに時間/日単位等の長い時間範囲を設定できるようにしてほしいという要望。nginx.tmplの設定オプション追加で対応可能。 |
| [#1359](https://github.com/nginx-proxy/nginx-proxy/issues/1359) | client_max_body_size to specific location | feature | feasible | 0 | 2019-11-11 | 特定のlocationブロックに対してのみclient_max_body_sizeを設定したい要求。パスごとのカスタム設定ファイルや環境変数で対応できるかどうかの検討が必要。 |
| [#1208](https://github.com/nginx-proxy/nginx-proxy/issues/1208) | pwritev() "/var/cache/nginx/client_temp/0000000014" failed (28: No space left on device) | bug | feasible | 6 | 2018-12-07 | nginxのクライアント一時ディレクトリがディスクフルになるエラー。client_body_temp_pathのtmpfsマウントやキャッシュ設定の改善で対応可能。 |
| [#2577](https://github.com/nginx-proxy/nginx-proxy/issues/2577) | Exposing gRPC service through SSL results in protocol error | bug | feasible | 5 | 2025-01-24 | SSL経由でgRPCサービスを公開するとプロトコルエラーが発生する問題。nginx.tmplのgRPC/HTTP2設定の修正で対応できる可能性。 |
| [#1190](https://github.com/nginx-proxy/nginx-proxy/issues/1190) | 503 on Chrome DevTools connection request | bug | feasible | 3 | 2018-11-01 | Chrome DevToolsのプロトコル接続リクエストが503になる問題。WebSocket/HTTP CONNECT系の特殊なリクエストへの対応が必要。 |
| [#1331](https://github.com/nginx-proxy/nginx-proxy/issues/1331) | Websocket connection quick abort after 101 response sent | bug | feasible | 1 | 2019-09-18 | 101 Upgradeレスポンス送信直後にWebSocket接続が切断されるバグ。proxy_read_timeoutやWebSocketのヘッダー設定が原因の可能性。 |
| [#981](https://github.com/nginx-proxy/nginx-proxy/issues/981) | jwilder/nginx-proxy upload limits? | question-support | feasible | 44 | 2017-11-16 | アップロードサイズ制限（client_max_body_size）の設定方法についての質問。44コメントと非常に高いエンゲージメント。ドキュメントに明記すべき。 |
| [#1247](https://github.com/nginx-proxy/nginx-proxy/issues/1247) | Optimise for Emby | feature | needs-discussion | 0 | 2019-03-16 | Embyメディアサーバー向けのプロキシ設定最適化要求。特定アプリ向けのプリセット設定は汎用化が難しく議論が必要。 |
| [#1642](https://github.com/nginx-proxy/nginx-proxy/issues/1642) | ERR_HTTP2_SERVER_REFUSED_STREAM with PHP web application on 0.9.0 | bug | needs-discussion | 26 | 2021-05-27 | PHPアプリでHTTP/2のストリームが拒否されるエラー。needs-more-infoラベルで26コメントと大きな関心を集めているHTTP/2関連の問題。 |
| [#1406](https://github.com/nginx-proxy/nginx-proxy/issues/1406) | When nginx does TCP proxy forwarding, memory will not be released after disconnection | bug | needs-discussion | 0 | 2020-03-17 | TCPプロキシ転送後に切断してもメモリが解放されないメモリリークの報告。nginx-proxyの通常用途外（stream module）であり、再現手順の確認が必要。 |
| [#2600](https://github.com/nginx-proxy/nginx-proxy/issues/2600) | reverse proxy is override status code on development enviroment!!! | question-support | needs-discussion | 2 | 2025-03-12 | 開発環境でリバースプロキシがバックエンドのHTTPステータスコードを上書きするという問題。設定ミスの可能性が高くサポート系。 |
| [#1075](https://github.com/nginx-proxy/nginx-proxy/issues/1075) | http2 push support added | feature | likely-obsolete | 7 | 2018-02-20 | HTTP/2プッシュのサポート要求。HTTP/2 pushはnginxで非推奨になったため現在は実装価値が低い。 |
| [#139](https://github.com/nginx-proxy/nginx-proxy/issues/139) | gzip_types has no effect without setting gzip and gzip_proxied | bug | likely-obsolete | 23 | 2015-04-04 | gzip_typesの設定がgzip/gzip_proxied未設定では無効になるバグ。nginx.tmplのデフォルト設定にgzip onとgzip_proxied anyを追加するだけで対応可能だが、コメント数23と古いイシューで現在は修正済みの可能性が高い。 |
| [#516](https://github.com/nginx-proxy/nginx-proxy/issues/516) | Occasional Error 502 Bad Gateway | bug | likely-obsolete | 21 | 2016-07-27 | コンテナ再起動時などに散発する502エラー。docker-genの設定再生成タイミングとnginxリロードの競合が原因と思われるが、コメント数が多く古いイシュー。 |
| [#420](https://github.com/nginx-proxy/nginx-proxy/issues/420) | Nginx redirection is causing 504 Time-Outs for half of all requests. | bug | likely-obsolete | 10 | 2016-04-15 | リクエストの半数が504タイムアウトになる問題。IPv4/IPv6の解決順やupstreamの設定問題と思われるが、コメント数が多く2026年まで更新されている点は注目。 |
| [#1462](https://github.com/nginx-proxy/nginx-proxy/issues/1462) | Static files coming sometimes broken | bug | likely-obsolete | 6 | 2020-06-28 | 静的ファイルが断続的に壊れて配信される問題。プロキシのバッファリング設定や接続維持に関連する可能性があり、再現条件が不明確。 |
| [#912](https://github.com/nginx-proxy/nginx-proxy/issues/912) | client_max_body_size seems has no effect | bug | likely-obsolete | 3 | 2017-08-24 | client_max_body_sizeの設定が効かない問題。カスタム設定ファイルの配置場所または優先順位の問題で、現在は修正または文書化済みの可能性が高い。 |
| [#451](https://github.com/nginx-proxy/nginx-proxy/issues/451) | COPY, MOVE operations result in 502 Bad Gateway when proxying nginx/apache | bug | likely-obsolete | 1 | 2016-05-10 | WebDAVのCOPY/MOVEメソッドプロキシ時に502が発生するバグ。nginx設定でのメソッド許可またはproxy_pass設定の問題と思われる。 |
| [#1032](https://github.com/nginx-proxy/nginx-proxy/issues/1032) | Massive Performance Problem with 'Upgrade Headers' | bug | likely-obsolete | 0 | 2018-01-18 | Upgradeヘッダー処理による大幅なパフォーマンス低下の報告。 |
| [#846](https://github.com/nginx-proxy/nginx-proxy/issues/846) | Video files "hang" - no content received | bug | likely-obsolete | 0 | 2017-06-09 | 動画ファイルのプロキシ時にコンテンツが受信されずハングする問題。バッファリング設定（proxy_buffering off等）で対応可能な可能性。 |
| [#647](https://github.com/nginx-proxy/nginx-proxy/issues/647) | Troubleshooting latency issues | question-support | likely-obsolete | 10 | 2016-12-02 | nginx-proxy経由でのレイテンシ問題のトラブルシューティングを求めている。DNS解決やバッファリング設定のチューニングが対応策。 |
| [#490](https://github.com/nginx-proxy/nginx-proxy/issues/490) | Help: 502 Bad Gateway | question-support | likely-obsolete | 6 | 2016-06-25 | 502エラーのサポート質問。ネットワーク設定や起動順序の問題と思われるが、サポート系のため実装不要。 |
| [#1229](https://github.com/nginx-proxy/nginx-proxy/issues/1229) | Elements cannot be load behind reverse proxy | question-support | likely-obsolete | 6 | 2019-02-04 | リバースプロキシ背後で特定のリソース（JS/CSS等）が読み込めない問題。サブパスのルーティングやcontent-typeの扱いが原因。 |
| [#800](https://github.com/nginx-proxy/nginx-proxy/issues/800) | how to enable caching? | question-support | likely-obsolete | 4 | 2017-04-24 | nginxプロキシキャッシュを有効にする方法の質問。カスタムtemplate修正かvhost.dの設定追加で対応可能。 |
| [#1250](https://github.com/nginx-proxy/nginx-proxy/issues/1250) | 502 Bad Gateway with Alfresco | question-support | likely-obsolete | 1 | 2019-03-21 | Alfresco特有の大きなリクエストや特殊ヘッダーによる502エラー。タイムアウト・バッファ設定のカスタマイズで対応可能。 |

</details>

<details>
<summary><b>Headers / CORS / forwarded-for</b> — 20件</summary>

| Issue | タイトル | 種別 | 容易度 | コメント | 作成 | メモ |
|---|---|---|---|---|---|---|
| [#1125](https://github.com/nginx-proxy/nginx-proxy/issues/1125) | Custom param for noindex header | feature | good-first | 0 | 2018-04-24 | X-Robots-Tag: noindexヘッダーを環境変数で制御できるようにしてほしい機能要求。テンプレートへの小規模追加で実装可能。 |
| [#1974](https://github.com/nginx-proxy/nginx-proxy/issues/1974) | Prevent X-Forwareded-For header passing from untrusted upstreams | feature | feasible | 4 | 2022-04-26 | 信頼できないアップストリームからのX-Forwarded-Forヘッダーを除去・制御する機能の要望。セキュリティ上重要でnginx.tmpl修正で対応可能。 |
| [#1150](https://github.com/nginx-proxy/nginx-proxy/issues/1150) | Make CORS with Basic Auth possible by including _location files after basic_auth | feature | feasible | 1 | 2018-07-17 | Basic Auth設定後に_locationファイルをインクルードしてCORSヘッダーを有効にできるようにしてほしい機能要求。テンプレートの順序変更で対応可能。 |
| [#1530](https://github.com/nginx-proxy/nginx-proxy/issues/1530) | can't set header, need more_set_headers or something | feature | feasible | 0 | 2020-11-10 | カスタムレスポンスヘッダーを設定したいがmore_set_headersモジュール等がないという要望。vhost.d経由の回避策を提案するかモジュール追加を検討。 |
| [#891](https://github.com/nginx-proxy/nginx-proxy/issues/891) | Add an UUID to every incoming request | feature | feasible | 0 | 2017-07-31 | 受信リクエストにUUIDを付与するヘッダー追加機能の要求。nginx.tmplにX-Request-IDヘッダーを追加することで実装可能。 |
| [#918](https://github.com/nginx-proxy/nginx-proxy/issues/918) | CloudFlare Flexible SSL breaks redirections because of X-Forwarded-Port | bug | feasible | 4 | 2017-09-01 | CloudFlare Flexible SSL使用時にX-Forwarded-Portヘッダーが誤った値を送り、リダイレクトが壊れる問題。テンプレートでヘッダー処理を修正することで対応可能。 |
| [#694](https://github.com/nginx-proxy/nginx-proxy/issues/694) | AWS upstream sent too big header while reading response header from upstream | bug | feasible | 4 | 2017-01-26 | AWSバックエンドからの大きなレスポンスヘッダーでエラーになる問題。proxy_buffer_sizeとproxy_buffers設定のデフォルト値引き上げまたは設定可能化が解決策。 |
| [#1333](https://github.com/nginx-proxy/nginx-proxy/issues/1333) | Modifying headers appends values | bug | feasible | 1 | 2019-09-19 | カスタムヘッダーを上書き設定しても既存値に追記されてしまうバグ。nginx.tmplのheader_upstream処理を修正する必要がある。 |
| [#1005](https://github.com/nginx-proxy/nginx-proxy/issues/1005) | Get missing HttpOnly or Secure cookie flags | feature | needs-discussion | 0 | 2017-12-19 | プロキシ経由のレスポンスでHttpOnlyやSecureクッキーフラグが欠落する問題の修正要求。 |
| [#1087](https://github.com/nginx-proxy/nginx-proxy/issues/1087) | Headers More | feature | needs-discussion | 0 | 2018-03-07 | nginx-headersMoreモジュールを組み込んでほしい機能要求。カスタムビルドが必要でイメージサイズや管理コストが増える。 |
| [#2409](https://github.com/nginx-proxy/nginx-proxy/issues/2409) | 405 (cors error) on specific location | question-support | needs-discussion | 0 | 2024-03-07 | 特定ロケーションでCORSエラー405が発生する問題。カスタム設定でのCORSヘッダー処理の不備かサポート系。 |
| [#1412](https://github.com/nginx-proxy/nginx-proxy/issues/1412) | URL is "- " when it arrives at web server | bug | likely-obsolete | 1 | 2020-03-25 | バックエンドに届くURLが「- 」になってしまう問題。X-Original-URIヘッダーまたはproxy_passのパス書き換えに関連する可能性。 |
| [#1299](https://github.com/nginx-proxy/nginx-proxy/issues/1299) | Incompatible with cookies | bug | likely-obsolete | 0 | 2019-07-02 | Cookieの扱いで非互換があるとの報告。詳細不明で再現条件も不明確なため対応困難。 |
| [#867](https://github.com/nginx-proxy/nginx-proxy/issues/867) | Proxy don't return Basic auth required from Artifactory | bug | likely-obsolete | 0 | 2017-06-28 | Artifactoryからの401 Basic auth必要レスポンスがプロキシ経由で正しく返されない問題。ヘッダー処理の問題だが詳細不明でコメント0件。 |
| [#659](https://github.com/nginx-proxy/nginx-proxy/issues/659) | nginx dropping header "Location" for redirects of webserver behide the proxy | bug | likely-obsolete | 0 | 2016-12-13 | プロキシ背後のwebサーバーがリダイレクトする際にLocationヘッダーが失われる問題。proxy_redirect設定が不適切な可能性。 |
| [#885](https://github.com/nginx-proxy/nginx-proxy/issues/885) | Proxying without host header?! | question-support | likely-obsolete | 2 | 2017-07-26 | Hostヘッダーなしのプロキシ動作についての質問。古い問題でドキュメントまたはカスタム設定で対応可能。 |
| [#1153](https://github.com/nginx-proxy/nginx-proxy/issues/1153) | Can GET from Google Chrome Browser but not from Firefox Browser or curl or wget | question-support | likely-obsolete | 2 | 2018-07-22 | ChromeでのみGETが成功しFirefoxやcurlでは失敗するという問題。ヘッダーやSSL設定の違いが原因と思われる古いサポート質問。 |
| [#1360](https://github.com/nginx-proxy/nginx-proxy/issues/1360) | CORS - Non-Authoritative-Reason: HSTS - Error: 307 | question-support | likely-obsolete | 1 | 2019-11-15 | HSTSとCORSの組み合わせで307リダイレクトが発生する問題。HSTS設定とHTTP→HTTPS強制リダイレクトの設定ミスと思われる。 |
| [#1209](https://github.com/nginx-proxy/nginx-proxy/issues/1209) | How to add content type | question-support | likely-obsolete | 0 | 2018-12-11 | Content-Typeヘッダーの追加方法についての基本的な質問。vhost.dカスタム設定で対応可能。 |
| [#1070](https://github.com/nginx-proxy/nginx-proxy/issues/1070) | Issue with container/redirects needing cookies | question-support | likely-obsolete | 0 | 2018-02-16 | クッキーを必要とするリダイレクトに関する問題。proxy_cookie_domainやSameSite設定の問題と思われる古いサポート質問。 |

</details>

<details>
<summary><b>Multi-port / multi-network / IPv6</b> — 19件</summary>

| Issue | タイトル | 種別 | 容易度 | コメント | 作成 | メモ |
|---|---|---|---|---|---|---|
| [#1148](https://github.com/nginx-proxy/nginx-proxy/issues/1148) | Proxying container exposing multiple ports (but only one HTTP) | feature | feasible | 7 | 2018-07-17 | 複数ポートを公開するコンテナで1つだけHTTPポートを自動選択してプロキシしてほしい機能要求。コメント数が多く需要がある。 |
| [#1334](https://github.com/nginx-proxy/nginx-proxy/issues/1334) | Ability to set the NETWORK_ACCESS default to prevent accidental service exposure | feature | feasible | 0 | 2019-09-20 | NETWORK_ACCESSのデフォルト値をコンテナ単位で上書き可能にし、意図せぬサービス公開を防ぐグローバルデフォルト設定オプションの追加要求。 |
| [#1937](https://github.com/nginx-proxy/nginx-proxy/issues/1937) | No upstreams generated when container has it's own IP | bug | feasible | 1 | 2022-04-02 | コンテナが独自のIPを持つ場合にアップストリームが生成されないバグ。docker-genのネットワーク検出ロジックの問題の可能性。 |
| [#1380](https://github.com/nginx-proxy/nginx-proxy/issues/1380) | Dockerfile EXPOSE overrides --expose argument | bug | feasible | 1 | 2020-01-13 | DockerfileのEXPOSE命令が`docker run --expose`引数より優先されてしまいポート選択が誤動作する問題。docker-genのポート選択ロジックの修正が必要。 |
| [#1030](https://github.com/nginx-proxy/nginx-proxy/issues/1030) | Way to add proxy entry for a network IP | feature | needs-discussion | 2 | 2018-01-16 | DockerコンテナではなくネットワークIPに対してプロキシエントリを追加できるようにする機能要求。 |
| [#1138](https://github.com/nginx-proxy/nginx-proxy/issues/1138) | Expose different containers on different ports of the url | feature | needs-discussion | 1 | 2018-05-29 | URLの異なるポートで別々のコンテナを公開したい機能要求。現設計とは異なるアーキテクチャが必要になる可能性。 |
| [#1178](https://github.com/nginx-proxy/nginx-proxy/issues/1178) | NETWORK_ACCESS=internalorhtaccess | feature | needs-discussion | 0 | 2018-10-17 | NETWORK_ACCESSにinternalとhtaccess両方を組み合わせる新しい値を追加してほしいという機能要求。設計議論が必要。 |
| [#1199](https://github.com/nginx-proxy/nginx-proxy/issues/1199) | NETWORK_ACCESS=internal doesn't make sense for docker | bug | needs-discussion | 4 | 2018-11-18 | NETWORK_ACCESS=internalの動作がDockerのネットワーク概念と矛盾しているとの指摘。実装の意図と現実の乖離について設計議論が必要。 |
| [#986](https://github.com/nginx-proxy/nginx-proxy/issues/986) | [Question] Multiple services with different ports and the same domain | question-support | needs-discussion | 4 | 2017-11-23 | 同一ドメインで異なるポートの複数サービスをプロキシする方法についての質問。 |
| [#1009](https://github.com/nginx-proxy/nginx-proxy/issues/1009) | SSH port 22 nginx stream proxy setting possible on this image? | question-support | needs-discussion | 1 | 2017-12-22 | このイメージでSSHポート22のnginx streamプロキシが可能かどうかの質問。 |
| [#1058](https://github.com/nginx-proxy/nginx-proxy/issues/1058) | Is it possible to exclude service access within the same Docker network? | question-support | needs-discussion | 0 | 2018-02-05 | 同一Dockerネットワーク内のサービスアクセスを除外できるかという質問。NETWORK_ACCESS=internalが部分的に対応しているが完全な解答が必要。 |
| [#2637](https://github.com/nginx-proxy/nginx-proxy/issues/2637) | extend multiports to finally have a proper raw / non-http ports API | feature | complex | 4 | 2025-07-13 | HTTPではない生TCPポートをVIRTUAL_PORTで扱えるようにするAPI拡張の要望。nginxのstreamモジュール対応も含む大きな変更。 |
| [#370](https://github.com/nginx-proxy/nginx-proxy/issues/370) | Multiple IP for the reverse proxy listen on | feature | likely-obsolete | 1 | 2016-02-22 | nginxが複数のIPアドレスでリッスンできるようにしてほしい要望。nginx.tmplのlistenディレクティブ拡張が必要。 |
| [#133](https://github.com/nginx-proxy/nginx-proxy/issues/133) | nginx-proxy only sees docker's virtual interface (and IP) | bug | likely-obsolete | 30 | 2015-03-26 | nginx-proxyがDockerの仮想インターフェース（docker0）のIPしか認識しない問題。ネットワーク設定またはdocker-genのIP解決方法の問題。コメント数30と高エンゲージメントだが非常に古い。 |
| [#1050](https://github.com/nginx-proxy/nginx-proxy/issues/1050) | Issues with macvlan driver & multiple networks? | bug | likely-obsolete | 10 | 2018-01-31 | macvlanドライバーと複数ネットワーク使用時の問題。コメント数が多いが古く、現在のDockerバージョンでは状況が変わっている可能性。 |
| [#699](https://github.com/nginx-proxy/nginx-proxy/issues/699) | Exposing and listening to port 3306 breaks nginx-proxy under certain circumstances | bug | likely-obsolete | 1 | 2017-01-27 | 3306番ポートを公開するコンテナがある場合にnginx-proxyが壊れる問題。非HTTPポート露出時のdocker-genの挙動の問題。 |
| [#916](https://github.com/nginx-proxy/nginx-proxy/issues/916) | Port not exposed | question-support | likely-obsolete | 5 | 2017-08-30 | コンテナのポートが公開されない問題。ポート設定ミスのサポート質問で、VIRTUAL_PORTの使い方に関する内容。 |
| [#797](https://github.com/nginx-proxy/nginx-proxy/issues/797) | One fqdn, two containers with one port each | question-support | likely-obsolete | 3 | 2017-04-19 | 1つのFQDNに対して異なるポートを持つ2つのコンテナをルーティングしたい。VIRTUAL_PATHの活用が解決策。 |
| [#1312](https://github.com/nginx-proxy/nginx-proxy/issues/1312) | Connection Refused Connecting Two Containers | question-support | likely-obsolete | 1 | 2019-08-08 | コンテナ間通信で接続拒否される問題。Dockerネットワーク設定の誤りが原因の典型例。 |

</details>

<details>
<summary><b>Load balancing / upstream / health checks</b> — 18件</summary>

| Issue | タイトル | 種別 | 容易度 | コメント | 作成 | メモ |
|---|---|---|---|---|---|---|
| [#987](https://github.com/nginx-proxy/nginx-proxy/issues/987) | Feature Proposal: Opt-in support for HEALTHCHECK option | feature | feasible | 3 | 2017-11-24 | コンテナのHEALTHCHECKステータスを考慮してプロキシ対象を制御するオプトイン機能。2025年まで更新あり、継続的な関心が確認できる。 |
| [#968](https://github.com/nginx-proxy/nginx-proxy/issues/968) | nginx-proxy health check | feature | feasible | 1 | 2017-11-07 | nginx-proxy自体のヘルスチェックエンドポイントの追加要求。 |
| [#1383](https://github.com/nginx-proxy/nginx-proxy/issues/1383) | Intermittent 502 bad gateway error | bug | feasible | 8 | 2020-01-22 | 断続的な502エラーの問題。コンテナ再起動時のupstream切り替えタイミングやdocker-genの再生成遅延が原因と疑われ、コメント数が多く再現事例が集まっている。 |
| [#909](https://github.com/nginx-proxy/nginx-proxy/issues/909) | Loadbalancing one vhost with multiple listening ports | feature | needs-discussion | 0 | 2017-08-22 | 一つのvhostに対して複数ポートをリスニングしてロードバランシングする機能要求。テンプレートの大幅変更が必要。 |
| [#419](https://github.com/nginx-proxy/nginx-proxy/issues/419) | Waiting for healthy containers | feature | needs-discussion | 0 | 2016-04-12 | ヘルスチェックをパスしたコンテナのみをupstreamに追加してほしい要望。Dockerのhealthcheck機能との連携が必要で、実装方針の議論が必要。 |
| [#886](https://github.com/nginx-proxy/nginx-proxy/issues/886) | Allow upstream options | feature | needs-discussion | 0 | 2017-07-26 | nginxのupstreamブロックにオプション（keepalive等）を追加できる機能の要求。テンプレート変更で対応可能だが、どのオプションを許可するか設計議論が必要。 |
| [#979](https://github.com/nginx-proxy/nginx-proxy/issues/979) | How to configure TCP load balancing over 5222 port? | question-support | needs-discussion | 3 | 2017-11-15 | TCPポート5222でのロードバランシング設定についての質問（nginx streamモジュールが必要）。 |
| [#708](https://github.com/nginx-proxy/nginx-proxy/issues/708) | Support for Zero downtime deployment - asking for guidance for PR | feature | complex | 3 | 2017-02-08 | ゼロダウンタイムデプロイのサポートを実装したい（PRガイダンスを求めている）。nginx graceful reloadとupstreamのヘルスチェック連携が必要。 |
| [#221](https://github.com/nginx-proxy/nginx-proxy/issues/221) | session affinity | feature | complex | 3 | 2015-08-20 | スティッキーセッション（セッションアフィニティ）のサポート要望。nginxのip_hashやstickyモジュールが必要で、テンプレートの大幅な変更が必要。 |
| [#509](https://github.com/nginx-proxy/nginx-proxy/issues/509) | incorporate http and tcp/udp loadbalancing | feature | complex | 1 | 2016-07-16 | HTTPだけでなくTCP/UDPのロードバランシングをnginxのstream moduleで対応してほしい要望。nginx.tmplの大幅な拡張が必要で複雑。 |
| [#1400](https://github.com/nginx-proxy/nginx-proxy/issues/1400) | Upstream connection pooling | feature | complex | 0 | 2020-02-19 | 上流コンテナへの接続プーリング（keepalive）を有効にする機能要求。nginx.tmplのupstreamブロックにkeepaliveディレクティブを追加することで実装可能だが、パラメータ管理が複雑になる。 |
| [#1253](https://github.com/nginx-proxy/nginx-proxy/issues/1253) | 503 connect() failed (111: Connection refused) while connecting to upstream | bug | likely-obsolete | 9 | 2019-03-25 | アップストリームへの接続拒否503エラー。コンテナ起動タイミングの問題が多く、ヘルスチェック機能の不足が根本原因として議論されている。 |
| [#327](https://github.com/nginx-proxy/nginx-proxy/issues/327) | interference when one web server is down in two web server hosting environment | bug | likely-obsolete | 9 | 2015-12-31 | 2つのコンテナのうち1つが停止すると両方が影響を受ける問題。upstreamのヘルスチェックなしでの障害伝播と思われる。 |
| [#820](https://github.com/nginx-proxy/nginx-proxy/issues/820) | error: upstream server temporarily disabled while connecting to upstream | bug | likely-obsolete | 8 | 2017-05-14 | アップストリームへの接続失敗時に一時的に無効化されるエラー。nginxのmax_failsやfail_timeout設定の調整が対応策。 |
| [#871](https://github.com/nginx-proxy/nginx-proxy/issues/871) | USE_IP_HASH seems to have zero impact to the default.conf | bug | likely-obsolete | 2 | 2017-07-04 | USE_IP_HASH環境変数がdefault.confに反映されないバグ。テンプレートのバグの可能性があるが現在は修正済みの可能性が高い。 |
| [#1364](https://github.com/nginx-proxy/nginx-proxy/issues/1364) | how implementat load balance? | question-support | likely-obsolete | 2 | 2019-12-01 | ロードバランシングの実装方法についての質問。同一VIRTUAL_HOSTを持つ複数コンテナ起動で自動的にround-robinになる既存機能で対応可能。 |
| [#306](https://github.com/nginx-proxy/nginx-proxy/issues/306) | Multiple containers with same VIRTUAL_HOST without load balancing | question-support | likely-obsolete | 2 | 2015-12-01 | 同じVIRTUAL_HOSTを持つ複数コンテナでロードバランシングなしに特定コンテナにルーティングする方法の質問。 |
| [#1210](https://github.com/nginx-proxy/nginx-proxy/issues/1210) | Nginx-proxy behind a load balancer (AWS ELB for example) | question-support | likely-obsolete | 1 | 2018-12-13 | AWS ELB等のロードバランサー背後でnginx-proxyを使う構成についての質問。X-Forwarded-Forの扱いやヘルスチェックの設定が焦点。 |

</details>

<details>
<summary><b>DNS / hostnames / aliases / default host</b> — 16件</summary>

| Issue | タイトル | 種別 | 容易度 | コメント | 作成 | メモ |
|---|---|---|---|---|---|---|
| [#2699](https://github.com/nginx-proxy/nginx-proxy/issues/2699) | [Bug/Docs] Environment variable RESOLVERS is overwritten by entrypoint script, contradicting documentation | bug | good-first | 0 | 2026-01-31 | RESOLVERSエンバイロメント変数がエントリポイントスクリプトで上書きされてしまいドキュメントと矛盾するバグ。スクリプト修正で対応可能。 |
| [#1285](https://github.com/nginx-proxy/nginx-proxy/issues/1285) | Improve / alternate aliases handling | feature | feasible | 4 | 2019-06-05 | エイリアスホスト（VIRTUAL_HOST複数指定）の処理を改善する要求。サーバー名ごとに個別設定を持てるようにする実装案あり。 |
| [#1506](https://github.com/nginx-proxy/nginx-proxy/issues/1506) | Using a variable in the resolver directive | feature | feasible | 2 | 2020-10-05 | resolverディレクティブに変数を使いたいという要望。RESOLVERS環境変数との統合や動的DNS解決の改善につながる。 |
| [#1254](https://github.com/nginx-proxy/nginx-proxy/issues/1254) | Seamless non-www and www | feature | feasible | 1 | 2019-03-25 | wwwあり/なしのドメインを単一リダイレクトでシームレスに処理する機能要求。VIRTUAL_HOSTエイリアスとリダイレクト設定の組み合わせで実現可能。 |
| [#1555](https://github.com/nginx-proxy/nginx-proxy/issues/1555) | nginx-proxy uses default host without DEFAULT_HOST set | bug | feasible | 7 | 2021-02-01 | DEFAULT_HOST未設定時に意図せずデフォルトホストが使用されるバグ。7コメントで継続的な関心あり、nginx.tmplのロジック改善で対応可能。 |
| [#1142](https://github.com/nginx-proxy/nginx-proxy/issues/1142) | FEATURE working with windows hosts file | feature | needs-discussion | 4 | 2018-06-17 | Windowsのhostsファイルと連携する機能要求。Windowsでの開発環境向けだが実現方法の検討が必要。 |
| [#835](https://github.com/nginx-proxy/nginx-proxy/issues/835) | DEFAULT_HOST as IP of another server | feature | needs-discussion | 1 | 2017-05-26 | DEFAULT_HOSTに別サーバーのIPアドレスを指定してプロキシする機能要求。現在はドメイン名のみ対応でIP直接指定の対応が必要。 |
| [#1264](https://github.com/nginx-proxy/nginx-proxy/issues/1264) | Use container_name as part of hostname? | feature | needs-discussion | 0 | 2019-04-10 | Dockerのcontainer_nameを自動的にホスト名として使用できるようにする機能要求。VIRTUAL_HOST不要化につながるが設計議論が必要。 |
| [#126](https://github.com/nginx-proxy/nginx-proxy/issues/126) | [feature request] Handle DNS lookups | feature | likely-obsolete | 3 | 2015-03-16 | upstreamのDNS解決を動的に処理してほしい要望。resolverディレクティブの追加で対応可能だが、古いイシュー。 |
| [#350](https://github.com/nginx-proxy/nginx-proxy/issues/350) | Support using docker hostname option | feature | likely-obsolete | 0 | 2016-01-27 | Dockerの--hostnameオプションをVIRTUAL_HOSTとして自動認識してほしい要望。docker-genテンプレートでコンテナのHostnameフィールドを参照する方法で実装可能。 |
| [#994](https://github.com/nginx-proxy/nginx-proxy/issues/994) | DNS issue when running as service. Works fine as single container | bug | likely-obsolete | 9 | 2017-12-08 | サービスとして実行時にDNS解決が失敗するが単体コンテナでは正常に動く問題。 |
| [#1397](https://github.com/nginx-proxy/nginx-proxy/issues/1397) | docker containers cannot use virtual domain | question-support | likely-obsolete | 7 | 2020-02-16 | DockerコンテナからVIRTUAL_HOSTドメインへアクセスできない問題。内部DNSの解決ができずループバック設定が必要なケース。 |
| [#883](https://github.com/nginx-proxy/nginx-proxy/issues/883) | Proxy without a domain possible? | question-support | likely-obsolete | 3 | 2017-07-21 | ドメインなし（IPアドレス直接）でのプロキシ利用についての質問。DEFAULT_HOSTやカスタム設定で対応可能な内容。 |
| [#1434](https://github.com/nginx-proxy/nginx-proxy/issues/1434) | Curl Container A to Container B using virtual_host | question-support | likely-obsolete | 1 | 2020-05-12 | コンテナ間でVIRTUAL_HOSTを使って通信する際のDNS解決に関する質問。内部ネットワーク設定の問題。 |
| [#1439](https://github.com/nginx-proxy/nginx-proxy/issues/1439) | Access a site with another domain | question-support | likely-obsolete | 0 | 2020-05-24 | 別ドメインからサイトにアクセスする設定（エイリアスやリダイレクト）についての質問。 |
| [#1192](https://github.com/nginx-proxy/nginx-proxy/issues/1192) | DNS not resolving despite seemingly correct config | question-support | likely-obsolete | 0 | 2018-11-07 | 設定は正しいのにDNSが解決できないとの質問。コンテナネットワーク設定の問題で詳細不明。 |

</details>

<details>
<summary><b>Build / image / base / dependencies</b> — 12件</summary>

| Issue | タイトル | 種別 | 容易度 | コメント | 作成 | メモ |
|---|---|---|---|---|---|---|
| [#1161](https://github.com/nginx-proxy/nginx-proxy/issues/1161) | generate-dhparam becomes a zombie process | bug | feasible | 6 | 2018-08-20 | generate-dhparamスクリプトがゾンビプロセスになる問題。コメント数が多く影響が広い。プロセス管理の修正が必要。 |
| [#785](https://github.com/nginx-proxy/nginx-proxy/issues/785) | Google PageSpeed Module | feature | needs-discussion | 6 | 2017-04-06 | Google PageSpeedモジュールをイメージに追加するリクエスト。ビルドの複雑化やメンテナンスコスト増加のトレードオフがある。 |
| [#1201](https://github.com/nginx-proxy/nginx-proxy/issues/1201) | Brotli Compression | feature | complex | 3 | 2018-11-26 | Gzipに加えてBrotli圧縮をサポートする機能要求。nginxにbrotliモジュールを追加する必要があり、ビルドプロセスの変更が必要。 |
| [#114](https://github.com/nginx-proxy/nginx-proxy/issues/114) | Support Docker via TLS | feature | likely-obsolete | 2 | 2015-03-04 | Docker APIへの接続をTLS経由でサポートしてほしい要望。docker-gen側の機能で既にサポートされている可能性が高い。 |
| [#931](https://github.com/nginx-proxy/nginx-proxy/issues/931) | Error when trying to run image | bug | likely-obsolete | 5 | 2017-09-17 | イメージ起動時のエラー報告。2017年の古いバージョン特有の問題で現在は修正済みの可能性が高い。 |
| [#1482](https://github.com/nginx-proxy/nginx-proxy/issues/1482) | Custom Let's Encrypt Workflow behind nginx-proxy broken. Missing numeric version for latest docker image. | bug | likely-obsolete | 0 | 2020-07-29 | 「latest」タグに数値バージョンが付いていないため、Let's Encryptカスタムワークフローが壊れるという問題。イメージタグ管理の改善要求。 |
| [#1280](https://github.com/nginx-proxy/nginx-proxy/issues/1280) | CVE-2019-5021: Alpine Docker Image 'null root password' Vulnerability | bug | likely-obsolete | 0 | 2019-05-23 | Alpine Linuxの旧CVEによるrootパスワードnull脆弱性の報告。ベースイメージ更新で解消済みの古い問題。 |
| [#1238](https://github.com/nginx-proxy/nginx-proxy/issues/1238) | Setting WORKDIR in Dockerfile breaks ENTRYPOINT script. | bug | likely-obsolete | 0 | 2019-02-28 | Dockerfileでカスタムのイメージ作成時にWORKDIRを設定するとENTRYPOINTスクリプトが失敗するバグ。絶対パスを使用することで回避可能。 |
| [#1068](https://github.com/nginx-proxy/nginx-proxy/issues/1068) | Alpine version image size is larger | bug | likely-obsolete | 0 | 2018-02-13 | Alpineベースイメージがdebianより大きいという報告。現在は改善済みの可能性が高い古い問題。 |
| [#1061](https://github.com/nginx-proxy/nginx-proxy/issues/1061) | Sharing volumes of nginx with right permissions | question-support | likely-obsolete | 2 | 2018-02-08 | nginxボリュームを適切なパーミッションで共有する方法に関する質問。ユーザー権限設定の一般的な問題。 |
| [#1065](https://github.com/nginx-proxy/nginx-proxy/issues/1065) | Nginx command failure | question-support | likely-obsolete | 1 | 2018-02-13 | nginxコマンドが失敗するというサポート質問。設定エラーや権限問題が原因と思われる古い報告。 |
| [#1367](https://github.com/nginx-proxy/nginx-proxy/issues/1367) | Docker Image Tags | meta | likely-obsolete | 4 | 2019-12-07 | Dockerイメージのタグ体系（バージョン付きタグ、alpine版タグなど）の改善要求。現在は既に対応済みの可能性が高い。 |

</details>

<details>
<summary><b>Documentation</b> — 7件</summary>

| Issue | タイトル | 種別 | 容易度 | コメント | 作成 | メモ |
|---|---|---|---|---|---|---|
| [#998](https://github.com/nginx-proxy/nginx-proxy/issues/998) | Howto proxy an app running without docker | question-support | feasible | 11 | 2017-12-12 | Dockerなしで動くアプリをnginx-proxy経由でプロキシする方法についての質問。ドキュメントに事例追加が望ましい。 |
| [#967](https://github.com/nginx-proxy/nginx-proxy/issues/967) | [Question/Docs] `vhost.d/example.com` examples and redirects | docs | likely-obsolete | 1 | 2017-11-05 | vhost.dディレクトリ内の設定ファイルの例とリダイレクト設定のドキュメントが不足しているという指摘。 |
| [#907](https://github.com/nginx-proxy/nginx-proxy/issues/907) | Clarification on virtual_port | docs | likely-obsolete | 1 | 2017-08-17 | VIRTUAL_PORTの動作についての説明要求。ドキュメント改善で対応可能だが古い問題。 |
| [#825](https://github.com/nginx-proxy/nginx-proxy/issues/825) | Clarification on HTTP headers | docs | likely-obsolete | 1 | 2017-05-17 | HTTPヘッダーの動作についての説明要求。ドキュメント改善で対応可能だが古い問題。 |
| [#877](https://github.com/nginx-proxy/nginx-proxy/issues/877) | Best practices for migrating to nginx-proxy | docs | likely-obsolete | 0 | 2017-07-10 | nginx-proxyへの移行に関するベストプラクティスのドキュメント要求。コメント0件で古い要求。 |
| [#956](https://github.com/nginx-proxy/nginx-proxy/issues/956) | How to use it? | question-support | likely-obsolete | 7 | 2017-10-27 | 使い方全般についての漠然とした質問。ドキュメントの改善で対応できる。 |
| [#959](https://github.com/nginx-proxy/nginx-proxy/issues/959) | Expose port required or not? | question-support | likely-obsolete | 6 | 2017-10-28 | コンテナのポートをexposeする必要があるかどうかについての質問。ドキュメントを明確にすることで解消できる。 |

</details>

<details>
<summary><b>Logging / monitoring / debugging</b> — 4件</summary>

| Issue | タイトル | 種別 | 容易度 | コメント | 作成 | メモ |
|---|---|---|---|---|---|---|
| [#1455](https://github.com/nginx-proxy/nginx-proxy/issues/1455) | Logrotate | feature | feasible | 4 | 2020-06-21 | nginxのアクセスログをlogrotateで管理する機能要求。コンテナ内にlogrotateの設定を同梱するか、USR1シグナルによるログローテーション対応の追加を求めている。 |
| [#654](https://github.com/nginx-proxy/nginx-proxy/issues/654) | Access and Error logs not being logged | bug | likely-obsolete | 11 | 2016-12-09 | アクセスログとエラーログが記録されない問題。ログパスの設定やDockerのログドライバー設定との競合が原因と思われる。 |
| [#1389](https://github.com/nginx-proxy/nginx-proxy/issues/1389) | disabling error_log causes SIGTERM | bug | likely-obsolete | 0 | 2020-02-06 | error_logを無効化するとSIGTERMが発生しnginxが停止する問題。カスタム設定でerror_logをoffにした際のnginx設定検証エラーの可能性。 |
| [#321](https://github.com/nginx-proxy/nginx-proxy/issues/321) | ping in syslog | bug | likely-obsolete | 0 | 2015-12-19 | nginxのsyslogにpingリクエストのログが大量に出力される問題。ヘルスチェックのログフィルタリングが必要。 |

</details>

<details>
<summary><b>Other</b> — 3件</summary>

| Issue | タイトル | 種別 | 容易度 | コメント | 作成 | メモ |
|---|---|---|---|---|---|---|
| [#1147](https://github.com/nginx-proxy/nginx-proxy/issues/1147) | too much bike shedding? | meta | needs-discussion | 3 | 2018-06-30 | プロジェクトの意思決定プロセスに関するメタ的な議論。実装タスクではなく開発文化の問題提起。 |
| [#1214](https://github.com/nginx-proxy/nginx-proxy/issues/1214) | Web page | question-support | likely-obsolete | 1 | 2018-12-18 | タイトルが「Web page」のみで内容不明な質問。ほぼ確実にスパムまたは無効なissue。 |
| [#990](https://github.com/nginx-proxy/nginx-proxy/issues/990) | Unable to access MariaDB Externally | question-support | likely-obsolete | 0 | 2017-11-28 | MariaDBに外部からアクセスできないという質問（nginx-proxyの範疇外の可能性が高い）。 |

</details>

---

## 統計・分布

### PR: 推奨アクション別

| 推奨 | 件数 |
|---|---|
| needs-discussion | 12 |
| reimplement-fresh | 9 |
| revive-rebase | 7 |
| likely-obsolete | 6 |
| low-priority | 4 |

### PR: 価値別

| 価値 | 件数 |
|---|---|
| high | 8 |
| medium | 18 |
| low | 12 |

### Issue: 種別

| 種別 | 件数 |
|---|---|
| question-support | 112 |
| bug | 99 |
| feature | 75 |
| docs | 4 |
| meta | 2 |

### Issue: 実装容易度

| 容易度 | 件数 |
|---|---|
| good-first | 4 |
| feasible | 52 |
| needs-discussion | 46 |
| complex | 12 |
| likely-obsolete | 178 |

---

## 進捗ログ

### 陳腐化Issueへのコメント campaign(2026-06-18〜20)

likely-obsolete 候補を現行コード/docs/test で**検証してから**、根拠付きの控えめなコメントを本家に投稿する取り組み(本家権限は READ のみ=コメントのみ)。

**成果:** 計14件にコメント投稿 → **12件をメンテナがクローズ**(`resolved`→completed / `environment-obsolete`→not planned と判定が整合)。検証で「実は未解決/部分的」と判明した候補は投稿せず却下=検証ステップが有効と実証。検証データ全文: `research/data/verify-pilot-2026-06-18.json` / `verify-batch2-2026-06-19.json`。

**未クローズ・フォロー対象(2件):**

| Issue | 判定 | 根拠の要点 |
|---|---|---|
| [#1051](https://github.com/nginx-proxy/nginx-proxy/issues/1051) | resolved | 起動/停止で自動reload(Procfile `-watch`/`-notify`)が中核機能。再作成不要 |
| [#1300](https://github.com/nginx-proxy/nginx-proxy/issues/1300) | resolved | proxy_pass の proto は `VIRTUAL_PROTO` 既定 `http`(報告はサードパーティfork) |

**却下(再投稿しないこと):**

| Issue | 判定 | 却下理由 |
|---|---|---|
| [#133](https://github.com/nginx-proxy/nginx-proxy/issues/133) | not-resolved | Docker NAT/ホストネットワーク依存の環境問題。コード修正で直らない |
| [#139](https://github.com/nginx-proxy/nginx-proxy/issues/139) | partially | 現行テンプレに「効かない gzip_types」が残存=未修正 |
| [#912](https://github.com/nginx-proxy/nginx-proxy/issues/912) | partially | 原因は PHP/WP 側の既定値の疑い |
| [#1143](https://github.com/nginx-proxy/nginx-proxy/issues/1143) | not-resolved | 実体は Swarm 非対応の文書化要望(未対応) |
| [#263](https://github.com/nginx-proxy/nginx-proxy/issues/263) | not-resolved | FORCE_HTTPS/HSTS 強制トグル未実装(X-Forwarded-Proto 判定なし) |
| [#350](https://github.com/nginx-proxy/nginx-proxy/issues/350) | not-resolved | docker hostname からの vhost 自動導出は未実装 |
| [#597](https://github.com/nginx-proxy/nginx-proxy/issues/597) | not-resolved | 任意設定の env 注入(VIRTUAL_HOST_CONFIG*)未実装 |
| [#621](https://github.com/nginx-proxy/nginx-proxy/issues/621) | partially | 同一コンテナ内のホスト別 HTTPS_METHOD は未実装 |
| [#1238](https://github.com/nginx-proxy/nginx-proxy/issues/1238) | not-resolved | WORKDIR/ENTRYPOINT 構造温存=未修正 |
| [#1365](https://github.com/nginx-proxy/nginx-proxy/issues/1365) | partially | nginx の map_hash 宣言順という設定上の制約。コードバグではない |

**次の一手:** 次バッチを残りの likely-obsolete から同基準(機能要望・具体バグで現行機能を引用できるもの)で検証→投稿。漠然とした 502/504/DNS サポート系は対象外。

### 2026-06-19 — 実装PR(本家へ提出)

カタログのロードマップ/good-first 候補を実装し、upstream へPRを提出。各PRは main 起点の独立ブランチ。

| PR | 内容 | Issue | 状態 |
|---|---|---|---|
| [#2750](https://github.com/nginx-proxy/nginx-proxy/pull/2750) | `RESOLVERS` 環境変数がエントリポイントで上書きされるバグ修正 | [#2699](https://github.com/nginx-proxy/nginx-proxy/issues/2699) | ✅ **MERGED**(2026-06-19) |
| [#2751](https://github.com/nginx-proxy/nginx-proxy/pull/2751) | `VIRTUAL_INDEX`(FastCGI の `index` ディレクティブ) | [#1117](https://github.com/nginx-proxy/nginx-proxy/issues/1117) / supersedes #1118 | OPEN |
| [#2752](https://github.com/nginx-proxy/nginx-proxy/pull/2752) | `X_ROBOTS_TAG`(per-vhost の `X-Robots-Tag` ヘッダ) | [#1125](https://github.com/nginx-proxy/nginx-proxy/issues/1125) | OPEN |
| [#2753](https://github.com/nginx-proxy/nginx-proxy/pull/2753) | redirect サーバーブロックで `vhost.d` を include | [#1613](https://github.com/nginx-proxy/nginx-proxy/issues/1613) / supersedes #1618 | OPEN |
| [#2754](https://github.com/nginx-proxy/nginx-proxy/pull/2754) | `test_server-down` のフレーキー対策(`@pytest.mark.flaky`) | — | ✅ **MERGED**(2026-06-20) |

**マージ済み2件: [#2750](https://github.com/nginx-proxy/nginx-proxy/pull/2750)(2026-06-19)・[#2754](https://github.com/nginx-proxy/nginx-proxy/pull/2754)(2026-06-20)**。#2750 で upstream の `main` が前進(#2699 を含む)ため、以降のPRは新しい `main` を起点にする。残り3件(#2751/#2752/#2753)はオープン・レビュー待ち。

**メモ:**

- #2753 は当初フレーキー対策を同梱していたが、buchdag の「無関係な変更を混ぜない」レビューで **#2754 に分離 → #2754 はマージ済み**。
- #2754 の `@pytest.mark.flaky` に buchdag が「pytest-rerunfailures が要るのでは」と指摘 → 本repoは `pytest-ignore-flaky` + CI の `pytest --ignore-flaky` で機能する旨を返信(コード変更なし)してマージに至った。
- buchdag の要望で **PRタイトル/コミットメッセージは簡潔な1文に統一**(2026-06-20、全PR短縮済み)。

---

_生成: 並列トリアージワークフロー(15エージェント / Opus 4.8 + Sonnet)。元データは `research/data/triage-result.json`。進捗ログは随時追記。_
