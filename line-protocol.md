LINEインスタントメッセンジャープロトコル
===============================

Matti Virkkunen <mvirkkunen@gmail.com>

文書は2015年4月20日現在のものです。

この非公式の文書は、LINE（LINE Corporation / Naverによる）インスタントメッセンジャープロトコルについて説明しています。
情報は主にリバースエンジニアリングに基づいているため、精度は保証されていません。

また、この文書は未完成です。私は私が行くようにそれを拡大しています。

概要
--------

LINE全体としては、多数のWebサービスが多数あります。
デスクトップIMクライアントの複製に必要な唯一のサービス
機能はTalkServiceです。

このプロトコルは、HTTP（S）を介した要求応答アーキテクチャに基づいており、長いポーリングリターン
チャネル。 Apache Thriftは、メッセージデータのシリアル化に使用されます。

ワイヤプロトコル
-------------

ファイル：line.thrift（リバースエンジニアリングによって得られた恐らく完全なThriftインターフェースファイル）

プロトコルはApache Thrift TCompactProtocolからHTTPS経由でgd2.line.naver.jp:443になります。 HTTPリクエスト
パスはほとんどのリクエストで/ S4です。いくつかの特定の要求は、異なるパスを使用します。
関連性があります。

暗号化されていないHTTPも現時点では機能しているようですが、それを使用するのはセキュリティ上の問題です。
ネイバー自体は現在、100％HTTPSに移行しているようです。

キープアライブ接続を使用している場合は、最初の要求のヘッダーを永続化できます。ヘッダー
後続のリクエストで永続化された値を一時的にオーバーライドできます。 X-LSと呼ばれるHTTPヘッダーもあります
関与する。ヘッダーを永続化する場合は、サーバーが提供するX-LSヘッダー値を覚えておく必要があります
次のリクエストでそれを送り返します。値は整数のようです。名前は短いかもしれない
"Line Server"のために使用されています。おそらくロードバランサが次の応答を指示できるように使用されています
ヘッダーを知っているサーバーと同じサーバーに戻します。

永続ヘッダーを使用することにより、X-LSとX-LSの2つのヘッダーだけで各要求を送信することができます。
コンテンツ長

公式のプロトコルは、最初に認証キーを取得するように要求してから、
新しい接続を開いて、認証キーを残りの
次の要求のヘッダー。

タイプとコンセプト
------------------

友人、チャット、グループは、32文字の16進数のGUIDによって識別されます。
タイプ。

内部的には、どのユーザーも連絡先と呼ばれます。連絡先は「ミッド」と接頭辞によって識別されます
"u"（おそらく "user"の場合）です

チャットはルームと呼ばれ、 "ミッド"の前に "r"が付いています（ "ルーム"の場合）。部屋
余分なユーザーをプレーンなIMに招待したときに作成される軽量のマルチユーザーチャットです
連絡先との会話。グループは内部的にグループと呼ばれ、
接頭辞に "c"が付いた "id"（おそらく "チャット"のため）。グループはチャットより永続的です
それらに関連付けられた名前やアイコンなどの余分なデータがあります。

どのメッセージもMessageオブジェクトで表されます。メッセージIDは数値ですが、
文字列。

タイムスタンプは、64ビット整数で表されるミリ秒精度のUNIX時間です（TODO：
タイムゾーンはちょうど）

メッセージ認証
----------------------

要求が成功するには、次のHTTPヘッダーが必要です。

    X-Line-Application: DESKTOPWIN\t3.2.1.83\tWINDOWS\t5.1.2600-XP-x64
    X-Line-Access: authToken

\tエスケープシーケンスはタブ文字を表します。その他のX-Line  - アプリケーション名は存在しますが、これは
1つは現在働いています。アプリケーション名が無効であると、エラーが発生します。 authTokenは
ログイン手順で取得します。

ストレージオブジェクトサーバー
---------------------

メディアファイルは、http://os.line.naver.jp/ の別のサーバーに保存されています。
を「オブジェクト保管サーバー」とします。一部のファイル（メッセージ添付ファイルなど）には、
上記と同じプロトコルで認証しますが、一部のファイル（バディアイコンなど）は
認証が必要です。

これは、上記と同じ認証プロトコルを使用して、HTTPとHTTPSの両方でファイルを処理します。

ログイン手続き
---------------

