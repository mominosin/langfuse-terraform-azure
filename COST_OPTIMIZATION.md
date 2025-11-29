# コスト最適化ガイド

このドキュメントでは、Langfuse on Azure Container Appsのコスト削減方法を説明します。

## アーキテクチャ構成と要件

### Langfuse 共通要件（AKS / Container Apps 共通）

Langfuseでは以下のコンポーネントが必要です：

| コンポーネント | 役割 | 要件 |
|--------------|------|------|
| **Web** | API + UI | Langfuseのメインアプリケーション |
| **Worker** | 非同期処理 | Redis Bullキューからイベントを取得し、ClickHouseに書き込み |
| **ClickHouse** | 分析DB | トレースデータの保存・クエリ |
| **PostgreSQL** | メタデータDB | ユーザー、プロジェクト、設定等 |
| **Redis** | キュー + キャッシュ | **非クラスタモード必須**（Bull キューのCROSSLOT制約） |
| **Blob Storage** | ファイル保存 | イベントデータ、メディアファイル等 |

#### Redis非クラスタ要件について

LangfuseはBullキューを使用しており、Redisクラスタモードでは`CROSSSLOT`エラーが発生します。
これはAzure/AKS/Container Apps共通の制約です：

- ❌ Azure Managed Redis (OSSCluster) → CROSSSLOT エラー
- ✅ Azure Cache for Redis Standard (非クラスタ) → 動作OK
- ✅ Dragonfly / Valkey (非クラスタ) → 動作OK

---

### Container Apps版 固有の要件

| 要件 | 理由 |
|-----|------|
| **Application Gateway** | 内部Container Apps環境は直接外部公開不可。Application Gateway経由で公開が必要 |
| **Premium NFS FileStorage** | Container AppsでNFSマウントするにはPremium FileStorageが必須 |
| **専用Container App (ClickHouse)** | サイドカーパターンが使えないため、専用Container Appとして分離 |

### AKS版との構成比較

| 項目 | AKS版 | Container Apps版 |
|-----|-------|-----------------|
| **外部公開** | AGIC + Application Gateway | Application Gateway (別途必要) |
| **ClickHouse配置** | サイドカー or 専用Pod | 専用Container App |
| **Worker配置** | 同一Pod or 専用Pod | 専用Container App |
| **Redis** | 非クラスタ必須（共通） | 非クラスタ必須（共通） |
| **ClickHouse Storage** | 通常File Share | Premium NFS FileStorage |
| **運用複雑度** | 高（Kubernetes知識必要） | 低（マネージド） |

---

## (参考) Upstreamのデフォルト構成コスト

もし Upstream (AKS) リポジトリを **デフォルト設定**（`terraform.tfvars` による最適化なし）でデプロイした場合、コストは非常に高額になります。これが「コスト最適化」の出発点です。

| リソース | デフォルト設定 | 月額概算コスト |
| :--- | :--- | :--- |
| **DDoS Protection** | 有効 (`true`) | **~$3,000** |
| **AKS Nodes** | Standard_D8s_v6 (x2) | **~$800** |
| **PostgreSQL** | General Purpose (HA) | ~$280 |
| **合計** | | **~$4,450+** |

> [!CAUTION]
> **注意**: Upstreamのデフォルト設定のままデプロイすると、月額 70万円近い請求が発生する可能性があります。本リポジトリでは、DDoS保護の無効化やSKUの適正化により、これを大幅に削減しています。

---

## コスト比較（AKS版 vs Container Apps版）

ここでは、現実的な「最小構成（Minimum Viable）」同士で比較します。
Upstream (AKS) も最適化（DDoS無効化、SKUダウンサイジング）を行った場合と、本リポジトリ（Container Apps）の比較です。

### 開発環境

| リソース | AKS版 | Container Apps版 | 差額 | 備考 |
|---------|-------|-----------------|------|------|
| **コンピュート基盤** | $30-40 (AKS 1node B2s) | - | - | Container Appsは個別課金 |
| **Application Gateway** | $20-30 (AGIC共用) | $20-30 | ±0 | 両方必要 |
| **Web** | (AKS内) | $5-20 | - | CPU 0.5-1.0 |
| **Worker** | (AKS内) | $10-30 | - | CPU 1.0, 常時起動 |
| **ClickHouse** | (AKS内) | $30-60 | - | CPU 2.0, 常時起動 |
| **PostgreSQL** | $10-30 | $10-30 | ±0 | B_Standard_B1ms |
| **Redis** | $40-60 | $40-60 | ±0 | Standard C1（非クラスタ必須） |
| **Storage (Blob)** | $2-3 | $2-3 | ±0 | LRS |
| **Storage (ClickHouse)** | $2-5 (通常File Share) | $15-25 (Premium NFS) | **+$10-20** | Container Apps要件 |
| **Log Analytics** | $5 | $5 | ±0 | 30日保持 |
| **Private Endpoints** | $2 | $2 | ±0 | PostgreSQL, Redis |
| **合計** | **$100-145** | **$139-265** | **+$39-120** | |

