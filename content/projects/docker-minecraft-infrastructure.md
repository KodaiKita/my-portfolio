---
title: Docker-based Minecraft Server Infrastructure
tags:
  - docker
  - docker-compose
  - minecraft
  - waterfall
  - papermc
  - rcon
  - devops
---

## 概要

Docker コンテナ技術を活用し、Minecraft サーバーの可搬性向上・マルチバージョン対応・複数サーバーネットワーク（プロキシ + バックエンド）を一元管理するためのインフラ構築構想プロジェクトです。  
単一サーバーの動的 Java 切替環境 (`simple-docker-minecraft-server`) から、Waterfall プロキシを介したマルチサーバーネットワーク (`docker_minecraft_servers_for_gymk_ubuntu`) への発展的なアーキテクチャ設計・自動化運用ツール群の検証を行いました。  
※本プロジェクトは短期間の実運用を経た、コンテナ基盤・ネットワークセキュリティの構想・検証をメインとした設計プロジェクトです。

---

## 開発背景・課題意識

従来の手動構築による Minecraft サーバー運用には、インフラ・保守面で以下の課題が存在していました：

1. **Java バージョンの依存関係問題**: Minecraft バージョン（1.16.5 以下 〜 1.20+）ごとに必要な Java バージョン（Java 8 / 16 / 17 / 21）が異なり、ホスト環境での共存・切り替えが煩雑。
2. **マルチサーバー管理の複雑化**: ロビー・サバイバル・PVP など用途別サーバーを個別に起動・管理する際、ポート管理や内部ネットワークセキュリティ（直接続のバイパスリスク）の担保が困難。
3. **データ保存・運用の手動化**: サーバー間での一括アナウンスや自動ワールド保存（RCON）などの運用保守が手動コマンド依存になりがち。

これらの課題に対し、**Docker による環境分離・安全なプロキシ通信・RCON 自動化スクリプトによる省力運用インフラ** の構想・構築を行いました。

---

## 技術スタック & アーキテクチャ

| カテゴリ | 採用技術 |
| :--- | :--- |
| **コンテナ / オーケストレーション** | Docker, Docker Compose, Multi-stage Build |
| **プロキシ / リバースプロキシ** | Waterfall (BungeeCord 拡張) / Velocity |
| **サーバーソフトウェア** | PaperMC (1.8.8 〜 1.20+ Latest), Vanilla Server |
| **転送・認証セキュリティ** | BungeeGuard (トークン認証), Modern Forwarding (PaperMC 秘密鍵認証) |
| **プロトコル / リモート管理** | RCON Protocol (Remote Console), PowerShell (`rcon.ps1`) |
| **ホスト環境 / ネットワーク** | Ubuntu Server / WSL2, Docker Bridge Network (`minecraft-net`) |

---

## コア技術と設計・構想の工夫点

### 1. バージョン連動型 Java 動的選択 ＆ 汎用コンテナ設計 (`simple-docker-minecraft-server`)

単一サーバー構築における基本基盤として、Minecraft バージョンに応じた依存環境の隠蔽と自動化を検証しました：

- **Java バージョン自動切替マッピング**:
  - `MC_VERSION`（1.16.5 / 1.17 / 1.18〜1.20.4 / 1.20.5+）に応じ、適正な Java ランタイム（Java 8 / 16 / 17 / 21）をコンテナビルド時に選択。
- **`entrypoint.sh` による起動自動化**:
  - EULA の自動承認 (`eula.txt`)、`server.properties` 内の RCON 設定（ポート・パスワード）・メモリ割り当て (`MEMORY=2G` 等) の動的注入。
- **データ領域の完全永続化**:
  - `server/` ディレクトリをホストへマウントし、ワールドデータ・ログ・プラグイン設定の保全とポータビリティを確保。

### 2. Waterfall プロキシを介したマルチサーバーネットワーク構想 (`docker_minecraft_servers_for_gymk_ubuntu`)

複数サーバー（Lobby, Survival, PVP）を統合・接続するためのネットワークアーキテクチャ設計を行いました：

- **トラフィックの一元受給と内部ネットワーク分離**:
  - 外部アクセス（Port `25565`）はフロントエンドの **Waterfall プロキシ** のみ公開。
  - バックエンド（`lobby:25565`, `survival:25565`, `pvp:25565`）は Docker ブリッジネットワーク (`minecraft-net`) 内に閉塞し、外部からの直接接続を遮断。
- **レガシー ＆ モダン混在環境での転送認証セキュリティ**:
  - **1.8.8 レガシーサーバー (Lobby / PVP)**: `BungeeGuard` プラグインと共通トークン認証を導入し、プロキシを回避した不正接続や IP 偽装を防御。
  - **Latest サーバー (Survival)**: PaperMC の `Modern Forwarding` (Forwarding Secret 鍵認証) を適用し、プレイヤーデータ（UUID / IP）を安全に暗号化転送。

### 3. PowerShell / RCON による一括運用保守スクリプト (`rcon.ps1`)

コンテナ外部から複数サーバーを遠隔制御するための運用の仕組みを実装・構想しました：

- **マルチターゲット RCON スクリプト**:
  - 個別サーバー (`.\rcon.ps1 lobby say "Welcome!"`) へのコマンド投入に加え、`all` ターゲット指定により全バックエンドサーバーへの一括保存 (`.\rcon.ps1 all save-all`) やメンテナンス通知を一括配信。
- **コンテナ状態に依存しない遠隔操作**:
  - コンテナ内にログインすることなく、ホスト OS や外部スクリプトから自動バックアップ・定期メンテナンスの自動化が可能な構成。

---

## ネットワーク & コンテナ構成図

```mermaid
graph TD
    Client["Minecraft Client (Port 25565)"] -->|Public Access| Proxy["Waterfall Proxy Container\n(Modern / BungeeGuard Forwarding)"]

    subgraph Internal Docker Network [minecraft-net]
        Proxy -->|Internal Route| Lobby["Lobby Server (Paper 1.8.8)\n+ BungeeGuard"]
        Proxy -->|Internal Route| Survival["Survival Server (Paper Latest)\n+ Modern Forwarding"]
        Proxy -->|Internal Route| PVP["PVP Server (Paper 1.8.8)\n+ BungeeGuard"]
    end

    Admin["Administrator / Script"] -->|RCON (25575)| Lobby
    Admin -->|RCON (25575)| Survival
    Admin -->|RCON (25575)| PVP
```

---

## 運用状況・今後の展望

本プロジェクトは短期間の実運用およびローカル/Ubuntu 環境での検証を経て、**「コンテナインフラによるマルチサーバー管理とセキュリティ自動化」の技術的ブループリント** として整理されました。  
実運用の期間は短かったものの、Docker コンテナによる抽象化、プロキシによるネットワーク分離、RCON を用いた自動保守スクリプトの組み合わせは、規模の異なるゲームサーバー基盤のテンプレートとして応用可能な設計知見となっています。