このThriftメソッドは、電子メールアドレスとパスワードの組み合わせに対して新しいauthTokenを発行します。ザ
authトークンを指定する必要がないように、リクエストをパス/api/v4/TalkService.doに送信する必要があります
まだ存在しない場合（/S4では常に認証トークンが必要です）。

    loginWithIdentityCredentialForCertificate(
        IdentityProvider.LINE, // identityProvider
        "test@example.com", // identifier (e-mail address)
        "password", // password (in plain text)
        true, // keepLoggedIn
        "127.0.0.1", // accesslocation (presumably local IP?)
        "hostname", // systemName (will show up in "Devices")
        "") // certificate (empty on first login - see login verification procedure below)

結果の構造は次のとおりです。

    struct LoginResult {
        1: string authToken;
        2: string certificate;
        3: string verifier;
        4: string pinCode;
        5: LoginResultType type;
    }

ログインに成功すると、タイプはSUCCESS（1）に等しく、authTokenフィールドには
X-Line - 後続の要求で使用するアクセス値。

公式のデスクトップクライアントは、RSAを含む暗号化された電子メール/パスワードを送信し、X-Line-Accessは使用しません
ヘッダーがありますが、プレーンテキストの場合と同じように機能します。 （TODO：RSAログイン手順の説明を含む）

ログインの確認手順
----------------------------

現在のバージョンでは、ログインするときにPINコードを使用してIDを確認する必要があります
はじめてデスクトップクライアントに送信します。これは部分的にジオIPに基づいているようです。
検証手続きが追加される前にログインしていれば、自分自身を検証する必要はありません。新しい
ログインはすべて確認される必要があります。

PIN検証が必要な場合、loginメソッドはREQUIRE_DEVICE_CONFIRM（3）のタイプを返します。
SUCCESS（1）の代わりに。 pinCodeフィールドには、ユーザーに表示するPINコードと、
検証者フィールドは、この検証セッションを識別するために使用されるランダムトークンに設定される。ザ
トークンはプロセス全体で同じままです。

次に、クライアントは、X-Line-AccessヘッダーをHTTPパス/ Qに設定してHTTPパス/ Qに空の要求を発行します。
検証者トークン。この要求は、ユーザーがモバイルに正しいPINコードを入力するまでブロックされます
デバイス。

モバイルデバイス上でPINの入力が間違っているとは限りませんが、
現在3分の制限時間です。その後、トークンは失効します。クライアントは、
時間制限がローカルにあり、要求が終了した時点で要求を中止します。

/ Qからの成功応答は、以下を含むJSONです。

    {
        "timestamp": "946684800000",
        "result": {
            "verifier": "the_verifier_token",
            "authPhase": "QRCODE_VERIFIED"
        }
    }

この応答が受信されると、クライアントはloginWithVerifierForCertificate()呼び出しを発行します。
検証者トークンをパラメータとして使用します。サーバーは通常のLoginReplyメッセージを返します
authToken。 LoginReplyメッセージには証明書の値（ランダムな16進文字列）も含まれています。
将来のloginWithIdentityCredentialForCertificate()の呼び出しで格納されて使用され、
検証ステップ。証明書が使用されていない場合は、すべてのログインでPINの確認が求められます。

トークンがすでに期限切れになっている場合、/Qからの応答は次のようになります。

    {
        "timestamp": "946684800000",
        "errorCode": "404",
        "errorMessage": "key+is+not+found%3A+the_verifier_token+NOT_FOUND"
    }

初期同期
------------

ログイン後、クライアントはサーバーと同期する一連の要求を送信します。それ
一部のメッセージが常に送信されるとは思われません - クライアントはどこかにローカルにデータを格納している可能性があります。
getLastOpRevision()のリビジョンIDと比較します。クライアントは、複数の同期シーケンスを
それをより速くするために平行してください。

これらの同期操作のすべてまたはいずれかをサードパーティのクライアントに実装する必要はありません。

### シーケンス1

これは主な同期シーケンスのようです。

    getLastOpRevision()

後で長いポールリターンチャネルに使用するリビジョンIDを取得します。それが最初にフェッチされて確実に
同期処理中に何かが発生しても何も見逃されません。

    getDownloads()

神秘。おそらくそれは別々の呼び出しであるため、ソフトウェアの更新には関係しません。スタンプのダウンロードに関連する可能性があります。

    getProfile()

