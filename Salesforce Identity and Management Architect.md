# Salesforce Identity and Access Management (IAM) Architect

## TODO

- [x] YouTube視聴
- [ ] ハンズオン実施
- [ ] 問題集を解く
- [ ] SAMLアサーションの具体的な内容を確認する
- [ ] SAMLプロトコルにおいてSPとidPに設定する証明書について勉強する
  - [ ] JWTの署名とかもあわせて勉強
- [ ] APEX、LWCを簡単に勉強

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
    - モバイルアプリなどで認可コードの横取りを防止する
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

- OpenID Connect
  - OAuth2.0をベースに構築されたSSOプロトコル
  - IDトークンという身元照会用のトークンを発行することがポイント
  - Salesforceでは認証プロバイダー機能でOpenID Connectを設定できる
    - アプリケーション側がOIDCに対応していないならカスタムAPEXを利用する

**OIDC Flow**

---
```mermaid
sequenceDiagram
    autonumber
    actor User as ユーザー
    participant Client as クライアントアプリ<br>(Web / モバイル)
    participant AuthServer as 認可サーバー<br>(IdP: Google / Entra ID 等)
    participant ResourceServer as リソースサーバー<br>(API)

    Note over User, ResourceServer: 1. 認可リクエスト（認証の要求）
    User->>Client: アプリにアクセス / ログインボタン押下
    Client->>AuthServer: 認証リクエスト送信<br>(response_type=code, client_id, scope=openid profile, redirect_uri)

    Note over User, ResourceServer: 2. ユーザーの認証と同意
    AuthServer->>User: ログイン画面の表示（ID/パスワード・MFA等）
    User->>AuthServer: 資格情報の入力
    AuthServer->>User: アクセス許可の確認（同意画面）
    User->>AuthServer: 同意

    Note over User, ResourceServer: 3. 認可コードの返却
    AuthServer->>Client: リダイレクト + 認可コード (authorization_code) 返却

    Note over User, ResourceServer: 4. トークン交換（バックチャネル）
    Client->>AuthServer: トークンリクエスト送信<br>(code, client_secret, redirect_uri)
    AuthServer->>Client: **IDトークン (JWT)** + アクセストークン返却

    Note over User, ResourceServer: 5. 認証の検証とAPIアクセス
    Client->>Client: IDトークン（署名・有効期限）を検証してユーザー識別子（sub）を取得
    Client->>ResourceServer: APIリクエスト (Bearer アクセストークン)
    ResourceServer->>Client: 保護されたリソースを返却
    Client->>User: ログイン完了・画面表示
```
---

- Salesforce IAMの基本機能
  - MFA：本番環境（Experience Cloudを利用する外部ユーザーの除く）は必須
    - AuthアプリやYubikey、FaceIDなどの組み込み生体認証が使える
    - U2F：物理的なキーを挿入する方式
    - WebAuthn
    - FIDO2
  - Lightning Login
    - パスワードレスでSF Authアプリを利用してログイン
    - MFAを満たしているとみなされる
  - 証明書ベース
    - クライアント証明書をデバイスにインストールして認証
  - ログインフロー
    - 認証プロセス中にカスタムロジックを組み込む
    - 特定のIPからの接続の時のみセッションベースの権限セットを与えるなど（セッション切れ時に権限セットははく奪）
  - SCIM
    - ID管理規格
    - REST APIでユーザー情報を扱える
  - 接続アプリケーションのユーザープロビジョニング
  - アプリケーションランチャーに各アプリを置けばSSOできる
- Experience Cloud
  - SFのデータとシームレスに統合できるwebサイトを構築できる機能（主にB2B向けのポータルサイト）
    - 例えばイベント管理系の会社がSFを導入したとして、チケット購入サイトをExperience Cloudで構築してデータはSFオブジェクトに保持みたいな感じ
  - IAM関連
    - Identity Only：ID管理特化ライセンス
    - External Identity：Experience Cloudを利用する外部向けライセンス
    - ID検証クレジットアドオンライセンス：SMSを利用した検証用の追加ライセンス
  - セルフ登録
    - ポータルがある前提で、顧客やパートナーが自身でアカウント登録できる機能
    - プロファイルの設定なども自動化可能
  - 動的URLによるIDページのブランド設定
    - ユーザーの属性に応じてユーザーに最適なページを表示する機能
  - パスワードレスログイン
    - パスワード管理無しでSMS認証などでログインできる
    - こういった機能を実装するためのAPEXインターフェースが用意されているので、APEXクラスとして実装する
  - ヘッドレスAPI
- Summer24での補足
  - 代理認証：外部認証システムが認証を行う仕組み
    - SFでユーザー名を入力するとSFは外部認証サーバーに流す
    - SOAPベースでRESTはない
    - SAMLがないときに役に立つ
    - 認証情報が外部にあるので、SF上でパスワードリセットできない
  - Identity Connect
    - Active DierctoryからSFへユーザーの同期をできる機能
    - 廃止済み（試験には出る）、EntraIDなどを利用して同様の機能を実現できる
  - Experience Cloud組み込みログイン（非推奨）
    - Experience Cloudのログインを外部サイトに統合する機能
    - 現在はOAuthでの
  - CANVAS
    - 外部アプリとSFを統合する機能
    - LWCが一般的で優先度がひくい