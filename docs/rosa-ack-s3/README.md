## AWS Controllers for Kubernetes (ACK) による AWS S3の利用

### \[デモ\] AWS Controllers for Kubernetes (ACK) - Amazon S3 Operatorのインストール

ROSAに含まれるRed Hat OpenShiftのOperatorHubでは、Kubernetes/OpenShiftからAWSのリソースを簡単に利用できるようになっている、[AWS Controllers for Kubernetes (ACK)](https://aws-controllers-k8s.github.io/community/docs/user-docs/openshift/)というOperatorを用意しています。ACKを利用して、AWS S3を利用するための設定を行います。

ROSAクラスターの管理者権限を持つ cluster-admin ユーザでログインして、OperatorHubを選択すると、利用可能なACKのリストを確認できます。

![OperatorHubにあるACKのリスト](./images/ack-s3-install1.png)
<div style="text-align: center;">OperatorHubにあるACKのリスト</div>　　

ACKを利用するために必要となる、設定情報とAWS認証情報を登録します。これには、cluster-admin権限によるOpenShift CLI (oc) を利用します。  
最初に設定情報とAWS認証情報を保存するプロジェクト、「ack-system」を作成して、作業場所を「ack-system」に移動します。

```
$ oc new-project ack-system; oc project ack-system
```

設定情報をこのプロジェクト内に作成します。OpenShiftでの設定情報は、プロジェクトごとにconfigmapというリソースとして作成されます。ACKが利用するconfimapとして、「ack-user-config」という名前が指定されますので、この名前を使ってconfigmapを作成します。

```
$ cat <<EOF > config.txt
ACK_ENABLE_DEVELOPMENT_LOGGING=true
ACK_LOG_LEVEL=debug
ACK_WATCH_NAMESPACE=
AWS_REGION=ap-southeast-1
AWS_ENDPOINT_URL=
ACK_RESOURCE_TAGS=hellofromocp
EOF
$ oc create configmap --from-env-file=config.txt ack-user-config
configmap/ack-user-config created
```

AWS認証情報を作成します。OpenShiftでの認証情報は、プロジェクトごとにsecretというリソースとして作成されます。ACKが利用するsecretとして、「ack-user-secrets」という名前が指定されますので、この名前を使ってsecretsを作成します。

```
$ cat <<EOF > secrets.txt 
AWS_ACCESS_KEY_ID=XXXXXXX
AWS_SECRET_ACCESS_KEY=XXXXXXXXXXXXXXXXXXX
$ oc create secret generic --from-env-file=secrets.txt ack-user-secrets
secret/ack-user-secrets created
```

作成した　configmapとsecretsは、ROSAのWebコンソールで確認できます。

![ack-systemプロジェクトに作成したack-user-config](./images/ack-user-config.png)
<div style="text-align: center;">ack-systemプロジェクトに作成したack-user-config</div>　　

![ack-systemプロジェクトに作成したack-user-secrets](./images/ack-user-secrets.png)
<div style="text-align: center;">ack-systemプロジェクトに作成したack-user-secrets</div>　　

あとは、cluster-adminユーザで、OperatorHubから「ACK - Amazon S3」Operatorをインストールします。全てデフォルトの値を利用して、インストールを進めていきます。

![ACK Amazon S3のインストール](./images/ack-s3-install2.png)
![ACK Amazon S3のインストール](./images/ack-s3-install3.png)
![ACK Amazon S3のインストール](./images/ack-s3-install4.png)
![ACK Amazon S3のインストール](./images/ack-s3-install5.png)
<div style="text-align: center;">ACK Amazon S3のインストール</div>　

この状態が表示されれば、インストールが完了しています。ROSAのローカルユーザがインストールされたOperatorを利用して、S3のバケットをAWS上に作成できるようになります。

### \[ハンズオン\] AWS S3のバケット作成

ROSAクラスターにローカルユーザでログインしなおして、「インストールされたOperator」から ACK Amazon S3 Operatorを利用してバケットを作成します。このOperatorを選択して、「インスタンスの作成」をクリックすると、パラメータ入力画面に移動します。「名前」は任意の名前(ここではexample20)を入力し、「Name」には作成するAWS S3バケット名を入力します。AWS S3バケットは、グローバル名前空間が共通しているため、名前の重複が許可されていません。そのため、重複しないような名前を指定する必要があります。最後に一番下にある「作成」ボタンをクリックすれば、S3バケットの作成が完了します。

![Amazon S3バケットの作成](./images/s3-bucket-create1.png)
![Amazon S3バケットの作成](./images/s3-bucket-create2.png)
![Amazon S3バケットの作成](./images/s3-bucket-create3.png)
![Amazon S3バケットの作成](./images/s3-bucket-create4.png)
<div style="text-align: center;">Amazon S3バケットの作成</div>　

S3バケットの作成が完了すると、ROSAやAWSコンソールで、バケットが作成されていることを確認できます。

![Amazon S3バケットの確認](./images/s3-bucket-confirm1.png)
![Amazon S3バケットの確認](./images/s3-bucket-confirm2.png)
<div style="text-align: center;">Amazon S3バケットの作成</div>　

ACKによって作成したS3バケットを削除したい場合、当該バケットを選択して、「Bucketの削除」を選択すると削除できます。ROSAとAWSの両方から削除されます。

![Amazon S3バケットの削除](./images/s3-bucket-delete.png)
<div style="text-align: center;">Amazon S3バケットの削除</div>　