現在ログインしているユーザーのプロファイルを取得します。プロファイルには、表示名とステータスメッセージが含まれます。
等々。

    getSettingsAttributes(8458272)
    
保存された設定の一部を取得します。(the bits are NOTIFICATION_INCOMING_CALL, IDENTITY_IDENTIFIER,
NOTIFICATION_DISABLED_WITH_SUB and PRIVACY_PROFILE_IMAGE_POST_TO_MYHOME)

    getAllContactIds()

友人として追加されたすべての連絡先IDを取得します。

    getBlockedContactIds()

ブロックされたユーザーIDのリスト。

    fetchNotificationItems()

神秘

    getContacts(contactIds) - 連絡先IDを取得した以前のメソッドのID

そのユーザーの詳細を取得

    getGroupIdsJoined()

現在のユーザーがメンバーであるすべてのグループを取得します。

    getGroupIdsInvited()

ユーザーが保留中の招待状を持つすべてのグループを取得します。

    getGroups(groupIds) - 以前のメソッドのID

グループの詳細を取得します。これにはメンバーリストが含まれていました。

    getMessageBoxCompactWrapUpList(1, 50)

「現在アクティブなチャット」を持つ複雑な構造を返します。これは部屋のリストを返します。
グループには、部分的な情報と最新のメッセージが含まれています。この呼び出しは、
別のgetRoomsがないので、現在のユーザーがメンバーである部屋のリストを取得する唯一の方法です
方法。

### シーケンス2

    getActivePurchaseVersions(0, 1000, "en", "US")

神秘。おそらくスタンプのバージョンに関連しています。

    getConfigurations(...) - パラメータには国コードが含まれます

文字列キーを使用した構成設定のマップを返します。私はこのための正確なメタデータを持っていません
関数。例：

    {
      "function.linecall.mobile_network_expire_period": "604800",
      "function.linecall.store": "N",
      "contact_ui.show_addressbook": "N",
      "function.music": "N",
      "group.max_size": "200",
      "function.linecall.validate_caller_id": "N",
      "function.linecall.spot": "N",
      "main_tab.show_timeline": "N",
      "function.linecall": "N",
      "room.max_size": "100"
    }

設定の多くは、機能が有効または無効になっていること、および
それら。

    getRecommendationIds()

提示されたフレンドIDのリスト。

    getBlockedRecommendationIds()

友達から外された友人IDのリスト（以前の方法では戻れない
これら...？）

    getContacts(contactIds) - 以前のメソッドのID

連絡先リストの管理
-------------------------

Contacts have multiple statuses.

FRIEND = appears on friend list, unless hidden.

RECOMMEND = appears on "recommended contacts" list.

DELETE = used in notifications only AFAIK, to notify that a friend has been completely deleted.

Each state also has a _BLOCKED version where the current user will not receive messages from the
user. Friends also have a "hidden" status, that is set via the CONTACT_SETTING_CONTACT_HIDE setting
flag. Blocking is done via blockContact/unblockContact.

There is no separate function to delete a contact for some reason, instead it's done by setting the
CONTACT_SETTING_DELETE setting. Even though it's a setting, this is equivalent to really deleting
the friend - they won't appear on getallContactIds() anymore. (FIXME: actually test this...)

メッセージ送信の送信
----------------

メッセージは、sendMessage()関数を使用して送信されます。

    sendMessage(seq, msg)

seqパラメータは重要ではないようで、ゼロとして送信することができます。 msgパラメータは、
送信するメッセージオブジェクト。

テキストメッセージの唯一の必須フィールドは "to"です。これは有効なメッセージのIDです
受信者（ユーザ、チャットまたはグループ）、および送信するテキストコンテンツである「テキスト」フィールドが含まれます。その他
メッセージタイプにはcontentMetadataフィールドが含まれ、別のサーバーにファイルをアップロードする可能性があります。

sendMessageからの戻り値は、フィールド "id"のみを含む部分的なMessageオブジェクトです。
"createdTime"と "from"。 IDは、後でそのメッセージを参照するために使用できる数値の文字列です。

メッセージのタイプ
-------------

LINEは、簡単なテキストメッセージから写真やビデオへのさまざまなタイプのメッセージをサポートします。各
メッセージにはコンテンツタイプを指定するcontentTypeフィールドがあり、一部のメッセージには
さまざまな場所から添付ファイル。

