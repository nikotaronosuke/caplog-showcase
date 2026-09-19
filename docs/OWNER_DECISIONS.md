# Owner Decision Log

日本語 | [English](OWNER_DECISIONS.en.md)

Caplog は、Web と Mobile の両方を持つお出かけプロダクトですが、
開発では「何を共通化し、何を分けるか」を何度も決め直してきました。

この文書では機能一覧ではなく、プロジェクトオーナーとして
**どの体験を主役にし、どこで安全側へ倒し、どの責務をクライアントから外したか**
が分かる判断だけをまとめます。

---

## 1. Webを捨てずに、開発の主役をMobileへ移した

### 課題

Caplog はもともとWeb版があり、本番運用も続いています。

一方で、

- 地図を見ながら移動する
- 写真からプラン候補を作る
- 位置情報を使う
- 外出先で記録する

といった中心体験は、Mobileの方が自然でした。

### 判断

Webを停止してMobileへ全面移行するのではなく、

- **Mobile = 現在の主開発クライアント**
- **Web = 継続運用するブラウザクライアント**
- **データ = Supabaseで共有**

という形にしました。

クライアントは別でも、プラン・スポット・投稿を二重管理しない方針です。

「新しい方へ全部作り直す」より、
**利用中のWebを残しながら、中心体験だけMobileへ寄せる**判断です。

