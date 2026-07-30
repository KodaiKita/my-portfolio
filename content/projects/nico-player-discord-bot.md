---
title: NicoNico Player on Discord
tags:
  - discord-bot
  - typescript
  - bun
  - niconico
  - ffmpeg
  - puppeteer
  - docker
---

## 概要

ニコニコ動画の動画・音声コンテンツを Discord のボイスチャンネルでリアルタイム再生・共有するための高機能 Discord Bot です。  
ニコニコ動画の新配信システム（Domand / HLS）への完全対応に加え、既存の音楽 Bot では再生が困難だった「会員限定・ログイン必須動画」の再生やセッション自動更新、障害時の Discord 通知機能を備え、実運用を続けています。

---

## 開発背景・課題意識

Discord 上で友人コミュニティとニコニコ動画のコンテンツを楽しむ際、一般的な音楽 Bot では以下の課題がありました：

1. **ログイン必須動画・会員限定コンテンツの再生不可**: 認証セッションがないリクエストは拒否される。
2. **仕様変更（Domand移行）への追従不足**: 従来の HTTP 直リンクや旧 DMC 配信仕様が廃止され、HLS 暗号化/セッション制御への対応が必要。
3. **Cookie 期限切れによる運用中断**: 手動で Cookie を取得し直す手間が発生する。

これらの課題を解決し、**Discord 内でシームレスかつ安定してニコニコ動画を同時視聴できる体験** を実現するために開発しました。

---

## 技術スタック & アーキテクチャ

| カテゴリ | 採用技術 |
| :--- | :--- |
| **言語 / ランタイム** | TypeScript (ESModules), Bun 1.3 |
| **Discord API** | `discord.js` v14 (Slash Commands, Gateway Intents) |
| **音声処理** | `@discordjs/voice`, `ffmpeg` (リアルタイム WebM/Opus トランスコーディング) |
| **音声コーデック / 暗号化** | `@discordjs/opus`, `libsodium-wrappers`, `opusscript` |
| **ブラウザ自動化 / 認証** | `puppeteer` v21, Custom Cookie Manager (Netscape format) |
| **コンテナ・インフラ** | Docker, Docker Compose (UDP ポートレンジ制御: 50000-50100) |

---

## コア技術と実装の工夫点

### 1. ニコニコ動画 (Domand / HLS) 配信システムの解析とリアルタイム変換

ニコニコ動画の最新配信システムである **Domand 形式** に対応するため、以下のフローを独自実装しています (`NicoNicoProvider.ts`)：

1. **HTML/メタデータ解析**:
   - 視聴ページ HTML の `server-response` メタタグ内 JSON をデコードし、トラッキング ID (`watchTrackId`) やアクセスキー (`accessRightKey`) を抽出。
2. **帯域最適化と最高音質の自動選択**:
   - 利用可能な音声ストリーム (`domand.audios`) から **最高ビットレートの音声** を自動選択。
   - ストリーム要求に必要な映像ストリーム (`domand.videos`) は **最低画質の映像** を選択して通信帯域を最小化。
3. **HLS API 認証 & セッションリクエスト**:
   - ニコニコ公式 API (`nvapi.nicovideo.jp/v1/watch/.../access-rights/hls`) へ POST リクエストを送り、HLS プレイリスト URL (`contentUrl`) と動的セッション Cookie を取得。
4. **FFmpeg によるリアルタイム・ストリーミングパイプ**:
   - 取得した Cookie を `ffmpeg` の HTTP ヘッダーに動的注入し、HLS (`.m3u8`) から WebM / Opus 形式へのトランスコーディング処理を `pipe:1` (標準出力) 経由で `@discordjs/voice` の `AudioPlayer` へストリーミング供給。
   - ネットワーク寸断対策として `-reconnect 1 -reconnect_streamed 1 -reconnect_delay_max 30` パラメータを設定し、高い再生安定性を維持。

### 2. 自動ログイン & Cookie 自動更新・障害通知メカニズム

ログイン必須動画の再生と長期間の無人運用を両立するため、自動セッション維持機構を構築しています：

- **Netscape Cookie フォーマット管理 (`CookieManager.ts`)**:
  - ドメイン単位での Cookie 保存・適用ロジックを自作し、安全な HTTP リクエストヘッダー生成を実現。
- **定期有効期限監視 & 自動再ログイン (`CookieRefreshManager.ts` / `CookieAuthHelper.ts`)**:
  - `NicoSessionChecker` により定期的にセッション状態を確認し、Cookie の有効期限切れ（残り3日未満）や無効化を検知。
  - ニコニコ認証 API または `Puppeteer` ヘッドレスブラウザにより自動でバックグラウンドログインを実行し、新しい `user_session` Cookie を取得・保存。
- **Discord 障害通知システム (`CookieErrorNotifier.ts`)**:
  - 万が一自動更新に失敗した場合、Discord のログ用チャンネル（`#bot-logs` 等）へ自動で警告メッセージを送信し、管理者へ即座に通知。

### 3. オブジェクト指向なマルチソース・プレイヤー設計

将来的な機能拡張を見越し、プロバイダーパターンを採用した堅牢なディレクトリ構造をとっています：

- **`AudioProvider` 抽象クラス**: `NicoNicoProvider` のほか、直接 URL 再生 (`UrlProvider`) やローカルファイル再生 (`FileProvider`) を統一インタフェースで操作。
- **`PlayerService` / `QueueManager` / `ConnectionManager`**: サーバー（Guild）ごとの接続状態管理、再生キュー（Next / Previous / Skip / AutoPlay）、エラー時の安全なリソース破棄（Stop / Clean-up）を分離・カプセル化。

---

## インフラ & Docker 運用

- **Bun ベースの軽量 Docker コンテナ**:
  - `oven/bun:1.3.0` をベースイメージに採用。ビルドツール、FFmpeg、libsodium、Puppeteer 実行用 Linux ライブラリを最適化してインストール。
- **Discord 音声用 UDP ポートマッピング**:
  - `docker-compose.yml` にて Discord voice 通信用 UDP ポート (`50000-50100:50000-50100/udp`) を適切に開放し、コンテナ環境での低遅延音声伝送を実現。
- **`restart: unless-stopped`** による常時稼働設計。

---

## 現在の運用状況

自身の Discord サーバーで常時稼働中であり、友人コミュニティ内での日常的な動画・音楽共有ツールとして活用されています。単なるプロトタイプにとどまらず、API 仕様変更やエラー時のセッション復旧など、実運用における保守性・可用性を考慮した本格的なプロダクトとして運用しています。