### 本番環境

| リソース | AKS版 | Container Apps版 | 差額 | 備考 |
|---------|-------|-----------------|------|------|
| **コンピュート基盤** | $200-400 (AKS 2-3node D4s) | - | - | Container Appsは個別課金 |
| **Application Gateway** | $40-80 | $40-80 | ±0 | 両方必要 |
| **Web** | (AKS内) | $50-100 | - | CPU 2.0, min 2 |
| **Worker** | (AKS内) | $20-50 | - | CPU 2.0, min 2 |
| **ClickHouse** | (AKS内) | $60-120 | - | CPU 4.0 |
| **PostgreSQL (HA)** | $100-300 | $100-300 | ±0 | GP_Standard_D4s_v3 |
| **Redis** | $80-150 | $80-150 | ±0 | Standard C2-C3 |
| **Storage (Blob)** | $20 | $20 | ±0 | GRS |
| **Storage (ClickHouse)** | $5-10 (通常File Share) | $30-50 (Premium NFS) | **+$20-40** | Container Apps要件 |
| **Log Analytics** | $20-50 | $20-50 | ±0 | 大量ログ |
| **Private Endpoints** | $2-4 | $2-4 | ±0 | |
| **合計** | **$430-960** | **$433-935** | **ほぼ同等** | |

### 差額の要因

| 要因 | 差額 | 説明 |
|-----|------|------|
| **Premium NFS** | +$10-40/月 | Container AppsでNFSマウントに必須 |
| **コンピュート効率** | ±0〜+$30 | AKSはノード共有で効率的、Container Appsは個別課金 |

**共通コスト（差額なし）**:
- Application Gateway（両方必要）
- Redis Standard（非クラスタ必須はLangfuse要件）
- PostgreSQL、Storage (Blob)、Log Analytics

### まとめ

| 環境 | AKS版 | Container Apps版 | 主な差額要因 |
|-----|-------|-----------------|------------|
| 開発 | $100-145 | $139-265 | Premium NFS (+$10-20)、コンピュート効率差 |
| 本番 | $430-960 | $433-935 | Premium NFS (+$20-40)、ほぼ相殺 |

**Container Apps版のメリット**（コスト以外）:
- Kubernetes知識不要
- デプロイ時間短縮（10-18分 vs 20-30分）
- Helmチャート管理不要
- 自動スケーリング設定が簡単

---

## さらなるコスト削減案

現在の構成からさらにコストを削減したい場合の選択肢を示します。

### 🥇 優先度: 高（大きなコスト削減）

#### 1. Redisの代替案（月額 $35-55削減）

**現状**: Azure Cache for Redis Standard C1 = $40-60/月

**代替案A: Dragonfly on Container Apps**

Dragonflyは高性能でRedis互換のメモリストア（非クラスタモード対応）

新規ファイル `dragonfly.tf`:
```hcl
resource "azurerm_container_app" "dragonfly" {
  name                         = "dragonfly"
  container_app_environment_id = azapi_resource.container_app_environment.id
  resource_group_name          = azurerm_resource_group.this.name
  revision_mode                = "Single"

  template {
    container {
      name   = "dragonfly"
      image  = "docker.dragonflydb.io/dragonflydb/dragonfly:latest"
      cpu    = 0.5
      memory = "1Gi"

      env {
        name  = "DFLY_requirepass"
        secret_name = "dragonfly-password"
      }
    }

    min_replicas = 1
    max_replicas = 1
  }

  secret {
    name  = "dragonfly-password"
    value = random_password.dragonfly_password.result
  }

  ingress {
    external_enabled = false
    target_port      = 6379
    transport        = "tcp"
  }
}

resource "random_password" "dragonfly_password" {
  length  = 32
  special = false
}
```

**コスト**: Container Apps料金のみ（約 $5-10/月）

**注意**: LangfuseはBullキューを使用するため、CROSSSLOT対応が必要。Dragonflyは非クラスタモードで動作するため対応可能。

**代替案B: Valkey on Container Apps (Redis fork)**

Redis 7.2.4のフォーク、完全互換、非クラスタモード対応

```hcl
resource "azurerm_container_app" "valkey" {
  # 同様の構成
  template {
    container {
      image = "valkey/valkey:7.2"
      cpu   = 0.5
      memory = "1Gi"
      # ...
    }
    min_replicas = 1
    max_replicas = 1
  }
}
```

**コスト**: Container Apps料金のみ（約 $5-10/月）

**代替案C: Azure Cache for Redis Basic**

⚠️ **非推奨**: Basic SKUは非クラスタですが、SLAなし・永続性なしのため本番非推奨

