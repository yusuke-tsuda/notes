# Databricks Certified Data Engineer Associate 学習


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

## Databricks Asset Bundles

- 基本ファイル
  - ./databricks.yml：リリースターゲットなどのDABsの基本設定を記載
  - ./resources/xx.job.yml, xx.paipeline.yml：ジョブやETLパイプラインの定義を記載
  - ./src：SQLファイル、Pythonファイル、パイプラインの処理などのソースコードを記載
