tus-translate
=============

tus.io上的传输协议的翻译：

原文地址在[这里](http://tus.io/protocols/resumable-upload.html#file-creation)

下面只对其中的核心思想做一下自我的解读，如果有不明白的，去参看原文版本：

### 协议核心

协议核心描述了怎么样去回复中断的上传。

所有的客户端与服务器必须（MUST）执行这个协议核心。

#### 举例说明

一个HEAD请求用来确定上传应该在何处（OFFSET）继续。

下面的例子展示了：100byte的数据在传输了70字节后中断的恢复过程。

请求：

	HEAD /files/24e533e02ec3bc40c387f1a0e460e216 HTTP/1.1
	Host: tus.example.org
	Response:

响应（Response）：
	
	HTTP/1.1 200 Ok
	Offset: 70
	回传Offset值，客户端通过这个值用PATCH方法恢复上传。

请求：

	PATCH /files/24e533e02ec3bc40c387f1a0e460e216 HTTP/1.1
	Host: tus.example.org
	Content-Type: application/offset+octet-stream
	Content-Length: 30
	Offset: 70

	［上传剩下的30bytes］

响应：

	HTTP/1.1 200 Ok
	

#### 5.2报头（Headers）

##### 5.2.1 Offset--已上传量

	Offset参数在一个请求和响应的头部被定义，用以设定一个资源的已上传量，这个值必须是一个等于0或大于0的整数。

#### 5.3. 请求

##### 5.3.1. HEAD

服务器必须为每个针对特定资源的HEAD请求返回一个Offset值，即使是0,或者上传已经完成。

##### 5.3.2. PATCH

服务器必须接受PATCH请求，而且把请求信息中的字节数值加到给定的Offset上。所有的PATCH请求的Content-Type设定为：application/offset+octet-stream.

Offset值应该等于，但是也可能小于当前资源的offset,服务器必须在相同数据的相同的offset值处处理PATCH.Offset值大于当前上传的offset值的情况未定义。
请参看：see the Parallel Chunks extension.

客户端应该在PATCH请求里发送剩下的所有数据，但是也可以在需要的场景下使用多个小请求。（比如：NGINX buffering requests before they reach their backend）。

Servers MUST acknowledge successful PATCH operations using a 200 Ok status, which implicitly means that clients can assume that the new Offset = Offset + Content-Length.
服务器必须使用200 ok来应答一次成功的PATCH操作，以让客户端能够确定新的OffSet= Offset + Content-Length.

客户端和服务器都应该尝试网络可预见的错误检测以及处理。他们可以通过检查读写socket错误（ read/write socket errors），以及设定读写超时（read/write timeouts），来执行这些任务。客户端和服务器都应该使用一个30秒的超时。. A timeout SHOULD be handled by closing the underlaying connection.


服务器应该总是去处理信息的正文，以存储接受到的数据流。



### 6.协议扩展 

客户端和服务器鼓励去尽量实现下面定义的扩展需求。

#### 6.1. File Creation文件创造

所有的客户端和服务器应该执行文件创造的API。如果没有这样的API，那么必须自己定义一个。

##### 6.1.1. 举例

一个空的POST请求用来创造一个新的上传资源。The Entity-Length header 设定为上传文件的大小值。

请求：
	
	POST /files HTTP/1.1
	Host: tus.example.org
	Content-Length: 0
	Entity-Length: 100

响应:

	HTTP/1.1 201 Created
	Location: http://tus.example.org/files/24e533e02ec3bc40c387f1a0e460e216

新的资源有一个隐含的Offset值为0，允许客户端使用核心协议执行的实际上传。

##### 6.1.2. 报头

###### 6.1.2.1. Entity-Length（实体长度）


实体长度值是一个要上传的文件（或资源。或实体）字节长度。这样服务器就能知道一个上传具体在何时能够完成了。值必须是一个非负整数。

##### 6.1.3. 请求

###### 6.1.3.1. POST

客户端必须使用一个POST动作来初始化一个新的文件资源的上传。请求必须包含一个Entity-Length参数。

服务器必须使用201来应答一个成功的文件创造的请求，而且还必须在Location header里面为创建的资源设定绝对的url。

客户端继续使用核心协议执行实际的上传过程。

6.2. Checksums校验

这个扩展用来定义怎么为一个上传的文件的每个包或子文件提供校验。

6.3. Parallel Chunks并行块

这个扩展用来定义怎么并行的上传一个文件的多个块，以解决一个单独的tcp连接数的限制问题。

6.4. Metadata文件元数据
这个扩展用来定义在上传文件的过程怎么提供文件元信息。

6.5. Streams流


这个扩展用来定义怎么上传不知道数据大小的上传场景。

