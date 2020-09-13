# 本文介绍nucypher演示

## 1.工作流程
nucypher主要思想，本例使用四个节点分别模仿alice，bob，enrico和ursula，每一个角色所代笔的含义参见前文。

## 2.本例目的
alice有一些数据，想要发送给一个或一些bob，这些数据可以假设是由enrico产生的。但是alice并不想直接使用bob的公钥对数据进行加密，因为下次alice想要把数据发送给另一个bob时，alice需要用另一个bob的公钥对数据再次进行加密，不仅浪费存储资源还浪费计算资源。所有引入了一个第三方代理重加密节点，即ursula。alice通过对bob进行授权，设置策略公钥（基于过期时间、标签分类等策略）。alice不用每次都用bob的公钥加密，但是bob可以通过自己私钥对数据进行解密。  
本例中通过简单的一个ursula节点，对数据进行重加密，bob可以拿到alice的数据。

## 3.操作流程
### 3.1系统环境
ubuntu20.04  
python3.8.2  
nucypher 3.0.0-beta.0
### 3.2节点运行
分别打开四个终端，进入nucypher工作的虚拟环境。
#### 3.2.1 运行ursula节点
```sh
(my_nucy_env) aha@aha:~/Desktop/nucypher$ nucypher ursula run --dev --federated-only --console-logs
```
这一步至关重要，因为后续的alice，bob，enrico的运行都要使用ursula提供的服务。  
运行后可以看到如下界面

（图片）

#### 3.3.2 运行alice节点
```sh
(my_nucy_env) aha@aha:~/Desktop/nucypher$ nucypher alice run --dev --federated-only --teacher localhost:10151 --console-logs
```
运行后显示如下。在这里可以看到Alice_Verifying_Key。

（图片）

#### 3.3.3 运行enrico节点
在运行enrico之前，我们需要拿到alice给enrico分配的策略密钥。  
在这里我们使用python+jupyter进行演示如何拿到策略加密密钥。  
```python
alice = "http://localhost:8151"
Alice_Verifying_Key = "03bebf3d9631d53c4797be3246c49dcc213b2cb3843abf35449430030e7c7339b6"
derivation_response = requests.post(f"{alice}/derive_policy_encrypting_key/songs")
```
上一步的接口调用过程建议使用postman进行测试。  
```python
derivation_response.content
#可以看到其内容，policy_encrypting_key
b'{"result": {"policy_encrypting_key": "0224aa3b7e9f35d9fd0c7ab335068d9f43d20d14ec7473a0ca5848681acbb21454", "label": "songs"}, "version": "3.0.0-beta.0"}'
```
拿到`policy_encrypting_key`后在终端即可运行enrico节点。

```sh
nucypher enrico run --policy-encrypting-key 0224aa3b7e9f35d9fd0c7ab335068d9f43d20d14ec7473a0ca5848681acbb21454 --http-port 5152 --console-logs
```
运行成功后，可以看到下图输出。  

（图片）

#### 3.3.4 运行bob节点
> note:如果alice想要将数据发送给其他的人，其使用角色仍是bob

```sh
(my_nucy_env) aha@aha:~/Desktop/nucypher$ nucypher bob run --dev --federated-only --teacher localhost:10151 --controller-port 11151 --console-logs
```
成功运行后，可以看到下图输出。这里的`Bob_Verifying_Key`和`Bob_Encrypting_Key`在下一步中都要用到。

（图片）

### 3.3 alice向bob发送消息
#### 3.3.1 python需要使用的包
```python
from base64 import b64decode,b64encode
import json
import requests
```
#### 3.3.2 构建alice对象
```python
alice = "http://localhost:8151"
Alice_Verifying_Key = "03bebf3d9631d53c4797be3246c49dcc213b2cb3843abf35449430030e7c7339b6"
derivation_response = requests.post(f"{alice}/derive_policy_encrypting_key/songs")
```
这里通过`post`请求向alice发送数据，alice能够生成策略密钥，用于enrico加密数据。加入alice有很多数据，但是她不想把所有的数据都发送给bob，所有可以使用标签对数据进行分类，这里的`songs`即时标签`label`,可以是任意字符,表示alice仅想向bob分享此类数据。

#### 3.3.3 构建enrico对象
```python
enrico = "http://localhost:5152"
```

#### 3.3.4 构段一段消息
构造一段明文消息，并封装成nucypher可以使用的数据类型（使用base64编码）。
```python
message_to_encrypt = b"This is some plaintext"
encryption_request = { "message":b64encode(message_to_encrypt).decode() }
```
这里可以输出看到明文信息使用base64编码后`encryption_request`的具体数值`{'message': 'VGhpcyBpcyBzb21lIHBsYWludGV4dA=='}`。

