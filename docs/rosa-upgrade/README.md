## ROSAクラスターのアップグレード

### コンソールを使用したアップグレード

OpenShift Cluster Manager (OCM)を使用して、AWS STSを使用するROSAクラスターを手動でアップグレードできます。なお、ROSAの場合、通常のOpenShiftとは異なり、OpenShiftのWebコンソールとCLI(ocコマンド)によるアップグレードができないように制限がかけられています。そのため、OCMや後述するROSA CLIによるアップグレードを実施する必要があります。

[OCMにログイン](https://console.redhat.com/openshift/)して、アップグレードするROSAクラスターを選択し、Settingsタブをクリックして、「Update」ボタンをクリックします。

![ROSAクラスターの設定](./images/rosa-settings.png)
<div style="text-align: center;">ROSAクラスターの設定画面</div>　　

アップグレードするバージョンを選択して、「Next」をクリックします。

![バージョンの選択](./images/version-select.png)
<div style="text-align: center;">バージョンの選択</div>　　

クラスターのアップグレードをスケジュールします。 現在の時刻から1時間以内にアップグレードを開始するには、「Update now」を選択し、「Next」をクリックします。
指定した時間にアップグレードするには、「Schedule a different time」を選択し、アップグレードの日時を設定します。「Next」をクリックして確認ダイアログに進みます。

![アップグレードのスケジュール](./images/schedule.png)
<div style="text-align: center;">アップグレードのスケジュール</div>　　

アップグレードするバージョンとスケジュールを確認したら、「Confirm Update」をクリックして、アップグレードをスケジュールします。

![アップグレードの確認](./images/confirm.png)
<div style="text-align: center;">アップグレードの確認</div>　

アップグレードのステータスが「Update status」ペインに表示されます。

![アップグレードのステータス](./images/status.png)
<div style="text-align: center;">アップグレードのステータス</div>　　

アップグレードをキャンセルしたい場合、「Cancel this update」を選択します。

![アップグレードのキャンセル](./images/cancel.png)
<div style="text-align: center;">アップグレードのキャンセル</div>　　

### ROSA CLIを使用したアップグレード

OCMのコンソールの他に、ROSA CLIを使用してROSAクラスターをアップグレードできます。次のコマンドを実行して、利用可能なアップグレードを確認します。
```
$ rosa list upgrade --cluster test-cluster01
VERSION  NOTES
4.10.15  recommended
```

ここで確認したアップグレードを適用します。「Node draining」では、アップグレードのために、コンピュートノードからPodを停止させる猶予時間を指定できます。デフォルトは1時間です。
```
$ rosa upgrade cluster --cluster test-cluster01
? Version:  [Use arrows to move, type to filter, ? for more help]
> 4.10.15
I: Ensuring account and operator role policies for cluster 'XXXXXXX' are compatible with upgrade.
I: Account and operator roles for cluster 'test-cluster01' are compatible with upgrade
? Please input desired date in format yyyy-mm-dd: 2022-06-07
? Please input desired UTC time in format HH:mm: 08:00
? Node draining:  [Use arrows to move, type to filter, ? for more help]
  15 minutes
  30 minutes
  45 minutes
> 1 hour
  2 hours
  4 hours
  8 hours
I: Upgrade successfully scheduled for cluster 'test-cluster01'
```

指定したアップグレードのスケジュールを確認できます。
```
$ rosa list upgrade cluster --cluster test-cluster01
VERSION  NOTES
4.10.15  scheduled for 2022-06-07 08:00 UTC
```

現時点で、ROSA CLIによるアップグレードのキャンセルはできませんので、アップグレードのキャンセルをしたい場合、OCMのコンソールから実施してください。アップグレードを指定またはキャンセルすると、Red Hat SREチームからメールの通知がきます。

- アップグレードの指定をした場合の通知メールの例

```
Hello Hiforumi,

This notification is for your test-cluster01 cluster.
Your Cluster is scheduled for upgrade maintenance to version '4.10.15' on 2022-06-07 at 08:00 UTC.

Please contact Red Hat support if you have any questions.
Thank you for choosing Red Hat OpenShift Service on AWS,
OpenShift SRE
```

- アップグレードのキャンセルをした場合の通知メールの例

```
Hello Hiforumi,

This notification is for your test-cluster01 cluster.
Your Cluster upgrade maintenance to version '4.10.15' on 2022-06-07 at 06:00 UTC has been cancelled.

If you have any questions, please contact us. Review the support process for guidance on working with Red Hat support.
Thank you for choosing Red Hat OpenShift Service on AWS,
OpenShift SRE
```

なお、実際にアップグレードが開始されると、次のような通知メールが届きます。
```
Hello Hiforumi,

This notification is for your test-cluster01 cluster.
Your Cluster is currently being upgraded to version '4.10.15'.

Please contact Red Hat support if you have any questions.

What can you expect?
Your cluster capacity may increase briefly during the course of the upgrade but will never decrease.
Your cluster will remain operational during the upgrade.
If your applications are not designed as highly available, they may experience brief outages as we roll through the upgrade.
We will send reminders as we get closer to the upgrade date.
The maintenance window for the upgrade is variable and should take approximately 90 minutes for the control plane nodes and 10 minutes for each worker node.

What should you do to minimize impact?
You can minimize the impact on your applications by scaling your services to more than one pod.
In general, for applications to be able to continue to service clients, they should be scaled.
Some pod workloads are not appropriate for scaling, such as a single-instance, non-replicated database using a persistent volume claim.
In this situation, a deployment strategy of 'recreate' will ensure the pod is restarted after migration, although a brief outage will be experienced.
For more information, refer to our highly available deployment guide.

If you have any questions, please contact us. Review the support process for guidance on working with Red Hat suppor
Thank you for choosing Red Hat OpenShift Service on AWS,
OpenShift SRE
```

この通知メールにもあるように、アップグレードについては、コントローラノード 90分、コンピュートノード 10分/1台 が目安となります。コンピュートノード上のユーザーPodの動作状況や、「Node draining」の時間指定次第で、アップグレード時間はこれより多くかかることもあります。
```
The maintenance window for the upgrade is variable and should take approximately 90 minutes for the control plane nodes and 10 minutes for each worker node.
```

アップグレードが正常に完了すると、次の通知メールが届きます。
```
Hello Hiforumi,

This notification is for your test-cluster01 cluster.
Your Cluster has been successfully upgraded to version '4.10.15'.

If you have any questions, please contact us. Review the support process for guidance on working with Red Hat support.

Thank you for choosing Red Hat OpenShift Service on AWS,
OpenShift SRE
```

参考情報ですが、実際にROSAクラスターの最小構成(Controller x3, Infra x2, Compute x2, Single-Avialability zone)で、ユーザーアプリを動かしていない構成をアップグレードした場合、アップグレード開始メールを受け取ってからアップグレード完了メールを受け取るまで、100分でした。概ね前述した目安の時間に沿ったものとなっています。


### ROSAクラスターのマイナーリリース間のアップグレード

ROSAクラスターは、4.9から4.10など、マイナーリリース間のアップグレードも可能です。Red Hatによって、マイナーリリース間のアップグレードパスが提供されている場合、前述の手順と同じ手順でアップグレードできます。

[4.9.40からアップグレードするバージョンの選択](./images/minor-upgrade1.png)
[4.9.40からアップグレードするバージョンの選択](./images/minor-upgrade2.png)
[4.9.40からアップグレードするバージョンの選択](./images/minor-upgrade3.png)
<div style="text-align: center;">4.9.40からアップグレードするバージョンの選択</div>　　

マイナーリリース間のアップグレードを実施する際に、ROSAクラスターで利用されるIAMロールのアップグレードを先に実施しておく、などの前提条件が指定される場合があります。このときは、上記画像で提示される手順(「Learn more」と書かれている箇所)を実施すると、アップグレードを実施できるようになります。下記のナレッジベースがその1例となりますので、参考にしてください。

**[参考情報]** [ROSA STS requires user action before install or upgrade](https://access.redhat.com/solutions/6808671)


これで、ROSAクラスターアップグレードスケジューリングのデモ紹介は終了です。次は、インストラクターによる、[ROSAクラスター削除](../rosa-delete)のデモ紹介です。

[HOME](../../README.md)