メッセージには、contentMetadataマップに余分な属性を含めることができます。 1つのグローバルに使用される属性は
私が受け取った自動メッセージの値が「1」に含まれている「BOT_CHECK」
「公式アカウント」 - これは自動応答のインジケータになる可能性があります。

### NONE (0)

最初のcontentTypeはNONEと呼ばれますが、実際はテキストです。それは最も簡単なコンテンツです
タイプ。テキストフィールドには、UTF-8でエンコードされたメッセージコンテンツが含まれます。

注意すべき唯一のことは、Unicode専用エリアのコードポイントとして送信される絵文字です。

例：

    client.sendMessage(0, line.Message(
        to="u0123456789abcdef0123456789abcdef",
        contentType=line.ContentType.NONE,
        text="Hello, world!"))

TODO: 絵文字のリストの作成

### IMAGE (1)
受信
#### 

画像メッセージのコンテンツは、2つの方法のいずれかで配信できます。

通常の画像メッセージの場合、プレビュー画像はプレーンJPEGとしてcontentPreviewフィールドに含まれます。
しかし、何らかの理由で、公式のデスクトップクライアントはそれを無視して、むしろオブジェクト保管サーバーからダウンロードします

プレビュー画像URLはhttp://os.line.naver.jp/os/m/MSGID/preview で、フルサイズの画像URL
http://os.line.naver.jp/os/m/MSGID MSGIDはメッセージのIDフィールドです。

「公式アカウント」は一度に多くのクライアントにメッセージをブロードキャストするので、その画像メッセージデータは
（現在はアカマイのCDNと思われる）公開されているサーバーに保存されています。これらのメッセージの場合、
埋め込みプレビューが含まれ、画像のURLはcontentMetadataマップに
以下のキー：

