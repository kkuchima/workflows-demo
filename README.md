# Workflows Demo
 
## 0. 前提条件
- Google Cloud Project が作成されていること
- 以降の手順は Cloud Shell 上での実行を想定しています

## 1. API 有効化
### 1-1. API の有効化
本手順で利用する API を有効化します
```bash
gcloud services enable \
  cloudfunctions.googleapis.com \
  run.googleapis.com \
  workflows.googleapis.com \
  cloudbuild.googleapis.com \
  storage.googleapis.com
```

## 2. Cloud Functions のデプロイ(乱数生成アプリケーション)
### 2-1. 作業ディレクトリの作成
```bash
mkdir ~/randomgen
cd ~/randomgen
```

### 2-2. 乱数を生成するアプリケーションの作成
以下の内容で `main.py` というファイルを作成します
```bash
import random, json
from flask import jsonify

def randomgen(request):
    randomNum = random.randint(1,100)
    output = {"random":randomNum}
    return jsonify(output)
```

以下の内容で `requirements.txt` というファイルを作成します
```bash
flask>=1.0.2
```

### 2-3. Cloud Functions へのデプロイ
作成したアプリケーションを `randomgen` という名前で Cloud Functions にデプロイします
```bash
gcloud functions deploy randomgen \
    --runtime python37 \
    --trigger-http \
    --allow-unauthenticated
```

デプロイが完了したら、出力された URL にブラウザからアクセスするか  
もしくは以下コマンドを実行し、アプリケーションが動作していることを確認します
（乱数が表示されていれば OK です）
```bash
curl $(gcloud functions describe randomgen --format='value(httpsTrigger.url)')
```

生成された URL (下記フォーマット)は後ほど使うので書き留めておいてください  
https://us-central1-< project-id >.cloudfunctions.net/randomgen  

## 3. Cloud Functions のデプロイ(掛け算アプリケーション)
### 3-1. 作業ディレクトリの作成
```bash
mkdir ~/multiply
cd ~/multiply
```

### 3-2. 掛け算をするアプリケーションの作成
以下の内容で `main.py` というファイルを作成します
```bash
import random, json
from flask import jsonify

def multiply(request):
    request_json = request.get_json()
    output = {"multiplied":2*request_json['input']}
    return jsonify(output)
```

以下の内容で `requirements.txt` というファイルを作成します
```bash
flask>=1.0.2
```

### 3-3. Cloud Functions へのデプロイ
作成したアプリケーションを `multiply` という名前で Cloud Functions にデプロイします
```bash
gcloud functions deploy multiply \
    --runtime python37 \
    --trigger-http \
    --allow-unauthenticated
```

デプロイが完了したら、以下コマンドを実行し、アプリケーションが動作していることを確認します
（input として渡した値 5 の 2 倍の数(10)が返ってきたら OK です）
```bash
curl $(gcloud functions describe multiply --format='value(httpsTrigger.url)') \
-X POST \
-H "content-type: application/json" \
-d '{"input": 5}'
```

生成された URL (下記フォーマット)は後ほど使うので書き留めておいてください   
https://us-central1-< project-id >.cloudfunctions.net/multiply

## 4. Cloud Workflows を使った Cloud Functions の接続
### 4-1. 作業ディレクトリの作成
```bash
mkdir ~/workflows
cd ~/workflows
```

### 4-2. Cloud Workflows 構成ファイルの作成
以下の内容で `workflow.yaml` というファイルを作成します  
`<project-id>` には現在利用されている Google Cloud Project ID を入力してください  
`randomgen` で生成された乱数を `multiply` に渡し、2 倍にして表示します  
```bash
- randomgenFunction:
    call: http.get
    args:
        url: https://us-central1-<project-id>.cloudfunctions.net/randomgen
    result: randomgenResult
- multiplyFunction:
    call: http.post
    args:
        url: https://us-central1-<project-id>.cloudfunctions.net/multiply
        body:
            input: ${randomgenResult.body.random}
    result: multiplyResult
- returnResult:
    return: ${multiplyResult}
```

### 4-2. Cloud Workflows のデプロイ
以下コマンドを実行し、 Cloud Workflows をデプロイします
```bash
gcloud beta workflows deploy workflow --source=workflow.yaml
```

