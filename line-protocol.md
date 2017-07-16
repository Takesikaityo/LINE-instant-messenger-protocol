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

### Sequence 2

    getActivePurchaseVersions(0, 1000, "en", "US")

Mystery. Probably related to sticker versions.

    getConfigurations(...) - parameters involve country codes

Returns a map of configuration settings with string keys. I do not have exact metadata for this
function. Example:

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

Many of the settings seem to be related to features being enabled or disabled and maximum limits for
them.

    getRecommendationIds()

List of suggested friend IDs.

    getBlockedRecommendationIds()

List of suggested friend IDs that have been dismissed (why can't the previous method just not return
these...?)

    getContacts(contactIds) - with IDs from the previous methods

Managing the contact list
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

Sending messages
----------------

Messages are sent using the sendMessage() function.

    sendMessage(seq, msg)

The seq parameter doesn't seem to be matter, and can be sent as zero. The msg parameter is the
Message object to send.

The only required fields for a text message are "to", which can be the ID for any valid message
recipient (user, chat or group), and the "text" field which is the text content to send. Other
message types involve the contentMetadata fields and possibly uploading files to a separate server.

The return value from sendMessage is a partial Message object that only contains the fields "id",
"createdTime" and "from". The ID is a numeric string that can be used to refer to that message
later.

Message types
-------------

LINE supports various types of messages from simple text messages to pictures and video. Each
message has a contentType field that specifies the type of content, and some messages include
attached files from various locations.

Messages can contain extra attributes in the contentMetadata map. One globally used attribute is
"BOT_CHECK" which is included with a value of "1" for automatic messages I've received from
"official accounts" - this could be an auto-reply indicator.

### NONE (0)

The first contentType is called NONE internally but is actually text. It's the simplest content
type. The text field contains the message content encoded in UTF-8.

The only thing to watch out for is emoji which are sent as Unicode private use area codepoints.

Example:

    client.sendMessage(0, line.Message(
        to="u0123456789abcdef0123456789abcdef",
        contentType=line.ContentType.NONE,
        text="Hello, world!"))

TODO: make a list of emoji

### IMAGE (1)

#### Receiving

Image message content can be delivered in one of two ways.

For normal image messages, a preview image is included as a plain JPEG in the contentPreview field.
However, for some reason the official desktop client seems to ignore it and rather downloads
everything from the object storage server.

The preview image URLs are http://os.line.naver.jp/os/m/MSGID/preview and the full-size image URL
are http://os.line.naver.jp/os/m/MSGID where MSGID is the message's id field.

"Official accounts" broadcast messages to many clients at once, so their image message data is
stored on publicly accessible servers (currently seems to be Akamai CDN). For those messages no
embedded preview is included and the image URLs are stored in the contentMetadata map with the
following keys:

* PREVIEW_URL = absolute URL for preview image
* DOWNLOAD_URL = absolute URL for full-size image
* PUBLIC = "TRUE" (haven't seen other values)

As an example of a publicly available image message, have a Pikachu:

http://dl-obs.official.line.naver.jp/r/talk/o/u3ae3691f73c7a396fb6e5243a8718915-1379585871

#### Sending

Sending an image message is done in two steps. First a Thrift sendMessage call is used to obtain a
message ID, and then the image data is uploaded to the Object Storage Server with a separate HTTP
upload request.

The message will not be delivered to the recipient until the HTTP upload is complete. The official
client displays messages in the order of the sendMessage calls, even if the image data is uploaded
much later. It might be possible to have fun by "reserving" a spot for an image message in a
conversation and then filling it in later. It's unknown if there's an internal timeout for uploading
the image data.

In order to send an image message, first send a message normally with contentType=1 (IMAGE) and make
note of the returned message ID. The official client also puts "1000000000" in the text field. The
meaning of this is unknown and it's not required.

The upload HTTP request is a multipart/form-data ("file upload") POST request to the URL:

https://os.line.naver.jp/talk/m/upload.nhn

The request uses the usual X-Line-Application and the X-Line-Access headers for authentication.
There are two fields in the multipart request. The first field is called "params" and the content is
a JSON object containing among other things the message ID. There is on Content-Type header.

{"name":"1.jpg","oid":"1234567890123","size":28878,"type":"image","ver":"1.0"}

The name field does not seem to be used for anything. oid should be set to the message ID obtained
earlier. size should be set to the size of the image file to upload.

The second field is called "file" with a filename argument, has a Content-Type header, and contains
the image data itself. The filename and Content-Type headers seem to be ignored and the image format
is automatically detected. At least JPEG and PNG data is supported for uploading, but everything is
converted to JPEG by the server.

Example sendMessage call:

    # First send the message by using sendMessage
    result = client.sendMessage(0, line.Message(
        to="u0123456789abcdef0123456789abcdef",
        contentType=line.ContentType.IMAGE))

    # Store the ID
    oid = result.id

Example HTTP upload:

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

### STICKER (7)

Sticker messages are simply a reference to a separately hosted image file. The information required
to reference a sticker is contained in the contentMetadata map.

A sticker reference consists of three numbers, the STKVER (sticker version), STKPKGID (sticker
package ID) and STKID (sticker ID within package). To send a sticker, a message with contentType=7
and these three values specified are enough. When receiving a sticker some stickers also return a
meaningful textual name for the sticker in the STKTXT metadata field - this is added automatically
and does not need to be specified when sending.

Sticker image files are hosted on yet another CDN server at dl.stickershop.line.naver.jp. The CDN
server does not require authentication and can be viewed with a plain browser for testing. The base
URL for a sticker package is formed from the STKVER and STKPKGID values. First, the version is split
into three numbers as follows:

    VER = floor(STKVER / 1000000) + "/" + floor(STKVER / 1000) + "/" + (STKVER % 1000)

Using this the package base URL can be determined:

http://dl.stickershop.line.naver.jp/products/{VER}/{STKPKGID}/{PLATFORM}/

PLATFORM is a platform identifier which is presumably used to deliver different image sizes etc to
different platforms. The "WindowsPhone" platform seems to have most interesting files. Other known
platforms are "PC". Not all platforms contain all file types.

Sticker package version 100, package 1 ("Moon & James") is used as an example in the following URLs.
Substitute another package base URL to see other packages.

The sticker package contains a metadata file for a listing its contents:

http://dl.stickershop.line.naver.jp/products/0/0/100/1/WindowsPhone/productInfo.meta

This is a JSON file with metadata about the stickers in this package including names in multiple
languages, the price in multiple currencies and a list of stickers.

Each package also has an icon:

http://dl.stickershop.line.naver.jp/products/0/0/100/1/WindowsPhone/tab_on.png - active

http://dl.stickershop.line.naver.jp/products/0/0/100/1/WindowsPhone/tab_off.png - dimmed

Each referenced sticker image is available in the subdirectory "stickers" as a PNG image. The
filename is {STKID}.png for the full image and {STKID}_key.png for a thumbnail.

http://dl.stickershop.line.naver.jp/products/0/0/100/1/WindowsPhone/stickers/13.png - full size

http://dl.stickershop.line.naver.jp/products/0/0/100/1/WindowsPhone/stickers/13_key.png - thumbnail

All sticker images as well as the icons can be downloaded as a single package from:

http://dl.stickershop.line.naver.jp/products/0/0/100/1/WindowsPhone/stickers.zip

The ShopService is used with the path /SHOP4 to get a list of sticker packages the current user has.
TODO: specify more

Interestingly the official client sends some emoji as a sticker message instead of a plain text
message if the message content consists only of the single emoji. Emoji as stickers are in package
number 5: (TODO: figure out how they're mapped)

http://dl.stickershop.line.naver.jp/products/0/0/100/5/WindowsPhone/productInfo.meta

The official clients also contain references to "old" stickers that have no STKVER or STKPKGID and
use a URL of the format:

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
