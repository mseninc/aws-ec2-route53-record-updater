# aws-ec2-route53-record-updater

EC2 インスタンス起動時にパブリック IP アドレスを取得し、Route 53 のレコードを更新します。

## 概要

1. EC2 インスタンスが起動すると、EventBridge のイベントルールから Step Functions ステートマシンが起動します。
2. Step Functions ステートマシンは、EC2 インスタンスのパブリック IP アドレスを取得し、Route 53 のレコードを更新します。

```mermaid
flowchart TB

subgraph AWSCloud[AWS]
    subgraph VPC[VPC]
        subgraph SubnetPublic1[Public Subnet 1]
            EC2Instance("EC2 Instance"):::AWSCompute
        end
    end
    subgraph GroupEventBridge[EventBridge]
        EventRule1("UpdateRoute53RecordOnEC2Running"):::AWSCompute
    end
    subgraph GroupRoute53[Route53]
        Route53Record("DNS レコード"):::AWSCompute
    end
    subgraph GroupStepFunctions[Step Functions ステートマシン]
        StateMachine("aws-ec2-route53-record-updater-task"):::AWSCompute
    end
    EC2Instance -->|①イベント発火| EventRule1
    EventRule1 -->|②ステートマシン起動| StateMachine
    StateMachine -->|③パブリック IP アドレス, DNS レコード取得| EC2Instance
    StateMachine -->|④更新| Route53Record
end

%% ---スタイルの設定---

%% デフォルトスタイル
classDef default fill:none,color:#666,stroke:#aaa

%% AWS Cloudのスタイル
classDef AWSCloud fill:none,color:#345,stroke:#345
class AWSCloud AWSCloud

%% Regionのスタイル
classDef AWSRegion fill:none,color:#59d,stroke:#59d,stroke-dasharray:3
class RegionTokyo AWSRegion

%% VPCのスタイル
classDef AWSVPC fill:none,color:#0a0,stroke:#0a0
class VPC AWSVPC

%% Availability Zoneのスタイル
classDef AWSAZ fill:none,color:#59d,stroke:#59d,stroke-width:1px,stroke-dasharray:8
class GA AWSAZ

%% Private subnetのスタイル
classDef AWSPrivateSubnet fill:#def,color:#07b,stroke:none
class SubnetPrivate1 AWSPrivateSubnet

%% Public subnetのスタイル
classDef AWSPublicSubnet fill:#efe,color:#092,stroke:none
class SubnetPublic1 AWSPublicSubnet

%% Network関連のスタイル
classDef AWSNetwork fill:#84d,color:#fff,stroke:none

%% Compute関連のスタイル
classDef AWSCompute fill:#e83,color:#fff,stroke:none

%% DB関連のスタイル
classDef AWSDatabase fill:#46d,color:#fff,stroke:#fff

%% AWSStorage関連のスタイル
classDef AWSStorage fill:#493,color:#fff,stroke:#fff

%% 外部要素のスタイル
classDef OtherElement fill:#aaa,color:#fff,stroke:#fff
```

## デプロイ

### 準備: CloudWatch Logs の Resource based policy 更新

- [cloudwatch-logs-resource-policy.md](cloudwatch-logs-resource-policy.md) を参照して、 `cloudwatch-logs-resource-policy.json` を編集して、 CloudWatch Logs の Resource based policy を更新する。

### デプロイ

`ROUTE53_HOSTED_ZONE_ID` に設定する DNS レコードのホストゾーン ID を指定して Serverless Framework でデプロイします。

```bash
npm install
ROUTE53_HOSTED_ZONE_ID=Z******************** npx serverless deploy --region us-east-1
```

### EC2 インスタンスへのタグの設定

Route 53 に登録する DNS レコード名を EC2 インスタンスのタグに設定します。

```json
{
    "Key": "Route53Record",
    "Value": "hogehoge.example.com"
}
```

※このタグキー (デフォルトでは `Route53Record`) はデプロイ時に `domainNameTagKey` で指定した値を指定します。デプロイ時にこのキーを変更するには `--param="domainNameTagKey=新しいタグキー"` オプションを指定します。（参考: [Serverless Framework - Parameters](https://www.serverless.com/framework/docs-guides-parameters)）
