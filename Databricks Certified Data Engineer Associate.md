# Databricks Certified Data Engineer Associate 学習


## 基礎知識
- コンピュートの種類
  - インタラクティブコンピュート：高速起動でサクッとつかえる
  - ジョブコンピュート：ジョブやパイプラインを実行するためのコンピュート
  - SQLウェアハウス：SQLの高速実行に特化したコンピュート

## 構文
- データマスクはユーザーグループと関数で実行する

```sql
-- patientてーぶる# Databricks Certified Data Engineer Associate 学習


## 基礎知識
- コンピュートの種類
  - インタラクティブコンピュート：高速起動でサクッとつかえる
  - ジョブコンピュート：ジョブやパイプラインを実行するためのコンピュート
  - SQLウェアハウス：SQLの高速実行に特化したコンピュート

## 構文
- データマスクはユーザーグループと関数で実行する

```sql
--doctorグループにだけ項目が見えるようにする関数
CREATE FUNCTION not_doctor_mask(val STRING)
RETURN
CASE
  WHEN is_account_group_member('doctors') THEN val
  ELSE 'CONFIDENTIAL'
END
```

  - これをVIEWで`SELECT not_doctor_mask(culumn_name) FROM ...`として利用することでマスクする。
