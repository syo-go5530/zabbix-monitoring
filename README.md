# Zabbix監視スタック(Docker Compose)

## 概要

Docker Compose上にZabbixの監視基盤を構築し、監視項目の自作からSlack通知連携までを一人で設計・実装した記録。

以前構築した monitoring-portfolio(Prometheus/Grafana/Alertmanager構成)では、監視を収集・判定・通知・可視化という役割に分解して理解した。今回はそれらを一つの製品にまとめて提供するZabbixに取り組み、実務で広く使われているオールインワン型の監視ツールを内側から理解することを目的とした。

行ったこと:

- Docker Composeで複数コンテナ構成(Server / Web / Agent / DB)を構築
- UserParameterで監視項目(nginxの死活監視)を自作
- しきい値を設計し、トリガーを自作
- アクション設定でSlack通知を連携し、障害発生時・復旧時の両方を通知
- 構築・検証中に起きた障害(ネットワークタイムアウト、設定漏れなど)を切り分けて解決

## 構成

Docker Composeで5つのコンテナを動かしている。

- zabbix-web: ブラウザからアクセスする管理画面(ポート8888)
- zabbix-server: 監視データの収集、トリガー判定、アクション実行を行う中枢
- mysql-server: 監視設定・履歴データの保存先
- zabbix-agent: 監視対象ホスト上で動き、データをサーバーに送る
- sample-nginx: 監視対象として用意した疑似Webサービス

データの流れとしては、zabbix-agentがsample-nginxの状態を取得してzabbix-serverに送信し、serverがmysql-serverにデータを保存しつつトリガー判定を行う。トリガーが発火すると、アクション経由でSlackに通知が送られる。管理画面(zabbix-web)からは、これらの設定や履歴を確認できる。

## Prometheus構成との対比

以前のPrometheus構成と比べると、Zabbixは分解された機能を一つの製品にまとめたオールインワン型という違いがある。

収集はPrometheus構成ではNode ExporterやBlackbox Exporterが担っていたが、Zabbixではzabbix-agentが担う。判定はPrometheusからzabbix-serverへ、通知はAlertmanagerからzabbix-serverのアクション機能へ、可視化はGrafanaからzabbix-webへ、それぞれ役割が引き継がれる形になる。

Prometheus構成を先に分解して理解していたことで、Zabbixの画面の裏で何が起きているかを想像しながら学べた。たとえば「Zabbix Agentが停止している」というエラーに遭遇したとき、これはExporterが落ちているのと同じ状態だとすぐに理解でき、切り分けがスムーズに進んだ。

## 監視実装

sample-nginxの死活監視を、Zabbix標準の監視項目ではなくUserParameterで自作した。

custom-agent/userparameter_nginx.conf:

```
UserParameter=custom.nginx.up,wget -q -T 2 -O /dev/null http://sample-nginx:80/ 2>/dev/null && echo 1 || echo 0
```

nginxに2秒でアクセスし、成功すれば1、失敗すれば0を返す。Zabbixに標準で用意されていない監視項目を、コマンドを組んで追加する経験として作成した。

トリガーは、監視項目の最新値(last関数)が0(停止)であることを条件とし、重要度はHighに設定した。単純なUp/Down監視だが、どの関数を使うか、重要度をどう割り当てるかを自分で考えて設定した。瞬間的なノイズを拾いたくない監視対象であればavg関数を使うなど、監視対象の特性によって関数を使い分けるべきだと理解している。

## 通知設計

トリガーが発火した際の通知方法をアクションとして設定した。Zabbixのアクションは発生時の通知(Operations)と復旧時の通知(Recovery operations)を別々に設定する必要があり、両方を設定することで障害の発生と復旧の両方をSlackに通知する仕組みを構築した。

Slack連携はIncoming WebhookではなくBot Token方式を採用した。Slack APIアプリを作成し、必要なOAuth Scope(chat:write, channels:read, groups:read, chat:write.public)を付与した上で、Zabbix側のMedia typeにBot Tokenを設定している。

実際に届いた通知は以下の通り。

障害発生時: Problem: sample-nginxが停止している
復旧時: Resolved in 26s: sample-nginxが停止している

## トラブルシューティング

構築・検証の過程で発生した問題と、切り分け・解決の記録。

### カスタム監視項目がUnsupported item keyになる

症状: Zabbix画面上で、自作した監視項目がサポートされていないキーとして扱われる。

原因: docker-compose.ymlのzabbix-agentサービスに、UserParameter設定ファイルをコンテナ内にマウントするvolumesの指定が抜けていた。

対応: 以下を追加してコンテナを再起動した。

