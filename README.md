# KongBootCamp　デモ環境構築
[0. 事前準備](https://github.com/MasakiGogami-i4/kong-training/blob/main/README.md#0%E4%BA%8B%E5%89%8D%E6%BA%96%E5%82%99)<br>
[1. Kong CP・DPの構築](https://github.com/MasakiGogami-i4/kong-training/blob/main/README.md#1kong-cpdp%E3%81%AE%E6%A7%8B%E7%AF%89)<br>
[2. Prometheus/Grafanaのデプロイ](https://github.com/MasakiGogami-i4/kong-training/blob/main/README.md#2-prometheusgrafana%E3%81%AE%E3%83%87%E3%83%97%E3%83%AD%E3%82%A4)<br>
[3. サンプルアプリのデプロイ （BookInfo）](https://github.com/MasakiGogami-i4/kong-training/blob/main/README.md#3%E3%82%B5%E3%83%B3%E3%83%97%E3%83%AB%E3%82%A2%E3%83%97%E3%83%AA%E3%81%AE%E3%83%87%E3%83%97%E3%83%AD%E3%82%A4-bookinfo)<br>
[4. 監査ログの取得](https://github.com/MasakiGogami-i4/kong-training/blob/main/README.md#3%E3%82%B5%E3%83%B3%E3%83%97%E3%83%AB%E3%82%A2%E3%83%97%E3%83%AA%E3%81%AE%E3%83%87%E3%83%97%E3%83%AD%E3%82%A4-bookinfo)<br>
[5. APIOpsの実装](https://github.com/MasakiGogami-i4/kong-training/blob/main/README.md#3%E3%82%B5%E3%83%B3%E3%83%97%E3%83%AB%E3%82%A2%E3%83%97%E3%83%AA%E3%81%AE%E3%83%87%E3%83%97%E3%83%AD%E3%82%A4-bookinfo)<br>

## 0.事前準備
1. k8sクラスタの用意　※本手順の資材はaks前提
- k8sクラスタ構築
- Ingress Controller（Contour）インストール
  - 参考：https://qiita.com/ipppppei/items/c15acc5c7f7af3e7c289#contour%E3%81%AE%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB
2. k8sクラスタ、インターネットに接続可能な端末の用意
- 端末作成
- 各種ツールのインストール
  - kubectl
  - helm
  - deck
3. Konnectのアカウント作成
- Konnectアカウント作成

## 1.Kong CP・DPの構築
使用リポジトリ：MasakiGogami-i4/kong-training（本リポジトリ）

### Kong CP構築
- Konnect上でCPを作成（Gateway Manager > New Gateway > Self-Managed Hybrid）
### Kong DP構築
- GHA workflow作成（.github/workflow/deploy_dp.yml）
  - docker hubからKongコンテナイメージをpull
  - TrivyでKongイメージの脆弱性をスキャン
  - Github Container Resistory(GHCR)にタグを付与してKongイメージをpush（ゴールデンイメージ）
  - GHCRのimage pull用のsecretをk8sに作成
  - TLS証明書を作成してKonnectに登録
  - k8sにTLS証明書のsecretを作成
  - Kong DP用values.yaml作成
  - k8sにhelmでKong DPをデプロイ
- Githubに変数設定（vars,secrets）
- workflow実行　※トリガーは手動実行
- Konnect上でDPが作成されたことを確認（Gateway Manager > CP選択 > Data Plane Nodes）

## 2. Prometheus/Grafanaのデプロイ
- 下記ページを参考にしてk8sにhelmでPrometheus/Grafanaをデプロイ
  - https://qiita.com/ipppppei/items/c15acc5c7f7af3e7c289#promethes-operator%E3%81%AE%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB
  - ※本デモではcert-managerはインストールしていないため、ingressのenabled、ingressClassName、hosts部分のみ修正でok
- ブラウザから動作確認
  - http://prometheus.20-204-106-218.nip.io/ <br>
  - http://grafana.20-204-106-218.nip.io/login <br>
※外部IPは自分の環境に合わせて変更すること  
※Grafanaはadmin/prom-operatorでログイン可能  
※Kong(official)のGrafanaダッシュボードは以下から取得可能  
https://grafana.com/grafana/dashboards/7424-kong-official/

## 3.サンプルアプリのデプロイ （BookInfo）
使用リポジトリ：MasakiGogami-i4/bookinfo
fork元：https://github.com/imurata/bookinfo

### BookInfo用Kongリソース作成
- BookInfoアプリ用プラグイン定義作成（kong-plugins/）
  - 現状はprometheusとratelimit advancedのみグローバルスコープで定義　※その他プラグインは手動設定前提
- BookInfoアプリ用API Spec作成（docs/openapi/api-spec.yaml）
- BookInfoアプリ用API Productドキュメント作成（docs/product.md）

- api-spec.yamlのmainリポジトリ更新時にworkflow実行（.github/workflow/deploy_oas.yaml、upload_spec.yaml）
  - deploy_oas.yaml
    - API SpecからDeck形式に変換
    - kong-plugins配下のKongプラグイン定義ファイルもDeck形式に変換
    - デプロイ
  - upload_spec.yaml
    - API SpecをKonnectのAPI Productsの該当Product Versionsにアップロード
- Konnect上で反映確認
  - Kongリソースが作成されたことを確認（Gateway Manager > CP選択 > Gateway Services, Routes, Upstreams, Targets, Consumers, Plugins）
  - API Specが作成されたことを確認


### BookInfoアプリデプロイ
- k8sリソース定義ファイル修正（platform/kube/bookinfo.yaml）
  - BookInfoをインターネット公開するHTTPProxyを追加
    -  productpageをcontourの外部IPで公開するHTTPProxyを追加　※外部IPは自分の環境に合わせて変更すること
  - BookInfo details,ratings,reviews宛のリクエストを各マイクロサービスではなくkongに転送させるように修正（下図参考）
    - productpageのDeploymentのspec.template.spec.envにKongのhost,port等を追加

初期状態  
<img width="724" alt="image" src="https://github.com/user-attachments/assets/dd734a4d-db71-44d1-ad60-7c235d3d8b9d" />  

details,ratings,reviews宛のリクエストの向き先をKongに変更した状態  
<img width="721" alt="image" src="https://github.com/user-attachments/assets/238b19d8-4256-4acf-8faa-305dd035a900" />  
※引用：https://qiita.com/ipppppei/items/0c235f9ae9c50131a7c6  

-  GHA workflow作成（.github/workflow/deploy_bookinfo.yml）
  -  Bookinfoアプリのコンテナイメージ作成
  -  GHCRにタグを付与してBookinfoイメージをpush
  -  k8sにkubectlでBookinfoをデプロイ

-  ブラウザから動作確認 （http://productpage.74-225-133-33.nip.io/productpage）　※外部IPは自分の環境に合わせて変更すること

## 4.監査ログの取得
使用リポジトリ：MasakiGogami-i4/kong-training（本リポジトリ）

- Konnect監査ログを受信するアプリをデプロイ（audit-logs/webhook-script.yaml、webhook-server.yaml）
```
kubectl apply -f audit-logs/webhook-server.yaml

kubectl apply -f audit-logs/webhook-script.yaml

kubectl get ing -n audit-logs

kubectl get pod -n audit-logs

kubectl logs webhook-server -n audit-logs -f
```

- Konnect上で監査ログ設定（Organization　> Audit Logs Setup >　Konnect）
  - 下記設定
    - Endpoint：http://webhook.20-204-106-218.nip.io/　※外部IPアドレスは各自の設定に変更すること
    - Authorization Header：Bearer hoge　※audit-logsはデモ用のため認証かけていないのでダミーの値でOK
    - Log Format：Json
  - View Advanced Fields　>　Disable SSL Verification (Do not recommend)を有効化（disable → enableに変更してSave）
  - SatusタブでActive、200、timeを確認

- BookInfoにアクセスして動作確認
  - audit-logs PodのログにKonnect監査ログが見えることを確認
```
kubectl logs webhook-server -n audit-logs | tail -f
```
  
## 5.APIOpsの実装
使用リポジトリ：MasakiGogami-i4/konnect-apiops-template
fork元：https://github.com/imurata/konnect-apiops-template

以下のリポジトリのREADME及び./github/workflowsを参照。<br>
https://github.com/MasakiGogami-i4/kong-training/tree/main
