# cf-pacemaker
pacemaker setting test on AWS EC2

## 概要
AWS EC2 on RHEL8 High Availability Add-OnインスタンスをCloudformationで構築後、
Pacemakerを利用してクラスタ間でPrivate IPリソースのAct/Stb構成を組む手順です。

## 前提条件
- Cloudformation実行端末にAWS-CLIがインストール済みであること
- 管理者権限（or 必要な権限）をもつIAMアクセスキーが設定済みであること

## Cloudformationでインフラ構築

### テンプレートをチェックする
```
aws cloudformation validate-template --template-body file://cf-pacemaker.yml
```


### スタックを作成する
```
aws cloudformation create-stack \
--stack-name cf-pacemaker \
--region ap-northeast-1 \
--template-body file://pacemaker.yml \
--parameters ParameterKey=KeyName,ParameterValue=[your key name] \
ParameterKey=MyIP,ParameterValue=[XXX.XXX.XXX.XXX/32] \
--capabilities CAPABILITY_NAMED_IAM
```

[your key name]にはEC2作成時に利用するシークレットキー名を指定してください。

[XXX.XXX.XXX.XXX/32]にはEC2へのSSH接続元IPアドレスを指定してください。

### スタック情報を表示する
```
watch aws cloudformation describe-stacks --stack-name cf-pacemaker
```

CREATE-COMPLETEが表示するまで待ちます。

## Pacemaker設定

EC2インスタンスにログインする

### クラスタノードの認証設定　★片方のノードでのみ実施
```
pcs host auth ec2-pacemaker-1-cf ec2-pacemaker-2-cf -u hacluster -p HaPas_123
```
###クラスタの作成　★上記コマンド実行したノードで
```
pcs cluster setup test-cluster ec2-pacemaker-1-cf ec2-pacemaker-2-cf
```
### クラスタの有効化と起動 ★上記コマンド実行したノードで
```
pcs cluster enable --all
pcs cluster start --all
```

### フェンシング設定
起動した2ノードでそれぞれ以下のコマンドを実行してインスタンスIDを取得
```
curl -s http://169.254.169.254/latest/meta-data/instance-id
```
片側のノードで以下の環境変数に代入
```
INSTANCE_ID1=[instance-id1]
INSTANCE_ID2=[instance-id2]
```
片側のノードで以下を実行
```
pcs stonith create clusterfence fence_aws region=ap-northeast-1 pcmk_host_map="ec2-pacemaker-1-cf:$INSTANCE_ID1;ec2-pacemaker-2-cf:$INSTANCE_ID2" power_timeout=240 pcmk_reboot_timeout=480 pcmk_reboot_retries=4
```

### pacemaker resourceの作成
```
pcs resource create privip awsvip secondary_private_ip=10.5.10.13 --group networking-group
pcs resource create vip IPaddr2 ip=10.5.10.13 --group networking-group　
```
※awsvipおよびIPaddr2はPacemaker標準添付のスクリプトです。
各スクリプトの利用方法は以下のコマンドで確認してください。
```
pcs resource describe awsvip
pcs resource describe Ipaddr2
```

ここまでで設定完了です。
以下のコマンドで2ノードともonlineかつ片側のノードでnetworking-groupリソースが起動していることを確認してください。
```
pcs status
```


## 動作確認
## .11ノードのフェンシング
pcs stonith fence ec2-pacemaker-1-cf
　→.11インスタンスが再起動されるはず
pcs status

## .12ノードのフェールオーバー
pcs node standby ec2-pacemaker-2-cf
→.12ノードがstadby状態に

pcs node unstandby ec2-pacemaker-2-cf
→.12ノードがOnlineに