#### 3.3.5 enrico对数据进行加密
```python
encryption_response = requests.post( f"{enrico}/encrypt_message",data=json.dumps(encryption_request) )
```
使用post请求将数据传送给enrico，可以拿到加密后的数据。

```python

encryption_response.content

b'{"result": {"message_kit": "Ag7I5VUWoDpUwTcZ1aEmi36lIBFMHrv47iNQYbW7SJGcAjEIt9B6inZcprzdXURBLR71myktmLZYRiryZqe4fEWCTwYmxOsCSSGXt2FjTaJWR0mwx6pP1ZNIpQA7XOp4HXcDWvWthNC5bxbzEXTTl+4yDgFo8AonRyqqjZ+HHUONm+kAAACEvQDpeQff8QhoQkFnXZt0k/+S1/uZAkf3FFYj3Ur2Bl83B5zF3t0c2jVPMp787dvQoMR54wRVNnCw6YsNZUGS0BWzhOlm4oO9RHo5NV8lXIrqfsY7DCl4JR/yzj6NW8dc2HiImG1af5LTq325HxZjUsh7M7FDS0c27QYUYB11LFBB2oGm", "signature": "U6URG6VM9TOLH67lBcxOLQfOXzHNamqUS4QMoLvmdTh8gTFTzog0/FSsZNHtEYshYb0nUj2z12h5rZJ8cju11A=="}, "version": "3.0.0-beta.0"}'
```
`message_kit`即原数据经过enrico加密的数据。取出`message_kit`后续使用。

#### 3.3.6 构建bob对象
```python
bob = "http://localhost:11151"
Bob_Verifying_Key = "0384a576c6370e37690d594681423528ac1c6c5a772ee8c6d20b47e550f41f6fd6"
Bob_Encrypting_Key =  "02684c17f80d25d3d625397faa12fa537a9be5a6406f42920dd19886827e319139"
```

#### 3.3.7 构建需要检索的数据对象
```python
retrieval = {}
retrieval["label"] = "songs"
retrieval["policy_encrypting_key"] = policy_encrypting_key
retrieval["alice_verifying_key"] = Alice_Verifying_Key
retrieval["message_kit"] = message_kit
```

#### 3.3.7 alice对bob进行授权
下面是授权所需要的基本信息，包括bob的签名密钥、加密密钥、数据标签、过期时间、以及基于阈值门县加密的m和n（本例只运行一个ursula节点），所以都设置成1。
```python
grant = {}
grant["bob_verifying_key"] = Bob_Verifying_Key
grant["bob_encrypting_key"] = Bob_Encrypting_Key
grant["m"] = 1
grant["n"] = 1
grant["label"] = "songs"
grant["expiration"] = 5
```
通过put请求，alice建立授权信息，授权后bob可以拿到加密的数据。
```pyhton
requests.put(f"{alice}/grant",data=json.dumps(grant))
```
> 注意：这里的put只能运行一次，多次运行会返回500错误，提示'Policy already exists in active_policies.'。

#### 3.3.8 bob获取数据
```python
retrieved =  requests.post(f"{bob}/retrieve",data=json.dumps(retrieval))
#retrieved.content
#b'{"result": {"cleartexts": ["VGhpcyBpcyBzb21lIHBsYWludGV4dA=="]}, "version": "3.0.0-beta.0"}'
```
这里的`cleartexts`里即包含了上文的`message_kit`。如果发送所条信息，也将包含在`cleartexts`中。


#### 3.3.9 数据解码
```python
retrieved_b64 = json.loads(retrieved.content)['result']['cleartexts']
#retrieved_b64
#['VGhpcyBpcyBzb21lIHBsYWludGV4dA==']
b64decode(retrieved_b64[0])
#b'This is some plaintext'
```
即顺利拿到alice发送的数据。


## 4. 总结
### 4.1 代理重加密
本文中ursula充当代理重加密节点，具体细节全部由项目内部进行处理。使用过程中只需要将alice，bob等节点建立的ursula节点的服务之上即可。后续的操作过程中，并没有具体涉及到代理重加密的过程。只要通过alice进行授权，bob即可拿到数据。

### 4.2 阈值门限
在nucypher工作过程中，为了充分避免风险，确保ursula节点的可靠，通常会建立多个节点，把重加密密钥保存在多个节点之上，而授权后的bob只需要拿到规定数量的密钥片段即可组合成整个密钥，对秘文数据进行解密。

### 4.3 多ursula节点
目前还没有尝试。

### 4.4 自行搭建节点
难