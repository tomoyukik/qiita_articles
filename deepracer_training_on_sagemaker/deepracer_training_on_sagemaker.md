# DeepRacerのモデルをSageMakerで学習する

SageMakerを利用してDeepRacerのモデルを学習する手順をまとめている。
基本的な手順は以下の公式ドキュメントにまとまっているものの、一部理解が足りていなかった部分を補足する内容をまとめている。

<https://docs.aws.amazon.com/ja_jp/deepracer/latest/developerguide/train-evaluate-models-using-sagemaker-notebook.html>

まだ理解の甘い部分、誤っている部分が多分に残っているが、今後理解を深めるための第一歩としてご容赦いただきたい。

## 目標

SageMakerを活用してDeepRacerのモデル学習を行い、DeepRacerにエントリーできるモデルが構築できることを目標とする。

これを足がかりとして、 AWSの各サービスへの理解の深化やより柔軟なDeepRacerのモデル構築を進めていきたい。

## 目次

1. DeepRacerとは何か
1. なぜSageMakerを使いたいか。 SageMakerを使うことのメリット。
1. 必要な要素と構成
    - SageMakerとは
    - RoboMakerとは
    - Kinesis Video Streamとは
1. SageMakerを活用した学習の仕組み
1. 環境の構築
    - Imageの用意(SageMaker/RoboMaker)
    - RoboMakerアプリケーションの作成
    - Kinesis Video Streamの作成
1. 学習の方法
    - SageMakerでのジョブの作成
    - logの確認方法
1. モデルのインポート

## [DeepRacer](https://docs.aws.amazon.com/deepracer/latest/developerguide/what-is-deepracer.html)とは

自動運転のアプリケーションを構築することで、強化学習について学べるサービスである。
自動運転車に搭載するモデルに対して深層強化学習を適用し、学習したモデルばバーチャルレースなどのレースに出場させることができる。

