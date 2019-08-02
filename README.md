# S3UploaderV2Demo

## 概要

- [ブラウザから Amazon S3 への写真のアップロード](https://docs.aws.amazon.com/ja_jp/sdk-for-javascript/v2/developer-guide/s3-example-photo-album.html) のソースコードをベースに CloudFormation で環境を構築するデモ
- S3に画像をアップロードしたことをトリガーに Step Functions を起動
- そもそも、API Gateway から画像をPOSTした際に Step Functions を起動させようとしたが、画像をBASE64した値をStep Functions のパラメータとして渡せないことから、考えた構成
  - refs [タスクの実行に関連する制限](https://docs.aws.amazon.com/ja_jp/step-functions/latest/dg/limits.html#service-limits-task-executions)
- Step Functionsをファイルをトリガーに起動させたい場合のひな型

### 構成

![構成図](https://raw.githubusercontent.com/ot-nemoto/S3UploaderV2Demo/feature/images/S3UploaderV2Demo.jpg)

- Websiteから画像をアップロード。
  - アップロードする際にRoleはIDPoolで管理
  - アップロード先の Bucket は `Album`
- `Album` にアップロードした Event を CloudWatch Events で監視
  - S3のオブジェクトのイベントを監視する際は、CloudTrailを有効にする必要がある
- CloudWatch Events のターゲットで Step Functions を起動
  - ステートマシンの入力は、CloudWatch Envents のイベントソースを渡す
- Step Functions で起動した Lambda では、Step Functions の入力をそのまま渡しデバッグ
  - デバッグした値から S3 へアップロードしたバケットやオブジェクトを取得できる（あとはLambda側で要実装）

## デプロイ

デプロイ用のバケット作成

```sh
aws cloudformation create-stack \
    --stack-name s3-uploader-v2-demo-bucket-stack \
    --template-body file://bucket-template.yaml

DEPLOY_BUCKET_NAME=$(aws cloudformation describe-stacks \
    --stack-name s3-uploader-v2-demo-bucket-stack \
    --query 'Stacks[].Outputs[?OutputKey==`DeployBucket`].OutputValue' \
    --output text)
echo ${DEPLOY_BUCKET_NAME}
  #
```

デプロイ

```sh
sam package \
    --s3-bucket ${DEPLOY_BUCKET_NAME} \
    --output-template-file packaged.yml

sam deploy \
    --template-file packaged.yml \
    --stack-name s3-uploader-v2-demo-stack \
    --capabilities CAPABILITY_NAMED_IAM
```

フロントエンドをデプロイ

```sh
export BUCKET_NAME=$(aws cloudformation describe-stacks \
    --stack-name s3-uploader-v2-demo-stack \
    --query 'Stacks[].Outputs[?OutputKey==`AlbumBucket`].OutputValue' \
    --output text)
export REGION=$(aws cloudformation describe-stacks \
    --stack-name s3-uploader-v2-demo-stack \
    --query 'Stacks[].Outputs[?OutputKey==`Region`].OutputValue' \
    --output text)
export IDENTITY_POOL_ID=$(aws cloudformation describe-stacks \
    --stack-name s3-uploader-v2-demo-stack \
    --query 'Stacks[].Outputs[?OutputKey==`IdentityPool`].OutputValue' \
    --output text)

envsubst < public/app.js.tpl > public/app.js

WEBSITE_BUCKET_NAME=$(aws cloudformation describe-stacks \
    --stack-name s3-uploader-v2-demo-stack \
    --query 'Stacks[].Outputs[?OutputKey==`WebsiteBucket`].OutputValue' \
    --output text)

aws s3 sync public/ s3://${WEBSITE_BUCKET_NAME}/ --delete --exclude app.js.tpl
```

## アンデプロイ

```sh
aws cloudformation delete-stack \
    --stack-name s3-uploader-v2-demo-stack
```

```sh
aws cloudformation delete-stack \
    --stack-name s3-uploader-v2-demo-bucket-stack
```
