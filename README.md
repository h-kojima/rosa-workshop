## Red Hat OpenShift Service on AWS (ROSA) Workshops

Red Hat OpenShift Service on AWS (ROSA) Workshops プロジェクトは、インストラクター主導のデモ紹介またはセルフペースの演習で、ROSAを効果的に紹介及び体感していただくことを目的としています。

最初に受講者は、コンテナ/Kubernetes/OpenShift/ROSAの概要を学習します。

<embed src="docs/pdf/2022-rosa-workshop-lecture.pdf#&scrollbar=0&view=Fit&viewrect=0,0,570,0" width="640" height="360" hspace="0" vspace="0">

PDFの資料は[こちら](docs/pdf/2022-rosa-workshop-lecture.pdf)からダウンロードできます。

<iframe width="560" height="315" src="https://www.youtube.com/embed/DlrDb-hK2Vo" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

次に下記のデモ紹介及び演習を実施します。コンテンツは以下の種類に分かれます。

- \[デモ\]: インストラクターによるデモ紹介です。受講者は、コマンド実行やGUIの操作をする必要はありません。
- \[ハンズオン\]: セルフペースの演習です。受講者は、コマンド実行やGUIの操作をしてワークショップを進めます。
- \[デモとハンズオン\]: 上記の両方を含みます。

### 演習の前提要件

- AWS/GitHubにアクセス可能なネットワーク環境
- RHELサーバにSSHログインして、コマンドが実行可能
   - ROSA/OpenShift CLIがインストールされているRHELサーバを利用するために、SSHログインが必要です。
- GitHubアカウント(個人アカウントも可)が利用可能
   - ROSAクラスターの認証情報として利用します。
- ROSAクラスターにアクセス可能なWebブラウザ
   - [「Browsers and Client Tools」表](https://access.redhat.com/articles/4763741)にある、最新版のOpenShiftに対応したWebブラウザ(Firefox/MS Edge/Chrome/Safari)のいずれかを利用します。


### コンテンツ

1. [\[デモ\] ROSAクラスターの作成](docs/rosa-create)
1. [\[ハンズオン\] GitHubを利用したROSAクラスターへのアクセス](docs/rosa-access)
1. [\[ハンズオン\] アプリケーションのデプロイのクイックスタート](docs/rosa-app-deploy-quickstart)
1. [\[ハンズオン\] 永続ボリュームとしての Amazon EBS の利用設定](docs/rosa-volume)
1. [\[デモとハンズオン\] AWS Controllers for Kubernetes (ACK) による Amazon S3の利用](docs/rosa-ack-s3)
1. [\[ハンズオン\] コンピュートノードの追加/削除とオートスケールの設定](docs/rosa-nodes)
1. [\[デモ\] ROSAクラスターのアップグレード](docs/rosa-upgrade)
1. [\[デモ\] ROSAクラスターの削除](docs/rosa-delete)
2. [\[ハンズオン\] ROSAクラスターでのJavaアプリケーション開発 スターターラボ](docs/rosa-sample-app-develop)
