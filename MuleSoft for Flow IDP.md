# MuleSoft for Flow : IDP
## 参考資料

- [デモ動画](https://videos.mulesoft.com/watch/PtGxds2xs4DjVHUKY1fGpo)
- [ハンズオン資料](https://jpn-codelabs.pub.codelabs.tools.mulesoft.com/codelabs/a0AKY000000A9k42AC/index.html#0)（実際に実行するには専用のライセンスが必要）

## IDPの基本機能

- Salesforce core上で動くIDP
  - 名前にMuleSoftが入っているのは、どうやらMuleSoft IDPをSalesforce向けにカスタムしたものだからぽい
- 基本的にSalesforceにアップロードされたファイルからデータを抽出する
- あらかじめ読み取り項目と項目ごとのプロンプトを定義しておき、定義した内容をフロービルダーから呼び出すことが基本
- 読み取り精度が低い場合はIDP側でフローを中断しhuman in the loopを実行することが可能

## ユースケース

(1) 標準のファイル機能にファイルアップロード後、レコードトリガーで起動してファイル内容からレコード作成

```mermaid
---
title: MuleSoft for Flow:IDP（レコードトリガーフローで起動のイメージ）
---

sequenceDiagram
autonumber
    actor U as ユーザー
    participant FS as Salesforce ファイル
    participant RD as レコードトリガーフロー
    participant IDP as IDP
    participant O as Salesforce オブジェクト

    U ->> FS:ファイルアップロード（PDF/jpg/png）
    FS ->>+ RD: ファイルアップロードをトリガーにフロー起動
    RD ->>+ IDP: トリガーとなったファイルに対してIDP実行
    Note over IDP:対象ファイルの読み取りを実行
    alt 読み取り精度が事前設定の閾値より低い場合
        IDP ->> U: ユーザーに確認、内容修正依頼
        U -->> IDP: 修正内容、承認
    end

    IDP -->>- RD: 読み取り結果を返却
    RD ->>- O: IDPの読み取り結果をレコードに登録

```

(2) 画面フローからファイルをアップロードして、レコードを作成するイメージ

```mermaid
---
title: MuleSoft for Flow:IDP（画面フローで起動のイメージ）
---

sequenceDiagram
autonumber
    actor U as ユーザー
    participant GF as 画面フロー
    participant FS as Salesforce ファイル
    participant IDP as IDP
    participant O as Salesforce オブジェクト

    U ->>+ GF: 画面フローにファイル（PDF/jpg/png）をアップロード
    GF ->> FS: ファイル保存
    GF ->>+ IDP: 対象ファイルの読み取りを実行
    IDP -->>- GF: 読み取り結果返却
    GF ->> U: 読み取り内容の確認、承認依頼
    U -->> GF: 承認
    GF ->>- O: 読み取り結果をレコードに登録
```