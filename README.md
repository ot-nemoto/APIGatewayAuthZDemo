# APIGatewayAuthZDemo

## 概要

- API GatewayのリクエストにCognitoによる認証を組み込んだデモ

## 構成

![構成](https://github.com/ot-nemoto/APIGatewayAuthDemo/blob/images/APIGatewayAuthDemo.png)

## 前提条件

- aws-cli で構築するため、aws configure の設定は適宜実施していること

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
  #   "message": "Hello"
  # }
```

## ポイント

- ユーザプールクライアントの設定で、ユーザ名とパスワードによる認証を有効にする (`USER_PASSWORD_AUTH`)
  - ref https://github.com/ot-nemoto/APIGatewayAuthNDemo/blob/master/template.yaml#L15
- API Gateway の swagger の定義では、securityDefinitions に認証方法を定義する
  - ここでリクエストヘッダーに `Authentication` を指定するように定義しているので、認証用のヘッダー名を変えたい場合は、ここの定義を修正する
  - ref https://github.com/ot-nemoto/APIGatewayAuthNDemo/blob/master/template.yaml#L60
















IDENTITY_POOL_ID=$(aws cloudformation describe-stacks \
    --stack-name api-gateway-authz-demo \
    --query 'Stacks[].Outputs[?OutputKey==`IdentityPoolId`].OutputValue' \
    --output text)
echo ${IDENTITY_POOL_ID}
  # ap-northeast-1:d02c87b3-2db8-4d8d-8d40-a7985feb9c88

USER_POOL_ID=$(aws cloudformation describe-stacks \
    --stack-name api-gateway-authz-demo \
    --query 'Stacks[].Outputs[?OutputKey==`UserPoolId`].OutputValue' \
    --output text)
echo ${USER_POOL_ID}
  # ap-northeast-1_x6EoTroO6

USER_POOL_CLIENT_ID=$(aws cloudformation describe-stacks \
    --stack-name api-gateway-authz-demo \
    --query 'Stacks[].Outputs[?OutputKey==`UserPoolClientId`].OutputValue' \
    --output text)
echo ${USER_POOL_CLIENT_ID}
  # 5rgp27dcco2gq1k8n5f31mn8v5

TOKEN=$(aws cognito-idp initiate-auth \
     --auth-flow USER_PASSWORD_AUTH \
     --client-id ${USER_POOL_CLIENT_ID} \
     --auth-parameters USERNAME=${USERNAME},PASSWORD=${NEW_PASSWORD} \
     --query 'AuthenticationResult.IdToken' \
     --output text)
echo ${TOKEN}

IDENTITY_POOL_ID=$(aws cloudformation describe-stacks \
    --stack-name api-gateway-authz-demo \
    --query 'Stacks[].Outputs[?OutputKey==`IdentityPoolId`].OutputValue' \
    --output text)
echo ${IDENTITY_POOL_ID}
  # ap-northeast-1:d02c87b3-2db8-4d8d-8d40-a7985feb9c88

PROVIDER_NAME=$(aws cloudformation describe-stacks \
    --stack-name api-gateway-authz-demo \
    --query 'Stacks[].Outputs[?OutputKey==`UserPoolProviderName`].OutputValue' \
    --output text)
echo ${PROVIDER_NAME}
  # cognito-idp.ap-northeast-1.amazonaws.com/ap-northeast-1_HYeuIrhic

IDENTITY_ID=$(aws cognito-identity get-id \
    --identity-pool-id ${IDENTITY_POOL_ID} \
    --logins ${PROVIDER_NAME}=${TOKEN} \
    --output text)
echo ${IDENTITY_ID}
  # ap-northeast-1:52cc7b28-2785-4c1e-b861-dba004ee0898

aws cognito-identity get-credentials-for-identity \
    --identity-id ${IDENTITY_ID} \
    --logins ${PROVIDER_NAME}=${TOKEN}







IDENTITY_ID=$(aws cognito-identity get-id \
    --identity-pool-id ${IDENTITY_POOL_ID} \
    --output text)
echo ${IDENTITY_ID}
  # ap-northeast-1:57b1d910-3623-4c07-9784-cf38bd42d005

aws cognito-identity get-credentials-for-identity \
    --identity-id ${IDENTITY_ID}


refs
- https://dev.classmethod.jp/cloud/aws/aws-cli-credentials-using-amazon-cognito/
- https://www.wakuwakubank.com/posts/696-aws-cognito/
- https://qiita.com/y13i/items/1923b47079bdf7c44eec