```yaml
zabbix-agent:
  volumes:
    - ./custom-agent/userparameter_nginx.conf:/etc/zabbix/zabbix_agentd.d/userparameter_nginx.conf:ro
```

### 監視項目の値に文字列エラーが混入する

症状: Value of type "string" is not suitable for value type "Numeric (unsigned)" というエラーが発生。

原因: wgetが失敗した際、標準エラー出力の文字列がそのまま監視項目の値に混入していた。

対応: wgetコマンドの末尾に2>/dev/nullを追加し、エラー出力を破棄するよう修正した。

### Zabbix Agent/Serverのネットワークエラーが繰り返される

症状: サーバーログにnetwork error, wait for 15 secondsが繰り返し出力される。

原因: wgetの実行に時間がかかり、Zabbix Agent/Serverのデフォルトタイムアウト(3秒)を超過していた。

対応: Agent側・Server側の両方にZBX_TIMEOUT: "10"を設定し、タイムアウトまでの許容時間を延長した。最終的にはServer側の設定が根本原因だった。

### Slack通知が送信エラーになる(zabbix_url must contain a schema)

原因: Zabbixのグローバルマクロ{$ZABBIX.URL}が未設定だった。

対応: Administration > General > Macrosでhttp://localhost:8888を設定した。

### Slack通知がmissing_scopeエラーになる

原因: Slack Bot Tokenに必要なOAuth Scopeが不足していた。

対応: channels:read, groups:read, chat:write.publicを追加し、アプリを再インストールして新しいTokenに更新した。

### 復旧通知だけSlackに届かない

症状: 障害発生時の通知(PROBLEM)は届くが、復旧時の通知(RESOLVED)が届かない。

原因: アクション設定のOperations(発生時)のみ設定しており、Recovery operations(復旧時)の設定が抜けていた。

対応: Recovery operationsに同じSlack送信先の設定を追加した。これによりPROBLEM/RESOLVED両方の通知が届くようになった。

## セットアップ手順

このリポジトリをクローンした後、以下の手順で再現できる。

1. 環境変数ファイルを用意する

```
cp .env.example .env
```

必要に応じて `ZABBIX_WEB_PORT` などの値を環境に合わせて変更する。

2. コンテナを起動する

```
docker compose up -d
```

起動後、`docker compose ps` で5つのコンテナがすべて起動していることを確認する。

3. Zabbix管理画面にログインする

ブラウザで `http://localhost:8888`(ポートは.envの値に合わせる)にアクセスし、初期アカウントでログインする。

4. 監視対象ホストを登録する

Data collection > Hosts から、`docker-host-01` というホスト名でzabbix-agentを登録する。

5. カスタム監視項目とトリガーを設定する

登録したホストに対して、UserParameterで定義した `custom.nginx.up` の監視項目を追加し、値が0になったら発火するトリガーを設定する。

6. アクションを設定する

Alerts > Actions > Trigger actions から、トリガー発火時にSlackへ通知するアクションを作成する。OperationsとRecovery operationsの両方に、同じSlack送信先の設定を追加する。

7. Slack側の準備

Slack APIで新しいアプリを作成し、Bot Tokenの方式でOAuth Scope(chat:write, channels:read, groups:read, chat:write.public)を付与する。発行したTokenをZabbixのMedia typeに設定し、通知先チャンネルにBotを参加させる。

8. 動作確認

```
docker stop zbx-sample-nginx
```

で障害を発生させ、Slackに通知が届くことを確認する。

```
docker start zbx-sample-nginx
```

で復旧させ、復旧通知が届くことを確認する。

## 学んだこと

Zabbixは、Prometheus構成で分解して理解していた「収集・判定・通知・可視化」という役割を、一つの製品の中でどう実現しているかを確認しながら学べた。設定項目はPrometheus構成より多いが、対応関係を意識することで迷わず進められた。

トラブルシューティングの面では、エラーメッセージを読んで層ごとに切り分ける(アプリケーション層の設定か、ネットワーク層のタイムアウトか、権限の問題か)という進め方が、Prometheus構成のときと同様に有効だった。特にRecovery operationsの設定漏れは、Zabbixのアクションが発生時と復旧時で別々の設定を持つという仕様を理解していなければ気づけなかった点で、実際に手を動かしたからこそ得られた知識だった。

## 今後の拡張予定

現時点では、死活監視とSlack通知の基本的な流れを構築した段階にある。今後は以下を実施する予定。

- エスカレーション設定(一定時間解決しない場合に再通知や別の担当者への通知を行う)
- メンテナンスモード(意図的な停止時にアラートを抑制する)
- Webシナリオ監視、ログ監視など、監視対象の拡張
- Discoveryによるホストの自動検出、テンプレートの自作
- ダッシュボードの作成