# APIGatewayAuthZDemo

## 概要

- API GatewayのリクエストにCognitoによる認証・認可を組み込んだデモ

## 構成

![構成](https://github.com/ot-nemoto/APIGatewayAuthZDemo/blob/images/APIGatewayAuthZDemo.png)

- APIを叩く前に、CognitoからTokenを取得する（このデモでは取得する箇所は aws-cli を使用し、実装してはいない）
- APIへはCognitoから取得したTokenをヘッダーに付与し、リクエストを投げる
- API GatewayはヘッダーのTokenでLambda（GetCredentialsFunction）へ認証を行う
- 認証が通った場合、API GatewayはLambda（DescribeInstancesFunction）を起動し、EC2のインスタンスID一覧を取得する

## 前提条件

- aws-cli で構築するため、aws configure の設定は適宜実施していること
- templateではAPI Gatewayのログを出力するようにCloudFormationに定義しています。該当RegionのAPI GatewayでAPI GateawyのRoleの設定が必要です。
  - 未設定の場合は [api-gateway-settings](https://github.com/ot-nemoto/api-gateway-settings) でも設定できます。

## デプロイ

```sh
aws cloudformation create-stack \
    --stack-name api-gateway-authz-demo \
    --capabilities CAPABILITY_AUTO_EXPAND CAPABILITY_IAM \
    --template-body file://template.yaml
```

## 使い方

### ユーザ作成

ユーザプールIDを取得

```sh
USER_POOL_ID=$(aws cloudformation describe-stacks \
    --stack-name api-gateway-authz-demo \
    --query 'Stacks[].Outputs[?OutputKey==`UserPoolId`].OutputValue' \
    --output text)
echo ${USER_POOL_ID}
  # ap-northeast-1_ydrdhvOdG
```

ユーザ名は `ot-nemoto@example.com` / 初期パスワードは `Passw0rd?`

```sh
USERNAME=ot-nemoto@example.com
PASSWORD=Passw0rd?

aws cognito-idp admin-create-user \
    --user-pool-id ${USER_POOL_ID} \
     --username ${USERNAME} \
    --temporary-password ${PASSWORD}
```

### 初回認証

ユーザプールのクライアントIDを取得

```sh
USER_POOL_CLIENT_ID=$(aws cloudformation describe-stacks \
    --stack-name api-gateway-authz-demo \
    --query 'Stacks[].Outputs[?OutputKey==`UserPoolClientId`].OutputValue' \
    --output text)
echo ${USER_POOL_CLIENT_ID}
  # 4rt9kn7rd5c7av4686tec5p21k
```

認証

```sh
aws cognito-idp initiate-auth \
     --auth-flow USER_PASSWORD_AUTH \
     --client-id ${USER_POOL_CLIENT_ID} \
     --auth-parameters USERNAME=${USERNAME},PASSWORD=${PASSWORD}
  # {
  #     "ChallengeName": "NEW_PASSWORD_REQUIRED",
  #     "ChallengeParameters": {
  #         "USER_ID_FOR_SRP": "d0b2ec76-ac3b-47b8-ad14-3aa7fac7c102",
  #         "requiredAttributes": "[]",
  #         "userAttributes": "{\"email\":\"ot-nemoto@example.com\"}"
  #     },
  #     "Session": "1a2lI3uS03...JzBV0cyng4"
  # }
```

### パスワード変更

ユーザ作成時には、ユーザのステータスが `FORCE_CHANGE_PASSWORD` であるため、APIの認証に使うためのトークンIDを取得できない
Sessionを利用し、初期パスワードを更新する

Sessionを取得

```sh
SESSION=$(aws cognito-idp initiate-auth \
     --auth-flow USER_PASSWORD_AUTH \
     --client-id ${USER_POOL_CLIENT_ID} \
     --auth-parameters USERNAME=${USERNAME},PASSWORD=${PASSWORD} \
     --query 'Session' --output text)
echo ${SESSION}
  # 1a2lI3uS03...JzBV0cyng4
```

パスワードを `qwerT1234%` に変更

```sh
NEW_PASSWORD=qwerT1234%

aws cognito-idp respond-to-auth-challenge \
    --client-id ${USER_POOL_CLIENT_ID} \
    --challenge-name NEW_PASSWORD_REQUIRED \
    --challenge-responses USERNAME=${USERNAME},NEW_PASSWORD=${NEW_PASSWORD} \
    --session "${SESSION}"
  # {
  #     "AuthenticationResult": {
  #         "ExpiresIn": 3600,
  #         "IdToken": "eyJraWQiOi...QJuwtGQ_dQ",
  #         "RefreshToken": "eyJjdHkiOi...4fhSK18PfA",
  #         "TokenType": "Bearer",
  #         "AccessToken": "eyJraWQiOi...5u3c0Xibsg"
  #     },
  #     "ChallengeParameters": {}
  # }
```

### APIを実行

認証機能を有効にしたAPIのエンドポイントを取得

```sh
INVOKE_URL=$(aws cloudformation describe-stacks \
    --stack-name api-gateway-authz-demo \
    --query 'Stacks[].Outputs[?OutputKey==`InvokeURL`].OutputValue' \
    --output text)
echo ${INVOKE_URL}
  # https://a76l5zr17l.execute-api.ap-northeast-1.amazonaws.com/v1/
```

トークンID未設定でAPIを実行

```sh
curl -s -XGET ${INVOKE_URL} | jq
  # {
  #   "message": "Unauthorized"
  # }
```

トークンIDを取得

```sh
TOKEN=$(aws cognito-idp initiate-auth \
     --auth-flow USER_PASSWORD_AUTH \
     --client-id ${USER_POOL_CLIENT_ID} \
     --auth-parameters USERNAME=${USERNAME},PASSWORD=${NEW_PASSWORD} \
     --query 'AuthenticationResult.IdToken' \
     --output text)
echo ${TOKEN}
  # eyJraWQiOi...lJcHl_o_kg
```

トークンIDをリクエストヘッダーに付与しAPIを実行

```sh
curl -s -XGET ${INVOKE_URL} -H "Authentication:${TOKEN}" | jq
  # {
  #   "instanceIds": [
  #     "i-01234567",
  #     "i-89012345"
  #   ]
  # }
```

## ポイント

- ユーザプールクライアントの設定で、ユーザ名とパスワードによる認証を有効にする (`USER_PASSWORD_AUTH`)
  - ref https://github.com/ot-nemoto/APIGatewayAuthNDemo/blob/master/template.yaml#L15
- API Gateway の swagger の定義では、securityDefinitions に認証方法を定義する
  - ここでリクエストヘッダーに `Authentication` を指定するように定義しているので、認証用のヘッダー名を変えたい場合は、ここの定義を修正する
  - ref https://github.com/ot-nemoto/APIGatewayAuthNDemo/blob/master/template.yaml#L60
- 権限自体はCognitoのIDプールに持たせる
  - 認証時、IDプールから認証成功時に付与されるRoleのcredentialsをAPI Gatewayへ返す
  - credentialsは、`AccessKeyId`、`SecretKey`、`SessionToken`
  - このcredentialsをAPI Gatwayから実行するLambda（DescribeInstancesFunction）へ渡す
  - DescribeInstancesFunction はec2:DescribeInstancesのPolicyを持つサービスロールは持っていないが、このcredentialsを使って処理を実行する
