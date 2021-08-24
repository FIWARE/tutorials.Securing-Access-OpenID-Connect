[![FIWARE Banner](https://fiware.github.io/tutorials.Securing-Access/img/fiware.png)](https://www.fiware.org/developers)

[![FIWARE Security](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/security.svg)](https://github.com/FIWARE/catalogue/blob/master/security/README.md)
[![License: MIT](https://img.shields.io/github/license/fiware/tutorials.Securing-Access.svg)](https://opensource.org/licenses/MIT)
[![Support badge](https://img.shields.io/badge/tag-fiware-orange.svg?logo=stackoverflow)](https://stackoverflow.com/questions/tagged/fiware)
[![OpenID 1.0](https://img.shields.io/badge/OpenID-1.0-ff7059.svg)](https://openid.net/specs/openid-connect-core-1_0.html)
<br/>
[![Documentation](https://img.shields.io/readthedocs/fiware-tutorials.svg)](https://fiware-tutorials.rtfd.io)

このチュートリアルは、以前の
[セキュリティで保護されたアクセスのチュートリアル](https://github.com/FIWARE/tutorials.Securing-Access)
を補足するものです。このチュートリアルでは、FIWARE アプリケーションへのアクセスも保護しますが、さまざまな
OpenID Connect フローを使用してユーザを認証します。

## コンテンツ

<details>
<summary><strong>詳細</strong></summary>

-   [ID の認証](#authenticating-identities)
    -   [:arrow_forward: ビデオ: OpenID Connect とは何ですか？](#arrow_forward-video-what-is-openid-connect)
    -   [JSON Web Tokens の標準概念](#standard-concepts-of-json-web-tokens)
-   [前提条件](#prerequisites)
    -   [Docker](#docker)
    -   [Cygwin](#cygwin)
-   [アーキテクチャ](#architecture)
    -   [チュートリアルのセキュリティ設定](#tutorial-security-configuration)
-   [起動](#start-up)
    -   [登場人物 (Dramatis Personae)](#dramatis-personae)
-   [OIDC フロー](#oidc-flows)
    -   [OpenID Connect の有効化](#enable-openid-connect)
        -   [GUI](#gui)
        -   [REST API](#rest-api)
    -   [認可コード・フロー (Authorization Code Flow)](#authorization-code-flow)
        -   [認可コード - サンプル・コード](#authorization-code---sample-code)
        -   [認可コード - サンプルの実行](#authorization-code---running-the-example)
    -   [暗黙フロー](#implicit-flow)
        -   [暗黙フロー - サンプル・コード](#implicit-flow---sample-code)
        -   [暗黙フロー - サンプルの実行](#implicit-flow---running-the-example)
    -   [ハイブリッド・フロー](#hybrid-flow)
        -   [ハイブリッド - サンプル・コード](#authorization-code---sample-code)
        -   [ハイブリッド - サンプルの実行](#authorization-code---running-the-example)

</details>

<a name="authenticating-identities"/>

# ID の認証 (Authenticating Identities)

> "Yes, your home is your castle, but it is also your identity
> and your possibility to be open to others.
>
> — David Soul

デジタル ID (Digital identities) は、人々の特性とインターネット上で実行されるアクションの両方を表します。
アプリケーションを保護するためには、ID が本当に本人であることを認証する必要があります。FIWARE **Keyrock**
generic enabler は OAuth 2.0 に加えて、[OpenID Connect](https://openid.net/connect/) (OIDC) をサポートし、
サードパーティ・アプリケーションがユーザを認証できるようにします。**OpenID Connect** は、OAuth 2.0 プロトコルの上に
あるシンプルな ID レイヤです。[JSON Web Tokens](https://jwt.io/) を使用して、ユーザの身元を確認し、
これらのユーザに関する基本的なプロファイルを取得できます。

OpenID Connect フローは、次の3つの OAuth 2.0 グラント・フローの上に構築されています:

-   [認可コード (Authorization Code)](https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth)
-   [暗黙 (Implicit)](https://openid.net/specs/openid-connect-core-1_0.html#ImplicitFlowAuth)
-   [ハイブリッド (Hybrid)](https://openid.net/specs/openid-connect-core-1_0.html#HybridFlowAuth)

認可 (Authorization) と認証 (Authentication) は2つのまったく異なるものです。1つ目は特定のデータへのアクセスを
許可または禁止し、2つ目はサイン・インについてです。OAuth2.0 は認可プロセスを有効にしますが、ユーザを識別および
認証する方法がありません。OIDC は、OAuth 2.0 認証の問題を解決するために作成されました。OAuth 2.0 と OIDC
は、ユーザ名とパスワードの公開を避けてユーザを識別するトークンを生成します。特に、OIDC は JSON Web Token (JWT)
を生成します。これは、アプリケーションが本質的にそれ自体からユーザ情報を検証して直接取得できるものです。

<a name="arrow_forward-video-what-is-openid-connect"/>

## :arrow_forward: ビデオ: OpenID Connect とは何ですか？

[![](https://fiware.github.io/tutorials.Step-by-Step/img/video-logo.png)](https://www.youtube.com/watch?v=Kb56GzQ2pSk "OpenID connect")

上の画像をクリックして、OpenID Connect と Identity に関するビデオをご覧ください。

OAuth2 は、アクセス権を付与するためのメカニズムです。具体的には、**認証** (Authorization) です -
(_これを実行できますか？_)。技術的には、OAuth プロトコル内には、**ID** (Identity) 自体の概念がないため、
モバイル・アプリのログインのような、特定の**認証**のユースケースを実行できる場合でも、実際には**認証**
(_私は User X です_) 向けに設計されていません。OpenID は OAuth2 の拡張機能を提供し、アプリケーションが
標準的な方法でユーザ情報を取得できるようにします。

OpenID 接続は、(**Keyrock** など) 複数のエンティティ・プロバイダで機能し、JSON Web tokens を使用して
操作されます。いくつかの基本的なユーザ情報を保持する追加の ID token をレスポンスに追加します。
追加のユーザ情報は、標準化された `/userinfo` エンドポイントからリクエストできます。

OpenID connect リクエストは、OAuth2 リクエストと非常によく似たフローに従います。これらは、最初の
リクエストを行うときに `openid` スコープを使用して区別されます。レスポンスには、以下に説明する要素を
保持するエンコードされた JSON Web Token (JWT) が含まれます。

| 名前  | 説明                                         |
| ----- | -------------------------------------------- |
| `iss` | レスポンスの発行者の発行者 ID                |
| `sub` | サブジェクト識別子                           |
| `aud` | この ID token の対象となるオーディエンス (s) |
| `exp` | 有効期限                                     |
| `iat` | JWT が発行された時刻                         |

他のエントリも追加される場合があります。完全な OpenID 仕様は
[こちら](https://openid.net/specs/openid-connect-core-1_0.html) にあります。

<a name="standard-concepts-of-json-web-tokens"/>

## JSON Web Tokens の標準概念

JSON Web Token (JWT) の構造は次のとおりです:

-   ヘッダー。これは JSON Web Token の署名に使用されるアルゴリズムを識別します。

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

-   ペイロード。トークンが作成された時期と作成者に関する情報だけでなく、ユーザ・データも含まれています。

```json
{
  "sub": "1234567890",
  "iss": "https://fiware-idm.com",
  "iat": 1516239022,
  "username": "Alice",
  "gravatar": true
}
```

-   署名 (Signature)。次のように生成されます:

```text
Crypto-Algorithm ( base64urlEncoding(header) + '.' + base64urlEncoding(payload), secret)
```

JWT は、Base64 を使用して各部分をエンコードし、それらをポイント (.) で連結した結果です。例えば：

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwiaXNzIjoiaHR0cHM6Ly9maXdhcmUtaWRtLmNvbSIsImlhdCI6MTUxNjIzOTAyMiwidXNlcm5hbWUiOiJBbGljZSIsImdyYXZhdGFyIjp0cnVlfQ.dZ7z0u_4FZC7xiVQDtGAl7NRT0fK8_5hJqYa9E-4xGE
```

<a name="prerequisites"/>

# 前提条件

<a name="docker"/>

## Docker

物事を単純にするために、両方のコンポーネントが [Docker](https://www.docker.com) を使用して実行されます。**Docker**
は、さまざまコンポーネントをそれぞれの環境に分離することを可能にするコンテナ・テクノロジです。

-   Docker Windows にインストールするには、
    [こちら](https://docs.docker.com/docker-for-windows/)の手順に従ってください
-   Docker Mac にインストールするには、
    [こちら](https://docs.docker.com/docker-for-mac/)の手順に従ってください
-   Docker Linux にインストールするには、
    [こちら](https://docs.docker.com/install/)の手順に従ってください

**Docker Compose** は、マルチコンテナ Docker アプリケーションを定義して実行するためのツールです。
[YAML file](https://raw.githubusercontent.com/Fiware/tutorials.Securing-Access-OpenID-Connect/master/docker-compose.yml)
ファイルは、アプリケーションのために必要なサービスを構成するために使用します。つまり、すべてのコンテナ・サービスは
1つのコマンドで呼び出すことができます。Docker Compose は、デフォルトで Docker for Windows と Docker for Mac の一部と
してインストールされますが、Linux ユーザは[ここ](https://docs.docker.com/compose/install/)に記載されている手順に従う
必要があります。

<a name="cygwin"/>

## Cygwin

シンプルな bash スクリプトを使用してサービスを開始します。Windows ユーザは [cygwin](http://www.cygwin.com/) を
ダウンロードして、Windows 上の Linux ディストリビューションと同様のコマンドライン機能を提供する必要があります。

<a name="architecture"/>

# アーキテクチャ

このアプリケーションは、最初の[セキュリティ・チュートリアル](https://github.com/FIWARE/tutorials.Identity-Management/)
で作成されたデータを使用してプログラムで読み取ることにより、
[以前のチュートリアル](https://github.com/FIWARE/tutorials.Identity-Management/)で作成された既存の在庫管理および
センサ・ベースのアプリケーションに OIDC ドリブンのセキュリティを追加します。これは、1つの FIWARE コンポーネント -
[Keyrock](https://fiware-idm.readthedocs.io/en/latest/) Generic enabler を使用します。**Keyrock** は独自の
[MySQL](https://www.mysql.com/) データベースを使用します。 このチュートリアルでは、OIDC を使用して JWT
を付与することにのみ焦点を当てています。
[セキュリティで保護されたアクセスのチュートリアル](https://github.com/FIWARE/tutorials.Securing-Access)
では、トークンを使用してセンサ情報に安全にアクセスする方法を学習できます。

したがって、全体的なアーキテクチャは次の要素で構成されます:

-   FIWARE [Keyrock](https://fiware-idm.readthedocs.io/en/latest/) は、以下を含んだ、補完的な ID 管理システムを
    提供します:
    -   アプリケーションとユーザのための OAuth2 認可システム
    -   アプリケーションとユーザのための OIDC 認証システム
    -   ID 管理のための Web サイトのグラフィカル・フロントエンド
    -   HTTP リクエストによる ID 管理用の同等の REST API
-   [MySQL](https://www.mysql.com/) データベース :
    -   ユーザ ID、アプリケーション、ロール、および権限を保持するために使用されます
-   **在庫管理フロントエンド**には、次のことを行います:
    -   店舗情報を表示します
    -   各店舗でどの商品を購入できるかを示します
    -   ユーザが製品を"購入"して在庫数を減らすことができます
    -   許可されたユーザを制限されたエリアに入れることができます

要素間のすべての対話は HTTP リクエストによって開始されるため、エンティティはコンテナ化され、公開されたポートから実行
されます。

![](https://fiware.github.io/tutorials.Securing-Access-OpenID-Connect/img/architecture.png)

**在庫管理フロントエンド**にセキュリティを追加するために必要な設定情報は、関連する `docker-compose.yml` ファイルの
`tutorial` セクションにあります。関連する変数を以下に示します。

<a name="tutorial-security-configuration"/>

## チュートリアルのセキュリティ設定

```yaml
tutorial:
    image: fiware/tutorials.context-provider
    hostname: iot-sensors
    container_name: fiware-tutorial
    networks:
        default:
            ipv4_address: 172.18.1.7
    expose:
        - "3000"
        - "3001"
    ports:
        - "3000:3000"
        - "3001:3001"
    environment:
        - "DEBUG=tutorial:*"
        - "SECURE_ENDPOINTS=true"
        - "OIDC_ENABLED=true"
        - "WEB_APP_PORT=3000"
        - "KEYROCK_URL=http://localhost"
        - "KEYROCK_IP_ADDRESS=http://172.18.1.5"
        - "KEYROCK_PORT=3005"
        - "KEYROCK_CLIENT_ID=tutorial-dckr-site-0000-xpresswebapp"
        - "KEYROCK_CLIENT_SECRET=tutorial-dckr-site-0000-clientsecret"
        - "KEYROCK_JWT_SECRET=jsonwebtokenpass"
        - "CALLBACK_URL=http://localhost:3000/login"
```

`tutorial` コンテナは、2 つのポートでリッスンしています :

-   ポート `3000` が公開されているので、ダミー IoT デバイスを表示する Web ページが表示されます
-   ポート `3001` は純粋にチュートリアルアクセスのために公開されているため、cUrl または Postman は同じネットワークの
    一部ではなくても、UltraLight コマンドを作成できます

`tutorial` コンテナは、次に示すように環境変数によってドライブされます:

| キー                  | 値                                     | 説明                                                                                           |
| --------------------- | -------------------------------------- | ---------------------------------------------------------------------------------------------- |
| DEBUG                 | `tutorial:*`                           | ロギングに使用されるデバッグ・フラグ                                                           |
| OIDC_ENABLED          | `true`                                 | チュートリアルでOpenID Connectを有効化                                                         |
| KEYROCK_CLIENT_ID     | `tutorial-dckr-site-0000-xpresswebapp` | このアプリケーションで Keyrock によって定義された、Client ID                                   |
| KEYROCK_CLIENT_SECRET | `tutorial-dckr-site-0000-clientsecret` | このアプリケーションで Keyrock によって定義された、Client Secret                               |
| KEYROCK_JWT_SECRET    | `jsonwebtokenpass`                     | このアプリケーションが id_tokens を検証するために Keyrock によって定義された JWT Secret        |
| CALLBACK_URL          | `http://localhost:3000/login`          | チャレンジが成功したときに Keyrock が使用するコールバック URL                                  |

YAML ファイルに記述されている、他の `tutorial`コンテナの設定値は、以前のチュートリアルで説明しています

<a name="start-up"/>

# 起動

インストールを開始するには、次の手順を実行します:

```console
git clone https://github.com/FIWARE/tutorials.Securing-Access-OpenID-Connect.git
cd tutorials.Securing-Access-OpenID-Connect
git checkout NGSI-v2

./services create
```

> **注** Docker イメージの最初の作成には最大 3 分かかります

その後、リポジトリ内で提供される
[services](https://github.com/FIWARE/tutorials.Securing-Access-OpenID-Connect/blob/NGSI-v2/services) Bash
スクリプトを実行することによって、コマンドラインからすべてのサービスを初期化することができます:

```console
./services <command>
```

ここで、`<command>` は、私たちがアクティベートしたいエクササイズに応じて変わります。

> :information_source: **注:** クリーンアップをやり直したい場合は、次のコマンドを使用して再起動することができます:
>
> ```console
> ./services stop
> ```

<a name="dramatis-personae"/>

### 登場人物 (Dramatis Personae)

次の `test.com` のメンバは、合法的にアプリケーション内にアカウントを持っています

-   Alice, **Keyrock** アプリケーションの管理者です
-   Bob, スーパー・マーケット・チェーンの地域マネージャで、数人のマネージャがいます :
    -   Manager1 (マネージャ 1)
    -   Manager2 (マネージャ 2)
-   Charlie, スーパー・マーケット・チェーンのセキュリティ責任者。彼の下に数人の警備員がいます :
    -   Detective1 (警備員 1)
    -   Detective2 (警備員 2)

次の`example.com` のメンバはアカウントにサインアップしましたが、アクセスを許可する理由はありません

-   Eve - 盗聴者のイブ
-   Mallory - 悪意のある攻撃者のマロリー
-   Rob - 強盗のロブ

<details>
  <summary>
   詳細 <b>(クリックして拡大)</b>
  </summary>

| 名前       | E メール                  | パスワード |
| ---------- | ------------------------- | ---------- |
| alice      | alice-the-admin@test.com  | `test`     |
| bob        | bob-the-manager@test.com  | `test`     |
| charlie    | charlie-security@test.com | `test`     |
| manager1   | manager1@test.com         | `test`     |
| manager2   | manager2@test.com         | `test`     |
| detective1 | detective1@test.com       | `test`     |
| detective2 | detective2@test.com       | `test`     |

| 名前    | E メール            | パスワード |
| ------- | ------------------- | ---------- |
| eve     | eve@example.com     | `test`     |
| mallory | mallory@example.com | `test`     |
| rob     | rob@example.com     | `test`     |

</details>

Alice によって 2 つの組織 (organizations) が設定されました:

| 名前       | 説明                                | UUID                                   |
| ---------- | ----------------------------------- | -------------------------------------- |
| Security   | Security Group for Store Detectives | `security-team-0000-0000-000000000000` |
| Management | Management Group for Store Managers | `managers-team-0000-0000-000000000000` |

適切なロールと権限を持つ 1 つのアプリケーション も作成されました:

| Key           | Value                                  |
| ------------- | -------------------------------------- |
| Client ID     | `tutorial-dckr-site-0000-xpresswebapp` |
| Client Secret | `tutorial-dckr-site-0000-clientsecret` |
| JWT Secret    | `jsonwebtokenpass`                     |
| URL           | `http://localhost:3000`                |
| RedirectURL   | `http://localhost:3000/login`          |

時間を節約するために、
[以前のチュートリアル](https://github.com/FIWARE/tutorials.Roles-Permissions)の users と organizations を作成する
データがダウンロードされ、起動時に自動的に MySQL データベースに保存されるため、割り当てられた UUIDs は変更されず、
データを再度入力する必要もありません。

**Keyrock** MySQL データベースは、ユーザ、パスワードなどの格納を含むアプリケーション・セキュリティのあらゆる側面を
扱います。アクセス権を定義し、OAuth2 認証プロトコルを扱います。完全なデータベース関係図は
[ここ](https://fiware.github.io/tutorials.Securing-Access/img/keyrock-db.png)にあります。

ユーザ (users) や組織 (organizations)、アプリケーション (applications) を作成する方法について思い出すには、
アカウント `alice-the-admin@test.com` と パスワード `test` を使用して、`http://localhost:3005/idm`
にログインします。

![](https://fiware.github.io/tutorials.Securing-Access/img/keyrock-log-in.png)

そして、周りを見回してください。

<a name="oidc-flows"/>

# OIDC フロー (OIDC Flows)

FIWARE **Keyrock** は、[OpenID Connect 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
で説明されている OIDC 標準に準拠しています。そこで定義されている3つの標準認証フローすべてをサポートします。

OIDC は OAuth 2.0 の最上位に構築されているため、OAuth トークン・エンドポイントにリクエストを送信すると、
`Authorization` ヘッダは、**Keyrock** によって提供されるアプリケーションの Client ID と Client Secret
の認証情報を `:` で区切り、Base64 でエンコードして作成されます。 値は次のように生成できます:

```console
echo tutorial-dckr-site-0000-xpresswebapp:tutorial-dckr-site-0000-clientsecret | base64
```

```
dHV0b3JpYWwtZGNrci1zaXRlLTAwMDAteHByZXNzd2ViYXBwOnR1dG9yaWFsLWRja3Itc2l0ZS0wMDAwLWNsaWVudHNlY3JldAo=
```

<a name="enable-openid-connect"/>

## OpenID Connect の有効化

OpenID Connect は、GUI または REST API を介して Keyrock のアプリケーションで有効にできます。

<a name="gui"/>

### GUI

サイン・インすると、ユーザは Web ページを通じてアプリケーションで OIDC をアクティブ化できます。

![](https://fiware.github.io/tutorials.Securing-Access-OpenID-Connect/img/edit-OIDC.png)

JSON Web トークンを検証するときに使用される Secret は、アプリケーション情報の Web ページにあります。

![](https://fiware.github.io/tutorials.Securing-Access-OpenID-Connect/img/jwtsecret-OIDC.png)

JWT secret は、OAuth2 クレデンシャル・セクションの "Secret をリセット" ボタンをクリックして更新することもできます。

![](https://fiware.github.io/tutorials.Securing-Access-OpenID-Connect/img/jwtsecret-reset-OIDC.png)

<a name="rest-api"/>

### REST API

Keyrock でアプリケーションを作成するときに、OIDC を有効にすることもできます。
[ロールとパーミッションのチュートリアル](https://github.com/FIWARE/tutorials.Roles-Permissions) で説明されているように、
`/v1/applications` への POST リクエストを介して作成でき、`scope` 属性に `openid` を含めます。

```console
curl -iX POST \
  'http://localhost:3005/v1/applications' \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa' \
  -d '{
  "application": {
    "name": "Tutorial Application",
    "description": "FIWARE Application protected by OAuth2 and Keyrock",
    "redirect_uri": "http://tutorial/login",
    "url": "http://tutorial",
    "grant_type": [
      "authorization_code",
      "implicit",
      "password"
    ],
    "scope": "openid",
    "token_types": ["permanent"]
  }
}'
```

アプリケーションがすでに作成されている場合は、PATCH リクエストを行うことにより、コマンドラインから行うこともできます。

```console
curl -X PATCH \
  'http://localhost:3005/v1/applications/{{application-id}}' \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa' \
  -d '{
  "application": {
    "scope": "openid"
  }
}'
```

<a name="authorization-code-flow"/>

## 認可コード・フロー (Authorization Code Flow)

[認可コード](https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth)・フローは、
認証メカニズムをサポートするように調整できます。 OIDC は、自動化コード自体のフローを変更するのではなく、
以下に示すように、認可エンドポイントへのリクエストにパラメータを追加するだけです。レスポンスは、id_token
と交換できるアクセス・コード (access-code) を返します。これにより、ユーザが識別されます。

![](https://fiware.github.io/tutorials.Securing-Access/img/authcode-flow.png)

これは、Travis-CI などのサード・パーティが GitHub アカウントを使用してログインするように要求するときに使用される
種類のフローの例です。 Travis があなたのパスワードにアクセスすることはありませんが、GitHub からあなたが主張している
人物であるという詳細を受け取ります。

<a name="authorization-code---sample-code"/>

### 認可コード - サンプル・コード

ユーザは最初に **Keyrock** にリダイレクトされ、 `code` をリクエストする必要があります。`oa.getAuthorizeUrl()` は
`/oauth/authorize?response_type=code&client_id={{client-id}}&state=oic&redirect_uri={{callback_url}}&scope=openid`
形式の URL を返します

"openid" の値は、これが OIDC リクエストであることを Keyrock に示すために、リクエストのスコープ・パラメータに
含まれています。このチュートリアルの状態値は、"oauth2" と "oic" の場合があります。 この値は、Keyrock
からの回答を管理する方法を示します。

```javascript
function authCodeOICGrant(req, res) {
    const path = oa.getAuthorizeUrl('code', 'openid', 'oic');
    return res.redirect(path);
}
```

ユーザがアクセスを承認した後、レスポンスは `redirect_uri` によって受信され、以下のコードで処理されます。
**Keyrock** から暫定アクセス・コードを受け取り、使用可能な `id_token` を取得するために2番目のリクエストを
行う必要があります。


```javascript
function authCodeOICGrantCallback(req, res) {
    return oa
        .getOAuthAccessToken(req.query.code, 'authorization_code')
        .then(results => {
            return getUserFromIdToken(req, results.id_token);
        })
        .then(user => {
            // Store user
        })
}
```

id_tokenは、環境変数を介してアプリケーションで事前構成した JWT Secret を使用して検証し、その id_token
からユーザ情報を取得できる JWT にすぎません。

```javascript
function getUserFromIdToken(req, idToken) {
  return new Promise(function(resolve, reject) {
    jwt.verify(idToken, jwtSecret, function(error, decoded) {
      // Decoded --> JSON with user, token and issuer information
    });
  });
}
```

デコードされた json は、次のように返されます:

```json
{
    "organizations": [],
    "displayName": "",
    "roles": [],
    "app_id": "tutorial-dckr-site-0000-xpresswebapp",
    "trusted_apps": [],
    "isGravatarEnabled": false,
    "email": "alice-the-admin@test.com",
    "id": "aaaaaaaa-good-0000-0000-000000000000",
    "app_azf_domain": "",
    "username": "alice",
    "trusted_applications": [],
    "iss": "https://fiware-idm.com",
    "sub": "aaaaaaaa-good-0000-0000-000000000000",
    "aud": "tutorial-dckr-site-0000-xpresswebapp",
    "exp": 1516238462,
    "iat": 1516239022,
}
```

<a name="authorization-code---running-the-example"/>

### 認可コード - サンプルの実行

`http://localhost:3000/` のページを表示し、Authorization Code ボタンをクリックすることで、
認可コード・グラント・フロー (Authorization Code grant flow) をプログラムで呼び出すことができます。

ユーザは最初に **Keyrock** にリダイレクトされ、ログインする必要があります

![](https://fiware.github.io/tutorials.Securing-Access/img/keyrock-log-in.png)

次に、ユーザはリクエストを承認する必要があります

![](https://fiware.github.io/tutorials.Securing-Access/img/keyrock-authorize.png)

レスポンスでは、画面の右上にユーザが表示され、トークンの詳細が画面に表示されます:

![](https://fiware.github.io/tutorials.Securing-Access-OpenID-Connect/img/authCode-OIDC-web.png)

> **注** **Keyrock** > `http://localhost:3005` から故意にログアウトしない限り、すでにアクセスを許可している既存の
> **Keyrock** セッションが後続の認証リクエストに使用されるため、 **Keyrock** ログイン画面が再び表示されることは
> ありません。

<a name="implicit-flow"/>

## 暗黙フロー (Implicit Flow)

[暗黙](https://openid.net/specs/openid-connect-core-1_0.html#ImplicitFlowAuth)フローは、
認証メカニズムをサポートするように調整することもできます。OIDC は、認可コード・グラントと同様に、
フローを変更せずに、リクエストの response_type を変更します。このフローは、暫定的なアクセスコードを返すのではなく、
`id_token` を直接返します。これは 認可コード・フローほど安全ではありませんが、一部のクライアント側
アプリケーションで使用できます。

![](https://fiware.github.io/tutorials.Securing-Access/img/implicit-flow.png)

<a name="implicit-flow---sample-code"/>

### 暗黙フロー - サンプル・コード

ユーザは最初に **Keyrock** にリダイレクトされ、`token` をリクエストする必要があります。`oa.getAuthorizeUrl()` は、
`/oauth/authorize?response_type=id_token&client_id={{client-id}}&state=oic&redirect_uri={{callback_url}}` 形式の
URL を返します。OIDC フローに従う場合、レスポンスのタイプは "id_token" であることに注意してください。

```javascript
function implicitOICGrant(req, res) {
    const path = oa.getAuthorizeUrl('id_token', null, 'oic');
    return res.redirect(path);
}
```

ユーザがアクセスを承認した後、レスポンスは `redirect_uri` によって受信され、以下のコードで処理され、
使用可能なアクセス・トークンが **Keyrock** から受信されます。

```javascript
function implicitOICGrantCallback(req, res) {
    return getUserFromIdToken(req, req.query.id_token)
        .then(user => {
          // Store User and return
        })
}
```

id_token は、認可コードのセクションで説明したように、JWT Secret を使用して検証できる単なる JWT です。

<a name="implicit-flow---running-the-example"/>

### 暗黙フロー - サンプルの実行

`http://localhost:3000/` のページを表示し、Implicit Grant ボタンをクリックすることにより、プログラムで
暗黙グラント・フロー (Implicit grant flow) を呼び出すことができます。

ユーザは最初に **Keyrock** にリダイレクトされ、ログインする必要があります。

![](https://fiware.github.io/tutorials.Securing-Access/img/keyrock-log-in.png)

ユーザはリクエストを承認する必要があります。

![](https://fiware.github.io/tutorials.Securing-Access/img/keyrock-authorize.png)

レスポンスでは、画面の右上にユーザが表示され、トークンの詳細も画面に表示されます:

![](https://fiware.github.io/tutorials.Securing-Access-OpenID-Connect/img/implicit-OIDC-web.png)

> **注** **Keyrock** > `http://localhost:3005` から故意にログアウトしない限り、すでにアクセスを許可している既存の
> **Keyrock** セッションが後続の認証リクエストに使用されます。

<a name="hybrid-flow"/>

## ハイブリッド・フロー (Hybrid Flow)

[ハイブリッド](https://openid.net/specs/openid-connect-core-1_0.html#HybridFlowAuth)・フローは、認可コードと
暗黙グラントを組み合わせています。 アプリケーションのフロント・エンドとバック・エンドでプロセスを並列化すると
便利な場合があります。フローは認可コード・グラントに似ていますが、この場合、トークンは認可エンドポイントと
トークン・エンドポイントの両方で生成されます。

<a name="authorization-code---sample-code"/>

### ハイブリッド - サンプル・コード

ユーザは最初に **Keyrock** にリダイレクトされ、`code` をリクエストする必要があります。`oa.getAuthorizeUrl()` は
`/oauth/authorize?response_type=code id_token token&client_id={{client-id}}&state=oic&redirect_uri={{callback_url}}&scope=openid`
形式の URL を返します。ハイブリッド・フローでは、すべての response_types (code, token, id_token)
を含める必要があることに注意してください。最初のリクエストでは、これにより認可コード、アクセス・トークン、および
id_token が生成されます。 スコープ "openid" も含める場合、以前に生成された認可コードを使用すると、Keyrock
は新しいアクセス・トークンと新しい id_token を生成します。

```javascript
function hybridOICGrant(req, res) {
  const path = oa.getAuthorizeUrl('code id_token token', 'openid', 'oic');
  return res.redirect(path);
}

```

ユーザがアクセスを承認した後、レスポンスは `redirect_uri` によって受信され、以下のコードで処理されます。
暫定アクセス・コード (A interim access code) は **Keyrock** から受信され、使用可能な `id_token`
を取得するために2番目のリクエストを行う必要があります。

```javascript
function authCodeOICGrantCallback(req, res) {
    return oa
        .getOAuthAccessToken(req.query.code, 'hybrid')
        .then(results => {
            return getUserFromIdToken(req, results.id_token);
        })
        .then(user => {
            // Store User and return
        })
}
```

id_token は、認可コードのセクションで説明したように、JWT Secret を使用して検証できる単なる JWT です。

<a name="authorization-code---running-the-example"/>

### ハイブリッド - サンプルの実行

`http://localhost:3000/` のページを表示し、Authorization Code ボタンをクリックすることにより、
プログラムでハイブリッド・フローを呼び出すことができます。

ユーザは最初に **Keyrock** にリダイレクトされ、ログインする必要があります。


![](https://fiware.github.io/tutorials.Securing-Access/img/keyrock-log-in.png)

ユーザはリクエストを承認する必要があります。

![](https://fiware.github.io/tutorials.Securing-Access/img/keyrock-authorize.png)

レスポンスでは、画面の右上にユーザが表示され、トークンの詳細も画面に表示されます:

![](https://fiware.github.io/tutorials.Securing-Access-OpenID-Connect/img/hybrid-OIDC-web.png)

> **注** **Keyrock** > `http://localhost:3005` から故意にログアウトしない限り、すでにアクセスを許可している既存の
> **Keyrock** セッションが後続の認証リクエストに使用されるため、 **Keyrock** ログイン画面が再び表示されることは
> ありません。

# 次のステップ

高度な機能を追加することで、アプリケーションに複雑さを加える方法を知りたいですか
？このシリーズ
の[他のチュートリアル](https://www.letsfiware.jp/fiware-tutorials)を読むことで見
つけることができます :

---

## License

[MIT](LICENSE) © 2018-2020 FIWARE Foundation e.V.