**Evidence:** [README — Mobile-first / Shared data](../README.md#shared-data-separate-clients)

---

## 2. 公開URLとDB内部IDを分けた

### 課題

DB内部のUUIDをそのまま公開URLに使えば実装は単純です。

しかし、

- URLが長い
- 地域との関係が見えない
- 共有時に人間が読んでも意味がない
- 将来URL設計を変えにくい

という問題がありました。

### 判断

内部のUUIDとは別に短い `public_id` を持たせ、
公開URLを

```text
/posts/{prefecture}/{city}/{public_id}
```

へ統一しました。

ただし `public_id` は**認可の仕組みにはしません**。

公開用identityとsecurity boundaryは分け、
非公開データの保護は認証・所有者判定・DB側制御へ任せています。

**Evidence:** [Public URL design](public-url-design.md)

---

## 3. URL移行で「きれいさ」より、既存リンクを壊さないことを優先した

### 課題

公開URLを新形式へ変更したとき、
旧UUID URLを切れば構成はきれいになります。

しかし、既に共有されたURLや検索エンジン上のURLが404になります。

### 判断

旧URLは残し、正規URLが作れるときだけ **308 Permanent Redirect** します。

一方で、

- 地域情報が不足している
- `public_id` が無い
- 移行途中のデータ

では、無理に新URLを組み立てずUUID形式へfallbackします。

redirect先が自分自身になる場合もredirectしません。

「新形式へ完全統一できないデータは壊す」ではなく、
**移行途中でも表示を継続する**方を選びました。

**Evidence:** [Public URL design — legacy redirect / fallback](public-url-design.md#旧-uuid-url-を壊さない)

---

## 4. 地図の経路が取れないとき、見栄えのための直線を描かなかった

### 課題

ルート情報が無いスポット間を直線で結べば、
地図は「完成しているように」見えます。

しかし徒歩・車でも実際の道路とは違い、
電車やバスではさらに誤解を生みます。

### 判断

**経路が無い区間は、線なしにする**ことにしました。

保存済みpolylineが壊れている場合も、

- decode失敗
- 座標範囲外
- 2点未満

なら描画しません。

UIを埋めることより、
**存在しない移動経路をそれらしく見せないこと**を優先しています。

**Evidence:** [Map system — 嘘の直線を描かない](map-system.md#嘘の直線を描かない)

---

## 5. 経路取得を「全部成功しないと失敗」にはしなかった

### 課題

複数スポットのプランでは、隣接区間のうち1つだけRoutes APIが失敗することがあります。

プラン全体を1つのroute requestとして扱うと、
1区間の失敗で全部の経路が失われます。

### 判断

経路を**隣接スポットごとのsegment**として扱い、
取得できた区間だけ保存します。

1区間失敗しても、

- 他区間のpolylineは残す
- 失敗区間はnull
- UIでは線なし

としました。

「全部そろわないと使えない」より、
**確実に取れた情報だけ使う部分成功**を選びました。

**Evidence:** [App API — Routesは区間ごとの部分成功](app-api.md#6-routes-は区間ごとの部分成功)

---

## 6. MobileにGoogleのserver credentialを置かず、App APIを境界にした

### 課題

MobileからPlaces / Routesを直接呼ぶ方が実装量は少なくなります。

ただしserver-side credentialをアプリbundleへ持たせることになり、

- key露出
- 課金制御
- provider responseの露出
- 利用者単位の制限

をクライアントへ背負わせます。

### 判断

Mobile専用の **Cloudflare Workers App API** を置きました。

MobileはSupabase sessionのBearerを渡し、
Worker側でユーザーを確認してから外部APIを呼びます。

credentialはWorkerより内側に留めます。

**Evidence:** [App API — API keyをMobile bundleに置かない](app-api.md#1-api-key-を-mobile-bundle-に置かない)

---

## 7. App APIをWeb本体とは別deploymentにした

### 課題

既存Web WorkerへMobile API endpointを追加すれば、
deploy先もコードベースも1つで済みます。

ただしMobileのAutocompleteやRoutesが増えたとき、
Web閲覧と同じCPU / request budget / 障害範囲を共有します。

### 判断

**Mobile App APIとWeb配信を別Workerへ分離**しました。

GoogleやSupabaseという共通依存は残りますが、

- deploy
- request traffic
- CPU budget
- incident boundary

を分けています。

コードを一か所へ集めることより、
**障害と課金トラフィックの境界を分けること**を優先しました。

**Evidence:** [App API — Web本体と障害境界を分ける](app-api.md#8-web-本体と障害境界を分ける)

---

## 8. 課金APIのrate limitはfail-openではなくfail-closedにした

### 課題

rate-limit基盤が落ちたときにfail-openへすると、
アプリ自体は動き続けます。

しかしPlaces / Routesは外部課金が発生するため、
制限が壊れた瞬間に無制限呼び出しへ流れる可能性があります。

### 判断

課金endpointは **fail-closed** にしました。

rate-limitを確認できなければproviderを呼びません。

一方で、課金の無いhealth / diagnosticsとはfailure policyを分けています。

可用性を一律に最大化するより、
**失敗したときのコストの大きさでpolicyを変える**判断です。

**Evidence:** [App API — fail-closed rate limit](app-api.md#3-課金対象-endpoint-は-fail-closed-の-rate-limit)

---

## 9. 外部APIのraw responseを、そのままMobileへ返さなかった

### 課題

App APIを単純proxyにすると実装は簡単です。

ただしGoogle側のresponse shape・不要field・raw errorまで
Mobileへ流れ込みます。

### 判断

Worker側で必要fieldだけを取り出し、
Mobile用の小さいresponse shapeへ変換します。

Google API側もField Maskを使い、

- Place Detailsは画面に必要なfieldだけ
- Routesはencoded polylineだけ

など取得範囲を絞りました。

目的はpayload削減だけでなく、
**APIコストやprovider依存範囲を必要以上に広げないこと**です。

**Evidence:** [App API — raw responseを返さない](app-api.md#4-google-raw-response-をそのまま-mobile-へ返さない) / [Field Mask](app-api.md#5-field-mask-で取得範囲を絞る)

---

## 10. 地図の診断性を残しつつ、位置情報をログへ残さない

### 課題

外部APIや地図の障害調査にはログが必要です。

一方で、

- 緯度・経度
- Place ID
- 検索語
- encoded polyline
- user id

まで残すと、診断ログが利用者の行動情報を持つことになります。

### 判断

ログは **counts-only / status中心** にしました。

request id、status、件数、短いerror codeなどは残しますが、
位置・検索内容・provider raw bodyは残しません。

「何もログしない」と「何でもログする」の中間として、
**障害調査に必要な形だけを最初から制限**しています。

**Evidence:** [App API — counts-only logs](app-api.md#7-counts-only-の診断ログ) / [Map system — 位置情報をログへ出さない](map-system.md#7-ログへ位置情報を出さない)

---

## 11. 地図UIは固定zoomではなく、実際のスポット間距離で動かした

### 課題

固定zoomは実装が単純ですが、

- 近いスポット同士ではピンが重なる
- 離れたスポットでは寄りすぎる

という問題がありました。

さらに下部カルーセルが地図へ重なるので、
単純にピン中心へ移動すると見づらくなります。

### 判断

選択スポットから**最も近い別スポットまでの距離**を使って表示範囲を計算し、
min / max内へclampしました。

さらにカルーセル分だけ中心を補正しています。

カード操作と地図focusも同期し、
「地図」と「プラン一覧」を別々のUIとして扱わない方針です。

**Evidence:** [Map system — dynamic zoom / carousel sync](map-system.md#2-カルーセルと地図を同期する)

---

## 12. 実機でしか分からない部分は、エミュレータ上の完成で終わらせなかった

地図・位置情報・写真・Androidのmarker描画などは、
ブラウザや静的mockだけでは確認できません。

そのためMobileはExpo dev clientを使い、
Android実機でUIと端末依存挙動を確認しながら進めています。

これは「実機対応」という機能ではなく、
**完成判定をコードやスクリーンショットだけに置かない**ための開発判断です。

**Evidence:** [README — Real-device verification](../README.md#real-device-verification)

---

## このプロジェクトで優先したもの

Caplogでは、実装の短さより次を優先しています。

- Webを壊さずMobile中心へ移行する
- 公開identityとsecurity identityを分ける
- URL移行で既存リンクを壊さない
- 分からない経路を見た目で補完しない
- 外部APIは部分成功を許す
- credentialと課金制御をMobileから外す
- WebとMobile APIの障害境界を分ける
- 課金系はfail-closedにする
- provider responseをそのままクライアントへ流さない
- 診断ログへ位置情報を残しすぎない
- 地図UIを実際の距離と操作に合わせる
- 最終確認を実機で行う

AIを使って実装していますが、
このプロジェクトで重要なのはコード量ではなく、
**便利さ・互換性・コスト・privacyの間で、どの境界を選んだか**です。