### 4-3. Cloud Workflows の実行
メニュー左側の Tools > Workflows を選択します  
![image01](https://user-images.githubusercontent.com/47690801/110273380-d321bb00-800f-11eb-8a11-f0093ad01f0e.png)

`workflow` という名前のワークフローを選択します  
![image02](https://user-images.githubusercontent.com/47690801/110273402-da48c900-800f-11eb-9573-30797f4d918c.png)

画面右上の `EXECUTE` を選択します  
![image03](https://user-images.githubusercontent.com/47690801/110273413-dddc5000-800f-11eb-8758-7ecac19852a9.png)

画面左下の `EXECUTE` をクリックしワークフローを実行します  
![image04](https://user-images.githubusercontent.com/47690801/110273416-de74e680-800f-11eb-90a5-964f80bb5ab0.png)

画面右上の更新ボタンを押し、実行結果を確認します  
![image05](https://user-images.githubusercontent.com/47690801/110273417-df0d7d00-800f-11eb-84d6-a56df4bd3f92.png)

## 5. Cloud Run のデプロイ
### 5-1. 作業ディレクトリの作成
```bash
mkdir ~/floor
cd ~/floor
```

### 5-2. アプリケーションの作成
以下の内容で app.py というファイルを作成します
```bash
import json
import logging
import os
import math

from flask import Flask, request

app = Flask(__name__)

@app.route('/', methods=['POST'])
def handle_post():
    content = json.loads(request.data)
    input = float(content['input'])
    return f"{math.floor(input)}", 200

if __name__ != '__main__':
    # Redirect Flask logs to Gunicorn logs
    gunicorn_logger = logging.getLogger('gunicorn.error')
    app.logger.handlers = gunicorn_logger.handlers
    app.logger.setLevel(gunicorn_logger.level)
    app.logger.info('Service started...')
else:
    app.run(debug=True, host='0.0.0.0', port=int(os.environ.get('PORT', 8080)))
```

以下の内容で Dockerfile というファイルを作成します
```bash
# Use an official lightweight Python image.
# https://hub.docker.com/_/python
FROM python:3.7-slim

# Install production dependencies.
RUN pip install Flask gunicorn

# Copy local code to the container image.
WORKDIR /app
COPY . .

# Run the web service on container startup. Here we use the gunicorn
# webserver, with one worker process and 8 threads.
# For environments with multiple CPU cores, increase the number of workers
# to be equal to the cores available.
CMD exec gunicorn --bind 0.0.0.0:8080 --workers 1 --threads 8 app:app
```

### 5-3. Cloud Run へのデプロイ
作成したアプリケーションをコンテナにビルドします
```bash
export SERVICE_NAME=floor
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/${SERVICE_NAME}
```

作成したアプリケーションを Cloud Run にデプロイします
```bash
gcloud run deploy ${SERVICE_NAME} \
  --image gcr.io/${GOOGLE_CLOUD_PROJECT}/${SERVICE_NAME} \
  --platform managed \
  --region us-central1 \
  --no-allow-unauthenticated
```
生成された URL (下記フォーマット)は後ほど使うので書き留めておいてください
https://floor-`<random-hash>`.run.app

## 6. Cloud Workflows との接続
Cloud Run と Cloud Workflows を接続する設定をします  
## 6-1. サービスアカウントの作成
Cloud Workflows 用のサービスアカウントを作成します
```bash
export SERVICE_ACCOUNT=workflows-sa
gcloud iam service-accounts create ${SERVICE_ACCOUNT}
```

サービスアカウントに権限を付与します
```bash
gcloud projects add-iam-policy-binding ${GOOGLE_CLOUD_PROJECT} \
    --member "serviceAccount:${SERVICE_ACCOUNT}@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com" \
    --role "roles/run.invoker"
```

### 6-2. 構成ファイルの編集
`workflows` ディレクトリに移動します
```bash
cd ~/workflows
```

Cloud Workflows の構成ファイルを以下のように編集します  
(`api.mathjs.org` の API で引き渡された値を log に変換する処理も加えています)
```bash
- randomgenFunction:
    call: http.get
    args:
        url: https://<region>-<project-id>.cloudfunctions.net/randomgen
    result: randomgenResult
- multiplyFunction:
    call: http.post
    args:
        url: https://<region>-<project-id>.cloudfunctions.net/multiply
        body:
            input: ${randomgenResult.body.random}
    result: multiplyResult
- logFunction:
    call: http.get
    args:
        url: https://api.mathjs.org/v4/
        query:
            expr: ${"log(" + string(multiplyResult.body.multiplied) + ")"}
    result: logResult
- floorFunction:
    call: http.post
    args:
        url: https://floor-<random-hash>.run.app
        auth:
            type: OIDC
        body:
            input: ${logResult.body}
    result: floorResult
- returnResult:
    return: ${floorResult}
```

### 6-3. Cloud Workflows のデプロイ
Cloud Workflows をデプロイします
```bash
gcloud beta workflows deploy workflow \
    --source=workflow.yaml \
    --service-account=${SERVICE_ACCOUNT}@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com
```

### 6-4. Cloud Workflows のデプロイ
サービスアカウントを指定した Cloud Workflows をデプロイします
```bash
gcloud beta workflows deploy workflow \
    --source=workflow.yaml \
    --service-account=${SERVICE_ACCOUNT}@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com
```

### 6-5. Cloud Workflows の実行
メニュー左側の Tools > Workflows を選択します  
image01  

`workflow` という名前のワークフローを選択します  
image02  

画面右上の `EXECUTE` を選択します  
image03  

画面左下の `EXECUTE` をクリックしワークフローを実行します  
image04  

画面右上の更新ボタンを押し、実行結果を確認します  
image05  

