# 本リポジトリについて
これは私が個人でクラウドに建てているAPIサーバーについての内容を記録したものになります。
ポートフォリオ掲載用（また備忘録として）を目的としているが、
別アカウントかつプライベートリポジトリで運用していることから、
ドキュメントとしてReadmeを記載してます。

## 構築内容・目的

個人開発のWebサービス向けに、APIサーバーをクラウド上に構築しました。
単にAPIを公開するだけではなく、コンテナ化したアプリケーションのデプロイ、データベース接続、認証情報の管理、CI/CDまでを一通り経験することを目的とし、Google Cloud上にGKEを中心とした構成としている。
アプリケーションコードの更新から本番環境への反映までを自動化し、手作業によるデプロイを減らすことも目的としました。

現状、API自体はブログ記事を参照するためのものになっている。
また、記事はMarkdownを編集してGitHubへpushすれば記事データまで自動的に更新されるようにしている。

## 構成

主に以下の技術・サービスで構成しています。

- Google Cloud
- Google Kubernetes Engine（GKE）
- Artifact Registry
- Cloud SQL（PostgreSQL）
- Secret Manager
- Docker
- Kubernetes Job
- GitHub Actions
- Go
- Gin
- GORM
- Markdown

システムは、大きく「記事参照」と「記事登録・更新」の2つの処理に分けています。

```text
【記事参照】

Web Frontend
      ↓
 Article API
   Go / Gin
      ↓
 PostgreSQL


【記事登録・更新】

Markdown
   ↓
GitHub
   ↓
GitHub Actions
   ↓
Kubernetes Job
   ↓
  md-sync
   ↓
PostgreSQL
```

BackendはDockerイメージとしてArtifact Registryへ保存し、GKE上でDeploymentとして稼働させています。
記事の登録処理と参照処理を分離し、Webサイトからの記事閲覧時にはMarkdownファイルを直接読み込まず、データベースに登録済みの記事をAPIから取得する構成としました。

## 実装

### GKEへのアプリケーション配置

GatewayとBackendをそれぞれDockerコンテナ化し、Artifact RegistryへpushしたイメージをGKE Autopilot上で実行しました。

KubernetesではDeploymentとServiceを定義し、BackendはClusterIPとしてクラスタ内部のみからアクセスできるようにしています。

GatewayのみLoadBalancer Serviceとして外部公開し、Backendを直接インターネットへ公開しない構成としました。

### Cloud SQLとの接続

データベースにはCloud SQL for PostgreSQLを使用しました。

Backend PodにはCloud SQL Auth ProxyをSidecarコンテナとして配置し、Backendからは `localhost:5432` に接続することでCloud SQLへアクセスできるようにしています。

```text
Backend Pod
├─ Backend
│    └─ localhost:5432
│
└─ Cloud SQL Auth Proxy
     └─ Cloud SQL
```

これにより、アプリケーション側ではCloud SQL固有の接続処理を持たず、通常のPostgreSQL接続として扱える構成にしました。

### Workload Identityによる認証

GKE上のPodからGoogle Cloudのリソースへアクセスする際には、Workload Identityを利用しました。
　
Kubernetes ServiceAccountとGoogle Service Accountを関連付け、BackendがCloud SQLへ接続するために必要な `Cloud SQL Client` 権限のみを付与しています。

### GitHub ActionsによるCI/CD

GitHubのmainブランチへのpushをトリガーとして、以下の処理を自動化しました。

```text
GitHub Push
  ↓
GitHub Actions
  ↓
Docker Image Build
  ↓
Artifact Registry Push
  ↓
GKE Credential取得
  ↓
Deployment更新
  ↓
Rollout確認
```

GitHub ActionsからGoogle Cloudへの認証にはWorkload Identity Federationを使用しています。

GitHub Actionsが発行するOIDCトークンをGoogle Cloud側で検証し、一時的にデプロイ用Service Accountの権限を取得する仕組みとしました。

## 工夫した点

特に意識したのは、アプリケーションの実行権限とCI/CDのデプロイ権限を分離したことです。

アプリケーション実行時は、

```text
Backend
↓
Kubernetes ServiceAccount
↓
Google Service Account
↓
Cloud SQL
```

という認証経路を使用しています。

一方、デプロイ時は、

```text
GitHub Actions
↓
GitHub OIDC
↓
Workload Identity Federation
↓
Deploy用Google Service Account
↓
Artifact Registry / GKE
```

という別の認証経路を使用しています。

用途ごとにService Accountを分け、それぞれ必要な権限のみを与えることで、認証情報や権限の範囲をできるだけ限定しました。

記事コンテンツについては、MarkdownファイルをGitで管理し、変更されたMarkdownをバッチ処理によってHTMLへ変換してDBへ登録する仕組みも検討・実装しました。

APIサーバーとは別に `md-sync` というWorkerを用意し、GitHub ActionsからKubernetes Jobとして実行する構成です。

## 課題

構築を進める中で、GKEを利用することでインフラ構成が複雑になりやすい点が課題になりました。

小規模な個人開発であっても、

- Kubernetes Deployment / Service
- IAM
- Workload Identity
- Artifact Registry
- Cloud SQL
- Cloud SQL Auth Proxy
- GitHub Actions

など、管理する要素が多くなります。

小規模サービスではサービス数が増え、運用コストが高くなるという問題もありました。

## 改善予定

現在の規模であれば、GKEを利用するメリットよりも運用負荷の方が大きくなる可能性があるため、APIサーバーをCloud Runへ移行する構成を検討しています。

```text
Client
  ↓
Cloud Run
  ↓
Cloud SQL
```

※
最小最低構成で建てたつもりだが普通にきつい
<img width="1332" height="328" alt="image" src="https://github.com/user-attachments/assets/9bff1e46-6929-42e2-af36-9aae49107496" />

<img width="1267" height="385" alt="image" src="https://github.com/user-attachments/assets/12f8ab35-bd1e-4ad3-bc41-e361c2857c05" />
