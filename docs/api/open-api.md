---
sidebar_position: 1
---

# Open API 教程与最佳实践

## 1. 准备工作

### 1.1 注册

在使用 open API 前，首先需要在金色传说（主网环境：[https://jinse.cc/](https://jinse.cc/) ，测试网环境：[https://staging.nervina.c](https://staging.nervina.cn)[n](https://staging.nervina.cn) ）上进行注册，该帐号将会成为通过 open API 设计和发行秘宝的创作者，相关数据可以通过在金色传说平台登录该帐号进行查看。

测试环境请求地址为：[https://goldenlegend.staging.nervina.cn/api/v1/](https://goldenlegend.staging.nervina.cn/api/v1/)

正式环境请求地址为：[https://goldenlegend.nervina.cn/api/v1/](https://goldenlegend.nervina.cn/api/v1/)

发行平台金色传说正式环境：[https://jinse.cc](https://jinse.cc)

发行平台金色传说测试环境：[https://staging.nervina.cn](https://staging.nervina.cn)

钱包正式环境：[https://mibao.net](https://mibao.net)

钱包测试环境：[https://wallet.staging.nervina.cn](https://wallet.staging.nervina.cn)

### 1.2 设置公开信息

完成注册后，必须首先设置创作者（issuer）公开信息才可以进行后续操作。这里设置的公开信息会在链上公开存储，上链操作由金色传说平台自动完成，设计和发行秘宝均置后于该步骤，因此注册后必须执行此操作。

### 1.3 申请 open API 的 key 和 secret

open API 的鉴权需要响应的 key-secret，在完成上述两步操作之后，可以发送邮件到 biz@nervina.io 进行申请，内容必须包含要申请 key-secret 的帐号的注册邮箱。

### 1.4 能量点充值

在金色传说平台设计和分发秘宝都需要消耗能量点（设计秘宝消耗 1 能量点；每分发一个秘宝消耗 1 能量点）；在进行创作和发行前需要进行能量点充值。

完成上述操作后，即可通过 open API 在金色传说平台进行操作。

## 2. 开放 API 签名算法

为了保证安全性，用户在 HTTP 请求中增加 Authorization 的 Header 来包含签名信息，表明这个消息已被授权。

### 2.1 Authorization 字段计算的方法

```plain
Authorization = 'NFT ' + AccessKeyId + ':' + Signature
Signature = base64(hmac-sha1(AccessKeySecret,
              VERB + '\n'
              + FULLPATH + '\n'
              + Content-MD5 + '\n'
              + Content-Type + '\n'
              + Date))
```
* `AccessKeyId` 即为通过邮件申请获取到的 open API 的 key
* `AccessKeySecret` 即为通过邮件申请获取到的 open API 的 secret
* `VERB` 表示请求的 method，包括 GET、POST、PUT 等
* `\n` 表示换行符
* `FULLPATH` 表示请求的 endpoint，如果有 query-string，也需要包含在内
* `Content-MD5` 表示请求内容数据的 MD5 值，对消息内容（不包括头部）计算 MD5 值获得 128 比特位数字，然后对该数字进行 base64 编码即可得到。该请求头可用于消息合法性的检查（消息内容是否与发送时一致）；当正文为空时，Content-MD5 留空即可。详情可参看 [RFC2616 Content-MD5](https://www.ietf.org/rfc/rfc2616.txt)
* `Content-Type` 表示请求内容的类型，如  `application/json` ，也可以为空
* `Date` 表示此次操作的时间，必须为 GMT 格式，如  `Sun, 22 Nov 2015 08:16:38 GMT` 

上述 headers 中， `Date` 和  `Content-Type` 为必须包含的字段，且  `Date` 中的时间和金色传说服务器的时间相差不能超过 15 分钟，否则金色传说服务器将拒绝该请求，并返回 403 错误。

在计算签名时， `Content-Type` 和  `Content-MD5` 可以为空字符，但不能省略。

### 2.2 计算规则

* 签名的字符串必须为  `UTF-8` 格式，含有中文字符的签名字符串必须先进行  `UTF-8` 编码，再进行后续计算。
* 签名的方法为 RFC 2014 中定义的  `HMAC-SHA1` 方法，其中 key 为  `AccessKeySecret` 
* `Content-Type` 和  `Content-MD5` 在请求中不是必须的，如果为空时，留空即可

### 2.3 Python 代码示例

Python 版本为 3.9.1。示例中， `AccessKeyId = '44CF9590006BF252F707'` ， `AccessKeySecret = 'OtxrzxIsfpFjA7SwPzILwy8Bw21TLhquhboDYROV'` 

```plain
# 请求
GET /api/v1/token_classes
Content-MD5: ''
Content-Type: 'application/json'
Date: Tue, 06 Jul 2021 00:00:34 GMT

# 签名字符串为
'GET\n/api/v1/token_classes\n\napplication/json\nTue, 06 Jul 2021 00:00:34 GMT'
```
Python 代码如下：
```python
import base64
import hmac
import hashlib
from datetime import datetime

def get_signature(secret, method, endpoint, content, gmt, content_type='application/json'):
    if content:
        content_md5 = base64.b64encode(
            hashlib.md5(content.encode('utf-8')).digest()).decode('utf-8')
    else:
        content_md5 = ''
    msg = f'{method}\n{endpoint}\n{content_md5}\n{content_type}\n{gmt}'
    h = hmac.digest(secret.encode('utf-8'), msg.encode('utf-8'), hashlib.sha1)
    signature = base64.b64encode(h).decode('utf-8')
    return signature

key = '44CF9590006BF252F707'
secret = 'OtxrzxIsfpFjA7SwPzILwy8Bw21TLhquhboDYROV'
method = 'GET'
endpoint = '/api/v1/token_classes'
content = ''
content_type = 'application/json'
gmt = 'Tue, 06 Jul 2021 00:00:34 GMT'

signature = get_signature(secret, method, endpoint, content, gmt, content_type)    # SXc3VHXXbU08qzYdAm1RvwMWaUw=
auth = f'NFT {key}:{signature}'
print(auth)    # NFT 44CF9590006BF252F707:SXc3VHXXbU08qzYdAm1RvwMWaUw=
```
得到  `auth` 后，最终的请求为：
```plain
GET /api/v1/token_classes
Content-MD5: ''
Content-Type: 'application/json'
Date: Tue, 06 Jul 2021 00:00:34 GMT
Authorization: NFT 44CF9590006BF252F707:SXc3VHXXbU08qzYdAm1RvwMWaUw=
```
### 2.4 常见错误

* 如果传入的  `AccessKeyId` 不存在或 inactive，返回 403 Forbidden, InvalidAccessKeyId
* 如果请求头中的  `Authorization` 格式不对，返回 400 Bad Request, InvalidArgument
* GMT 时间格式必须为  `%a, %d %b %Y %H:%M:%S GMT` 
    * 如果签名验证时，没有传入  `Date` 或格式不正确，返回 403 Forbidden, AccessDenied
    * 如果传入的时间与服务器时间差大于 15 分钟，返回 403 Forbidden, RequestTimeTooSkewed

## 3. 术语说明
* `token_class` : 可以理解为秘宝的模具，包含秘宝的基本信息，将会在链上存储；一个秘宝必须先被设计出秘宝模具，通过秘宝模具才能进行铸造和分发。
* `uuid` : 为了方便使用而被创建出来的一种 id，其中包括：
    * `token_class_uuid` : 关联到对应的  `token_class` 
    * `token_uuid` : 关联到具体的 NFT Token
    * `tx_uuid` : 关联分发和转让的交易
* `address` : ckb 地址
* `nft_type_args` : NFT Token Cell 的 Type Script 中的  `args` 字段

## 4. 设计秘宝、铸造和分发秘宝

相关代码可以参考： [https://github.com/nervina-labs/nft_open_api_demo_python](https://github.com/nervina-labs/nft_open_api_demo_python)

### 4.1 设计秘宝

相关文档：[POST /](https://app.swaggerhub.com/apis/ShiningRay/NftSaasOpenAPI/0.0.1#/default/post_token_classes)[t](https://app.swaggerhub.com/apis/ShiningRay/NftSaasOpenAPI/0.0.1#/default/post_token_classes)[ok](https://app.swaggerhub.com/apis/ShiningRay/NftSaasOpenAPI/0.0.1#/default/post_token_classes)[e](https://app.swaggerhub.com/apis/ShiningRay/NftSaasOpenAPI/0.0.1#/default/post_token_classes)[n](https://app.swaggerhub.com/apis/ShiningRay/NftSaasOpenAPI/0.0.1#/default/post_token_classes)[_](https://app.swaggerhub.com/apis/ShiningRay/NftSaasOpenAPI/0.0.1#/default/post_token_classes)[c](https://app.swaggerhub.com/apis/ShiningRay/NftSaasOpenAPI/0.0.1#/default/post_token_classes)[l](https://app.swaggerhub.com/apis/ShiningRay/NftSaasOpenAPI/0.0.1#/default/post_token_classes)[as](https://app.swaggerhub.com/apis/ShiningRay/NftSaasOpenAPI/0.0.1#/default/post_token_classes)[s](https://app.swaggerhub.com/apis/ShiningRay/NftSaasOpenAPI/0.0.1#/default/post_token_classes)[es](https://app.swaggerhub.com/apis/ShiningRay/NftSaasOpenAPI/0.0.1#/default/post_token_classes)

设计秘宝必须包含以下字段：

* name: 秘宝的名称，不超过 30 个字符（一旦创建，不可修改）
* description: 秘宝的简介，不超过 200 个字符（一旦创建，不可修改）
* total: 非负整数，为 0 时，表示秘宝不限量；大于 0 时，表示秘宝限量的数量
* renderer: 秘宝的媒体信息，必须为以  `https://` 开头、媒体信息有效的 url，且不超过 255 字符（秘宝设计完成后，30 天内可修改一次 renderer）
    * renderer 为图片：支持格式为 png, jpg, jpeg, gif, webp 和 svg 6 种格式，必须以  `https://` 开头，且结尾为 `png | jpg | jpeg | gif | webp | svg`。
    * renderer 为音频或视频：支持格式为 mp3, wav, mp4 和 webm 4 种格式，必须以  `https://` 开头，且结尾为  `mp3 | wav | mp4 | webm`。当 renderer 为音视频时，同时需要传入参数  `cover_image_url` 用于设置 NFT 封面，校验规则与图片格式的 renderer 一致。

设计的秘宝 token class 会上链，上链操作由平台完成。

代码示例：

```python
import json
import requests

key = ''
secret = ''
method = 'POST'
endpoint = '/token_classes'
content = {
    'name': 'New Token Class Example',
    'description': 'Create Token Class by Open API',
    'total': '0',
    'renderer': 'https://oss.jinse.cc/production/7ea62f75-bec0-4cdc-b81a-d59d7f40ace1.jpg'
}
content_type = 'application/json'

url = f'https://goldenlegend.staging.nervina.cn/api/v1{endpoint}'
signature, gmt = get_signature_and_gmt(secret, method, endpoint, content, content_type)
headers = {
    'Content-Type': content_type,
    'Date': gmt,
    'Authorization': f'NFT {key}:{signature}'
}

requests.request(method, url, headers=headers, data=json.dumps(content))
```
返回：
```json
{
    'name': 'New Token Class Example', 
    'description': 'Create Token Class by Open API', 
    'issued': '0', 
    'renderer': 'https://oss.jinse.cc/production/7ea62f75-bec0-4cdc-b81a-d59d7f40ace1.jpg', 
    'uuid': 'e6469650-6ac5-4477-a64b-2a97f4168356', '
    total': '0', 
    'tags': []
}
```
其中， `uuid` 为平台生成的唯一标识 token class id
```python
# 如果创建音频或视频格式的秘宝，则 content 如下所示：
content = {
    'name': 'New Token Class Example',
    'description': 'Create Token Class by Open API',
    'total': '0',
    'renderer': 'https://oss.jinse.cc/production/583b109e-1fc3-42bd-937f-f4935ae80167.mp4',
    'cover_image_url': 'https://oss.jinse.cc/production/7ea62f75-bec0-4cdc-b81a-d59d7f40ace1.jpg'
}
```
* 在设计秘宝时提供可选参数`configure`，用于设置 NFT class cell 的 `configure`，金色传说平台默认设置为 `11000000`， 相关信息可参考：[https://talk.nervos.org/t/rfc-multi-purpose-nft-draft-spec/5434](https://talk.nervos.org/t/rfc-multi-purpose-nft-draft-spec/5434)。 `configure` 格式应为类似  `11000000` 的 8 位二进制字符串，否则会返回  `configure field format error` 

请求示例： 

```python
# 可选参数 configure 设置 NFT class cell 的 configure
content = {
    'name': 'New Token Class with Configure Example',
    'description': 'Create Token Class by Open API',
    'total': '0',
    'configure': '00000000',
    'renderer': 'https://oss.jinse.cc/production/7ea62f75-bec0-4cdc-b81a-d59d7f40ace1.jpg'
}
```
### 4.2 查询 token class

相关文档：[GET /token_classes/{uuid}](https://app.swaggerhub.com/apis/ShiningRay/NftSaasOpenAPI/0.0.1#/default/get_token_classes__uuid__)

查询字段：

* `uuid` : 在创建 token class 时返回的 uuid

代码示例：

```python
import requests

key = ''
secret = ''
method = 'GET'
uuid = '7b1eb753-77a8-46ec-ad8a-d78bc204b8d3'
endpoint = f'/token_classes/{uuid}'
content = ''
content_type = 'application/json'

url = f'https://goldenlegend.staging.nervina.cn/api/v1{endpoint}'
signature, gmt = get_signature_and_gmt(secret, method, endpoint, content, content_type)
headers = {
    'Content-Type': content_type,
    'Date': gmt,
    'Authorization': f'NFT {key}:{signature}'
}

requests.request(method, url, headers=headers)
```
返回：
```json
{
  "name": "New token class",
  "description": "New token class description",
  "issued": "0",
  "renderer": "https://oss.jinse.cc/production/8ee71e29-3b10-4d15-b68c-380c7840c653.jpeg",
  "uuid": "7b1eb753-77a8-46ec-ad8a-d78bc204b8d3",
  "total": "0",
  "verified_info": {
    "is_verified": true,
    "verified_title": "金色传说官方认证",
    "verified_source": "official"
  },
  "tags": []
}
```
### 4.3 查询金色传说账户下所有的 token class

相关文档：[GET /token_classes](https://app.swaggerhub.com/apis/ShiningRay/NftSaasOpenAPI/0.0.1#/default/get_token_classes)

代码示例：

```python
import requests

key = ''
secret = ''
method = 'GET'
endpoint = '/token_classes'
content = ''
content_type = 'application/json'

url = f'https://goldenlegend.staging.nervina.cn/api/v1{endpoint}'
signature, gmt = get_signature_and_gmt(secret, method, endpoint, content, content_type)
headers = {
    'Content-Type': content_type,
    'Date': gmt,
    'Authorization': f'NFT {key}:{signature}'
}

requests.request(method, url, headers=headers)
```
返回：
```json
{
  "token_classes": [
    {
      "name": "New token class",
      "description": "New token class description",
      "issued": "0",
      "renderer": "https://oss.jinse.cc/production/8ee71e29-3b10-4d15-b68c-380c7840c653.jpeg",
      "uuid": "7b1eb753-77a8-46ec-ad8a-d78bc204b8d3",
      "total": "0",
      "is_banned": false,
      "verified_info": {
        "is_verified": true,
        "verified_title": "金色传说官方认证",
        "verified_source": "official"
      },
      "tags": []
    },
    ...
  ],
  "meta": {
    "next_cursor": "1314",
    "has_banned_items": true
  }
}
```
### 4.4 铸造并分发秘宝

相关文档：[POST /token_classes/{token_class_uuid}/tokens](https://app.swaggerhub.com/apis/ShiningRay/NftSaasOpenAPI/0.0.1#/default/post_token_classes__token_class_uuid__tokens)

查询字段：

* `token_class_uuid` : 设计秘宝时返回的 token class uuid

请求字段：

* `addresses` : 一个由 ckb 地址组成的列表

代码示例：

```python
import json
import requests

key = ''
secret = ''
method = 'POST'
uuid = 'e6469650-6ac5-4477-a64b-2a97f4168356'
address = 'ckt1q3vvtay34wndv9nckl8hah6fzzcltcqwcrx79apwp2a5lkd07fdxxqycw877rwy0uuwspsh9cteaf8kqp8nzjl0dxfp'
endpoint = f'/token_classes/{uuid}/tokens'
content = {
    'addresses': [
        address,
        address
    ]
}
content_type = 'application/json'

url = f'https://goldenlegend.staging.nervina.cn/api/v1{endpoint}'
signature, gmt = get_signature_and_gmt(secret, method, endpoint, content, content_type)
headers = {
    'Content-Type': content_type,
    'Date': gmt,
    'Authorization': f'NFT {key}:{signature}'
}

requests.request(method, url, headers=headers, data=json.dumps(content))
```
返回：
```json
[
    {
        'uuid': 'e7c87177-2461-4ea1-8b8f-35d8b2ca34ea', 
        'oid': '3eefc8278f217311c88c446731b703d7', 
        'version': 0, 
        'characteristic': '0000000000000000', 
        'issuer_id': 1, 
        'token_class_id': 626, 
        'state': 'pending', 
        'configure': 192, 
        'n_issuer_id': '0xf90f9c38b0ea0815156bbc340c910d0a21ee57cf', 
        'n_state': 0, 
        'created_at': '2021-07-06T01:57:48.140Z', 
        'updated_at': '2021-07-06T01:57:48.140Z'
    }, {
        'uuid': 'a46916b6-b180-4020-ab2b-59377fe3d9e9', 
        'oid': '3eefc8278f217311c88c446731b703d7', 
        'version': 0, 
        'characteristic': '0000000000000000', 
        'issuer_id': 1, 
        'token_class_id': 626, 
        'state': 'pending', 
        'configure': 192, 
        'n_issuer_id': '0xf90f9c38b0ea0815156bbc340c910d0a21ee57cf', 
        'n_state': 0, 
        'created_at': '2021-07-06T01:57:48.140Z', 
        'updated_at': '2021-07-06T01:57:48.140Z'
    }
]
```
其中，  `uuid` 是具体的 NFT token id

### 4.5 查询 NFT token

相关文档：[GET /tokens/{token_uuid}](https://app.swaggerhub.com/apis/ShiningRay/NftSaasOpenAPI/0.0.1#/default/get_tokens__uuid_)

查询参数：

* `token_uuid` : 铸造并分发秘宝时返回的 token uuid

代码示例：

```python
import requests

key = ''
secret = ''
method = 'GET'
token_uuid = '992179fc-515e-42d7-b731-79ae23c92f6c'
endpoint = f'/tokens/{token_uuid}'
content = ''
content_type = 'application/json'

url = f'https://goldenlegend.staging.nervina.cn/api/v1{endpoint}'
signature, gmt = get_signature_and_gmt(secret, method, endpoint, content, content_type)
headers = {
    'Content-Type': content_type,
    'Date': gmt,
    'Authorization': f'NFT {key}:{signature}'
}

requests.request(method, url, headers=headers)
```
返回：
```json
{
  "name": "First NFT 009😄",
  "description": "first nft 009\nasdfasdfas",
  "issued": "6",
  "total": "11",
  "bg_image_url": "https://img.ggg.com/xxxxxx.jpg",
  "issuer_info": {
    "name": "Alice",
    "avatar_url": null,
    "uuid": "6c7d8931-d3ef-4fbb-ae70-0db1c9353f81"
  },
  "tx_state": "committed",
  "class_uuid": "6bf5ae4f-c941-460a-bdc7-1e67115ee740",
  "from_address": "6c7d8931-d3ef-4fbb-ae70-0db1c9353f81",
  "to_address": "ckt1qyqt8lh56d05m9ylnme0rks5phx75pk7ppjqkjmndc",
  "is_class_banned": false,
  "is_issuer_banned": false,
  "n_token_id": 5,
  "verified_info": {
    "is_verified": null,
    "verified_title": "",
    "verified_source": ""
  },
  "class_likes": 0,
  "class_liked": false,
  "uuid": "992179fc-515e-42d7-b731-79ae23c92f6c",
  "created_at": "2021-07-22T08:46:22.589Z",
  "distribution_method": "send",
  "tx_hash": "0x01c308910bc4840c1cde69d15365a9f3b8c4756b955255843c5c7d5046b27f47",
  "tx_uuid": "71e5d500-f0b1-4d47-8206-54ec09a09897"
}
```
### 4.6 根据 token id 查询持有者地址

相关文档：[GET /tokens/{token_uuid}/address](https://app.swaggerhub.com/apis/ShiningRay/NftSaasOpenAPI/0.0.1#/default/get_tokens__uuid__address)

查询参数：

* `token_uuid` : 铸造并分发后返回的 token uuid

代码示例：

```python
import requests

key = ''
secret = ''
method = 'GET'
token_uuid = 'fd9b685b-a37f-45e5-81cf-0043433460b8'
endpoint = f'/tokens/{token_uuid}/address'
content = ''
content_type = 'application/json'

url = f'https://goldenlegend.staging.nervina.cn/api/v1{endpoint}'
signature, gmt = get_signature_and_gmt(secret, method, endpoint, content, content_type)
headers = {
    'Content-Type': content_type,
    'Date': gmt,
    'Authorization': f'NFT {key}:{signature}'
}

requests.request(method, url, headers=headers, data=content)
```
返回：
```json
{
    'address': 'ckt1qjda0cr08m85hc8jlnfp3zer7xulejywt49kt2rr0vthywaa50xws7nx9v0ycll73vnzpsc0nvm3rh8jkc5g2a7xm59'
}
```
 
## 5. NFT 持有者相关查询

### 5.1 查询地址所持有的 NFT

相关文档：[GET /indexer/holder_tokens/{address}](https://app.swaggerhub.com/apis/ShiningRay/NftSaasOpenAPI/0.0.1#/indexer/get_indexer_holder_tokens__address_)

查询参数：

* `address` : 要查询的 ckb 地址

代码示例：

```python
import requests

key = ''
secret = ''
method = 'GET'
address = 'ckt1q3vvtay34wndv9nckl8hah6fzzcltcqwcrx79apwp2a5lkd07fdxxqycw877rwy0uuwspsh9cteaf8kqp8nzjl0dxfp'
endpoint = f'/indexer/holder_tokens/{address}'
content = ''
content_type = 'application/json'

url = f'https://goldenlegend.staging.nervina.cn/api/v1{endpoint}'
signature, gmt = get_signature_and_gmt(secret, method, endpoint, content, content_type)
headers = {
    'Content-Type': content_type,
    'Date': gmt,
    'Authorization': f'NFT {key}:{signature}'
}

requests.request(method, url, headers=headers, data=content)
```
返回：
```json
{
    'holder_address': 'ckt1q3vvtay34wndv9nckl8hah6fzzcltcqwcrx79apwp2a5lkd07fdxxqycw877rwy0uuwspsh9cteaf8kqp8nzjl0dxfp',
    'meta': {
        'total_count': 9,
        'max_page': 1,
        'current_page': 1
    },
    'token_list': [
        {
            'token_uuid': 'a46916b6-b180-4020-ab2b-59377fe3d9e9',
            'n_token_id': 1,
            'class_uuid': 'e6469650-6ac5-4477-a64b-2a97f4168356',
            'class_name': 'New Token Class Example',
            'class_bg_image_url': 'https://examples.jpg',
            'class_description': 'Create Token Class by Open API',
            'class_total': '0',
            'class_issued': '2',
            'is_class_banned': false,
            'tx_state': 'committed',
            'issuer_name': '名字很长的创作者名字很长的创作者名很长的创作者名字很长的创',
            'issuer_avatar_url': 'https://images.pexels.com/photos/3396664/peels-photo-3396664.jpeg',
            'issuer_uuid': 'f0d044c2-9ddb-4406-87b3-b281fbb27a76',
            'is_issuer_banned': false,
            'from_address': 'f0d044c2-9ddb-4406-87b3-b281fbb27a76',
            'to_address': 'ckt1q3vvtay34wndv9nckl8hah6fzzcltcqwcrx79apwp2a5lkd07fdxxqycw877rwy0uuwspsh9cteaf8kqp8nzjl0dxfp',
            'token_outpoint': {
                'tx_hash': '0x24ec444afe9d44e664214f659109d4ff852dc72c065f0f12dc921a65df6e4b13',
                'index': 2
            },
            'verified_info': {
                'is_verified': true,
                'verified_title': '知名Rapper,Hiphop文化推广者',
                'verified_source': 'official'
            },
            'class_likes': 0
        }
    ]
}
```
### 5.2 查询 nft type args 对应的 token

相关文档：[GET /indexer/tokens/{nft_type_args}](https://app.swaggerhub.com/apis/ShiningRay/NftSaasOpenAPI/0.0.1#/indexer/get_indexer_tokens__nft_type_args_)

查询参数：

* `nft_type_args` : 在对应的 CKB Cell 的 Type Script 中的 args

代码示例：

```python
import requests

key = ''
secret = ''
method = 'GET'
nft_type_args = '0xf90f9c38b0ea0815156bbc340c910d0a21ee57cf0000002900000000'
endpoint = f'/indexer/tokens/{nft_type_args}'
content = ''
content_type = 'application/json'

url = f'https://goldenlegend.staging.nervina.cn/api/v1{endpoint}'
signature, gmt = get_signature_and_gmt(secret, method, endpoint, content, content_type)
headers = {
    'Content-Type': content_type,
    'Date': gmt,
    'Authorization': f'NFT {key}:{signature}'
}

requests.request(method, url, headers=headers, data=content)
```
返回：
```json
{
    'token': {
        'id': 3271, 
        'uuid': 'db144551-e65f-4dc4-9037-fc017734ef7b', 
        'issuer_id': 1, 
        'token_class_id': 620, 
        'n_issuer_id': '0xf90f9c38b0ea0815156bbc340c910d0a21ee57cf',
        'n_token_id': 0, 
        'version': 0, 
        'configure': 192, 
        'characteristic': '0000000000000000', 
        'n_state': 0, 
        'state': 'committed', 
        'created_at': '2021-07-05T11:40:02.071Z', 
        'updated_at': '2021-07-05T11:40:02.130Z', 
        'oid': '84cbe247c0792718523fecec799b662b', 
        'is_committed': True,
        'verified_info': {
            'is_verified': true,
            'verified_title': '知名Rapper,Hiphop文化推广者',
            'verified_source': 'official'
        },
    }, 
    'cell': {
        'block_number': 2024829, 
        'out_point': {
            'tx_hash': '0x511af426c9117d5040c05e95538ba3525d806d97f74a0688cbb835023dd30c7e', 
            'index': 1
        }, 
        'output': {
            'capacity': 13400000000, 
            'lock': {
                'code_hash': '0x58c5f491aba6d61678b7cf7edf4910b1f5e00ec0cde2f42e0abb4fd9aff25a63', 
                'args': '0x9b67065717f46e887d8b9647571db949beec9933', 
                'hash_type': 'type'
            }, 
            'type': {
                'code_hash': '0xb1837b5ad01a88558731953062d1f5cb547adf89ece01e8934a9f0aeed2d959f', 
                'args': '0xf90f9c38b0ea0815156bbc340c910d0a21ee57cf0000002900000000', 
                'hash_type': 'type'
            }
        }, 
    'output_data': '0x000000000000000000c000', 
    'tx_index': 1
    }
}
```
## 6. 转让交易 

创作者完成铸造和分发秘宝之后，持有者可以通过自己的密钥签名发起转让交易。

此处要求应用的开发人员自己使用 CKB 的 SDK 来生成地址，并在自己的系统中存储公私钥（金色传说平台不涉及持有人的密钥管理），并使用密钥在转账交易中进行签名。

### 6.1 生成转让交易

相关文档：[GET /tx/token_transfers/new](https://app.swaggerhub.com/apis/ShiningRay/NftSaasOpenAPI/0.0.1#/unsigned_tx_generators/get_tx_token_transfers_new)

查询参数：

* `from_address` : token 持有者的地址，同时也是需要对生成交易签名的私钥持有者
* `to_address` : 转让交易的目标地址，一旦转让，交易无法撤回
* `token_uuid` : 要转让 token 的 uuid
* `nft_type_args` : token 对应 cell 的 type script args
* `token_uuid` 和  `nft_type_args` 两者必须满足有且仅有一个

代码示例：

```python
import requests

key = ''
secret = ''
method = 'GET'
from_address = 'ckt1q3vvtay34wndv9nckl8hah6fzzcltcqwcrx79apwp2a5lkd07fdx83pmv9wj80kf0w5zfym9am9eply253tuu8v5lsn'
to_address = 'ckt1q3vvtay34wndv9nckl8hah6fzzcltcqwcrx79apwp2a5lkd07fdxxqycw877rwy0uuwspsh9cteaf8kqp8nzjl0dxfp'
# nft_type_args = 'cf12d78e-96b9-40d1-b6ca-1d5ff1d87287'
# endpoint = f'/api/v1/tx/token_transfers/new?from_address={from_address}&to_address={to_address}&nft_type_args={nft_type_args}'
token_uuid = 'a46916b6-b180-4020-ab2b-59377fe3d9e9'
endpoint = f'/tx/token_transfers/new?from_address={from_address}&to_address={to_address}&token_uuid={token_uuid}'
content = ''
content_type = 'application/json'

url = f'https://goldenlegend.staging.nervina.cn/api/v1{endpoint}'
signature, gmt = get_signature_and_gmt(secret, method, endpoint, content, content_type)
headers = {
    'Content-Type': content_type,
    'Date': gmt,
    'Authorization': f'NFT {key}:{signature}'
}

requests.request(method, url, headers=headers, data=content)
```
返回：
```json
{
    "unsigned_tx": {
        "version": "0x0",
        "cell_deps": [
            {
                "out_point": {
                    "tx_hash": "0xbd262c87a84c08ea3bc141700cf55c1a285009de0e22c247a8d9597b4fc491e6",
                    "index": "0x2"
                },
                "dep_type": "code"
            },
            {
                "out_point": {
                    "tx_hash": "0xd346695aa3293a84e9f985448668e9692892c959e7e83d6d8042e59c08b8cf5c",
                    "index": "0x0"
                },
                "dep_type": "code"
            },
            {
                "out_point": {
                    "tx_hash": "0x03dd2a5594ed2d79196b396c83534e050ba0ad07fa5c1cd61a7094f9fb60a592",
                    "index": "0x0"
                },
                "dep_type": "code"
            },
            {
                "out_point": {
                    "tx_hash": "0xf8de3bb47d055cdf460d93a2a6e1b05f7432f9777c8c474abf4eec1d4aee5d37",
                    "index": "0x0"
                },
                "dep_type": "dep_group"
            }
        ],
        "header_deps": [],
        "inputs": [
            {
                "previous_output": {
                    "tx_hash": "0x0e9cad38a05005dff037c5c1df917efe5be53409976405a330ed15e14f939d3b",
                    "index": "0x0"
                },
                "since": "0x0",
                "capacity": "0x31eb3ba4e",
                "lock": {
                    "code_hash": "0x58c5f491aba6d61678b7cf7edf4910b1f5e00ec0cde2f42e0abb4fd9aff25a63",
                    "args": "0xc43b615d23bec97ba8249365eecb90fc8aa457ce",
                    "hash_type": "type"
                },
                "type": {
                    "code_hash": "0xb1837b5ad01a88558731953062d1f5cb547adf89ece01e8934a9f0aeed2d959f",
                    "args": "0xf90f9c38b0ea0815156bbc340c910d0a21ee57cf0000002300000000",
                    "hash_type": "type"
                }
            }
        ],
        "outputs": [
            {
                "capacity": "0x31eb3b450",
                "lock": {
                    "code_hash": "0x58c5f491aba6d61678b7cf7edf4910b1f5e00ec0cde2f42e0abb4fd9aff25a63",
                    "args": "0x009871fde1b88fe71d00c2e5c2f3d49ec009e629",
                    "hash_type": "type"
                },
                "type": {
                    "code_hash": "0xb1837b5ad01a88558731953062d1f5cb547adf89ece01e8934a9f0aeed2d959f",
                    "args": "0xf90f9c38b0ea0815156bbc340c910d0a21ee57cf0000002300000000",
                    "hash_type": "type"
                }
            }
        ],
        "outputs_data": [
            "0x000000000000000000c000"
        ],
        "witnesses": ["0x"]
    }
}
```
### 6.2 对签名后的交易发送上链

相关文档：[POST /tx/token_transfers](https://app.swaggerhub.com/apis/ShiningRay/NftSaasOpenAPI/0.0.1#/unsigned_tx_generators/post_tx_token_transfers)

查询参数：

*  `from_address` : NFT token 持有者地址，也是需要对交易签名的私钥持有者
*  `to_address` : 转让交易的目标地址
*  `nft_type_args` : token 对应 cell 的 type script args
*  `token_uuid` : token 在金色传说平台对应的唯一 uuid
*  `signed_tx` : 签名完成的交易字符串

代码示例：

```python
import requests

key = ''
secret = ''
method = 'POST'
endpoint = '/tx/token_transfers'
content = {
    'from_address': 'ckt1qjda0cr08m85hc8jlnfp3zer7xulejywt49kt2rr0vthywaa50xws7nx9v0ycll73vnzpsc0nvm3rh8jkc5g2a7xm59', 
    'to_address': 'ckt1q3vvtay34wndv9nckl8hah6fzzcltcqwcrx79apwp2a5lkd07fdxxqycw877rwy0uuwspsh9cteaf8kqp8nzjl0dxfp',
    'nft_type_args': '0xf90f9c38b0ea0815156bbc340c910d0a21ee57cf0000003300000002',
    'token_uuid': 'f40588f2-7ed3-468c-afbf-ef92f2ecf99a',
    'signed_tx': '{"version":"0x0","cell_deps":[{"out_point":{"tx_hash":"0xbd262c87a84c08ea3bc141700cf55c1a285009de0e22c247a8d9597b4fc491e6","index":"0x2"},"dep_type":"code"},{"out_point":{"tx_hash":"0xd346695aa3293a84e9f985448668e9692892c959e7e83d6d8042e59c08b8cf5c","index":"0x0"},"dep_type":"code"},{"out_point":{"tx_hash":"0x03dd2a5594ed2d79196b396c83534e050ba0ad07fa5c1cd61a7094f9fb60a592","index":"0x0"},"dep_type":"code"},{"out_point":{"tx_hash":"0xf8de3bb47d055cdf460d93a2a6e1b05f7432f9777c8c474abf4eec1d4aee5d37","index":"0x0"},"dep_type":"dep_group"}],"header_deps":[],"inputs":[{"previous_output":{"tx_hash":"0x52419c2bf7131908052e7fbed374ca82f3b9a23c091bcfcb487c4ac1c4c90717","index":"0x1"},"since":"0x0"}],"outputs":[{"capacity":"0x31eb3c002","type":{"code_hash":"0xb1837b5ad01a88558731953062d1f5cb547adf89ece01e8934a9f0aeed2d959f","args":"0xf90f9c38b0ea0815156bbc340c910d0a21ee57cf0000003300000002","hash_type":"type"},"lock":{"code_hash":"0x58c5f491aba6d61678b7cf7edf4910b1f5e00ec0cde2f42e0abb4fd9aff25a63","args":"0x009871fde1b88fe71d00c2e5c2f3d49ec009e629","hash_type":"type"}}],"outputs_data":["0x000000000000000000c000"],"witnesses":["0x5500000010000000550000005500000041000000c3e23abd5f3fb6350d138c7e25933c1a7e150d6aa75f8c118e55233101052af079ba116a515180252b73e2cbe02a2924118ed710c7e5c5e80c7e5b80258be45800"]}'
}
content_type = 'application/json'

url = f'https://goldenlegend.staging.nervina.cn/api/v1{endpoint}'
signature, gmt = get_signature_and_gmt(secret, method, endpoint, content, content_type)
headers = {
    'Content-Type': content_type,
    'Date': gmt,
    'Authorization': f'NFT {key}:{signature}'
}

requests.request(method, url, headers=headers, data=content)
```
返回：
```json
{
    "tx_hash": "0x8ea665214de0a494f8df0125cbf492e96b2ec1df2cae8569bfe1d27759a3925b",
    "uuid": "396d73ca-f0b2-44ef-93a0-ea0d40e6fdc0"
}
```
 返回的  `tx_hash` 为该交易在链上的交易哈希，可在浏览器对应查看；  `uuid` 即为该交易在金色传说平台对应的交易 id，通过其他接口可查询交易状态。

### 6.3 查询交易状态

相关文档：[GET /tx/token_transfers/{uuid}](https://app.swaggerhub.com/apis/ShiningRay/NftSaasOpenAPI/0.0.1#/unsigned_tx_generators/get_tx_token_transfers__uuid_)

查询参数：

* `uuid` : 交易的 tx uuid

代码示例：

```python
import requests

key = ''
secret = ''
method = 'GET'
tx_uuid = '75fddac0-7cda-4d9f-a770-56ee8ab6489c'
endpoint = f'/tx/token_transfers/{tx_uuid}'
content = ''
content_type = 'application/json'

url = f'https://goldenlegend.staging.nervina.cn/api/v1{endpoint}'
signature, gmt = get_signature_and_gmt(secret, method, endpoint, content, content_type)
headers = {
    'Content-Type': content_type,
    'Date': gmt,
    'Authorization': f'NFT {key}:{signature}'
}

requests.request(method, url, headers=headers, data=content)
```
返回：
```json
{
    "from_address": "a2406443-4fbc-4058-ace2-ea2de19bace3",
    "to_address": "ckt1qsfy5cxd0x0pl09xvsvkmert8alsajm38qfnmjh2fzfu2804kq47v67zaqf9v82ypv7maevz5h70va4eevpggc8gjyw",
    "tx_type": "issue",
    "on_chain_timestamp": 1625492143,
    "tx_state": "committed",
    "issuer_name": "issuer_name",
    "issuer_avatar_url": "https://goldenlegend.oss-cn-hangzhou.aliyuncs.com/production/1620901052284.gif",
    "class_bg_image_url": "https://oss.jinse.cc/production/c8eb25b0-457c-4ac0-98b8-4308ac465893.png",
    "class_name": "Test",
    "class_total": "5",
    "class_uuid": "c2b0c5fb-6145-485d-995e-dd265b17471d",
    "n_token_id": 1
}
```