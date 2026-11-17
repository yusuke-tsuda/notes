# Databricks Certified Data Engineer Associate 学習

## todo

- auto cdcの機能について勉強
- jobとpipelineの具体的な違いについて勉強
  - 

## 基礎知識
- コンピュートの種類
  - インタラクティブコンピュート：高速起動でサクッとつかえる
  - ジョブコンピュート：ジョブやパイプラインを実行するためのコンピュート
  - SQLウェアハウス：SQLの高速実行に特化したコンピュート
 
- delta tableのpartition
  - delta tableにもpostgreSQLに似たpartitionをつけることができる
  - parquitを指定したカラムの値の分類に沿ってフォルダ分けして格納して、検索時はそれぞれのフォルダを検索する。
    - 上手く使えば必要なファイルだけにアクセスできるので処理が早くなる
    - 一方でカラムの値に偏りがあるような場合（data skew）、joinやgroup byなどのシャッフル処理が遅くなる可能性もある
  - Databricksは少量データ（1TB未満）はpartitionしないことを推奨
  - また新規のテーブルはLiquid Clusterの利用を推奨している

- Databricksと外部ストレージの接続
  - 自動セットアップ
    - [自動セットアップの方法](https://docs.databricks.com/aws/ja/connect/unity-catalog/cloud-storage/s3/automatedsetups3)
    - [IAM権限の委譲](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies-temporary-delegation.html)
  - 手動セットアップ
    - [手動セットアップの方法](https://docs.databricks.com/aws/ja/connect/unity-catalog/cloud-storage/s3/s3-external-location-manual)

- jobとpipeline
  - それぞれ色々な機能があるが、pipelineの最大の特色はマテビューと宣言的ストリーミングテーブルとのこと
  - 単純にマテビューやストリーミングテーブルを宣言的に作成しておくだけで、pipelineが静的解析を行い差分連携をしてくれる
    - いちいち実行順序を指定したりする必要もない
  - 一方で実行順序の制限や複雑なエラー処理、夜間バッチなどはjobが向いている

- VACUUM
  - Delte TableにもVCUUMがある
  - 実態はparquitファイルの整理で、ファイル整理によりtime tlavelが失われる
  - マネージドでAuto Vacuumしている他、`VACUUM table_name`で能動的に実行も可能

## 構文
- データマスクはユーザーグループと関数で実行する

```sql
-- doctorsグループにだけ項目が見えるようにする関数
CREATE FUNCTION not_doctor_mask(val STRING)
RETURN
CASE
  WHEN is_account_group_member('doctors') THEN val
  ELSE 'CONFIDENTIAL'
END;

-- VIEWで関数を利用することでマスクする
CREATE VIEW patient_masked_view AS
SELECT id, not_doctor_mask(name) FROM patient_records;
```

- 外部テーブルを管理テーブルに昇格させるALTER文は引っかかりやすいので注意
  - `ALTER TABLE xxx SET MANAGED; `

- df.summary()関数
  - 平均値、最大値、最小値など一通りの統計データを出力してくれるカラム
  - 引数でカラムを指定する（引数が空だと全カラムの統計情報を出力）

- stream file読み書きは専用のAPIを利用

```python
spark.readStream
      .format("cloudFiles")
      .option("cloudFiles.format","json")
      .load(orderLocation)
    .writeStream
      .option("checkpointLocation",checkpointPath)
      .table("uncleanedOrders")
```

- 権限付与のSQL

```sql
GRANT :privilege_name ON :table_name TO :role_name;
```

## Databricks Asset Bundles

- 基本ファイル
  - ./databricks.yml：リリースターゲットなどのDABsの基本設定を記載
  - ./resources/xx.job.yml, xx.paipeline.yml：ジョブやETLパイプラインの定義を記載
  - ./src：SQLファイル、Pythonファイル、パイプラインの処理などのソースコードを記載

- databricks.ymlの記述例

```yaml
bundle: # bundleの基本情報
  name: test_bundle
  uuid: fb9740e4-11a3-4774-ade0-9f9d04f7c429

variables: # bundle内で使える変数定義
  schema_name:
    description: xxx
    default: demo_schema


include: #resourceセクションで直接job定義も書けるが、resources配下に分けてincludeする方がスマート
  - resources/*.yml

target: #deploy設定を環境ごとに記載
  dev:
    mode: dvelopment
    default: true
    workspace:
      host: https://abcd.com
    variables:
      schema_name: dev_schema
  prod:
    mode: production
    host: https://xyz.com
    variables:
      schema_name: prod_schema
```

- DABsコマンド　※実態はdatabricks CLIのサブコマンド群
  - databricks bundle deploy：指定のワークスペースにデプロイ
  - databricks bundle run：指定のワークスペースにデプロイされているジョブを起動
  - databricks bundle validate：静的構文チェック
  - databricks bundle test；ワークスペースでテストを実行
    - testはワークスペースで実行されるため、先にdeployしないとできない
    - そのため静的構文チェック→pytestなどをローカルでやると良い（CICDに必須ではない）