# Azure コスト比較レポート

本レポートでは、**Upstream (オリジナルの [langfuse/langfuse-terraform-azure](https://github.com/langfuse/langfuse-terraform-azure) リポジトリ)** のデフォルト構成と、本リポジトリ (Container Apps) の最適化構成を比較し、それぞれのコスト構造を詳細に分析します。

> [!NOTE]
> **前提条件**: すべてのコスト試算は **東日本リージョン (Japan East / Tokyo)** の価格（2025年11月時点）に基づいています。

## 1. 構成別コスト比較（3パターン）

| 項目 | ① Upstream (AKS) デフォルト | ② AKS 最小構成 (Min) | ③ Container Apps 最小構成 (Min) |
| :--- | :--- | :--- | :--- |
| **月額概算** | **~$4,450** (約67万円) | **~$313** (約4.7万円) | **~$351** (約5.3万円) |
| **特徴** | ハイスペック・DDoS保護あり | 徹底的なコスト削減 (Bシリーズ) | サーバーレス・運用管理レス |
| **Compute** | Nodes: D8s_v6 x 2 | Node: B2s x 1 | Web: 0, Worker/CH: Min |
| **Ingress** | App Gateway v2 | App Gateway v2 | App Gateway v2 |
| **DB / Redis** | HA構成 / Standard | 最小構成 / Basic | 最小構成 / Basic |
| **DDoS保護** | **有効 (+$3,000)** | 無効 | 無効 |

---

## 2. 詳細コスト内訳（何にいくらかかるのか）

各構成におけるリソースごとのコスト内訳です。

### ① Upstream (AKS) デフォルト構成
**合計: ~$4,450 / 月**
*   **DDoS Protection**: **$3,000** (支配的コスト)
*   **Compute (AKS Nodes)**: **$1,200** (Standard_D8s_v6 x 2台)
*   **Database (PostgreSQL)**: **$280** (General Purpose, HA有効)
*   **Ingress (App Gateway)**: **$250** (Standard v2)
*   **Redis**: **$16** (Basic C1)

### ② AKS 最小構成 (Min)
**合計: ~$313 / 月**
*   **Ingress (App Gateway)**: **$250.00** (固定費・削減不可)
*   **Compute (AKS Node)**: **$30.37** (Standard_B2s x 1台)
*   **Redis**: **$15.00** (Basic C0)
*   **Database (PostgreSQL)**: **$10.00** (B_Standard_B1ms)
*   **Storage**: **$2.50** (Standard File Share)
*   **その他**: **$5.00** (Disk, IP等)

### ③ Container Apps 最小構成 (Min) - 本リポジトリ推奨
**合計: ~$351 / 月**
*   **Ingress (App Gateway)**: **$250.00** (固定費・削減不可)
*   **Compute (Container Apps)**: **$55.00**
    *   Web: $0 (Scale-to-Zero)
    *   Worker: ~$25 (0.5 vCPU 常時起動)
    *   ClickHouse: ~$30 (1.0 vCPU 常時起動)
*   **Storage (ClickHouse)**: **$16.20** (Premium NFS 100GB - 必須)
*   **Redis**: **$15.00** (Basic C0)
*   **Database (PostgreSQL)**: **$10.00** (B_Standard_B1ms)
*   **その他**: **$5.00** (Log Analytics等)

---

## 3. さらなるコスト削減ガイド (ACA版)

Container Apps 版のコスト (~$351) を、運用の手間を増やさずに下げるための設定例です。

### 📉 A. ClickHouse リソースの縮小
開発環境に限り、ClickHouse の CPU/メモリを削減します。
*   **設定**: `cpu = 0.5`, `memory = "1Gi"`
*   **削減額**: -$20 〜 -$30 / 月

### 🛑 B. 不要リソースの削除
*   **Private Endpoint**: 開発環境でVNet内閉域が不要な場合、Public Access + Firewall に変更 (-$2)
*   **Log Analytics**: 保持期間を 30日 → 7日に短縮 (微減)

### 最適化後の到達可能コスト

| 構成 | App Gateway | Compute/DB/Storage | 合計 |
| :--- | :--- | :--- | :--- |
| **標準 (ACA Min)** | $250 | ~$101 | **~$351** |
| **最適化後** | $250 | ~$70 | **~$320** |

---

## 4. サマリー

*   **Upstreamデフォルト**は月額 $4,000 超と非常に高額なため、そのまま利用するのは非推奨です。
    *   **高額な理由**: DDoS Protection ($3,000) が有効であること、およびハイスペックなVM (D8s_v6) とDBを使用しているためです。
*   **本リポジトリの最適化**:
    *   DDoS Protection を無効化 (-$3,000)
    *   開発環境向けに Compute/DB の SKU を最小化 (-$1,000以上)
*   **最小構成**で比較すると、インフラコスト単体では **AKS ($313)** が最安です。
*   **Container Apps ($351)** は月額 +$38 程度高くなりますが、Kubernetes の運用管理（アップグレード、ノード管理）が不要になるメリットがあります。
*   本リポジトリでは、この「運用コスト削減」を重視し、**Container Apps** を採用しています。
