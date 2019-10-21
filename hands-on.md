## CloudFormationi によるAWSリソースのプロビジョニング
- 活用するサービスが多いので CloudFormation でハンズオンの準備をする

### CloudFormation の特徴
- 変更のプレビュー
- 依存関係の管理
- クロスアカウントとクロスリージョンの管理


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

