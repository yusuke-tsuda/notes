# Salesforce Identity and Access Management (IAM) Architect

## TODO

- [ ] YouTube視聴
- [ ] ハンズオン実施
- [ ] 問題集を解く
- [ ] SAMLアサーションの具体的な内容を確認する
- [ ] SAMLプロトコルにおいてSPとidPに設定する証明書について勉強する
  - [ ] JWTの署名とかもあわせて勉強
- [ ] APEXを簡単に勉強

## YouTubeメモ

- IAMはユーザーの認証とアクセス権の管理を行う仕組み
- 認証
  - 認証されたユーザーに対して何ができるかを権限付与するプロセス
- 認可
  - リソースへのアクセスと操作の権限を付与するプロセス
- SSO
  - アプリ共通でログインできるようにする仕組み
  - 自動プロビジョニングで不適切なアクセスを防止したり、パスワードリセットなどの管理業務の削減のメリットがある
  - ※ ただしidPのアカウント流出リスクはある
- SAML
  - SSOを実現するプロトコル
  - idP
    - ユーザーの認証を行うプロットフォーム（EntraIDやOktaが代表格）
  - SP
    - ユーザーが利用するアプリケーションなどのサービス
  - SAMLアサーション
    - idPが生成する認証情報や属性情報が含まれるXMLデータ
    - これによりSP側でもユーザーが認証される
- SFはidPとしてもSPとしても使える

**SAML認証フロー**

---
```mermaid
---
title: SAML idP Initiated Flow
---
sequenceDiagram
    autonumber
    actor User as ユーザー（ブラウザ）
    participant IdP as IdP
    participant SP as SP

    User->>IdP: 1. IdPポータルにアクセス・認証
    IdP-->>User: 2. ログイン成功（SP一覧表示）
    
    User->>IdP: 3. SPアイコン（アプリ）をクリック
    Note over IdP: SAML Response (Assertion) を生成・署名
    IdP-->>User: 4. SAML Response を含むHTMLフォーム返却 (Auto-POST)

    Note over User: JavaScriptによる自動POST実行
    User->>SP: 5. SAML Response をPOST (ACS URL宛)
    Note over SP: 署名検証・Assertion解読・セッション作成

    SP-->>User: 6. ログイン完了（リソース画面の表示）

```
---
```mermaid
---
title: SAML SP Initiated Flow
---
sequenceDiagram
    autonumber
    actor User as ユーザー（ブラウザ）
    participant SP as SP
    participant IdP as IdP

    User->>SP: 1. SPの保護リソースへアクセス
    Note over SP: 未認証を検知し SAML Request を生成
    SP-->>User: 2. IdPへリダイレクト (SAML Request 添付)<br>どのIdPにリダイレクトするかはSP実装依存

    User->>IdP: 3. SAML Request を送信 (HTTP Redirect / POST)
    Note over IdP: ユーザーの未認証を確認
    IdP-->>User: 4. 認証画面（ログインフォーム）の表示

    User->>IdP: 5. 資格情報（ID/PW等）を送信
    Note over IdP: 認証・SAML Response (Assertion) を生成・署名
    IdP-->>User: 6. SAML Response を含むHTMLフォーム返却 (Auto-POST)

    Note over User: JavaScriptによる自動POST実行
    User->>SP: 7. SAML Response をPOST (ACS URL宛)
    Note over SP: 署名検証・Assertion解読・セッション作成

    SP-->>User: 8. ログイン完了（要求リソースへアクセス許可）
```
---

- OAuth 2.0
  - ユーザーの認証情報を共有せずにアプリケーションがリソースへアクセスするための認可プロトコル
  - 認証サーバーと認可サーバーは論理的には分かれているが、実装は基本同じサーバーにあることが多い

**OAuth認可フローの代表例**

---
```mermaid
---
title: OAuth 2.0 Authorization Code Flow
---
sequenceDiagram
    autonumber
    actor User as ユーザー (リソースオーナー)
    participant App as アプリケーション（Client）
    participant Auth as 認可サーバー
    participant Resource as リソースサーバー

    User->>App: 1. サービス利用開始（ログイン要求）
    App-->>User: 2. 認可サーバーへリダイレクト (client_id, redirect_uri 等)

    User->>Auth: 3. 認可エンドポイントへアクセス
    Auth-->>User: 4. 認証画面・権限同意画面の表示
    User->>Auth: 5. ログインおよびアクセス許可（同意）

    Note over Auth: 認可コード (Authorization Code) を発行
    Auth-->>User: 6. リダイレクトURIへ転送 (認可コードを付与)
    User->>App: 7. リダイレクトURIへアクセス (認可コードを渡す)

    Note over App: バックエンドから認可サーバーへ直接通信
    App->>Auth: 8. アクセストークン要求 (認可コード + client_secret)
    Note over Auth: 認可コードとクライアント検証
    Auth-->>App: 9. アクセストークン（およびリフレッシュトークン）返却

    App->>Resource: 10. リソース要求 (Authorization: Bearer <トークン>)
    Note over Resource: トークンの有効性検証
    Resource-->>App: 11. 保護リソース返却
    App-->>User: 12. サービス画面表示
```
---

