# Azure コスト比較レポート

本レポートでは、Upstream (AKS) のデフォルト構成と、本リポジトリ (Container Apps) の最適化構成を比較し、さらに開発環境における最小コスト構成を分析します。

## 1. サマリー：劇的なコスト削減

Upstream (AKS) をデフォルトでデプロイした場合と、本リポジトリの推奨構成（開発用）の比較です。

| 構成 | 月額概算 (Japan East) | 備考 |
| :--- | :--- | :--- |
| **Upstream (AKS) デフォルト** | **~$4,450 / 月** (約67万円) | DDoS保護有効、ハイスペックVM (D8s_v6 x2) |
| **本リポジトリ (ACA) 開発版** | **~$351 / 月** (約5.3万円) | DDoS無効、最小リソース、App Gateway込み |
| **削減率** | **約 92% 削減** | |

> [!IMPORTANT]
> **DDoS Protection の警告**: Upstream のデフォルト設定 (`use_ddos_protection = true`) は、それだけで **月額約 $3,000** かかります。本リポジトリではデフォルトで `false` に設定しています。

---

## 2. 詳細比較：Upstreamデフォルト vs 本リポジトリ

なぜこれほどの差が出るのか、主要リソースごとの内訳です。

| リソースカテゴリ | Upstream (AKS) デフォルト | 本リポジトリ (ACA) 開発版 | コストへの影響 |
| :--- | :--- | :--- | :--- |
| **DDoS 保護** | **有効 (~$3,000)** | **無効 ($0)** | 最大の削減要因です。 |
| **コンピューティング** | ノード: 2x `Standard_D8s_v6`<br>(8 vCPU, 32GB RAM)<br>**コスト: ~$1,200/月** | Web: Scale-to-Zero<br>Worker/CH: 最小vCPU (常時起動)<br>**コスト: ~$55/月** | AKSのデフォルトノードは開発用には過剰です。ACAは必要な分だけリソースを割り当てます。 |
| **データベース** | PostgreSQL (HA有効)<br>SKU: `GP_Standard_D2s_v3`<br>**コスト: ~$280/月** | PostgreSQL (HA無効)<br>SKU: `B_Standard_B1ms`<br>**コスト: ~$10/月** | 開発環境向けにSKUを最適化しました。 |
| **イングレス** | Application Gateway (v2)<br>**コスト: ~$250/月** | Application Gateway (v2)<br>**コスト: ~$250/月** | 外部公開のため、両方の構成で固定費としてかかります。 |

---

## 3. 開発環境における「最小構成」の比較

「もし AKS も限界までスペックを落としたら、ACA とどちらが安いのか？」
HA構成や冗長化を考慮しない、**Minimum Viable Configuration** 同士の比較です。

### 前提条件
*   **共通**: App Gateway ($250), PostgreSQL B1ms ($10), Redis Basic ($15)
*   **AKS (Min)**: `Standard_B2s` (2 vCPU, 4GB) x 1 ノード
*   **ACA (Min)**: Worker (0.5 vCPU), ClickHouse (1.0 vCPU) 常時起動 + Premium NFS

### コスト内訳表

| リソース | AKS版 (Min) | Container Apps版 (Min) | 差額 | 備考 |
| :--- | :--- | :--- | :--- | :--- |
| **Ingress** | **$250.00** | **$250.00** | ±0 | 固定費 (App Gateway Standard v2) |
| **Compute** | **$30.37**<br>(B2s Node x1) | **$55.00**<br>(Worker+CH 従量課金) | +$24.63 | ACAの常時起動はVM借り切りより割高 |
| **Storage (CH)** | **$2.50**<br>(Standard File) | **$16.20**<br>(Premium NFS 100GB) | +$13.70 | ACAでのClickHouseにはPremium NFSが必須 |
| **Database** | $10.00 | $10.00 | ±0 | PostgreSQL B1ms |
| **Redis** | $15.00 | $15.00 | ±0 | Basic C0 |
| **その他** | $5.00 | $5.00 | ±0 | Disk, Log Analytics |
| **合計** | **~$313 /月** | **~$351 /月** | **ACAが +$38** | |

### 結論
純粋なインフラコストでは **AKS (Min) の方が月額 $38 安い** です。
しかし、本リポジトリでは **「月額 $38 で Kubernetes の運用管理（アップグレード、ノード管理）を不要にする」** という判断で Container Apps を採用しています。

---

## 4. さらなるコスト削減ガイド (ACA版)

Container Apps 版のコスト (~$351) をさらに下げるためのオプションです。

### 🚀 A. Redis を Dragonfly に変更 (推奨)
Azure Cache for Redis ($15~$40) を、Container Apps 上の Dragonfly コンテナ ($5) に置き換えます。
*   **削減額**: -$10 〜 -$35 / 月
*   **適用**: `dragonfly.tf` を作成し、Redis リソースを削除。

### 📉 B. ClickHouse リソースの縮小
開発環境に限り、ClickHouse の CPU/メモリを削減します。
*   **設定**: `cpu = 0.5`, `memory = "1Gi"`
*   **削減額**: -$20 〜 -$30 / 月

### 🛑 C. 不要リソースの削除
*   **Private Endpoint**: Public Access + Firewall に変更 (-$2)
*   **Log Analytics**: 保持期間を 7日に短縮 (微減)

### 最適化後の到達可能コスト

| 構成 | App Gateway | Compute/DB/Storage | 合計 |
| :--- | :--- | :--- | :--- |
| **標準 (ACA Min)** | $250 | ~$101 | **~$351** |
| **最適化後 (Dragonfly等)** | $250 | ~$65 | **~$315** |
| **(参考) AppGWなし** | $0 | ~$65 | **~$65** |

> [!TIP]
> **Application Gateway について**: 開発環境でセキュリティ（WAF）や固定IPが不要であれば、Application Gateway を削除して Container Apps のパブリックエンドポイントを使用することで、**一気に $250 削減** できます（合計 ~$65〜$100）。ただし、セキュリティリスクについては十分検討してください。
