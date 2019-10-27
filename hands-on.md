## CloudFormationi によるAWSリソースのプロビジョニング
- 活用するサービスが多いので CloudFormation でハンズオンの準備をする

## 使うスクリプトの整理
#### m.yml
- MLOpsScikitImageRepo: Scikit-base のECRのリポジトリ、この段階ではイメージなし
- MLOpsIrisModelRepo: Iris-model のECRのリポジトリ
- MLOpsBucket:MLOps S3 バケット　この段階では中身なし
- MLOpsRepo: CodeCommit のリポジトリ
- IAWorkshopNotebookInstance: ノートブックインスタンス
- IAWorkshopNotebookInstanceLifecycleConfig: ノートブックインスタンスのlifecycle configで下記を設定。1ノートブック、1Codepipelineで運用。複数人でやる場合には？
    - サブネットの定義
    - セキュリティグループの定義
    - ノートブックインスタンスのライフサイクルの設定
        -  この中でのCFn使ってリソースつくている
            - build-image.yml sikit_base
            - build-image.yml iris_model
            - train-model-pipeline.yml 
- MLOpsCodeBuild: CloudBuild用のIAMロール
- MLOps: SageMaker用のIAMロール


#### build-image.yml
- CodeBuildのプロジェクト
- CodePipelineのパイプライン

#### train-model-pipeline.yml
- Lambda SageMakerの学習ジョブを立ち上げる関数
    - SageMaker: 学習ジョブ投げてる
    - CloudWatch: 学習ジョブをモニタリング
    - CodePipeline: 既存のcodepipelineのJobIDを確認、通知。学習が成功したかを通知
    
- Lambda CodePipeline の監視を行う関数
    - SageMaker: Job description を見に行ってる
    - CloudWatch: モニタリングをdisableしている
    - Codepipeline: pipelineのステータスを監視し、承認するかを表示

- Lambda permission: 監視関数がLambdaを叩けるようにする許可
- CloudWatch Events: 学習ジョブを関ししていて、終わったらpipelineに知らせる
- CodePipeline: deployのためのpipelineを作る
    - Source: S3 モデルが置かれたらに置かれたら
    - Train: 学習用Lambdaを発火、
    - TrainApproval: マニュアル操作、何を見て判断？
    - DeployDev: CloudFormationを呼び出す、deploy-model-dev.yml。こいつがSageMakerのEndpointをデプロイ。
    - DeployApproval: マニュアル操作、何を見て判断？
    - CloudFormationを呼び出す、deploy-model-prd.yml。こいつがSageMakerのEndpointをデプロイ。

#### deploy-model-dev.yml
- SageMaker のモデル: Endpointで使う
- EndpointConfig: エンドポイントの設定。Endpointで使う
- SageMaker Endpoint

#### deploy-model-prd.yml
- SageMaker EndpointConfig: エンドポイントの設定。Endpointで使う
- SageMaker Endpoint
- AutoScaling Target
- AutScaling Scaling policy


## ハンズオンの内容
- s3へはアウトプットとして何が残る？

- CloudFormation で m.yml の中身を実行

1. Select the below to launch CloudFormation stack.

Region| Launch
------|-----
US East (N. Virginia) | [![Launch MLOps solution in us-east-1](imgs/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=AIWorkshop&templateURL=https://s3.amazonaws.com/aws-ai-ml-aod-latam/mlops-workshop/m.yml)


#### build-image.yml scikit-base が走る
- CFn の scikit-base スタックができる
    - CodePipelineのパイプライン
        - GetSource: CodeCommitのブランチがソース
        - Build: Codebuildをキック  
    - CodeBuildのプロジェクト
        - lab1 で作る buildspec.yml を参している
        - lab1の中で下記スクリプトを作成してコミット
            - Dockerfile
            - app.py
            - buildspec.yml

#### build-image.yml scikit-base が走る
- CFn の iris-model スタックができる
    - DeployPipeline: CodePipelineのパイプライン
        - GetSource: CodeCommitのブランチがソース
        - Build: Codebuildをキック  
    - BuildImageProject: CodeBuildのプロジェクト
        - lab1 で作る buildspec.yml を参している
        - lab1の中で下記スクリプトを作成してコミット
            - Dockerfile
            - model.py
            - buildspec.yml
            
#### train-model-pipeline.yml　が走る           
- CFn の iris-train-pipeline スタックができる
    - iris-train-pipeline:CodePipeline のパイプライン
        - Source: Amazon S3 
        - Train: Lambda、mlops-job-launcher-{modelname}
        - TrainApproval: Manual approval 
        - DeployModelDev: CFn
            - deploy-model-dev.yml を参照
            - SageMaker のモデル: Endpointで使う
            - EndpointConfig: エンドポイントの設定。Endpointで使う
            - SageMaker Endpoint
        - DeployApproval: Manual approval
        - DeployPrd: CFn
            - deploy-model-prd.yml を参照
            - SageMaker EndpointConfig: エンドポイントの設定。Endpointで使う
            - SageMaker Endpoint
            - AutoScaling Target
            - AutScaling Scaling policy
           
- Lambda: mlops-job-launcher-{modelname}、SageMakerの学習ジョブを立ち上げる関数
    - SageMakerの学習ジョブ投げる
    - CloudWatchのmlops-job-monitor-${ModelName} で学習ジョブをモニタリング
    - CodePipelinedで既存のcodepipelineのJobIDを確認、通知。学習が成功したかを通知
    
- Lambda:mlops-job-monitor-${ModelName}、CodePipeline の監視を行う関数
    - SageMaker: Job description を見に行ってる
    - CloudWatch: モニタリングをdisableしている
    - Codepipeline: pipelineのステータスを監視し、承認するかを表示

- Lambda permission: 監視関数がLambdaを叩けるようにする許可
- CloudWatch Events Rule: 学習ジョブを関ししていて、終わったらpipelineに知らせる
    - Lambda の mlops-job-monitor-${ModelName} をターゲットにしている

### 01_BuildBaseImage
#### 01_Creating a Scikit-Learn Base Image
- このディレクトリで下記を作成。
    - 推論用のDockerfileを作成
    - 推論用の`app.py`ファイルを作成
    - CodeBuildが読み込むためのbuildspec.yml

- CodeCommitのgitリポジトリへ作成したファイルをコピー
- CodeCommitへpush
- CodePipeline が発火され、CodeBuildが走る

### 02_BuildModelImage
#### 01_Creating an Iris Model Image
- このディレクトリで下記を作成。
    - 推論用のDockerfileを作成
    - 推論用の`app.py`ファイルを作成
    - CodeBuildが読み込むためのbuildspec.yml

- CodeCommitのgitリポジトリへ作成したファイルをコピー
- CodeCommitへpush
- CodePipeline が発火され、CodeBuildが走る
    - CodeBuild の中でDocker Image がビルドされてECRへプッシュされる
 
 #### 03_Training our custom model 
 - Souce: s3へデータをアップロードすると、CodePipelineが発火
 - Train: Pipeline中のBuildでLambdaで学習
    - Lambda がsagemakerの学習と学習のチェックを監視
 - TrainApproval: マニュアルで学習を認証
 - DeployDev: Dev環境にモデルをデプロイ
 - DeployApproval: 承認
 - DeployPrd:
 #### 04_Check Progress and Test the endpoint
 - パイプラインの監視と、エンドポイントの動作確認
