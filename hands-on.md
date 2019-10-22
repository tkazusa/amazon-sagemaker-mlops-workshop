## CloudFormationi によるAWSリソースのプロビジョニング
- 活用するサービスが多いので CloudFormation でハンズオンの準備をする

## 使うスクリプトの整理
#### m.yml
- Scikit-base のECRのリポジトリ
- Iris-model のECRのリポジトリ
- MLOps S3 バケット
- CodeCommit のリポジトリ
- ノートブックインスタンス
    - サブネットの定義
    - セキュリティグループの定義
    - ノートブックインスタンスのライフサイクルの設定
        -  この中でのCFn使ってリソースつくている
            - build-image.yml sikit_base
            - build-image.yml iris_model
            - train-model-pipeline.yml 
- CloudBuild用のIAMロール
- SageMaker用のIAMロール

#### build-image.yml
- CodeBuildのプロジェクト
- CodePipelineのパイプライン

#### train-model-pipeline.yml
- Lambda 学習用のファンクション
    - SageMaker: 学習ジョブ投げてる
    - CloudWatch: 学習ジョブをモニタリング
    - CodePipeline: 学習が成功したかを通知
    


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
