## AWS Security Token Service (STS) を使用したROSAクラスターの作成

### 前準備

ROSAは、Red Hatによるお客様の既存AWSアカウントへのデプロイを可能にするモデルを提供します。Red Hatはセキュリティを考慮して、[AWS Secrity Token Service (STS)](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id_credentials_temp.html) を使用したROSAのデプロイを推奨しています。STSを使用したROSAデプロイの前提条件は、次のドキュメントをご参照ください。

- [第1章 STS を使用した ROSA の AWS 前提条件](https://access.redhat.com/documentation/ja-jp/red_hat_openshift_service_on_aws/4/html/prepare_your_environment/rosa-sts-aws-prereqs)
- [第2章 OpenShift Cluster Manager IAM ロールリソース](https://access.redhat.com/documentation/ja-jp/red_hat_openshift_service_on_aws/4/html/prepare_your_environment/rosa-sts-ocm-role)
- [第4章 STS を使用する ROSA クラスターの IAM リソースについて](https://access.redhat.com/documentation/ja-jp/red_hat_openshift_service_on_aws/4/html/introduction_to_rosa/rosa-sts-about-iam-resources)
- [第5章 必要な AWS サービスクォータ](https://access.redhat.com/documentation/ja-jp/red_hat_openshift_service_on_aws/4/html/prepare_your_environment/rosa-sts-required-aws-service-quotas)

ROSAクラスターをデプロイするには、AWSアカウントと[Red Hatアカウント](https://cloud.redhat.com/)を利用します。アカウントがない場合は、アカウントを作成する必要があります。

[AWSコンソール](https://console.aws.amazon.com/rosa/)でROSAサービスを有効にします。AWSアカウントにログインして、「Enable OpenShift」をクリックします。次のような画面になれば、ROSAサービスが有効になっています。

![ROSAサービスの有効化](./images/rosa-enable.png)
<div style="text-align: center;">ROSAサービスの有効化</div>　　

AWS CLIを[インストール](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)して、[設定](https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-chap-configure.html)します。このとき、「~/.aws/credentials」で、次の情報を設定する必要があります。

- aws_access_key_id
- aws_secret_key

「~/.aws/configure」で、デフォルトのAWSのリージョンを設定します。このリージョンが、後述のROSAクラスター作成コマンド実行時のデフォルトのリージョンとして扱われます。そして、次のコマンドを実行して、AWS APIをクエリーし、AWS CLIがインストールされて正しく設定されていることを確認します。次のような出力が表示されれば、AWS CLIの設定が完了しています。

```
$ aws sts get-caller-identity
{
    "UserId": "AIDXXXXXXXXXXXXX",
    "Account": "XXXXXXXXX",
    "Arn": "arn:aws:iam::XXXXXXXXX:user/testuser01"
}
```

AWSアカウントを利用してELBをインストールしたことがない場合、次のコマンドを実行してELB用のロールを作成します。
```
$ aws iam create-service-linked-role --aws-service-name "elasticloadbalancing.amazonaws.com"
```

IAMロールが作成されると、次のような画面を確認できます。

![AWS ELBのロール](./images/iam-elb-role.png)
<div style="text-align: center;">AWS ELBのロール</div>　　

続いて、[ROSA CLI](https://console.redhat.com/openshift/downloads)をダウンロードして、PATHを設定します。

![ROSA CLIのダウンロード画面](./images/rosa-cli-download.png)
<div style="text-align: center;">ROSA　CLIのダウンロード画面</div>　　

```
$ chmod +x rosa
$ sudo mv rosa /usr/local/bin/
$ rosa version
1.2.2
```

続いて、ROSA CLIでRed Hatアカウントにログインします。アクセストークンは、下記のコマンド実行により表示されているURLから入手できます。
```
$ rosa login
To login to your Red Hat account, get an offline access token at https://console.redhat.com/openshift/token/rosa
? Copy the token and paste it here: ********************
I: Logged in as '<Red Hatアカウントのユーザー名>' on 'https://api.openshift.com'
```

AWSアカウントにROSAデプロイに必要なクォータがあることを確認します。次のような出力が表示されたら、クォータのチェックはパスしています。

```
$ rosa verify quota  
I: Validating AWS quota...
I: AWS quota ok. If cluster installation fails, validate actual AWS resource usage against https://docs.openshift.com/rosa/rosa_getting_started/rosa-required-aws-service-quotas.html
```

もし、クォータのチェックに失敗したら、エラーメッセージをもとに、前述のAWSサービスクォータの増加をリクエストします。

次のコマンドを実行して、AWSおよびRed Hatの認証情報が正しく設定されていることを確認します。AWSアカウントID、デフォルトのリージョン、および、ARNが予測した内容と一致していることを確認します。

```
$ rosa whoami
AWS Account ID:               XXXXXXXXXX
AWS Default Region:           ap-northeast-1
AWS ARN:                      arn:aws:iam::XXXXXXXXXX:user/testuser01
OCM API:                      https://api.openshift.com
OCM Account ID:               XXXXXXXXXX
OCM Account Name:             Hiforumi Kojima
OCM Account Username:         <Red Hatアカウントのユーザー名>
OCM Account Email:            hkojima@redhat.com
OCM Organization ID:          XXXXXXXXXX
OCM Organization Name:        Hiforumi Kojima
OCM Organization External ID: XXXXXXXXXX
```

ROSA CLIを利用して、最新のOpenShift CLI (oc) をインストールします。
```
$ rosa download openshift-client
$ rosa verify openshift-client
I: Verifying whether OpenShift command-line tool is available...
I: Current OpenShift Client Version: 4.10.3
```

### STSを使用したROSAクラスターの作成

STSを使用してROSAクラスターを作成するために必要な、AWS IAMロールとポリシーを作成します。

最初にROSAクラスターの管理に利用する[OpenShift Cluster Manager (OCM)](https://cloud.redhat.com/openshift)(OpenShiftのテレメトリ情報などを管理するRed HatのSaaS)とAWSアカウントをリンクするために必要なロール「ocm-role」と、Red Hatアカウントのユーザー情報を確認するために必要なロール「user-role」の2つを作成します。
```
$ rosa create ocm-role --mode auto -y
I: Creating ocm role
I: Creating role using 'arn:aws:iam::XXXXXXXXX:user/testuser01'
I: Created role 'ManagedOpenShift-OCM-Role-XXXXXXXXX' with ARN 'arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-OCM-Role-XXXXXXXXX'
I: Linking OCM role
I: Successfully linked role-arn 'arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-OCM-Role-XXXXXXXXX' with organization account 'XXXXXXXXX'


$ rosa create user-role --mode auto -y
I: Creating User role
I: Creating ocm user role using 'arn:aws:iam::XXXXXXXXX:user/testuser01'
I: Created role 'ManagedOpenShift-User-<Red Hatアカウントのユーザー名>-Role' with ARN 'arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-User-<Red Hatアカウントのユーザー名>-Role'
I: Linking User role
I: Successfully linked role ARN 'arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-User-<Red Hatアカウントのユーザー名>-Role' with account 'XXXXXXXXX'
```

次に、実際にROSAクラスターを作成するために必要となる、AWS IAMロールとポリシーを作成します。

```
$ rosa create account-roles --mode auto -y
I: Logged in as '<Red Hatアカウントのユーザー名>' on 'https://api.openshift.com'
I: Validating AWS credentials...
I: AWS credentials are valid!
I: Validating AWS quota...
I: AWS quota ok. If cluster installation fails, validate actual AWS resource usage against https://docs.openshift.com/rosa/rosa_getting_started/rosa-required-aws-service-quotas.html
I: Verifying whether OpenShift command-line tool is available...
I: Current OpenShift Client Version: 4.10.3
I: Creating account roles
I: Creating roles using 'arn:aws:iam::XXXXXXXXX:user/testuser01'
I: Created role 'ManagedOpenShift-Installer-Role' with ARN 'arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-Installer-Role'
I: Created role 'ManagedOpenShift-ControlPlane-Role' with ARN 'arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-ControlPlane-Role'
I: Created role 'ManagedOpenShift-Worker-Role' with ARN 'arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-Worker-Role'
I: Created role 'ManagedOpenShift-Support-Role' with ARN 'arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-Support-Role'
I: Created policy with ARN 'arn:aws:iam::XXXXXXXXX:policy/ManagedOpenShift-openshift-cloud-credential-operator-cloud-crede'
I: Created policy with ARN 'arn:aws:iam::XXXXXXXXX:policy/ManagedOpenShift-openshift-image-registry-installer-cloud-creden'
I: Created policy with ARN 'arn:aws:iam::XXXXXXXXX:policy/ManagedOpenShift-openshift-ingress-operator-cloud-credentials'
I: Created policy with ARN 'arn:aws:iam::XXXXXXXXX:policy/ManagedOpenShift-openshift-cluster-csi-drivers-ebs-cloud-credent'
I: Created policy with ARN 'arn:aws:iam::XXXXXXXXX:policy/ManagedOpenShift-openshift-cloud-network-config-controller-cloud'
I: Created policy with ARN 'arn:aws:iam::XXXXXXXXX:policy/ManagedOpenShift-openshift-machine-api-aws-cloud-credentials'
I: To create a cluster with these roles, run the following command:
rosa create cluster --sts
```

AWSコンソールから、ROSAクラスター作成に必要なIAMロールとポリシーが作成されていることを確認できます。

![作成されたIAMロール](./images/managed-openshift-roles.png)
<div style="text-align: center;">作成されたIAMロール</div>　　

![作成されたIAMポリシー](./images/managed-openshift-policies.png)
<div style="text-align: center;">作成されたIAMポリシー</div>　　

どのようなAWSのアクションを許可するポリシーが作成されているかは、[こちらのドキュメント](https://docs.openshift.com/rosa/rosa_architecture/rosa-sts-about-iam-resources.html#rosa-sts-account-wide-roles-and-policies-creation-methods_rosa-sts-about-iam-resources)から確認することもできます。

ROSAクラスターの作成コマンドを実行します。

デフォルトで設定しているAWSリージョンに、OpenShift 4.10.14を利用した、シングルAZの最小構成(Control x3, Infra x2, Compute x2 の7台構成)のROSAクラスターを全自動で作成するには、次のコマンドを実行します。`--version`オプションを指定しない場合、自動的に最新版のOpenShiftが利用されます。`--dry-run`オプションを付けると、正常に実行可能かどうかを事前確認できます。
```
$ rosa create cluster -c test-cluster01 --version 4.10.14 --sts --mode auto -y 

I: Using arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-Installer-Role for the Installer role
I: Using arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-ControlPlane-Role for the ControlPlane role
I: Using arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-Worker-Role for the Worker role
I: Using arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-Support-Role for the Support role
I: Creating cluster 'test-cluster01'
I: To view a list of clusters and their status, run 'rosa list clusters'
I: Cluster 'test-cluster01' has been created.
I: Once the cluster is installed you will need to add an Identity Provider before you can login into the cluster. See 'rosa create idp --help' for more information.
Name:                       test-cluster01
ID:                         XXXXXXXXX
External ID:                
OpenShift Version:          
Channel Group:              stable
DNS:                        test-cluster01.9job.p1.openshiftapps.com
AWS Account:                XXXXXXXXX
API URL:                    
Console URL:                
Region:                     ap-northeast-1
Multi-AZ:                   false
Nodes:
 - Control plane:           3
 - Infra:                   2
 - Compute:                 2
Network:
 - Service CIDR:            172.30.0.0/16
 - Machine CIDR:            10.0.0.0/16
 - Pod CIDR:                10.128.0.0/14
 - Host Prefix:             /23
STS Role ARN:               arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-Installer-Role
Support Role ARN:           arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-Support-Role
Instance IAM Roles:
 - Control plane:           arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-ControlPlane-Role
 - Worker:                  arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-Worker-Role
Operator IAM Roles:
 - arn:aws:iam::XXXXXXXXX:role/test-cluster01-t1e5-openshift-machine-api-aws-cloud-credentials
 - arn:aws:iam::XXXXXXXXX:role/test-cluster01-t1e5-openshift-cloud-credential-operator-cloud-cr
 - arn:aws:iam::XXXXXXXXX:role/test-cluster01-t1e5-openshift-image-registry-installer-cloud-cre
 - arn:aws:iam::XXXXXXXXX:role/test-cluster01-t1e5-openshift-ingress-operator-cloud-credentials
 - arn:aws:iam::XXXXXXXXX:role/test-cluster01-t1e5-openshift-cluster-csi-drivers-ebs-cloud-cred
 - arn:aws:iam::XXXXXXXXX:role/test-cluster01-t1e5-openshift-cloud-network-config-controller-cl
State:                      waiting (Waiting for OIDC configuration)
Private:                    No
Created:                    Aug 24 2022 11:32:29 UTC
Details Page:               https://console.redhat.com/openshift/details/s/XXXXXXXXX
OIDC Endpoint URL:          https://rh-oidc.s3.us-east-1.amazonaws.com/XXXXXXXXX

I: Preparing to create operator roles.
I: Creating roles using 'arn:aws:iam::XXXXXXXXX:user/hkojima@redhat.com-XXXXXX-admin'
I: Created role 'test-cluster01-t1e5-openshift-cluster-csi-drivers-ebs-cloud-cred' with ARN 'arn:aws:iam::XXXXXXXXX:role/test-cluster01-t1e5-openshift-cluster-csi-drivers-ebs-cloud-cred'
I: Created role 'test-cluster01-t1e5-openshift-cloud-network-config-controller-cl' with ARN 'arn:aws:iam::XXXXXXXXX:role/test-cluster01-t1e5-openshift-cloud-network-config-controller-cl'
I: Created role 'test-cluster01-t1e5-openshift-machine-api-aws-cloud-credentials' with ARN 'arn:aws:iam::XXXXXXXXX:role/test-cluster01-t1e5-openshift-machine-api-aws-cloud-credentials'
I: Created role 'test-cluster01-t1e5-openshift-cloud-credential-operator-cloud-cr' with ARN 'arn:aws:iam::XXXXXXXXX:role/test-cluster01-t1e5-openshift-cloud-credential-operator-cloud-cr'
I: Created role 'test-cluster01-t1e5-openshift-image-registry-installer-cloud-cre' with ARN 'arn:aws:iam::XXXXXXXXX:role/test-cluster01-t1e5-openshift-image-registry-installer-cloud-cre'
I: Created role 'test-cluster01-t1e5-openshift-ingress-operator-cloud-credentials' with ARN 'arn:aws:iam::XXXXXXXXX:role/test-cluster01-t1e5-openshift-ingress-operator-cloud-credentials'
I: Preparing to create OIDC Provider.
I: Creating OIDC provider using 'arn:aws:iam::XXXXXXXXX:user/hkojima@redhat.com-XXXXXX-admin'
I: Created OIDC provider with ARN 'arn:aws:iam::XXXXXXXXX:oidc-provider/rh-oidc.s3.us-east-1.amazonaws.com/XXXXXXXXX'
I: To determine when your cluster is Ready, run 'rosa describe cluster -c test-cluster01'.
I: To watch your cluster installation logs, run 'rosa logs install -c test-cluster01 --watch'.
```

マルチAZの最小構成(Control x3, Infra x3, Compute x3 の9台構成)のROSAクラスターを作成する場合、`--multi-az`オプションを指定します。このオプションにより、3つのAZに跨ったROSAクラスターが作成されます。
```
$ rosa create cluster -c test-cluster01 --version 4.10.14 --sts --mode auto -y --multi-az
```

なお、対話モードによりオプションを指定しながらインストールすることもできます。
```
$ rosa create cluster --sts

I: Enabling interactive mode
? Cluster name: test-cluster01
? OpenShift version: 4.10.14
I: Using arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-Installer-Role for the Installer role
I: Using arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-ControlPlane-Role for the ControlPlane role
I: Using arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-Worker-Role for the Worker role
I: Using arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-Support-Role for the Support role
? External ID (optional): 
? Operator roles prefix: test-cluster01-b0e6
? Multiple availability zones (optional): No
? AWS region: ap-northeast-1
? PrivateLink cluster (optional): No
? Install into an existing VPC (optional): No
? Enable Customer Managed key (optional): No
? Compute nodes instance type (optional): m5.xlarge
? Enable autoscaling (optional): No
? Compute nodes: 2
? Machine CIDR: 10.0.0.0/16
? Service CIDR: 172.30.0.0/16
? Pod CIDR: 10.128.0.0/14
? Host prefix: 23
? Encrypt etcd data (optional): Yes
? Disable Workload monitoring (optional): No
I: Creating cluster 'test-cluster01'
I: To create this cluster again in the future, you can run:
   rosa create cluster --cluster-name test-cluster01 --sts --role-arn arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-Installer-Role --support-role-arn arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-Support-Role --controlplane-iam-role arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-ControlPlane-Role --worker-iam-role arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-Worker-Role --operator-roles-prefix test-cluster01-b0e6 --region ap-northeast-1 --version 4.10.14 --compute-nodes 2 --compute-machine-type m5.xlarge --machine-cidr 10.0.0.0/16 --service-cidr 172.30.0.0/16 --pod-cidr 10.128.0.0/14 --host-prefix 23 --etcd-encryption
I: To view a list of clusters and their status, run 'rosa list clusters'

I: Cluster 'test-cluster01' has been created.
I: Once the cluster is installed you will need to add an Identity Provider before you can login into the cluster. See 'rosa create idp --help' for more information.
Name:                       test-cluster01
ID:                         XXXXXXXXX
External ID:                
OpenShift Version:          
Channel Group:              stable
DNS:                        test-cluster01.xxx.xx.openshiftapps.com
AWS Account:                XXXXXXXXX
API URL:                    
Console URL:                
Region:                     ap-northeast-1
Multi-AZ:                   false
Nodes:
 - Control plane:           3
 - Infra:                   2
 - Compute:                 2
Network:
 - Service CIDR:            172.30.0.0/16
 - Machine CIDR:            10.0.0.0/16
 - Pod CIDR:                10.128.0.0/14
 - Host Prefix:             /23
STS Role ARN:               arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-Installer-Role
Support Role ARN:           arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-Support-Role
Instance IAM Roles:
 - Control plane:           arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-ControlPlane-Role
 - Worker:                  arn:aws:iam::XXXXXXXXX:role/ManagedOpenShift-Worker-Role
Operator IAM Roles:
 - arn:aws:iam::XXXXXXXXX:role/test-cluster01-b0e6-openshift-cloud-network-config-controller-cl
 - arn:aws:iam::XXXXXXXXX:role/test-cluster01-b0e6-openshift-machine-api-aws-cloud-credentials
 - arn:aws:iam::XXXXXXXXX:role/test-cluster01-b0e6-openshift-cloud-credential-operator-cloud-cr
 - arn:aws:iam::XXXXXXXXX:role/test-cluster01-b0e6-openshift-image-registry-installer-cloud-cre
 - arn:aws:iam::XXXXXXXXX:role/test-cluster01-b0e6-openshift-ingress-operator-cloud-credentials
 - arn:aws:iam::XXXXXXXXX:role/test-cluster01-b0e6-openshift-cluster-csi-drivers-ebs-cloud-cred
State:                      waiting (Waiting for OIDC configuration)
Private:                    No
Created:                    Jun  7 2022 03:09:25 UTC
Details Page:               https://console.redhat.com/openshift/details/s/XXXXXXXXX
OIDC Endpoint URL:          https://rh-oidc.s3.us-east-1.amazonaws.com/XXXXXXXXX

I: Run the following commands to continue the cluster creation:

	rosa create operator-roles --cluster test-cluster01
	rosa create oidc-provider --cluster test-cluster01

I: To determine when your cluster is Ready, run 'rosa describe cluster -c test-cluster01'.
I: To watch your cluster installation logs, run 'rosa logs install -c test-cluster01 --watch'.


$ rosa create operator-roles --cluster test-cluster01 --mode auto -y
$ rosa create oidc-provider --cluster test-cluster01 --mode auto -y
```

最後の2行のコマンドで、ROSAクラスター専用IAMロール(ROSAクラスターのOperatorが必要なAWSリソースを作成するために必要)と、認証するためにクラスターOperatorによって使用されるOpenID Connect (OIDC) プロバイダーを作成します。対話モードでROSAクラスター作成コマンドを実行した場合、これらを作成しないと、ROSAクラスター作成状態が「Waiting」のままで、デプロイが完了しません。


クラスターのデプロイ状態は、次のコマンドで確認できます。
```
$ rosa list clusters
ID                                NAME            STATE
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX  test-cluster01  installing
$ rosa describe cluster -c test-cluster01
...<省略>...
State:                      installing
...<省略>...
```

デプロイ状態(STATE)は、デプロイの進捗として次のように出力されます。
- waiting (Waiting for OIDC configuration)
- pending (Preparing account)
- installing (DNS setup in progress)
- installing
- ready

また、クラスターのデプロイログは次のコマンドで確認できます。
```
$ rosa logs install -c test-cluster01 --watch
...<省略>...
time="2022-06-07T03:52:52Z" level=info msg="Install complete!"
time="2022-06-07T03:52:52Z" level=info msg="To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/output/auth/kubeconfig'"
time="2022-06-07T03:52:52Z" level=info msg="Access the OpenShift web-console here: https://console-openshift-console.apps.test-cluster01.XXXXX.p1.openshiftapps.com"
REDACTED LINE OF OUTPUT
time="2022-06-07T03:52:52Z" level=debug msg="Time elapsed per stage:"
time="2022-06-07T03:52:52Z" level=debug msg="           cluster: 5m28s"
time="2022-06-07T03:52:52Z" level=debug msg="         bootstrap: 54s"
time="2022-06-07T03:52:52Z" level=debug msg="Bootstrap Complete: 13m11s"
time="2022-06-07T03:52:52Z" level=debug msg="               API: 3m32s"
time="2022-06-07T03:52:52Z" level=debug msg=" Bootstrap Destroy: 1m7s"
time="2022-06-07T03:52:52Z" level=debug msg=" Cluster Operators: 12m47s"
time="2022-06-07T03:52:52Z" level=info msg="Time elapsed: 33m33s"
time="2022-06-07T03:52:53Z" level=info msg="command completed successfully" installID=qcsg75m8
time="2022-06-07T03:52:53Z" level=info msg="saving installer output" installID=qcsg75m8
...<省略>...
time="2022-06-07T03:52:53Z" level=info msg="install completed successfully" installID=qcsg75m8
I: Cluster 'test-cluster01' is now ready
```

40分くらい待つとROSAクラスターのデプロイが完了して、STATEが「ready」状態になり、作成したROSAクラスターにアクセスできるようになります。

ROSAクラスターの作成完了を待っている間、予め作成済みである別のROSAクラスターを利用して、受講者は演習を進めます。[GitHubを利用したROSAクラスターへのアクセス](../rosa-access)に進んでください。

#### \[参考情報\]
- [Creating a ROSA cluster in STS mode](https://mobb.ninja/docs/rosa/sts/)

[HOME](../../README.md)
