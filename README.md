# zabbix-monitoring

Docker Composeで構築するZabbix監視スタックのハンズオンプロジェクトです。
Prometheus/Grafanaで構築した監視スタック([monitoring-portfolio](https://github.com/syo-go5530/monitoring-portfolio))と比較しながら、
エージェント型監視ツールであるZabbixの基本(ホスト登録・トリガー・アラート)を実践します。

## 構成

```
ブラウザ (http://localhost:8080)
    │
    ▼
zabbix-web (Web UI)
    │
    ▼
zabbix-server (収集・トリガー評価・アラート発行)
    │            ▲
    ▼            │ 監視データ送信
mysql-server   zabbix-agent (このDockerホストを監視)
(設定・履歴を保存)

sample-nginx (監視対象のサンプルWebサーバー, http://localhost:8081)
```

| コンテナ | 役割 |
|---|---|
| `mysql-server` | Zabbixの設定・監視履歴を保存するDB |
| `zabbix-server` | 監視データの収集、しきい値(トリガー)判定、アラート発行 |
| `zabbix-web` | ブラウザで見る管理画面 |
| `zabbix-agent` | 監視対象にインストールし、データをzabbix-serverに送る |
| `sample-nginx` | 監視対象のサンプル(あとでZabbix画面からホスト登録する) |

## セットアップ手順

### 1. 起動する

```bash
cd zabbix-monitoring
docker compose up -d
```

初回起動はDBの初期化が入るため、1〜2分ほどかかります。

### 2. 起動状況を確認する

```bash
docker compose ps
```

すべてのコンテナが `Up` になっていればOKです。`zabbix-web`だけ起動に少し時間がかかることがあります。

### 3. ブラウザでアクセスする

`http://localhost:8080` を開きます。

- 初期ログイン情報
  - ユーザー名: `Admin`
  - パスワード: `zabbix`

ログインしたら、**必ずパスワードを変更してください**(右上のユーザーアイコン → Profile → Change password)。

### 4. Agentが送ってきているホストを確認する

Zabbix画面の `Data collection` → `Hosts` を開きます。`docker-host-01` というホストが自動的に登録されているはずです(このリポジトリの`zabbix-agent`コンテナが監視データを送ってきているホストです)。

### 5. sample-nginxを監視対象に追加する(発展)

`sample-nginx`コンテナ自体にはまだZabbix Agentが入っていないため、監視するには以下のどちらかが必要です。

- `sample-nginx`にzabbix-agentをサイドカーとして追加する
- Zabbixの「HTTPエージェント」機能で、URL監視(生死確認)を設定する(エージェント不要)

まずは後者(URL監視)から試すのがおすすめです。`Data collection` → `Hosts` → `Create host` で新しいホストを作り、Itemとして `http://sample-nginx:80` へのHTTPチェックを追加します。

## 停止・削除

```bash
# 停止(データは残る)
docker compose down

# データも含めて完全に削除
docker compose down -v
```

## Prometheus/Grafana構成との違い(比較メモ)

| | Zabbix | Prometheus + Grafana |
|---|---|---|
| データ収集方式 | Agentからのプッシュ(push) | Prometheusがターゲットを取得しに行く(pull) |
| 得意な領域 | ホスト管理・障害検知・アラート運用 | メトリクスの可視化・柔軟なクエリ(PromQL) |
| 構成要素の数 | Server/Web/DB/Agentが密結合 | 各コンポーネントが疎結合(組み合わせ自由) |
| 現場での位置づけ | オンプレ・レガシー環境の監視で採用例が多い | クラウドネイティブ・コンテナ環境で主流 |

## 学んだこと(記入用)

- (ここに、実際に構築・検証して気づいたことを追記していく)
