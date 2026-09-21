## 開発者コンソールと匿名実行ウィンドウ

Apexクラスの開発及びテストをweb上で実行でできる機能
添付は生成したオブジェクトをdebugログに表示している

![alt text](imgs/image.png)

UI上からアクセスできない（レコードページに非表示）の項目のupdateなどにも使える

## コードビルダーとSalesforce CLI

Apex及びLWCの開発はVScodeのコードビルダー上でも実行できる。
コードビルダーではプロジェクト（一つの開発ディレクトリ）を作成してコーディングを行う。
通常のコードベースアプリケーションと異なり、プロジェクトとSalesforce組織は多対多でつながる。

```mermaid
flowchart LR
    subgraph Projects [コードビルダー プロジェクト]
        P1[プロジェクトA]
        P2[プロジェクトB]
    end

    subgraph Orgs [Salesforce組織]
        O1[スクラッチ組織 1]
        O2[サンドボックス組織 2]
        O3[開発者用組織 3]
    end

    P1 <-->|接続・デプロイ| O1
    P1 <-->|接続・デプロイ| O2
    P2 <-->|接続・デプロイ| O2
    P2 <-->|接続・デプロイ| O3
```

### コード管理

上記のように複数組織にプロジェクトをデプロイできるうえに、Web UI上でのカスタムオブジェクトの追加やフローの構築、APEXコードの追加が発生する。
そのため、開発プロジェクトでは変更方法にかかわらずgit repositoryを正とする運用を行うことがベストプラクティスとなっている。
Web UI上で構築した機能をコード化するには、Salesforce CLIまたはプロジェクトにて`SFDX: Retrieve Source from
  Org`を実行することでコードをプロジェクトに反映できる。

### スクラッチ組織

フローの構築などはxmlコードベースではなくフロービルダーで構築する方が圧倒的に早い。
しかし開発環境などCICDの対象となっている環境をWeb UIから変更してしまうと、CICDによるデプロイとコンフリクトしてしまう（実際には気が付かずに上書きされる）。
これを避けるためにスクラッチ組織を利用する。

スクラッチ組織とは有効期限30日間の一時的な組織である。
作成はCLIから`sf org create scratch`を実行するだけで簡単に構築できる。
フロービルダーでのフロー構築などはスクラッチ組織で行い、コードベースで実環境にデプロイするというのがベストプラクティスになっている。

## よくあるCICDフロー

開発環境や本番環境（開発組織）のデプロイ衝突を防ぐため、開発者は使い捨ての「スクラッチ組織」を活用して開発を行い、Gitリポジトリを唯一の「正」として管理。

```mermaid
---
title: Salesforce開発フロー (ソース駆動開発)
---
sequenceDiagram
    autonumber
    actor Dev as 開発者 (Local Git)
    participant Git as Git Repository
    participant Prod as 開発組織 (本番・実環境)

    Note over Git, Prod: 【初期状態】<br>開発組織の最新コード（全量）が<br>Gitリポジトリに配置済み

    %% 1. 開発の準備
    Dev->>Git: git clone / pull (最新コードの取得)

    %% 2. スクラッチ組織の生成（ライフサイクル開始）
    create participant Scratch as スクラッチ組織
    Dev->>Scratch: sf org create scratch (組織生成)

    rect rgba(172, 255, 252, 0.3)
        Note right of Dev: スクラッチ組織での開発・検証フェーズ
        %% 3. ソースのデプロイ
        Dev->>Scratch: sf project deploy start (既存コードを反映)

        %% 4. スクラッチ組織上での開発
        Dev->>Scratch: フローやメタデータを構築・変更

        %% 5. 変更の取得
        Dev->>Scratch: sf project retrieve start (変更分をローカルに取得)
        Scratch-->>Dev: メタデータ定義ファイル (.xml / .cls 等)
    end

    %% 6. リポジトリへの反映
    Dev->>Dev: git commit / test (変更のコミット・ローカル検証)
    Dev->>Git: git push (リポジトリへプッシュ)

    %% 7. スクラッチ組織の破棄（ライフサイクル終了）
    destroy Scratch
    Dev->>Scratch: sf org delete scratch (検証完了後の組織削除)

    %% 8. CI/CDデプロイ
    Note over Git, Prod: CI/CDワークフロー自動起動<br>(本番/実環境への自動デプロイ)
    Git->>Prod: sf project deploy start (本番/検証環境へデプロイ)
```

<br>
※変更セットはクラシカルなUI上でリリースする手段。アドミン向けの一切コードを書かない標準機能のみの構築以外では基本使わないと思ってよい。