* PREVIEW_URL = absolute URL for preview image
* DOWNLOAD_URL = absolute URL for full-size image
* PUBLIC = "TRUE" (haven't seen other values)

公開されている画像メッセージの例として、ピカチュウ：

http://dl-obs.official.line.naver.jp/r/talk/o/u3ae3691f73c7a396fb6e5243a8718915-1379585871

#### 送信

画像メッセージの送信は2つのステップで行われます。まず、ThriftのsendMessageコールを使用して、
メッセージIDに変換され、その後、イメージデータは別のHTTPを使用してオブジェクトストレージサーバーにアップロードされます
アップロードリクエスト。

HTTPアップロードが完了するまで、メッセージは受信者に配信されません。公式
クライアントは、画像データがアップロードされていても、sendMessage呼び出しの順番でメッセージを表示します
かなり後に。画像メッセージのためにスポットを「予約する」ことによって楽しい時間を過ごすことは可能かもしれません。
会話をし、後でそれを記入してください。アップロードに内部タイムアウトがあるかどうかは不明です
画像データ。

イメージメッセージを送信するには、まずcontentType = 1（IMAGE）のメッセージを通常送信し、
返されたメッセージIDの注記。公式のクライアントは、テキストフィールドに "1000000000"と入力します。ザ
この意味は不明であり、必須ではありません。

アップロードHTTPリクエストは、URLへのmultipart / form-data（「ファイルアップロード」）POSTリクエストです。

https://os.line.naver.jp/talk/m/upload.nhn

この要求では、通常のX-Line-ApplicationヘッダーとX-Line-Accessヘッダーが認証に使用されます。
マルチパートリクエストには2つのフィールドがあります。最初のフィールドは「params」と呼ばれ、コンテンツは
メッセージIDを含むJSONオブジェクト。 Content-Typeヘッダがあります。

{"name":"1.jpg","oid":"1234567890123","size":28878,"type":"image","ver":"1.0"}

名前フィールドは何のためにも使われていないようです。 oidは取得したメッセージIDに設定する必要があります
早くサイズは、アップロードするイメージファイルのサイズに設定する必要があります。

2番目のフィールドはfilename引数を持つ "file"と呼ばれ、Content-Typeヘッダーを持ち、
画像データそのもの。 filenameとContent-Typeヘッダーは無視され、イメージフォーマット
自動的に検出されます。アップロードには少なくともJPEGとPNGデータがサポートされていますが、すべてが
サーバーによってJPEGに変換されます。

sendMessage呼び出しの例:

    # First send the message by using sendMessage
    result = client.sendMessage(0, line.Message(
        to="u0123456789abcdef0123456789abcdef",
        contentType=line.ContentType.IMAGE))

    # Store the ID
    oid = result.id

HTTPアップロードの例:

    POST /talk/m/upload.nhn HTTP/1.1
    Content-Length: 29223
    Content-Type: multipart/form-data; boundary=separator-CU3U3JIM7B17R0C4SIWX1NS7I1G0LV6BF76GPTNN
    Host: obs-de.line-apps.com:443
    X-Line-Access: D82j....=
    X-Line-Application: DESKTOPWIN\t3.6.0.32\tWINDOWS 5.0.2195-XP-x64

    --separator-CU3U3JIM7B17R0C4SIWX1NS7I1G0LV6BF76GPTNN
    Content-Disposition: form-data; name="params"

    {"name":"1.jpg","oid":"1234567890123","size":28878,"type":"image","ver":"1.0"}
    --separator-CU3U3JIM7B17R0C4SIWX1NS7I1G0LV6BF76GPTNN
    Content-Disposition: form-data; name="file"; filename="1.jpg"
    Content-Type: image/jpeg

    ...image data...
    --separator-CU3U3JIM7B17R0C4SIWX1NS7I1G0LV6BF76GPTNN--

### スタンプ (7)

ステッカーメッセージは、個別にホストされたイメージファイルへの参照に過ぎません。必要な情報
ステッカーの参照はcontentMetadataマップに含まれています。

ステッカーの参照は、STKVER（ステッカーバージョン）、STKPKGID（ステッカー
パッケージID）とSTKID（パッケージ内のステッカーID）。ステッカーを送るには、contentType = 7のメッセージ
これらの3つの値で十分です。ステッカーを受け取ると、ステッカーも
STKTXTメタデータフィールドのステッカーの意味のあるテキスト名 - これは自動的に追加されます
送信時に指定する必要はありません。

ステッカーイメージファイルは、dl.stickershop.line.naver.jpのさらに別のCDNサーバーでホストされています。 CDN
サーバーは認証を必要とせず、テストのためにプレーンなブラウザで見ることができます。本拠
ステッカーパッケージのURLは、STKVERおよびSTKPKGIDの値から形成されます。まず、バージョンが分割されている
次の3つの数字に変換します。

    VER = floor(STKVER / 1000000) + "/" + floor(STKVER / 1000) + "/" + (STKVER % 1000)

これを使用して、パッケージのベースURLを決定することができます。

http://dl.stickershop.line.naver.jp/products/{VER}/{STKPKGID}/{PLATFORM}/

PLATFORMは、異なる画像サイズなどを配信するためにおそらく使用されるプラットフォーム識別子です。
異なるプラットフォーム。 「WindowsPhone」プラットフォームには、最も興味深いファイルがあるようです。その他の既知の
プラットフォームは「PC」です。すべてのプラットフォームにすべてのファイルタイプが含まれているわけではありません。

ステッカーパッケージバージョン100、パッケージ1（「Moon＆James」）は、次のURLの例として使用されています。
他のパッケージを見るには別のパッケージのベースURLに置き換えてください。

ステッカーパッケージには、内容を一覧表示するためのメタデータファイルが含まれています。

http://dl.stickershop.line.naver.jp/products/0/0/100/1/WindowsPhone/productInfo.meta

これは、このパッケージ内のステッカーに関するメタデータを含むJSONファイルで、複数の名前
言語、複数の通貨での価格、ステッカーのリスト。

各パッケージにはアイコンもあります：

http://dl.stickershop.line.naver.jp/products/0/0/100/1/WindowsPhone/tab_on.png - アクティブ

http://dl.stickershop.line.naver.jp/products/0/0/100/1/WindowsPhone/tab_off.png - 灰色

参照される各ステッカーイメージは、PNGイメージとしてサブディレクトリー「ステッカー」で使用できます。ザ
ファイル名はサムネイルの場合は{STKID} .png、サムネイルの場合は{STKID} _key.pngです。

http://dl.stickershop.line.naver.jp/products/0/0/100/1/WindowsPhone/stickers/13.png - フルサイズ

http://dl.stickershop.line.naver.jp/products/0/0/100/1/WindowsPhone/stickers/13_key.png - サムネイル

すべてのステッカーイメージとアイコンは、以下から単一のパッケージとしてダウンロードできます。

http://dl.stickershop.line.naver.jp/products/0/0/100/1/WindowsPhone/stickers.zip

ShopServiceはパス/ SHOP4とともに使用され、現在のユーザーが持つステッカーパッケージのリストを取得します。
TODO：もっと指定する

面白いことに、公式のクライアントは、平文の代わりにステレオメッセージとして絵文字を送信します
メッセージの内容が単一の絵文字のみで構成されている場合、メッセージステッカーとしての絵文字はパッケージに入っています
5番：（TODO：マッピング方法を理解する）

http://dl.stickershop.line.naver.jp/products/0/0/100/5/WindowsPhone/productInfo.meta

正式な顧客には、STKVERまたはSTKPKGIDを持たない「古い」ステッカーへの言及も含まれています。
次の形式のURLを使用します。

http://line.naver.jp/stickers/android/{STKID}.png

http://line.naver.jp/stickers/android/13.png

These seem to just be redirects to new URLs now.

Return channel
--------------

    fetchOperations(localRev, count)

For incoming events, fetchOperations() calls to the HTTP path /P4 is used. Using the /P4 path
enables long polling, where the responses block until something happens or a timeout expires. An
HTTP 410 Gone response signals a timed out poll, in which case a new request should be issued.

When new data arrives, a list of Operation objects is returned. Each Operation (except the end
marker) comes with a version number, and the next localRev should be the highest revision number
received.

The official client uses a count parameter of 50.

Operation data is contained either as a Message object in the message field, or in the string fields
param1-param3.

In general NOTIFIED_* messages notify the current user about other users' actions, while their
non-NOTIFIED counterparts notify the current user about their own actions, in order to sync them
across devices.

For many operations the official client doesn't seem to care about the fact that the param1-param3
fields contain the details of the operation and will rather re-fetch data with a get method instead.
For instance, many group member list changes will cause the client to do a getGroup(). This may be
either just lazy coding or a sign of the param parameters being phased out.

The following is a list of operation types.

### END_OF_OPERATION (0)

Signifies the end of the list. This presumably means all operations were returned and none were left
out due to the count param. This message contains no data, not even a revision number, so don't
accidentally set your localRev to zero.

### UPDATE_PROFILE (1)

The current user updated their profile. Refresh using getProfile().

* param1 = UpdateProfileAttributeAttr, which property was changed (possibly bitfield)

### NOTIFIED_UPDATE_PROFILE (2)

Another user updated their profile. Refresh using getContact[s]().

* param1 = the user ID
* param2 = UpdateProfileAttributeAttr, which property was changed (possibly bitfield)

### REGISTER_USERID (3)

(Mystery)

### ADD_CONTACT (4)

The current user has added a contact as a friend.

* param1 = ID of the user that was added
* param2 = (mystery - seen "0")

### NOTIFIED_ADD_CONTACT (5)

Another user has added the current user as a friend.

* param1 = ID of the user that added the current user

### BLOCK_CONTACT (6)

The current user has blocked a contact.

* param1 = ID of the user that was blocked
* param2 = (mystery, seen "NORMAL")

### UNBLOCK_CONTACT (7)

The current user has unblocked a contact.

* param1 = ID of the user that was unblocked
* param2 = (mystery, seen "NORMAL")

### CREATE_GROUP (9)

The current user has created a group. The official client immediately fetches group details with
getGroup().

* param1 = ID of the group.

### UPDATE_GROUP (10)

The current user has updated a group.

* param1 = ID of the group
* param2 = (Maybe a bitfield of properties? 1 = name, 2 = picture)

### NOTIFIED_UPDATE_GROUP (11)

Another user has updated group the current user is a member of.

* param1 = ID of the group
* param2 = ID of the user who updated the group
* param3 = (Maybe a bitfield of properties?)

###  INVITE_INTO_GROUP (12)

The current user has invited somebody to join a group.

* param1 = ID of the group
* param2 = ID of the user that has been invited

### NOTIFIED_INVITE_INTO_GROUP (13)

The current user has been invited to join a group.

* param1 = ID of the group
* param2 = ID of the user who invited the current user
* param3 = ID of the current user

### LEAVE_GROUP (14)

The current user has left a group.

* param1 = ID of the group

### NOTIFIED_LEAVE_GROUP (15)

Another user has left a group the current user is a member of.

* param1 = ID of the group
* param2 = ID of the user that left the group

### ACCEPT_GROUP_INVITATION (16)

The current user has accepted a group invitation.

* param1 = ID of the group

### NOTIFIED_ACCEPT_GROUP_INVITATION (17)

Another user has joined a group the current user is a member of.

* param1 = ID of the group
* param2 = ID of the user that joined the group

### KICKOUT_FROM_GROUP (18)

The current user has removed somebody from a group.

* param1 = ID of the group
* param2 = ID of the user that was removed

### NOTIFIED_KICKOUT_FROM_GROUP (19)

Another user has removed a user from a group. The removed user can also be the current user.

* param1 = ID of the group
* param2 = ID of the user that removed the current user
* param3 = ID of the user that was removed

### CREATE_ROOM (20)

The current user has created a room.

* param1 = ID of the room

### INVITE_INTO_ROOM (21)

The current user has invited users into a room.

* param1 = ID of the room
* param2 = IDs of the users, multiple IDs are separated by U+001E INFORMATION SEPARATOR TWO

### NOTIFIED_INVITE_INTO_ROOM (22)

The current user has been invited into a room. Invitations to rooms to others are not actually sent
until a message is sent to the room.

* param1 = ID of the room
* param2 = ID of the user that invited the current user
* param3 = IDs of the users in the room, multiple IDs are separated by U+001E INFORMATION SEPARATOR
           TWO. The user ID in param2 is not included in this list.

### LEAVE_ROOM (23)

The current user has left a room. Seems to be immediately followed by SEND_CHAT_REMOVED (41).

* param1 = ID of the room

### NOTIFIED_LEAVE_ROOM (24)

Another user has left a room.

* param1 = ID of the room
* param2 = ID of the user that left

### SEND_MESSAGE (25)

Informs about a message that the current user sent. This is returned to all connected devices,
including the one that sent the message.

* message = sent message

### RECEIVE_MESSAGE (26)

Informs about a received message that another user sent either to the current user or to a chat. The
message field contains the message.

The desktop client doesn't seem to care about the included message data, but instead immediately
re-requests it using getNextMessages().

* message = received message

### RECEIVE_MESSAGE_RECEIPT (28)

Informs that another user has read (seen) messages sent by the current user.

* param1 = ID of the user that read the message
* param2 = IDs of the messages, multiple IDs are separated by U+001E INFORMATION SEPARATOR TWO

### CANCEL_INVITATION_GROUP (31)

The current user has canceled a group invitation.

* param1 = ID of the group
* param2 = ID of the user whose invitation was canceled

### NOTIFIED_CANCEL_INVITATION_GROUP (32)

Another user has canceled a group invitation. The canceled invitation can also be that of the
current user.

* param1 = ID of the group
* param2 = ID of the user that canceled the request (or invited them in the first place?)
* param3 = ID of the user whose invitation was canceled

### REJECT_GROUP_INVITATION (34)

The current user has rejected a group infication.

* param1 = ID of the group

### NOTIFIED_REJECT_GROUP_INVITATION (35)

Presumably means another user has rejected a group invitation. However this message doesn't seem to
get sent.

### UPDATE_SETTINGS (36)

User settings have changed. Refresh with getSettingsAttributes() or getSettings()

* param1 = probably bitfield of changed properties
* param2 = probably new value of property (seem things like "bF"/"bT" for a boolean attribute)

### SEND_CHAT_CHECKED (40)

### SEND_CHAT_REMOVED (41)

The current user has cleared the history of a chat.

* param1 = user ID or group ID or room ID
* param2 = seen "990915482402" - maybe an ID, it's similar to IDs

### UPDATE_CONTACT (49)

The current user's settings (e.g. hidden status) for a contact has changed. Refresh with
getContact[s]().

* param1 = ID of the user that changed
* param2 = probably bitfield of changed properties

### (Mystery) 60

Meaning unknown. Has appeared after NOTIFIED_ACCEPT_GROUP_INVITATION and NOTIFIED_INVITE_INTO_ROOM.

Seen the following parameters:

* param1 = a group ID
* param2 = another user's ID

### (Mystery) 61

Meaning unknown. Has appeared after NOTIFIED_LEAVE_GROUP, KICKOUT_FROM_GROUP and
NOTIFIED_KICKOUT_FROM_GROUP and NOTIFIED_LEAVE_ROOM.

Seen the following parameters:

* param1 = a group ID
* param2 = another user's ID
* param3 = "0"