あらゆるレベルのユーザが楽しめるように設計されており、初学者であっても報酬関数の設計のみで強化学習を体験することができる。
[AWS DeepRacer コンソール](https://docs.aws.amazon.com/ja_jp/deepracer/latest/developerguide/deepracer-console-train-evaluate-models.html)というインターフェースも用意されており、
報酬関数の設計であれば環境構築のコストをかけずに参加できるようになっている。

今回はDeepRacerコンソールを使わずに、SageMakerを使ってモデルの学習を行なった。

## SageMakerとは

[SageMaker](https://docs.aws.amazon.com/ja_jp/sagemaker/latest/dg/whatis.html)はフルマネージドの機械学習サービスである。
名前の通り機械学習モデルの構築を行う際に利用され、機械学習のあらゆるアルゴリズムが組み込まれており、簡単に機械学習を開始することができる。
学習や推論に必要なリソースを必要なときに必要な分だけプロビジョニングするため金銭的なコストも必要最低限に抑えられ、またフルマネージドのサービスであるため、保守運用等のコストも不要である。

Dockerコンテナを使用すると、独自のアルゴリズムを用いてモデルを構築することもできる。

## RoboMakerとは

強化学習用のモデルの構築には、[RoboMaker](https://docs.aws.amazon.com/ja_jp/robomaker/latest/dg/gettingstarted-concepts.html)を組み合わせて使用する。
RoboMakerはシミュレーションを行うことができるサービスで、構築したモデルの走行をRoboMaker上でシミュレートし、その走行結果をSageMakerにフィードバックすることで強化学習を実現する。

## SageMakerを利用することのメリット

DeepRacerのモデル構築にSageMakerを使用することで、

1. より柔軟な条件下でモデルを構築することができる。
1. 走行データの情報量を増やすことができる。

2022/12/17日現在、DeepRacerコンソール上での学習では、学習の条件がある程度制限されている。
例えば、モデルの学習環境として、多数のコースが用意されているがコースごとに周回の方向が定められている。
ほとんどのコースが反時計回りとなっており、自分の場合、右折や右カーブの学習に苦労した。
ところがSageMaker・RoboMakerを使うとパラメータを調整することでシミュレーションの際の周回の方向を変更することができ、
学習の条件を柔軟に変更しながら、さまざまな状況で学習することが可能になる。

DeepRacerでは報酬関数を設計して自動走行車を作り上げていくが、その過程で走行ログの評価が必要になる。
報酬関数を設計し作り上げたモデルの走行ログを分析し、報酬関数の修正に活用していく。
これがDeepRacerの醍醐味の一つであると思う。
コンソール上の学習においても当然走行ログの取得はできるのだが、SageMaker (RoboMaker) を使用すると車から見えている画像データの取得をすることもできる。
DeepRacerのモデルは、車に搭載されているカメラで走行中の画像を取得し、その画像をもとに推論を行い次のアクションを決めているため、
走行ログの分析に画像を用い、モデルが何を見てアクションを決めているかなどを分析することで、報酬関数を設計するにあたり有用な情報が得られるのではないだろうか。

## 環境の構築

まずはモデルのトレーニングに必要となる環境を構築する。

1. RoboMakerのシミュレーションアプリケーションの作成。
2. SageMakerで使用するDockerイメージの作成。
3. Kinesis Video Streamの作成。

モデルのアルゴリズム変更等を行わない限り、ここで作成したリソースは今後の学習において使いまわすことができる。
マルチユーザモードでDeepRacerをホストしている場合、これらのリソースはユーザ間で共有することができる。

### RoboMakerシミュレーションアプリケーションの作成

RoboMakerアプリケーションの作成は、公開ECRからDeepRacer用のイメージを参照するだけである。
[参照先](https://github.com/aws-deepracer/aws-deepracer-workshops/blob/master/dpr401/deepracer_rl.ipynb)の情報の通りであるため、詳細は省略する。

以下は`boto3`を使用してアプリケーションを作成するコードの例である。

```py
robomaker = boto3.client("robomaker")
robomaker.create_simulation_application(
    name=app_name, # アプリケーション名
    environment={"uri": "public.ecr.aws/k1d3r4z1/deepracer-sim-public:latest"},
    simulationSoftwareSuite={"name": "SimulationRuntime"},
    robotSoftwareSuite={"name": "General"}
)
```

### SageMakerで使用するImageの作成

今回はCodeBuildを使用してイメージを構築した。
イメージのビルド時に`docker`コマンドを使用しているので、ビルドプロジェクトでは特権モードを有効化している。

イメージの構築に使用しているソースのディレクトリ構成は下記の通り。

```txt:tree
./
├─ buildspec.yaml
└─ sagemaker/
   ├─ Dockerfile
   └─ redis.conf
```

後掲の`buildspec.yaml`の通り、
RoboMakerのアプリケーション構築時に使用したイメージ (`public.ecr.aws/k1d3r4z1/deepracer-sim-public`) からモデルのソースを取得している。

`Dockerfile`と`redis.conf`は、それぞれ基本的には下記を使用している。

- `Dockerfile`: <https://github.com/aws-deepracer/aws-deepracer-workshops/blob/master/dpr401/Dockerfile>
- `redis.conf`: <https://github.com/aws-deepracer/aws-deepracer-workshops/blob/master/dpr401/src/lib/redis.conf>

ただし、redisの起動がうまくいかず学習画で止まっていたためredisの設定には変更を加えている。

Redisの3.2.0以降、Protected modeというものを導入しているようで([protected mode](https://redis.io/docs/management/security/#protected-mode))、
これの無効化、あるいはパスワード認証を行う必要があるらしい。

`redis.conf`に以下の修正を加えて、protected-modeを無効化している。

```conf:redis.conf
protected-mode no
```

修正を加えた`redis.conf`をDockerイメージに反映させるため、Dockerfileにも修正を加えている。

```dockerfile:Dockerfile
COPY ./sagemaker/redis.conf /etc/redis/redis.conf
```

#### `buildspec.yaml`

CodeBuildで参照する`buildspec.yaml`の例である。

```yaml:buildspec.yaml
version: 0.2

env:
  variables:
    SIMULATION_IMAGE: public.ecr.aws/k1d3r4z1/deepracer-sim-public
    SAGEMAKER_IMAGE: sagemaker-docker-cpu
    IMAGE_TAG: latest

phases:
  install:
    on-failure: ABORT
    commands:
      - apt-get install -y jq
  pre_build:
    on-failure: ABORT
    commands:
      - AWS_ACCOUNT_ID=$(echo ${CODEBUILD_BUILD_ARN} | sed -E s/^.\*\(\[0-9\]\{12\}\).\*\$/\\1/)
      - echo "${AWS_ACCOUNT_ID}:${AWS_DEFAULT_REGION}/${AWS_REGION}|${AWS_DEFAULT_REGION}"
      - aws ecr get-login-password --region "${AWS_DEFAULT_REGION}" | docker login --username AWS --password-stdin "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
      - docker pull "${SIMULATION_IMAGE}"
      - IMAGE_ID=$(docker images --format '{{ json . }}' | grep ${SIMULATION_IMAGE} | jq -r '.ID')
      - CONTAINER_ID=$(docker run -d -t "${IMAGE_ID}")
      - mkdir -p ./sagemaker/src/lib
      - docker cp "${CONTAINER_ID}:/opt/amazon/markov" ./sagemaker/src/
      - docker cp "${CONTAINER_ID}:/opt/amazon/rl_coach.patch" ./sagemaker/src/
      - docker cp "${CONTAINER_ID}:/opt/ml/code/." ./sagemaker/src/lib/
  build:
    on-failure: ABORT
    commands:
      - docker build -t "${SAGEMAKER_IMAGE}" -f ./sagemaker/Dockerfile ./sagemaker
      - docker tag "${SAGEMAKER_IMAGE}:latest" "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME_SAGEMAKER}:${IMAGE_TAG}"
      - docker tag "${SAGEMAKER_IMAGE}:latest" "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME_SAGEMAKER}:latest"
  post_build:
    on-failure: ABORT
    commands:
      - docker push "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${SAGEMAKER_IMAGE}:${IMAGE_TAG}"
      - docker push "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${SAGEMAKER_IMAGE}:latest"
```

## モデルのトレーニング

SageMakerのジョブとRoboMakerのジョブを起動しモデルの学習を行う。

### 報酬関数とメタデータの定義

SageMakerとRoboMakerのジョブで参照する3ファイルをS3にアップロードする。

```txt:tree
{s3-bucket}/
 └─ {s3-prefix}/
     ├─ reward_function.py
     ├─ model_metadata.json
     └─ training_param.yaml
```

1. `reward_function.py`
    - 報酬関数を定義したPythonファイル。SageMakerとRoboMakerの両ジョブにて参照される。
1. `model_metadata.json`
    - アクションスペースなどのモデルのメタデータを定義したJSONファイル。SageMakerとRoboMakerの両ジョブにて参照される。
1. `training_param.yaml`
    - RoboMakerのジョブで利用する。報酬関数やメタデータファイルのパス、ログの出力先、SageMakerジョブ・Kinesis Video StreamのARNなどを指定する。

#### アクションスペースの定義

アクションスペースはメタデータファイルに定義する。
離散アクションスペース、連続アクションスペースでやや定義方法が変わるため、それぞれの例を以下に記載する。

##### 離散アクションスペースの定義

`"action_space"`をキーに、モデルがとりうる `(ステアリング角, 速度)` の組の配列を指定する。

```json:model_metadata.json
{
    "action_space": [
        { "steering_angle": -30, "speed": 1.0 },
        { "steering_angle": -15, "speed": 1.0 },
        { "steering_angle":   0, "speed": 1.0 },
        { "steering_angle":  15, "speed": 1.0 },
        { "steering_angle":  30, "speed": 1.0 }
    ],
    "sensor": [ "STEREO_CAMERAS" ],
    "neural_network": "DEEP_CONVOLUTIONAL_NETWORK_SHALLOW"
}
```

##### 連続アクションスペースの定義

`"action_space"`をキーに、モデルがとりうるステアリング角の範囲、速度の範囲を指定する。
`"action_space_type"`に連続アクションスペースである旨を示す`"continuous"`を指定する。

```json:model_metadata.json
{
    "action_space": {
        "steering_angle": { "high": 30, "low": -30 },
        "speed":          { "high":  4, "low":   2 }
    },
    "sensor": [ "FRONT_FACING_CAMERA", "SECTOR_LIDAR" ],
    "neural_network": "DEEP_CONVOLUTIONAL_NETWORK_SHALLOW",
    "version": "4",
    "training_algorithm": "clipped_ppo",
    "action_space_type":  "continuous",
    "preprocess_type":    "GREY_SCALE",
    "regional_parameters": [ "0", "0", "0", "0" ]
}
```

### SageMakerジョブの作成

モデルの学習を行うSageMakerのジョブを作成する。
RoboMaker上のシミュレーション結果をフィードバックとして受け取り学習が開始されるため、RoboMakerのジョブがローンチされてから学習が開始される。

```py
# ジョブ作成時に指定する
metric_definitions = [
    {"Name": "reward-training", "Regex": "^Training>.*Total reward=(.*?),"},
    {"Name": "ppo-surrogate-loss", "Regex": "^Policy training>.*Surrogate loss=(.*?),"},
    {"Name": "ppo-entropy", "Regex": "^Policy training>.*Entropy=(.*?),"},
    {"Name": "reward-testing", "Regex": "^Testing>.*Total reward=(.*?),"},
]

# ハイパーパラメータの設定
custom_hyperparameter = {
    "s3_bucket":  s3_bucket, # S3バケットの指定
    "s3_prefix":  s3_prefix, # S3プレフィクスの指定
    "aws_region": aws_region,
    "model_metadata_s3_key":     f"s3://{s3_bucket}/{s3_prefix}/model_metadata.json",
    "reward_function_s3_source": f"s3://{s3_bucket}/{s3_prefix}/reward_function.json",
    "batch_size": "64",
    "num_epochs": "10",
    "stack_size": "1",
    "lr":         "0.0003",
    "exploration_type": "Categorical",
    "e_greedy_value":   "1",
    "epsilon_steps":    "10000",
    "beta_entropy":     "0.01",
    "discount_factor":  "0.999",
    "loss_type":        "Huber",
    "num_episodes_between_training": "20",
    "max_sample_count":              "0",
    "sampling_frequency":            "1",
    # 学習済みモデルのバケットとプレフィクス
    # "pretrained_s3_bucket": s3_bucket,
    # "pretrained_s3_prefix": f"{s3_username}/deepracer-notebook-sagemaker-200729-202318",
}

# SageMakerジョブの作成
sagemaker = boto3.client("sagemaker", region_name=aws_region)
traning_job = sagemaker.create_training_job(
    TrainingJobName=training_job_name,
    HyperParameters=custom_hyperparameter,
    AlgorithmSpecification={
        "TrainingImage": f"{custom_image_uri}:latest",
        "TrainingInputMode": "File",
        'MetricDefinitions': metric_definitions
    },
    RoleArn=sagemaker_role,
    OutputDataConfig={
        "S3OutputPath": f"s3://{s3_bucket}/{s3_prefix}/train-output/",
    },
    ResourceConfig={
        'InstanceType': instance_type,
        'InstanceCount': 1,
        'VolumeSizeInGB': 32
    },
    VpcConfig={
        'SecurityGroupIds': deepracer_security_groups,
        'Subnets': deepracer_subnets
    },
    StoppingCondition={
        'MaxRuntimeInSeconds': job_duration_in_seconds
    },
)
```

### Kinesis Video Streamの作成

Kinesis Video Streamはシミュレーションジョブ数分作成し、各RoboMakerのジョブにストリーム名を指定する。
学習終了後はストリームの削除を行わないと、起動し続けてしまう。

```sh
aws --region {aws_region} kinesisvideo create-stream --stream-name {kvs_stream_name} --media-type video/h264 --data-retention-in-hours 24
```

### RoboMakerジョブの作成

[`training_param.yaml`](https://github.com/aws-deepracer/aws-deepracer-workshops/tree/master/dpr401/src/artifacts/yaml)には、コースの指定やレースの種類などを指定し、RoboMakerのジョブにて参照する。
RoboMakerのジョブをローンチすることで学習が開始される。
ジョブのローンチのコードは[こちら](https://github.com/aws-deepracer/aws-deepracer-workshops/blob/master/dpr401/deepracer_rl.ipynb)の通りなので省略する。

## モデルのインポート

モデルの学習が完了すると、 DeepRacerコンソールからモデルをインポートすることができる。
ただし、インポートに必要なディレクトリの構成が決まっているようで (<https://docs.aws.amazon.com/ja_jp/deepracer/latest/developerguide/deepracer-troubleshooting-service-migration-errors.html#s3-folder-structure>) 、
それに従う必要がある。

出力の構成例を後掲しているが、`ip`ディレクトリと`model`ディレクトリが必要になるらしい。

また、残念ながら2022年12月17日現在マルチユーザモードで各ユーザがモデルをインポートする手段はないようで、マルチユーザモード下では管理者ユーザがユーザを指定してインポートする必要がある。

### 出力の構成例

```txt
s3://{buckket}/{prefix}/
└─ deepracer-training-sagemaker-*****-*****/
    │
    │  # モデルアーティファクト
    ├─ ip/
    │   ├─ done
    │   ├─ hyperparameters.json
    │   └─ ip.json
    ├─ model/
    │   ├─  .coach_checkpoint
    │   ├─  .ready
    │   ├─  {N}_Step-9605.ckpt.data-00000-of-00001
    │   ├─  {N}_Step-9605.ckpt.index
    │   ├─  {N}_Step-9605.ckpt.meta
    │   ├─  deepracer_checkpoints.json
    │   ├─  model_{N}.pb
    │   └─  model_metadata.json
    │
    │  # ログデータ
    ├─ iteration-data/training/
    │   ├─ camera-45degree/
    │   │  └─ {N}-video.mp4
    │   ├─ camera-pip/
    │   │  └─ {N}-video.mp4
    │   ├─ camera-topview/
    │   │  └─ {N}-video.mp4
    │   └─ training-simtrace/
    │      └─ {N}-iteration.csv
    ├─ sim-y41231383h9g/2022-12-14T13-40-11.900Z_2e900cea-0679-4053-99aa-81f54f6cc29c/
    │   ├─ gazebo-logs/server-11345/
    │   │   ├─ default.log
    │   │   └─ gzserver.log
    │   └─ ros-logs/
    │       ├─ 1fbeb178-7bb5-11ed-b6eb-0242a9fe0102/
    │       │   ├─ agents_video_editor-6.log
    │       │   ├─ car_reset_node-5.log
    │       │   ├─ download_params_and_roslaunch_agent_node-2.log
    │       │   ├─ master.log
    │       │   ├─ racecar-controller_manager-2.log
    │       │   ├─ racecar-racecar_spawn-1.log
    │       │   ├─ roslaunch-712974d3eac6-1.log
    │       │   ├─ roslaunch-712974d3eac6-435.log
    │       │   ├─ rosout-1-stdout.log
    │       │   ├─ rosout.log
    │       └─ rl_coach_545_1671025341644.log
    ├─ train-output/deepracer-training-sagemaker--*****-*****/output/model.tar.gz
    ├─ INITIAL_S3_WRITE_KEY
    │
    │  # 入力
    ├─ customer_reward_function.py
    ├─ training_metrics.json
    └─ training_params.yaml
```

## 参考

- <https://docs.aws.amazon.com/ja_jp/deepracer/latest/developerguide/train-evaluate-models-using-sagemaker-notebook.html>
- <https://github.com/aws-deepracer/aws-deepracer-workshops/blob/master/dpr401/deepracer_rl.ipynb>