- OAuth Client Credential Flowはユーザー情報を利用せずにclient_id/client_secretで認可するフロー（8以降のみ実施のイメージ）
  - 一時的なトークンをやり取るすることでsecret流出リスクを減らす構造

**OAuth 2.0の主要フロー** ※SFでの呼称が異なるので合わせて記載

# **OAuth 2.0 / 認証の主要フロー（拡張）**

| 一般的な呼称 | SFでの呼称 | 具体的な用途・ユースケース |
| :--- | :--- | :--- |
| **Authorization Code** | **Webサーバーフロー** | サーバーサイド言語で構築された**自社WebアプリやSaaS**から、**ユーザー個人の権限**でSalesforce APIを実行する場合。**Client Secretを安全に保持できる**構成で利用。 |
| **Implicit** | **ユーザーエージェントフロー** | **SPAやモバイルアプリ**など、ブラウザ/デバイス上で直接動作し**Client Secretを隠蔽できない**クライアント向け。（※現在は**PKCE付きWebサーバーフロー**への移行が推奨） |
| **JWT Bearer** | **JWTベアラーフロー** | **バッチ処理やバックエンドシステム**から、**画面操作なしで自動連携**する場合。**証明書（秘密鍵）**で署名し、**管理者による事前承認**のもと指定ユーザーの代理実行が可能。 |
| **Client Credentials** | **クライアントログイン情報フロー** | 個人の権限ではなく、**外部システムの専用サービスアカウント**として直接Salesforce APIを叩くインテグレーション。画面を介さない**純粋なシステム間（M2M）連携**。 |
| **SAML Bearer Assertion** | **SAML 2.0 ベアラーアサーションフロー** | **外部IdP（OktaやAzure ADなど）で既に認証済み**の場合に、その**SAMLアサーションを流用**してSalesforceのアクセストークンを取得するサーバー間・SSO連携。 |
| **Device Flow** | **デバイスフロー** | スマートTVやCLIツール、IoT機器など、**入力やブラウザ機能が制限されたデバイス**からアクセスする場合。**別の端末（スマホやPC）のブラウザで認証**を完了させる。 |
| **Asset Token Flow** | **アセットトークンフロー** | **IoTデバイスやコネクテッド機器**などの「モノ（アセット）」を対象とし、**デバイスごとのコンテキストや識別情報**を紐づけてSalesforceへ安全に接続する場合。 |

※ リフレッシュトークンフローは再認証をせずに、トークンを再発行することでセキュリティリスクを抑えつつ利便性を高める仕組み

- OAuthの設定 ※詳細はハンズオンで勉強
  - SFでは接続アプリケーションで行う
  - ユーザーの権限を設定
  - スコープの設定
  - アクセストークンの取り消し
    - 取り消し用エンドポイントにトークンを送付することで取り消しする
    - 管理者が画面から取り消すことも可能
  - PKCE (ピクシー)
    - モバイルアプリなどで認可コードの横取りを棒する
    - クライアントが認証時と認可時に同じかをcode_challenge, code_velifierを利用して確かめるフロー

**PKCEフロー**

---
```mermaid
---
title: OAuth 2.0 Authorization Code Grant with PKCE Flow
---
sequenceDiagram
    autonumber
    actor User as ユーザー（ブラウザ/アプリ）
    participant App as モバイル/SPAアプリ (Client)
    participant Auth as 認可サーバー (Salesforce等)
    participant RS as リソースサーバー (API)

    Note over App: 1. code_verifier (ランダム文字列) を生成<br/>2. SHA-256ハッシュ化して code_challenge を作成

    User->>App: 3. ログイン要求
    App-->>User: 4. 認可画面へリダイレクト<br/>(code_challenge, code_challenge_method=S256 添付)
    User->>Auth: 5. 認可エンドポイントへアクセス (code_challenge 送信)
    Auth-->>User: 6. 認証・アクセス許可画面の表示

    User->>Auth: 7. 資格情報入力 ＆ 承認
    Note over Auth: code_challenge と認可コードを紐づけて保持
    Auth-->>User: 8. リダイレクトURIへ戻す (認可コード 添付)

    User->>App: 9. アプリに認可コードが渡る
    App->>Auth: 10. アクセストークン要求<br/>(認可コード ＋ **code_verifier** 送信)

    Note over Auth: **【横取り防止の検証】**<br/>11. 受け取った code_verifier をS256ハッシュ化<br/>12. 保持していた code_challenge と一致するか検証

    Auth-->>App: 13. 検証成功：アクセストークン返却
    App->>RS: 14. API要求 (アクセストークン添付)
    RS-->>App: 15. リソース返却
    App-->>User: 16. 画面表示
```
---