**推奨**: Dragonfly または Valkey on Container Apps（月額 $35-55削減）

---

#### 2. ClickHouseリソース削減（月額 $10-30削減）

**現状**: ClickHouse Container App (CPU 2.0, Memory 4Gi = $30-60/月)

**代替案**: 開発環境ではリソースを削減

```hcl
# clickhouse.tf を編集
resources = {
  cpu    = 1.0   # 2.0 から削減
  memory = "2Gi" # 4Gi から削減
}
```

**影響**:
- ✅ 月額 $10-30 削減
- ⚠️ 大量データ処理時のパフォーマンス低下
- ⚠️ 開発/テスト環境のみ推奨

---

#### 3. PostgreSQL/Redis Private Endpointの削除（月額 $2削減）

**現状**: 2つのPrivate Endpoint（PostgreSQL, Redis）

**代替案**: 開発環境ではPublicアクセスを許可（ファイアウォールルールで制限）

**影響**:
- ✅ 月額 $2 削減（Private Endpoint x 2）
- ⚠️ セキュリティが若干低下（本番環境では非推奨）
- ✅ ファイアウォールルールで制御可能

**推奨**: 開発環境のみ適用

---

### 🥈 優先度: 中（中程度のコスト削減）

#### 3. PostgreSQLのサーバーレス化（月額 $5-20削減）

**現状**: Flexible Server (B_Standard_B1ms = 固定料金)

**代替案**: Azure SQL Database Serverless

Langfuseが必要とするのはPostgreSQL互換DBですが、Azure SQL DatabaseのServerlessプランを検討する価値があります。ただし、LangfuseはPostgreSQL前提のため、**PostgreSQL互換性の検証が必須**です。

別の選択肢として、**Supabase**や**Neon**などの外部PostgreSQLサービス（Serverless）を使用：

**Neon (Serverless Postgres)**:
- 無料枠: 0.5GB、月間191時間
- 有料: $19/月から（Autoscaling、Branching機能付き）

**Supabase**:
- 無料枠: 500MB、2 CPUまで
- 有料: $25/月から

**実装**: Terraformの外で管理し、`DATABASE_URL`のみ指定

**推奨**: 小規模プロジェクトや開発環境では検討の価値あり

---

#### 4. Log Analyticsの保持期間短縮（月額 $2-10削減）

**現状**: 30日保持

**代替案**: 7日保持に変更

`log_analytics.tf`:
```hcl
resource "azurerm_log_analytics_workspace" "this" {
  # ...
  retention_in_days   = 7  # 30から7に変更
}
```

**影響**:
- ✅ ログ保存コストが削減
- ⚠️ 過去のログが7日間しか見られない

**推奨**: 開発環境では7日、本番環境では30-90日

---

## コスト削減シナリオ

### シナリオ1: 現在の構成（月額 $139-265）

**構成**:
- ✅ Application Gateway: Standard_v2 capacity 1
- ✅ Container Apps (Web): CPU 0.5-1.0, min 0-1 replica
- ✅ Container Apps (Worker): CPU 1.0, 常時1台
- ✅ Container Apps (ClickHouse): CPU 2.0, 常時1台
- ✅ Redis: Azure Cache for Redis Standard C1（非クラスタ）
- ✅ PostgreSQL: B_Standard_B1ms
- ✅ Storage: LRS (Blob + Premium NFS 100GB)
- ✅ Private Endpoints: PostgreSQL, Redis用 (2個)
- ✅ Log Analytics: 30日保持

**月額コスト**: $139-265

**推奨**: 開発/テスト環境向け標準構成

---

### シナリオ2: コスト最適化開発環境（月額 $75-140）

**現在の構成からの変更**:
- Redis → Dragonfly on Container Apps（-$35-55）
- ClickHouseリソース削減（CPU 1.0, 2Gi）（-$10-30）
- Private Endpoint削除（-$2）
- Log Analytics: 7日保持（-$2-5）
- Web Container Apps: min 0 replicas（-$3-10）

**月額コスト**:
| リソース | 月額概算 |
|---------|---------|
| Application Gateway | $20-30 |
| Container Apps (Web) | $2-10 |
| Container Apps (Worker) | $10-20 |
| Container Apps (ClickHouse) | $15-30 |
| Container Apps (Dragonfly) | $5-10 |
| PostgreSQL | $10-20 |
| Storage (Blob + NFS) | $17-28 |
| Log Analytics | $2-3 |
| **合計** | **$75-140** |

**トレードオフ**:
- マネージドRedisなし（Dragonfly運用）
- ClickHouseパフォーマンス低下
- Private Endpoint なし（開発環境のみ）
- 短いログ保持期間

**削減額**: 現在の構成から約 $64-125削減

---

