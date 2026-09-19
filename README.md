# カプログ / Caplog

デートやお出かけを「見つける・作る・記録する」ためのモバイル中心のプロダクト。

現在のメインクライアントは Expo / React Native で開発している Mobile アプリです。

- **Mobile:** Expo / React Native — Android 実機で開発・確認中
- **Web:** https://caplog.jp — 継続運用中

<p align="center">
  <img src="screenshots/mobile-home.webp" width="23%" alt="Home" />
  <img src="screenshots/mobile-create-menu.webp" width="23%" alt="作成方法" />
  <img src="screenshots/mobile-plan-create.webp" width="23%" alt="プラン作成" />
  <img src="screenshots/mobile-spot-post.webp" width="23%" alt="スポット投稿" />
</p>

<p align="center">
  Home　/　作成方法　/　プラン作成　/　スポット投稿
</p>

## Product concept

行きたい場所をブックマークしておくアプリではありません。
次の 4 つがひと続きの循環になることを目指しています。

1. **見つける** — 次のお出かけ候補を探す
2. **プランにする** — 複数のスポットを 1 本のプランとしてまとめる
3. **記録する** — 実際に行った体験を写真と感想で残す
4. **つなげる** — その記録を次のプランの素材にする

「行った」で終わらせず、記録がそのまま次の計画の入力になる点が中心にあります。

## Engineering highlights

- **写真からプラン候補を作るとき、写真本体をアップロードしない**  
  端末内の写真から日時・位置情報だけを使って訪問スポット候補を起こし、候補生成のために写真そのものを外部へ送らない構成にしています。

- **Places / Routes を Mobile から直接呼ばない**  
  場所検索やルート取得は Cloudflare Workers の App API を経由し、外部 API の資格情報をクライアントに置かず、認証・キャッシュ・レート制限もこの層で扱います。

- **Mobile / Web で同じデータを共有する**  
  Supabase Auth / Database / Storage を共通基盤にし、Mobile と Web で別々のデータを持たず、同じプラン・スポット・投稿を扱います。

- **ネイティブ挙動は実機で確認する**  
  Expo dev client と Android 実機（Pixel 9）を使い、地図・位置情報・写真など端末依存の挙動を実機で確認しながら開発しています。

## Mobile features

現在実装されている範囲です。

**探す**

- 注目 / 新着 / 人気のプラン
- テーマ（半日・展望台・道の駅 など）や検索からの絞り込み

**作る**

- 複数スポットを順番に並べてプランを作成
- 写真から作成（写真の日時と位置からスポット候補を起こす）
- 記録から作成（デート記録のスポットをまとめてプラン化）

**記録する**

- スポット投稿（訪問したスポットを写真・感想付きで投稿）
- 写真は 1 投稿あたり複数枚まで添付可能

**管理する**

- プラン / スポットの保存
- 自分のプラン・自分のスポット・保存したプラン / スポットの一覧
- プランやスポットの位置の地図表示

## Tech stack

**Mobile**

- Expo
- React Native
- TypeScript
- Supabase
- React Navigation
- react-native-maps

**Web**

- Next.js
- React
- TypeScript
- Supabase
- Cloudflare Workers / OpenNext
- Google Maps

## Architecture

```mermaid
flowchart TD
    M["Mobile app<br/>(Expo / React Native)"]
    W["Web app<br/>(Next.js)"]
    A["App API<br/>(Cloudflare Workers)"]
    S["Supabase<br/>Auth / Database / Storage"]
    G["Google Maps Platform"]

    M -->|"auth・データ・写真"| S
    M -->|"場所検索・ルート"| A
    A --> G
    A --> S
    W -->|"auth・データ・写真"| S
    W -->|"地図表示"| G
```

Mobile と Web は同じ Supabase のデータを参照します。

App API は Mobile 向けのゲートウェイで、外部 API の呼び出しをサーバー側にまとめ、
結果をキャッシュする役割を持ちます。クライアントに置けない資格情報はこの層より内側にとどまります。

詳細は [docs/architecture.md](docs/architecture.md) を参照してください。

## Development

- Mobile-first で開発しています
- Android 実機（Pixel 9）で動作確認しています
- Expo dev client + Metro で、UI / JS の変更を実機へ即時反映しながら進めています
- Web / Mobile / 調査資産は 1 つの private monorepo で管理しています
- Web 版も継続して運用中です

## AI について

開発・調査には生成 AI を補助的に利用しています。
仕様・UX・実機確認・最終判断は人間が行っています。

---

> This repository is a public showcase of Caplog.
> Production source code and internal development materials are maintained privately.