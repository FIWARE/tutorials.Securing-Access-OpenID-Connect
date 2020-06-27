[![FIWARE Banner](https://fiware.github.io/tutorials.Securing-Access/img/fiware.png)](https://www.fiware.org/developers)

[![FIWARE Security](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/security.svg)](https://github.com/FIWARE/catalogue/blob/master/security/README.md)
[![License: MIT](https://img.shields.io/github/license/fiware/tutorials.Securing-Access.svg)](https://opensource.org/licenses/MIT)
[![Support badge](https://nexus.lab.fiware.org/repository/raw/public/badges/stackoverflow/fiware.svg)](https://stackoverflow.com/questions/tagged/fiware)
<br/>
[![Documentation](https://img.shields.io/readthedocs/fiware-tutorials.svg)](https://fiware-tutorials.rtfd.io)

<!-- prettier-ignore -->

このチュートリアルでは
、[以前のチュートリアル](https://github.com/FIWARE/tutorials.Roles-Permissions)で
作成したエンティティを使用して、FIWARE アプリケーションへのアクセスを保護します
。このチュートリアルでは、さまざまな OAuth2 グラント・フローの適切な使用方法と
、**Keyrock** generic enabler を認可サーバ (Authorization Server) として使用し、
ユーザを識別する方法について説明します。**Keyrock** はアクセスを制限するための
Policy Decision Point (PDP) としても使用されます。

チュートリアルでは、**Keyrock** を Web アプリケーションに統合する方法を示すコー
ドについて説明し、**Keyrock** GUI を使用した認可サーバとの対話の例を示します。い
くつかの [cUrl](https://ec.haxx.se/) コマンドは、**Keyrock** REST API へのアクセ
スにも使用しようします
。[Postman documentation](https://fiware.github.io/tutorials.Securing-Access/)
も利用できます。

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/66d8ba3abaf7319941b1)

# コンテンツ

<details>
<summary>詳細 <b>(クリックして拡大)</b></summary>

-   [アクセスの保護](#securing-access)
    -   [ID 管理の標準概念](#standard-concepts-of-identity-management)
-   [前提条件](#prerequisites)
    -   [Docker](#docker)
    -   [Cygwin](#cygwin)
-   [アーキテクチャ](#architecture)
    -   [チュートリアルのセキュリティ構成](#tutorial-security-configuration)
-   [起動](#start-up)
    -   [登場人物 (Dramatis Personae)](#dramatis-personae)
-   [OAuth2 グラント・フロー](#oauth2-grant-flows)
    -   [ユーザ資格情報のグラント (User Credentials Grant)](#user-credentials-grant)
        -   [パスワードによるログイン](#logging-in-with-a-password)
        -   [アクセス・トークンからのユーザ詳細の取得](#retrieving-user-details-from-an-access-token)
        -   [ユーザ資格情報 - サンプル・コード](#user-credentials---sample-code)
        -   [ユーザ資格情報 - サンプルの実行](#user-credentials---running-the-example)
    -   [認可コードのグラント (Authorization Code Grant)](#authorization-code-grant)
        -   [認可コード - サンプル・コード](#authorization-code---sample-code)
        -   [認可コード - サンプルの実行](#authorization-code---running-the-example)
    -   [暗黙のグラント (Implicit Grant)](#implicit-grant)
        -   [暗黙のグラント - サンプル・コード](#implicit-grant---sample-code)
        -   [暗黙のグラント - サンプルの実行](#implicit-grant---running-the-example)
    -   [クライアント資格情報のグラント (Client Credentials Grant)](#client-credentials-grant)
        -   [アプリケーションとしてのログイン](#logging-in-as-an-application)
        -   [クライアント資格情報のグラント - サンプル・コード](#client-credentials-grant---sample-code)
        -   [クライアント資格情報のグラント - サンプルの実行](#client-credentials-grant---running-the-example)
    -   [トークンをリフレッシュ](#refresh-token)
        -   [アベイラビリティ・チェック](#availability-check)
        -   [アクセス・トークンをリフレッシュ](#refresh-access-token)
        -   [トークンのリフレッシュ - サンプル・コード](#refresh-token---sample-code)
        -   [トークンのリフレッシュ - サンプルの実行](#refresh-token---running-the-example)
-   [PDP - アクセス制御](#pdp---access-control)
    -   [認証アクセス (Authentication Access)](#authentication-access)
        -   [認証アクセス - サンプル・コード](#authentication-access---sample-code)
    -   [基本認可 (Basic Authorization)](#basic-authorization)
        -   [基本認可 - サンプル・コード](#basic-authorization---sample-code)
    -   [PDP アクセス制御 - サンプルの実行](#pdp-access-control---running-the-example)

</details>

<a name="securing-access"></a>

# アクセスの保護

> "When a person or party approaches your post, you should challenge them at a
> distance that is sufficient for you to react if they turn out to have hostile
> intentions. You should say in a firm voice, loud enough to be easily heard,
> _"Halt! Who goes there?"_ (or _"Who is there?"_). Once the person answers, you
> should then say _"Advance to be recognized."_ ... If you have identified the
> person or persons approaching, permit them to pass. If you are not satisfied
> with that person's identification, you must detain the person and call the
> petty officer of the watch."
>
> — 11th General Order of the US Marine Corps

アプリケーションのリソースへのアクセスを保護するためには、2 つのことを知る必要が
あります。まず、誰が要求を出していますか、第二に、要求者がリソースにアクセスする
ことを許可されていますか？ FIWARE **Keyrock** generic enabler は
、[OAuth2](https://oauth.net/2/) を使用して、サードパーティのアプリケーションが
サービスへのアクセスを制限できるようにします。**OAuth2** は、アクセス権限を与え
るためのアクセス委任のためのオープン・スタンダードです。リソースの所有者 (例えば
、あなた) が、自分の情報 (例えば、エンティティのリスト) へのアクセスを第三者 (例
えば、Knowage Application) に許可することを、リソース・プロバイダ (例えば
、Knowage Generic Enabler) に通知することができます。

いくつかの一般的な OAuth 2.0 グラント・フローがあります。その詳細は以下のとおり
です :

-   [Authorization Code](https://oauth.net/2/grant-types/authorization-code) (認
    可コード)
-   [Implicit](https://oauth.net/2/grant-types/implicit) (暗黙)
-   [Password](https://oauth.net/2/grant-types/password) (パスワード)
-   [Client Credentials](https://oauth.net/2/grant-types/client-credentials) (ク
    ライアント資格情報)
-   [Device Code](https://oauth.net/2/grant-types/device-code) (デバイス・コード
    )
-   [Refresh Token](https://oauth.net/2/grant-types/refresh-token) (リフレッシュ
    ・トークン)

主なコンセプトは、**ユーザ** と **アプリケーション** の両方が、標準の OAuth2 チ
ャレンジ・レスポンス・メカニズムを使用して最初に自分自身を識別する必要があるとい
うことです。その後、ユーザは、その後のすべてのリクエストに追加するトークンを割り
当てられます。このトークンは、ユーザ、アプリケーション、およびユーザが実行できる
権利を識別します。**Keyrock** を使用して、他の Enabler がアクセスを制限し、ロッ
ク・ダウンすることができます。アクセス・フローの詳細については、以下および後続の
チュートリアルで説明します。

OAuth2 の背後にある理由は、ユーザに完全なアクセス権を与えるために、自分のユーザ
名とパスワードを第三者に公開する必要がないことです。関連するアクセスを許可するだ
けです。読み取り専用または読み書きのいずれかになる関連アクセスを許可するだけで、
このようなアクセスは細かいレベルまで定義できます。さらに、いつでもアクセスを取り
消すための規定があり、リソースのオーナーは誰に何にアクセスできるかを制御できます
。

アプリケーションがユーザを認証できると、アクセス制御メカニズムを使用してアクセス
をロック・ダウンすることもできます。アクセス制御にはアクセス・ポリシーが必要です
。つまり、誰が何をできるのかを定義する必要があります
。[前回のチュートリアル](https://github.com/FIWARE/tutorials.Roles-Permissions)で
はすでにロールとパーミッションを定義していましたが、簡単な Policy Decision Point
(PDP) を追加することで、このポリシーをプログラムで施行する必要があります。PDP は
、認可決定(authorization decisions)を評価して発行し、Policy Enforcement Point
(PEP）を使用して決定を施行することによってアクセスを保護します。

<a name="standard-concepts-of-identity-management"></a>

## ID 管理の標準概念

**Keyrock** ID 管理データベースには、次の共通オブジェクトがあります :

-   **User** - 電子メールとパスワードを使用して自分自身を識別できる、登録済みの
    ユーザ。ユーザには、個別にまたはグループとして権利を割り当てることができます
-   **Application** - 一連のマイクロ・サービスで構成された任意のセキュアな
    FIWARE アプリケーション
-   **Organization** - 一連の権利を割り当てることができるユーザのグループ。組織
    の権利を変更すると、その組織のすべてのユーザのアクセスが影響を受けます
-   **OrganizationRole** - ユーザは組織のメンバまたは管理者になることができます
    。管理者は組織にユーザを追加または削除できます。メンバは組織のロールと権限を
    取得するだけです。これにより、各組織はメンバに対して責任を持つことができ、ス
    ーパー管理者 (super-admin) がすべての権限を管理する必要がなくなります
-   **Role** - ロールは、一連のアクセス許可の説明的なバケットです。ロールは、単
    一のユーザまたは組織に割り当てることができます。サインインしたユーザは、自分
    のすべてのロールとその組織に関連付けられているすべてのロールのすべての権限を
    取得します
-   **Permission** - システム内のリソース上で何かを行う能力

さらに、FIWARE アプリケーション内で、2 つの人以外のアプリケーション (non-human
application) のオブジェクトを保護することができます。

-   **IoTAgent** - IoT センサとコンテキスト・ブローカー間のプロキシ
-   **PEPProxy** - ユーザの権利を確認する Generic Enabler 間での使用のためのミド
    ルウェア

オブジェクト間の関係は以下のようになります。赤でマークされたエンティティは、この
チュートリアルで直接使用されています :

![](https://fiware.github.io/tutorials.Securing-Access/img/entities.png)

<a name="prerequisites"></a>

# 前提条件

<a name="docker"></a>

## Docker

物事を単純にするために、両方のコンポーネントが [Docker](https://www.docker.com)
を使用して実行されます。**Docker** は、さまざまコンポーネントをそれぞれの環境に
分離することを可能にするコンテナ・テクノロジです。

-   Docker Windows にインストールするには
    、[こちら](https://docs.docker.com/docker-for-windows/)の手順に従ってくださ
    い
-   Docker Mac にインストールするには
    、[こちら](https://docs.docker.com/docker-for-mac/)の手順に従ってください
-   Docker Linux にインストールするには
    、[こちら](https://docs.docker.com/install/)の手順に従ってください

**Docker Compose** は、マルチコンテナ Docker アプリケーションを定義して実行する
ためのツールです
。[YAML file](https://raw.githubusercontent.com/Fiware/tutorials.Identity-Management/master/docker-compose.yml)
ファイルは、アプリケーションのために必要なサービスを構成するために使用します。つ
まり、すべてのコンテナ・サービスは 1 つのコマンドで呼び出すことができます
。Docker Compose は、デフォルトで Docker for Windows と Docker for Mac の一部と
してインストールされますが、Linux ユーザ
は[ここ](https://docs.docker.com/compose/install/)に記載されている手順に従う必要
があります。

<a name="cygwin"></a>

## Cygwin

シンプルな bash スクリプトを使用してサービスを開始します。Windows ユーザは
[cygwin](http://www.cygwin.com/) をダウンロードして、Windows 上の Linux ディスト
リビューションと同様のコマンドライン機能を提供する必要があります。

<a name="architecture"></a>

# アーキテクチャ

このアプリケーションは、最初
の[セキュリティ・チュートリアル](https://github.com/FIWARE/tutorials.Identity-Management/)で
作成したデータを使用し、それをプログラムで読み込むことで
、[以前のチュートリアル](https://github.com/FIWARE/tutorials.IoT-Agent/)で作成し
た既存の在庫管理およびセンサ・ベースのアプリケーションに OAuth2 主導のセキュリテ
ィを追加します
。[Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/),
[IoT Agent for UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/)
と[Keyrock](https://fiware-idm.readthedocs.io/en/latest/) Generic enabler の 3
つの FIWARE コンポーネントを使用し、これらを統合します。アプリケーションを
_"Powered by FIWARE"_ と認定するには、Orion Context Broker を使用するだけで十分
です。

Orion Context Broker と IoT Agent はオープンソースの
[MongoDB](https://www.mongodb.com/) 技術を利用して、保持している情報の永続性を保
ちます
。[以前のチュートリアル](https://github.com/FIWARE/tutorials.IoT-Sensors/)で作成
したダミー IoT デバイスも使用します。**Keyrock** は独自の
[MySQL](https://www.mysql.com/) データベースを使用します。

したがって、全体的なアーキテクチャは次の要素で構成されます :

-   FIWARE
    [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) は
    、[NGSI-v2](https://fiware.github.io/specifications/OpenAPI/ngsiv2) を使用して
    リクエストを受信します
-   [IoT Agent for UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/)
    は、[NGSI-v2](https://fiware.github.io/specifications/OpenAPI/ngsiv2) を使用し
    てサウスバウンド・リクエストを受信し、それをデバイスのため
    に[UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual)
    に変換します。
-   FIWARE [Keyrock](https://fiware-idm.readthedocs.io/en/latest/) は、以下を含
    んだ、補完的な ID 管理システムを提供します :
    -   アプリケーションとユーザのための OAuth2 認証システム
    -   ID 管理のための Web サイトのグラフィカル・フロントエンド
    -   HTTP リクエストによる ID 管理用の同等の REST API
-   [MongoDB](https://www.mongodb.com/) データベース :
    -   **Orion Context Broker** が、データ・エンティティ、サブスクリプション、
        レジストレーションなどのコンテキスト・データ情報を保持するために使用しま
        す
    -   デバイスの URLs や Keys などのデバイス情報を保持するために **IoT Agent**
        によって使用されます
-   [MySQL](https://www.mysql.com/) データベース :
    -   ユーザ ID、アプリケーション、ロール、および権限を保持するために使用され
        ます
-   **在庫管理フロントエンド**には、次のことを行います :
    -   店舗情報を表示します
    -   各店舗でどの商品を購入できるかを示します
    -   ユーザが製品を"購入"して在庫数を減らすことができます
    -   許可されたユーザを制限されたエリアに入れることができます
-   HTTP を介して実行されている
    [UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual)
    プロトコルを使用す
    る[ダミー IoT デバイス](https://github.com/FIWARE/tutorials.IoT-Sensors)のセ
    ットとして機能する Web サーバ。特定のリソースへのアクセスが制限されています
    。

要素間のすべての対話は HTTP リクエストによって開始されるため、エンティティはコン
テナ化され、公開されたポートから実行されます。

![](https://fiware.github.io/tutorials.Securing-Access/img/architecture.png)

**在庫管理フロントエンド**にセキュリティを追加するために必要な設定情報は、関連す
る `docker-compose.yml` ファイルの `tutorial` セクションにあります。関連する変数
を以下に示します。

<a name="tutorial-security-configuration"></a>

## チュートリアルのセキュリティ構成

```yaml
: image: fiware/tutorials.context-provider
  hostname: tutorial
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
      - "WEB_APP_PORT=3000"
      - "KEYROCK_URL=http://localhost"
      - "KEYROCK_IP_ADDRESS=http://172.18.1.5"
      - "KEYROCK_PORT=3005"
      - "KEYROCK_CLIENT_ID=tutorial-dckr-site-0000-xpresswebapp"
      - "KEYROCK_CLIENT_SECRET=tutorial-dckr-site-0000-clientsecret"
      - "CALLBACK_URL=http://localhost:3000/login"
```

`tutorial` コンテナは、2 つのポートでリッスンしています :

-   ポート `3000` が公開されているので、ダミー IoT デバイスを表示する Web ページ
    が表示されます
-   ポート `3001` は純粋にチュートリアルアクセスのために公開されているため、cUrl
    または Postman は同じネットワークの一部ではなくても、UltraLight コマンドを作
    成できます

`tutorial` コンテナは、次に示すように環境変数によってドライブされます :

| キー                  | 値                                     | 説明                                                                                 |
| --------------------- | -------------------------------------- | ------------------------------------------------------------------------------------ |
| DEBUG                 | `tutorial:*`                           | ロギングに使用されるデバッグ・フラグ                                                 |
| WEB_APP_PORT          | `3000`                                 | ログイン画面などを表示する web-app が使用するポート                                  |
| KEYROCK_URL           | `http://localhost`                     | ユーザを転送するときのリダイレクトに使用される、**Keyrock** Web Front-End 自体の URL |
| KEYROCK_IP_ADDRESS    | `http://172.18.1.5`                    | **Keyrock** OAuth 通信の URL                                                         |
| KEYROCK_PORT          | `3005`                                 | **Keyrock** がリッスンしているポート                                                 |
| KEYROCK_CLIENT_ID     | `tutorial-dckr-site-0000-xpresswebapp` | このアプリケーションで **Keyrock** によって定義された、Client ID                     |
| KEYROCK_CLIENT_SECRET | `tutorial-dckr-site-0000-clientsecret` | このアプリケーションで **Keyrock** によって定義された、Client Secret                 |
| CALLBACK_URL          | `http://localhost:3000/login`          | チャレンジが成功したときに **Keyrock** が使用するコールバック URL                    |

YAML ファイルに記述されている、他の `tutorial`コンテナの設定値は、以前のチュート
リアルで説明しています。

別々の `KEYROCK_URL` と `KEYROCK_IP_ADDRESS` は、チュートリアルで使用されている
単純化された Docker コンテナ化でのみ必要です。値 `localhost` を持つ
`KEYROCK_URL` 変数は、コンテナによって外部に公開されている場所を参照しています
。`KEYROCK_IP_ADDRESS` 変数は同じ場所を参照しますが、Docker ネットワークからアク
セスします。同様に、ブラウザは同じマシンからアクセスされると見なされるので
、`CALLBACK_URL` は `localhost` を含んでいます。これらの値はすべてプロダクション
環境に適したプロキシと DNS 設定に置き換えてください。プロダクション環境はこのチ
ュートリアルの対象外です。

<a name="start-up"></a>

# 起動

インストールを開始するには、次の手順を実行します :

```console
git clone https://github.com/FIWARE/tutorials.Securing-Access.git
cd tutorials.Securing-Access

./services create
```

> **注** Docker イメージの最初の作成には最大 3 分かかります

その後、リポジトリ内で提供される
[services](https://github.com/FIWARE/tutorials.Securing-Access/blob/master/services)
Bash スクリプトを実行することによって、コマンドラインからすべてのサービスを初期
化することができます :

```console
./services <command>
```

ここで、`<command>` は、私たちがアクティベートしたいエクササイズに応じてかわりま
す。

> :information_source: **注:** クリーンアップをやり直したい場合は、次のコマンド
> を使用して再起動することができます :
>
> ```console
> ./services stop
> ```

<a name="dramatis-personae"></a>

### 登場人物 (Dramatis Personae)

次の `test.com` のメンバは、合法的にアプリケーション内にアカウントを持っています

-   Alice, **Keyrock** アプリケーションの管理者です
-   Bob, スーパー・マーケット・チェーンの地域マネージャで、数人のマネージャがい
    ます :
    -   Manager1 (マネージャ 1)
    -   Manager2 (マネージャ 2)
-   Charlie, スーパー・マーケット・チェーンのセキュリティ責任者。彼の下に数人の
    警備員がいます :
    -   Detective1 (警備員 1)
    -   Detective2 (警備員 2)

次の`example.com` のメンバはアカウントにサインアップしましたが、アクセスを許可す
る理由はありません

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

Alice によって 2 つの組織 (organizations) が設定されました :

| 名前       | 説明                                   | UUID                                   |
| ---------- | -------------------------------------- | -------------------------------------- |
| Security   | 警備員のためのセキュリティ・グループ   | `security-team-0000-0000-000000000000` |
| Management | ストア・マネージャのための管理グループ | `managers-team-0000-0000-000000000000` |

適切なロールと権限を持つ 1 つのアプリケーション (applications) も作成されました
:

| Key           | Value                                  |
| ------------- | -------------------------------------- |
| Client ID     | `tutorial-dckr-site-0000-xpresswebapp` |
| Client Secret | `tutorial-dckr-site-0000-clientsecret` |
| URL           | `http://localhost:3000`                |
| RedirectURL   | `http://localhost:3000/login`          |

時間を節約するために
、[以前のチュートリアル](https://github.com/FIWARE/tutorials.Roles-Permissions)の
users と organizations を作成するデータがダウンロードされ、起動時に自動的に
MySQL データベースに保存されるため、割り当てられた UUIDs は変更されず、データを
再度入力する必要もありません。

**Keyrock** MySQL データベースは、ユーザ、パスワードなどの格納を含むアプリケーシ
ョン・セキュリティのあらゆる側面を扱います。アクセス権を定義し、OAuth2 認証プロ
トコルを扱います。完全なデータベース関係図
は[ここ](https://fiware.github.io/tutorials.Securing-Access/img/keyrock-db.png)に
あります。

ユーザ (users) や組織 (organizations)、アプリケーション (applications) を作成す
る方法について思い出すには、アカウント `alice-the-admin@test.com` と パスワード
`test` を使用して、`http://localhost:3005/idm` にログインします。

![](https://fiware.github.io/tutorials.Securing-Access/img/keyrock-log-in.png)

そして、周りを見回してください。

<a name="oauth2-grant-flows"></a>

# OAuth2 グラント・フロー

ドキュメントで説明されているように、FIWARE **Keyrock** は
[RFC 6749](http://tools.ietf.org/html/rfc6749) に記述された OAuth2 標準に準拠し
ており、そこに定義されている 4 つの標準のグラント・タイプをすべてサポートしてい
ます。

リクエストを行う際、`Authorization` ヘッダは、`:` と base-64 エンコードで区切ら
れた **Keyrock** によって提供されるアプリケーションの Client ID と Client Secret
の資格情報を組み合わせて構築されます。次のように値を生成できます :

```console
echo tutorial-dckr-site-0000-xpresswebapp:tutorial-dckr-site-0000-clientsecret | base64
```

```
dHV0b3JpYWwtZGNrci1zaXRlLTAwMDAteHByZXNzd2ViYXBwOnR1dG9yaWFsLWRja3Itc2l0ZS0wMDAwLWNsaWVudHNlY3JldAo=
```

4 つの主要グラント・フローはチュートリアル・アプリケーション内で実演できます。実
際に選択するフローは、独自のユース・ケースに依存します。

<a name="user-credentials-grant"></a>

## ユーザ資格情報のグラント (User Credentials Grant)

パスワード・グラントとしても知られている
、[ユーザ資格情報](https://tools.ietf.org/html/rfc6749#section-1.3.3)のグラント
・フローは、次の場合にのみ使用してください :

-   ユーザが Web アプリケーションのクライアント経由でアプリケーションにログイン
    しようとしている
-   Web アプリケーション・クライアントは絶対に信頼されている

![](https://fiware.github.io/tutorials.Securing-Access/img/user-credentials.png)

これは、Web アプリケーション が私たちが書いたので、スーパーマーケット・チュート
リアルのアプリケーション内で最も適切な使用法であり、私たちが所有している
**Keyrock** のインスタンスに資格証明を渡すことができると信じることができます。図
からわかるように、ユーザは自分自身のパスワードを Web アプリケーションのクライア
ント自体に入力する必要があります。

<a name="logging-in-with-a-password"></a>

### パスワードによるログイン

#### :one: リクエスト

ユーザ資格情報フローを使用してログインするには、POST リクエストを
`grant_type=password` を指定して、`oauth2/token` エンドポイントに送信します。

```console
curl -iX POST \
  'http://localhost:3005/oauth2/token' \
  -H 'Accept: application/json' \
  -H 'Authorization: Basic dHV0b3JpYWwtZGNrci1zaXRlLTAwMDAteHByZXNzd2ViYXBwOnR1dG9yaWFsLWRja3Itc2l0ZS0wMDAwLWNsaWVudHNlY3JldA==' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data "username=alice-the-admin@test.com&password=test&grant_type=password"
```

#### レスポンス

レスポンスは、ユーザを識別するためのアクセスコードを返します :

```json
{
    "access_token": "a7e22dfe2bd7d883c8621b9eb50797a7f126eeab",
    "token_type": "Bearer",
    "expires_in": 3599,
    "refresh_token": "05e386edd9f95ed0e599c5004db8573e86dff874"
}
```

<a name="retrieving-user-details-from-an-access-token"></a>

### アクセス・トークンからのユーザ詳細の取得

アクセスコードは、ユーザの詳細を取得するために、`/user` エンドポイントへの GET
リクエストと共に使用することができます。例えば、上記のレスポンスから
`access_token` を取得します。

#### :two: リクエスト

```console
curl -X GET \
  'http://localhost:3005/user?access_token=a7e22dfe2bd7d883c8621b9eb50797a7f126eeab'
```

#### レスポンス

次のように、ユーザ名 (Alice) が返されます :

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
    "trusted_applications": []
}
```

<a name="user-credentials---sample-code"></a>

### ユーザ資格情報 - サンプル・コード

このコードは、すべての OAuth2 呼び出しを別のライブラリ
[oauth2.js](https://github.com/FIWARE/tutorials.Step-by-Step/blob/master/context-provider/lib/oauth2.js)
に委譲します。すべてのリクエストには標準の OAuth2 ヘッダが含まれており、各リクエ
ストはアプリケーションのコードを単純化するという約束でラップされています。ユーザ
資格情報のフローは、`oa.getOAuthPasswordCredentials()` 関数を使用して呼び出され
ます。`access_token` が受信されると、次のように別々の `oa.get()` コールを使用し
てユーザの詳細が検索されます :

```javascript
function userCredentialGrant(req, res) {
    const email = req.body.email;
    const password = req.body.password;

    oa.getOAuthPasswordCredentials(email, password)
        .then(results => {
            logAccessToken(req, results.access_token);
            return getUserFromAccessToken(req, results.access_token);
        })
        .then(user => {
            // Store User and return
        });
}
```

```javascript
function getUserFromAccessToken(req, accessToken) {
    return new Promise(function(resolve, reject) {
        oa.get(keyrockIPAddress + "/user", accessToken)
            .then(response => {
                const user = JSON.parse(response);
                return resolve(user);
            })
            .catch(error => {
                req.flash("error", "User not found");
                return reject(error);
            });
    });
}
```

<a name="user-credentials---running-the-example"></a>

### ユーザ資格情報 - サンプルの実行

`http://localhost:3000/` のページをオープンし、ユーザ名とパスワードのフォームに
記入することで、ユーザ資格情報のグラント・フローをプログラムで呼び出すことができ
ます。

![](https://fiware.github.io/tutorials.Securing-Access/img/tutorial-log-in.png)

レスポンスにより、画面の右上にユーザが表示され、トークンの詳細も画面に表示されま
す :

![](https://fiware.github.io/tutorials.Securing-Access/img/tutorial-reponse.png)

<a name="authorization-code-grant"></a>

## 認可コードのグラント (Authorization Code Grant)

[認可コード](https://tools.ietf.org/html/rfc6749#section-1.3.1)のグラント・フロ
ーは、クライアント (ここではチュートリアルの Web アプリケーション) がパスワード
に直接アクセスする必要がない場合に使用できます。ユーザが誰であるかを知る必要があ
ります。認可コードのグラントにより、ユーザは **Keyrock** などの認可サーバ
(Authorization Server) にリダイレクトされ、そこにログインしてアクセスを許可しま
す。レスポンスはアクセス・トークンと交換可能なアクセス・コードを返し、アクセス・
トークンはユーザを識別します。

![](https://fiware.github.io/tutorials.Securing-Access/img/authcode-flow.png)

これは、Travis-CI などのサードパーティが Github アカウントを使用してログインする
ようにリクエストしたときに使用されるフローの例です。Travis はあなたのパスワード
にアクセスすることはできませんが、あなたが Github から来たと主張する人物の詳細を
受け取ることができます。

<a name="authorization-code---sample-code"></a>

### 認可コード - サンプル・コード

ユーザは最初に **Keyrock** にリダイレクトされ、`code` をリクエストし
、`oa.getAuthorizeUrl()` がフォーム
`/oauth/authorize?response_type=code&client_id={{client-id}}&state=xyz&redirect_uri={{callback_url}}`
の URL を返す必要があります。

```javascript
function authCodeGrant(req, res) {
    const path = oa.getAuthorizeUrl("code");
    return res.redirect(path);
}
```

ユーザがアクセスを許可した後、`redirect_uri` によってレスポンスが受信され、以下
のコードで処理された後、暫定的なアクセス・コードを **Keyrock** から受信し、2 番
目のリクエストが使用可能な `access_token` を取得する必要があります。

```javascript
function authCodeGrantCallback(req, res) {
    return oa
        .getOAuthAccessToken(req.query.code)
        .then(results => {
            return getUserFromAccessToken(req, results.access_token);
        })
        .then(user => {
            // Store User and return
        });
}
```

<a name="authorization-code---running-the-example"></a>

### 認可コード - サンプルの実行

`http://localhost:3000/` のページをオープンし、**Authorization Code** ボタンをク
リックすることで、プログラムで、ユーザ資格情報のグラント・フローを呼び出すことが
できます。

ユーザは最初に **Keyrock** にリダイレクトされ、ログインする必要があります

![](https://fiware.github.io/tutorials.Securing-Access/img/keyrock-log-in.png)

ユーザは、リクエストを承認する必要があります

![](https://fiware.github.io/tutorials.Securing-Access/img/keyrock-authorize.png)

レスポンスにより、画面の右上にユーザが表示され、トークンの詳細も画面に表示されま
す。

> **注** 意図的に　**Keyrock** の http://localhost:3005 からログアウトしない限り
> 、すでにアクセスを許可している既存の **Keyrock** セッションは、以降の認可リク
> エストに使用されるため、**Keyrock** のログイン画面は再び表示されません。

<a name="implicit-grant"></a>

## 暗黙のグラント (Implicit Grant)

[暗黙](https://tools.ietf.org/html/rfc6749#section-1.3.2)のグラント・フローは、
認可グラント・フローの簡略化された形式です。**Keyrock** は、暫定アクセスコードを
返すのではなく、直接 `access_token` を返します。これは、Authcode フローよりも安
全性は低くなりますが、一部のクライアント・サイドのアプリケーションで使用できます
。

![](https://fiware.github.io/tutorials.Securing-Access/img/implicit-flow.png)

<a name="implicit-grant---sample-code"></a>

### 暗黙のグラント - サンプル・コード

ユーザは最初に **Keyrock** にリダイレクトされ、`token` をリクエストし
、`oa.getAuthorizeUrl()` がフォーム
`/oauth/authorize?response_type=token&client_id={{client-id}}&state=xyz&redirect_uri={{callback_url}}`
の URL を返す必要があります。

```javascript
function implicitGrant(req, res) {
    const path = oa.getAuthorizeUrl("token");
    return res.redirect(path);
}
```

ユーザがアクセスを許可した後、`redirect_uri` によってレスポンスが受信され、以下
のコードで処理され、使用可能なアクセス・トークンを **Keyrock** から受信します。

```javascript
function implicitGrantCallback(req, res) {
    return getUserFromAccessToken(req, req.query.token).then(user => {
        // Store User and return
    });
}
```

<a name="implicit-grant---running-the-example"></a>

### 暗黙のグラント - サンプルの実行

暗黙のグラント・フローをプログラムで呼び出すには、`http://localhost:3000/` のペ
ージを表示して**Implicit Grant** ボタンをクリックします。

ユーザは最初に **Keyrock** にリダイレクトされ、ログインする必要があります

![](https://fiware.github.io/tutorials.Securing-Access/img/keyrock-log-in.png)

ユーザは、リクエストを許可する必要があります

![](https://fiware.github.io/tutorials.Securing-Access/img/keyrock-authorize.png)

レスポンスにより、画面の右上にユーザが表示され、トークンの詳細も画面に表示されま
す。

> **注:** 意図的に　**Keyrock** の http://localhost:3005 からログアウトしない限
> り、すでにアクセスを許可している既存の **Keyrock** セッションは、以降の認可リ
> クエストに使用されます。

<a name="client-credentials-grant"></a>

## クライアント資格情報のグラント (Client Credentials Grant)

最後のグラント・フローの
、[クライアント資格情報](https://tools.ietf.org/html/rfc6749#section-1.3.4)のグ
ラントにはユーザは必要ありません。アプリケーションでは、ユーザではなく、アプリケ
ーションにリソースへのアクセスが許可されるように、アプリケーションを識別する必要
があります。このチュートリアルでは、このような方法で保護されたリソースはありませ
んが、完全性のためにこのフローが含まれています。

![](https://fiware.github.io/tutorials.Securing-Access/img/client-credentials.png)

<a name="logging-in-as-an-application"></a>

### アプリケーションとしてのログイン

クライアント資格情報のフローを使用してログインするには、POST リクエストを
、`grant_type=client_credentials` を指定して、`oauth2/token` エンドポイントに送
信します。

#### :three: リクエスト

```console
curl -iX POST \
  'http://localhost:3005/oauth2/token' \
  -H 'Accept: application/json' \
  -H 'Authorization: Basic dHV0b3JpYWwtZGNrci1zaXRlLTAwMDAteHByZXNzd2ViYXBwOnR1dG9yaWFsLWRja3Itc2l0ZS0wMDAwLWNsaWVudHNlY3JldA==' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data "grant_type=client_credentials"
```

#### レスポンス

レスポンスは、アプリケーション自体を識別するための `access_token` を返します。

```json
{
    "access_token": "3816cf24b2d233546ae9244f6f6fdc327bfba69f",
    "token_type": "Bearer",
    "expires_in": 3599
}
```

<a name="client-credentials-grant---sample-code"></a>

### クライアント資格情報のグラント - サンプル・コード

このコードは、ユーザ資格情報のグラントに似ていますが、明示的なユーザ名またはパス
ワードはありません。

```javascript
function clientCredentialGrant(req, res) {
    oa.getOAuthClientCredentials().then(results => {
        // Store Access token
    });
}
```

<a name="client-credentials-grant---running-the-example"></a>

### クライアント資格情報のグラント - サンプルの実行

クライアント資格情報のグラント・フローをプログラムで呼び出すには
、`http://localhost:3000/` のページをオープンし、**Client Credentials** ボタンを
クリックします。

レスポンスにトークンの詳細が表示されます。ユーザは関与していません。

<a name="refresh-token"></a>

## トークンをリフレッシュ

ユーザーが、適切なグラント・タイプを使用して、自分自身を識別すると、それらが使用
している `access_token` が時間制限されていても、再度ログインする必要はありません
。継続的なアクセスを提供するために、
[リフレッシュ・トークン](https://tools.ietf.org/html/rfc6749#section-1.5)のフロ
ーが定義され、ユーザが期限切れにしたトークンを新しいトークンに交換できるようにな
りました。この交換を提供することは、OAuth2 認証サーバにとって必須ではなく、すべ
てのグラント・タイプには適切ではありません。

<a name="availability-check"></a>

### アベイラビリティ・チェック

リフレッシュ・トークンのフローが利用可能かどうかを確認するには、他のグラント・タ
イプの 1 つを使用してログインするだけです。たとえば、ユーザ・クレデンシャルのフ
ローを使用してログインする場合は、`grant_type=password` を使って POST リクエスト
を `oauth2/token` エンドポイントに送ります。

#### :four: Request

```console
curl -iX POST \
  'http://localhost:3005/oauth2/token' \
  -H 'Accept: application/json' \
  -H 'Authorization: Basic dHV0b3JpYWwtZGNrci1zaXRlLTAwMDAteHByZXNzd2ViYXBwOnR1dG9yaWFsLWRja3Itc2l0ZS0wMDAwLWNsaWVudHNlY3JldA==' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data "username=alice-the-admin@test.com&password=test&grant_type=password"
```

#### レスポンス

ユーザを識別する `refresh_token` とともに、レスポンスは `refresh_token` を返しま
す

```json
{
    "access_token": "a7e22dfe2bd7d883c8621b9eb50797a7f126eeab",
    "token_type": "Bearer",
    "expires_in": 3599,
    "refresh_token": "05e386edd9f95ed0e599c5004db8573e86dff874"
}
```

<a name="refresh-access-token"></a>

### アクセス・トークンをリフレッシュ

上記のレスポンスからの `refresh_token=05e386edd9f95ed0e599c5004db8573e86dff874`
は、後の使用のために保存されています。新しい `access_token` を取得するには (たと
えば、前回の有効期限が過ぎると)、`refresh_token` が OAuth2 リフレッシュ・トーク
ンのフローと `grant_type=refresh_token` で使われます。

#### :five: リクエスト

```console
curl -iX POST \
 'http://localhost:3005/oauth2/token' \
 -H 'Accept: application/json' \
 -H 'Authorization: Basic dHV0b3JpYWwtZGNrci1zaXRlLTAwMDAteHByZXNzd2ViYXBwOnR1dG9yaWFsLWRja3Itc2l0ZS0wMDAwLWNsaWVudHNlY3JldA==' \
 -H 'Content-Type: application/x-www-form-urlencoded' \
 --data "refresh_token=05e386edd9f95ed0e599c5004db8573e86dff874&grant_type=refresh_token"
```

#### レスポンス

レスポンスは以前のレスポンスと似ていますが、`access_token` と `refresh_token` が
更新され、有効期限のウィンドウが前方に移動しました。

```json
{
    "access_token": "298717d478980a2f0c3d2e9f9e222f1bb73e1c69",
    "token_type": "Bearer",
    "expires_in": 3599,
    "refresh_token": "4611e3ab68b5b606eb7a43db6835de646bb7d11d"
}
```

<a name="refresh-token---sample-code"></a>

### トークンのリフレッシュ - サンプル・コード

コードは、すべての OAuth2 呼び出しを別のライブラリ
[oauth2.js](https://github.com/FIWARE/tutorials.Step-by-Step/blob/master/context-provider/lib/oauth2.js)
に委譲します。すべてのリクエストには標準の OAuth2 ヘッダが含まれており、各リクエ
ストはアプリケーションのコードを単純化するという約束でラップされています。リクエ
スト・トークンのフローは、`oa.getOAuthRefreshToken()` 関数を使用して呼び出されま
す。以前のトークンが期限切れになったときに新しい `access_token` を受け取るために
使われた `refresh_token` です。

```javascript
function refreshTokenGrant(req, res) {
    return oa.getOAuthRefreshToken(req.session.refresh_token).then(results => {
        // Store new Access Token
    });
}
```

<a name="refresh-token---running-the-example"></a>

### トークンのリフレッシュ - サンプルの実行

`http://localhost:3000/` ページを立ち上げ、ユーザ名とパスワードのフォームを記入
することで、トークン・リフレッシュのフローをプログラムで呼び出すことができます。

![](https://fiware.github.io/tutorials.Securing-Access/img/tutorial-log-in.png)

レスポンスにより、画面の右上にユーザが表示され、トークンの詳細も画面に表示されま
す :

![](https://fiware.github.io/tutorials.Securing-Access/img/tutorial-reponse.png)

**Refresh Token** ボタンを押すと、ログインしたユーザの新しい `access_token` と
`refresh_token` が返されます。

<a name="pdp---access-control"></a>

# PDP - アクセス制御

PDP アクセス制御には 3 つのレベルがあります :

-   レベル 1 : 認証アクセス - サインインしたすべてのユーザにすべてのアクションを
    許可し、匿名ユーザにはアクションを許可しません
-   レベル 2 : 基本認証 - 現在ログインしているユーザがどのリソースと動詞にアクセ
    スする必要があるかを確認します
-   レベル 3 : 高度な認可 - [XACML](https://en.wikipedia.org/wiki/XACML) による
    詳細な制御

**Keyrock** は、シンプルなレベル 1 とレベル 2 の PDP を独自に提供するために使用
することができ、追加の Generic Enablers を組み合わせることでレベル 3 を提供がで
きます。このチュートリアルでは、ログインしているウェブサイト自体にのみ関心があり
ます。PEP Proxy と組み合わせて他のサービスを保護する方法については、後のチュート
リアルで扱います。

<a name="authentication-access"></a>

## 認証アクセス (Authentication Access)

**Keyrock** または、他の OAuth2 プロバイダが正常にログインした場合、ユーザが存在
することを示す `access_token` が提供されています。この情報は、ユーザーを **認
証** には十分です。

レベル 1 PDP は、任意のフローを使用する任意の OAuth2 プロバイダと組み合わせて使
用できます。

ユーザが **Keyrock** を使用して認証した場合、`/user` エンドポイントに GET リクエ
ストを行うことでアクセス・トークンの最新性を確認できます。

#### :six: リクエスト

```console
 curl -X GET \
  'http://localhost:3005/user?access_token={{access-token}}&app_id={{app-id}}'
```

成功したレスポンスは有効な `access_token` であることを示します。

<a name="authentication-access---sample-code"></a>

### 認証アクセス - サンプル・コード

ユーザがログインしたことを確認するには、ログインしたときにセッションに
`access_token` を保存して、その存在を確認します :

```javascript
function pdpAuthentication(req, res, next) {
    res.locals.authorized = req.session.access_token ? true : false;
    return next();
}
```

**Keyrock** `access_token` が期限切れになっているかどうかを確認するには、リクエ
ストに応じて現在のユーザの詳細を取得しようとすることができます :

```javascript
function pdpAuthentication(req, res, next) {
    const keyrockUserUrl =
        keyrockIPAddress +
        "/user" +
        "?access_token=" +
        req.session.access_token +
        "&app_id=" +
        clientId;
    return oa
        .get(keyrockUserUrl)
        .then(response => {
            res.locals.authorized = true;
            return next();
        })
        .catch(error => {
            debug(error);
            res.locals.authorized = false;
            return next();
        });
}
```

<a name="basic-authorization"></a>

## 基本認可 (Basic Authorization)

レベル 2 PDP は、通常はユーザ資格情報のグラント・フローを介して、自分たちの信頼
できる **Keyrock** インスタンスでのみ使用できます。

**Keyrock** の独自の信頼できるインスタンスを使用している場合、ユーザがサインイン
して `access_token` を取得すると、`access_token` はセッションに保存され、必要に
応じてユーザの詳細を取得するために使用されます。ユーザの詳細のリクエストは、リソ
ースのアクセス許可を含むように拡張できます。この情報を使用して、個々のリソースへ
のアクセスを許可または拒否することができます。

**Keyrock** の許可は URL などの `resource` と HTTP 動詞にマッピング可能な
`action` に基づいています。`/user` GET リクエストにパラメータを追加することで、
アクセス・パーミッションを含む拡張されたユーザの詳細を取得できます。

#### :seven: リクエスト

```console
 curl -X GET \
  'http://localhost:3005/user?access_token={{access-token}}&action={{action}}&resource={{resource}}&app_id={{app-id}}'
```

ここで :

-   `{{access-token}}` は、ログインしているユーザの現在のアクセス・トークンです
    。たとえば、`6c1f1ac938f644c655b9c46c67d9f8b068345e89`
-   `{{action}}` は、HTTP 動詞です。たとえば、`GET`
-   `{{resource}}` は、安全なエンドポイントです。たとえば、`/app/price-change`
-   `{{app-id}}`

#### レスポンス

レスポンスには、アクセスを許可または拒否する `authorization_decision` 属性が含ま
れます。

以下の例では、使用されたアクセス・トークンはマネージャの Bob に属し
、`tutorial-dckr-site-0000-xpresswebapp` 内の `/app/price-change` エンドポイント
へのアクセス権が与えられています。

```json
{
    "organizations": [],
    "displayName": "",
    "roles": [
        {
            "id": "managers-role-0000-0000-000000000000",
            "name": "Management"
        }
    ],
    "app_id": "tutorial-dckr-site-0000-xpresswebapp",
    "trusted_apps": [],
    "isGravatarEnabled": false,
    "email": "bob-the-manager@test.com",
    "id": "bbbbbbbb-good-0000-0000-000000000000",
    "authorization_decision": "Permit",
    "app_azf_domain": "",
    "username": "bob",
    "trusted_applications": []
}
```

<a name="basic-authorization---sample-code"></a>

### 基本認可 - サンプル・コード

したがって、**Keyrock** は単独で PDP として使用できます。ユーザがリソースにアク
セスしてフラグを設定するかどうかを確認するだけです :

```javascript
function pdpBasicAuthorization(req, res, next, url = req.url) {
    const keyrockUserUrl =
        keyrockIPAddress +
        "/user" +
        "?access_token=" +
        req.session.access_token +
        "&action=" +
        req.method +
        "&resource=" +
        url +
        "&app_id=" +
        clientId;
    return oa
        .get(keyrockUserUrl)
        .then(response => {
            const user = JSON.parse(response);
            res.locals.authorized = user.authorization_decision === "Permit";
            return next();
        })
        .catch(error => {
            debug(error);
            res.locals.authorized = false;
            return next();
        });
}
```

保護された Web ページは、`authorized` フラグがセットされているかどうかをチェック
し、許可されていなければユーザをリダイレクトする必要があります。これは、Policy
Enforcement Point (PEP) の一例です :

```javascript
function priceChange(req, res) {
    if (!res.locals.authorized) {
        req.flash("error", "Access Denied");
        return res.redirect("/");
    }
    /// Continue with the normal flow of execution...
}
```

同様に、セキュリティ保護されたコマンドは、ユーザが権限を与えられていない場合、早
く失敗し (Fail Fast)、エラー・コードを返す可能性があります。これは、Policy
Enforcement Point (PEP) の別の例です :

```javascript
function sendCommand(req, res) {
    if (!res.locals.authorized) {
        res.setHeader("Content-Type", "application/json");
        return res.status(403).send({ message: "Forbidden" });
    }
    /// Continue with the normal flow of execution...
}
```

<a name="pdp-access-control---running-the-example"></a>

## PDP アクセス制御 - サンプルの実行

> **注** レベル 2 では、4 つのリソースのみが保護されています :
>
> -   ドアのロックを解除するコマンドを送信
> -   ベルを鳴らすコマンドを送信
> -   価格変更のエリアへのアクセ
> -   注文在庫のエリアへのアクセス

#### 匿名アクセス (Anonymous Access)

-   ユーザとしてログインしていないことを確認してください

##### レベル 1 : 認証アクセス (Authentication Access)

-   任意のストアのページをクリックします - 匿名アクセスのためアクセスは**拒
    否**されます
-   `http://localhost:3000/device/monitor` で、デバイス・モニタをオープンします
    -   ランプのスイッチを入れます - 匿名アクセスのためアクセスが**拒否**されま
        す

##### レベル 2 : 認可アクセス (Authorization Access)

-   `http://localhost:3000` で制限付きアクセス・リンクをクリックしてください -
    アクセスが**拒否**されます
-   `http://localhost:3000/device/monitor` で、デバイス・モニタをオープンします
    -   ドアのロックを解除します - アクセスが**拒否**されます
    -   ベルを鳴らします - アクセスが**拒否**されます
    -   ランプのスイッチを入れます - アクセスが**拒否**されます

#### 盗聴者の Eve

Eve にはアカウントがありますが、アプリケーションにはロールはありません。

> **注** Eve は認識されたアカウントを持っているため、アカウントにはロールが割り
> 当てられていないにもかかわらず、完全な認証アクセスが得られます。

-   `http://localhost:3000` から、ユーザ `eve@example.com` と パスワード `test`
    でログインします

##### レベル 1 : 認証アクセス (Authentication Access)

-   任意のストアのページをクリックしてください - ログインしている任意のユーザに
    アクセスが**許可**されます
-   `http://localhost:3000/device/monitor` で、デバイス・モニタをオープンします
    -   ランプのスイッチを入れます - ログインしている任意のユーザにアクセス
        が**許可**されます

##### レベル 2 : 認可アクセス (Authorization Access)

-   `http://localhost:3000` で制限付きアクセス・リンクをクリックしてください -
    アクセスが**拒否**されます
-   `http://localhost:3000/device/monitor` で、デバイス・モニタをオープンします
    -   ドアのロックを解除します - アクセスが**拒否**されます
    -   ベルを鳴らします - アクセスが**拒否**されます

#### 地域マネージャの Bob

Bob には **management** のロールがあります

-   `http://localhost:3000` から、ユーザ `bob-the-manager@test.com` とパスワード
    `test` でログインします

##### レベル 1 : 認証アクセス (Authentication Access)

-   任意のストアのページをクリックしてください - ログインしている任意のユーザに
    アクセスが**許可**されます
-   `http://localhost:3000/device/monitor` で、デバイス・モニタをオープンします
    -   ランプのスイッチを入れます - ログインしている任意のユーザにアクセス
        が**許可**されます

##### レベル 2 : 認可アクセス (Authorization Access)

-   `http://localhost:3000` で制限されたアクセス・リンクをクリックしてください -
    アクセスは**許可**されます - これは management のみのアクセス許可です
-   `http://localhost:3000/device/monitor` で、デバイス・モニタをオープンします
    -   ドアのロックを解除します - アクセスは**拒否**されます - これは security
        のみの許可です
    -   ベルを鳴らします - アクセスが**許可**されます - これは management ユーザ
        に許可されます

#### セキュリティ管理者の Charlie

Charlie には **security** のロールがあります

-   `http://localhost:3000` から、ユーザ `charlie-security@test.com` とパスワー
    ド `test` でログインします

##### Level 1 : レベル 1 : 認証アクセス (Authentication Access)

-   任意のストアのページをクリックしてください - ログインしている任意のユーザは
    アクセスが**許可**されます
-   `http://localhost:3000/device/monitor` で、デバイス・モニタをオープンします
    -   ランプのスイッチを入れます - ログインしている任意のユーザにアクセス
        が**許可**されます

##### レベル 2 : 認可アクセス (Authorization Access)

-   `http://localhost:3000` で制限されたアクセス・リンクをクリックします。- アク
    セスは**拒否**されました - これは management のみのアクセス許可です
-   `http://localhost:3000/device/monitor` で、デバイス・モニタをオープンします
    -   ドアのロックを解除します - アクセスが**許可**されます - これは security
        のみの許可です
    -   ベルを鳴らします - アクセスが**許可**されます - これは security ユーザに
        許可されます

---

## License

[MIT](LICENSE) © 2018-2020 FIWARE Foundation e.V.