### シナリオ3: 本番環境（月額 $433-935）

**変更内容**:
- Application Gateway: capacity 2-4（冗長化）
- Container Apps: 全て min 2 replicas
- ClickHouse: CPU 4.0, Memory 8Gi
- Redis: Azure Cache for Redis Standard C2-C3
- PostgreSQL: GP_Standard_D4s_v3 + HA
- Storage (Blob): GRS
- Storage (NFS): 200GB以上
- Log Analytics: 90日保持
- NAT Gateway（オプション）
- カスタムドメイン + SSL

**月額コスト**: $433-935

**推奨**: 本番環境向け高可用性構成

---

## さらなる削減の実装優先順位

現在の構成からさらにコストを削減する場合の推奨順序：

### すぐに実装可能（リスク低）

1. **Log Analytics保持期間短縮** - `retention_in_days = 7` （月額 -$2～5）
2. **Web Container Apps スケールtoゼロ** - `min_replicas = 0` （月額 -$3～10）
3. **ClickHouseリソース削減** - CPU 1.0, Memory 2Gi （月額 -$10～30）

### 検討すべき（中リスク）

4. **Redis代替（Dragonfly/Valkey）** - 動作検証後 （月額 -$35～55）
5. **外部PostgreSQLサービス** - Neon/Supabase等、データガバナンス要件確認後 （月額 -$5～20）

### 慎重に検討（高リスク）

6. **PostgreSQL/Redis Private Endpoint削除** - 開発環境のみ、セキュリティ要件確認後 （月額 -$2）

---

## 実装例: 現在の構成

ファイル `terraform.tfvars` (開発環境の現在の設定):

```hcl
# 基本設定
location = "japaneast"
name     = "langfuse-dev"
# domain は未設定（Application Gateway経由でHTTPアクセス）

# Container Apps - Web（開発環境向け）
container_app_cpu          = 0.5
container_app_memory       = 1
container_app_min_replicas = 0  # スケールtoゼロ
container_app_max_replicas = 3
langfuse_image_tag         = "3"

# Container Apps - Worker（常時起動）
worker_cpu          = 1.0
worker_memory       = 2
worker_min_replicas = 1  # 常時1台起動
worker_max_replicas = 1

# PostgreSQL（最小構成、HAなし）
postgres_instance_count = 1
postgres_sku_name       = "B_Standard_B1ms"
postgres_storage_mb     = 32768

# Redis（Azure Cache for Redis Standard - 非クラスタ）
redis_sku_name = "Standard"
redis_family   = "C"
redis_capacity = 1

# セキュリティ（開発環境）
use_encryption_key  = true   # 暗号化キー有効
use_ddos_protection = false  # DDoS保護なし
```

**月額コスト**: 約 $139-265

---

## モニタリングとアラート

コスト削減後も、以下のモニタリングを推奨：

1. **Azure Cost Management**
   - 日次コストレポート
   - 予算アラート設定（$150, $300等）

2. **リソース使用状況**
   - Container Apps (Web, Worker, ClickHouse) のメトリクス監視
   - PostgreSQLのCPU/メモリ使用率
   - Redis のメモリ使用率
   - Storageの使用量

3. **コマンドでコスト確認**

```bash
# 現在月のコスト
az consumption usage list \
  --start-date $(date -u -d "$(date +%Y-%m-01)" '+%Y-%m-%d') \
  --end-date $(date -u '+%Y-%m-%d') \
  --query "[].{Service:instanceName,Cost:pretaxCost}" \
  --output table

# リソースグループ別コスト
az consumption usage list \
  --start-date 2025-11-01 \
  --end-date 2025-11-30 \
  | jq -r 'group_by(.instanceLocation) | .[] | {location: .[0].instanceLocation, total: (map(.pretaxCost|tonumber) | add)}'
```

---

## まとめ

### コスト削減の選択肢

**コスト最適化開発環境**:
1. Redis → Dragonfly/Valkey on Container Apps
2. ClickHouseリソース削減
3. Log Analytics 7日保持
4. Web Container Apps スケールtoゼロ
5. Private Endpoint削除（開発環境のみ）

→ **月額 $75-140** (現在の構成から -$64～125)

**本番環境**:
1. Application Gateway冗長化
2. Container Apps min 2 replicas
3. Redis: Standard C2-C3
4. PostgreSQL: GP_Standard_D4s_v3 + HA
5. Storage: GRS、NFS 200GB以上

→ **月額 $433-935**

---

## 次のステップ

1. 要件の確認（セキュリティ、可用性、パフォーマンス）
2. 開発環境でDragonfly/Valkeyを検証
3. コストモニタリング設定（予算アラート $200）
4. 段階的に本番環境へ適用

---

**最終更新**: 2025-11-29
**対象バージョン**: Langfuse on Container Apps（Web + Worker + ClickHouse）
