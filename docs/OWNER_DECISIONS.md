# Owner Decision Log

日本語 | [English](OWNER_DECISIONS.en.md)

Caplog の production code は private です。この文書は「公開コードの証明」ではなく、public showcase で公開している設計資料に残した判断の要約です。4件に絞ります。

## 1. Webを捨てずに、開発の中心だけMobileへ移した

Web版はすでに本番稼働していました。一方で、位置情報・写真・地図を使う中心体験はMobileの方が自然でした。

全面置換ではなく、

- Mobile = 主開発クライアント
- Web = 継続運用
- Supabase = 共通データ

という形にしました。

**Evidence:** [README](../README.md)

## 2. 公開URLを変えても旧URLを切らなかった

UUID直出しから `/posts/{prefecture}/{city}/{public_id}` へ移行しましたが、旧UUID URLは残しました。

新URLを作れるときだけ308 redirectし、移行途中で必要情報が無ければ旧URLへfallbackします。

きれいなURL構成より、既存リンクを壊さないことを優先した判断です。

**Evidence:** [Public URL design](public-url-design.md)

## 3. route が無い区間を直線で埋めなかった

見栄えだけならスポット間を直線で結べますが、実際の道路や電車経路を表していません。

routeが取れない区間は線なし。複数区間のうち1つが失敗しても、取れた区間は残す partial success にしました。

**Evidence:** [Map system](map-system.md) / [App API](app-api.md)

## 4. Mobile APIをWeb本体から分け、課金endpointをfail-closedにした

Google Places / Routes のserver credentialをMobileへ置かず、専用Cloudflare Workerを境界にしました。

さらにWeb本体とは別deploymentへ分離。rate-limit基盤を確認できないとき、課金endpointはproviderへ流さず fail-closed にします。

ログも座標・Place ID・検索文字列・raw responseを残さず、status / counts中心にしました。

**Evidence:** [App API](app-api.md)
