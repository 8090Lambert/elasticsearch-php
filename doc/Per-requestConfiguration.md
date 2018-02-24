# 请求层配置（Per-request）

除了配置连接层和客户端层，还可以配置请求层。配置请求层是在请求体中指定参数。

## 忽略异常

Elasticsearch-PHP的类库是会对普通的问题抛出异常的。这些异常跟Elasticsearch返回的HTTP响应码一一对应。例如，获取一个不存在的文档会抛出MissingDocument404Exception。

异常对于处理一些问题（如找不到文档、语法错误、版本冲突等）十分有用。但是有时候你只是想要处理返回的数据而不想捕获异常。

如果你想忽略异常，你可以配置ignore参数。ignore参数要作为client的参数配置在请求体中。例如下面的示例会忽略MissingDocument404Exception，返回的是Elasticsearch提供的JSON数据。

	$client = ClientBuilder::create()->build();
	
	$params = [
	    'index'  => 'test_missing',
	    'type'   => 'test',
	    'id'     => 1,
	    'client' => [ 'ignore' => 404 ] 
	];
	echo $client->get($params);
	
	> {"_index":"test_missing","_type":"test","_id":"1","found":false}


你可以通过数组的方式指定忽略多个HTTP状态码：

	$params = [
	    'index'  => 'test_missing',
	    'type'   => 'test',
	    'client' => [ 'ignore' => [400, 404] ] 
	];
	echo $client->get($params);
	
	> No handler found for uri [/test_missing/test/] and method [GET]

注意，返回的数据是字符串格式，而不是JSON数据。而在第一个示例中返回的是JSON数据，客户端会decode该JSON数据为数组。

一旦客户端无法得知返回的异常数据格式，客户端就不会decode返回结果。

## 自定义查询参数

有时候你要自己提供自定义参数，比如为第三方插件或代理提供认证token。在Elasticsearch-php的白名单中存储着所有的查询参数，这是为了防止你指定一个参数，而Elasticsearch却不接收。

如果你要自定义参数，你就要忽略掉这种白名单机制。为了达到这种效果，请增加custom参数：

	$client = ClientBuilder::create()->build();
	
	$params = [
	    'index' => 'test',
	    'type' => 'test',
	    'id' => 1,
	    'parent' => 'abc',              // white-listed Elasticsearch parameter
	    'client' => [
	        'custom' => [
	            'customToken' => 'abc', // user-defined, not white listed, not checked
	            'otherToken' => 123
	        ]
	    ]
	];
	$exists = $client->exists($params);

## 增加返回冗余

客户端默认返回响应体数据。如果你需要更多信息（如头信息、相应状态码等），你可以让客户端返回更多信息。通过verbose参数可以开启这个功能。

没有返回冗余，你看到的返回信息是这样的：

	$client = ClientBuilder::create()->build();
	
	$params = [
	    'index' => 'test',
	    'type' => 'test',
	    'id' => 1
	];
	$response = $client->get($params);
	print_r($response);
	
	
	Array
	(
	    [_index] => test
	    [_type] => test
	    [_id] => 1
	    [_version] => 1
	    [found] => 1
	    [_source] => Array
	        (
	            [field] => value
	        )
	
	)

如果加上返回冗余：

	$client = ClientBuilder::create()->build();
	
	$params = [
	    'index' => 'test',
	    'type' => 'test',
	    'id' => 1,
	    'client' => [
	        'verbose' => true
	    ]
	];
	$response = $client->get($params);
	print_r($response);
	
	
	Array
	(
	    [transfer_stats] => Array
	        (
	            [url] => http://127.0.0.1:9200/test/test/1
	            [content_type] => application/json; charset=UTF-8
	            [http_code] => 200
	            [header_size] => 86
	            [request_size] => 51
	            [filetime] => -1
	            [ssl_verify_result] => 0
	            [redirect_count] => 0
	            [total_time] => 0.00289
	            [namelookup_time] => 9.7E-5
	            [connect_time] => 0.000265
	            [pretransfer_time] => 0.000322
	            [size_upload] => 0
	            [size_download] => 96
	            [speed_download] => 33217
	            [speed_upload] => 0
	            [download_content_length] => 96
	            [upload_content_length] => -1
	            [starttransfer_time] => 0.002796
	            [redirect_time] => 0
	            [redirect_url] =>
	            [primary_ip] => 127.0.0.1
	            [certinfo] => Array
	                (
	                )
	
	            [primary_port] => 9200
	            [local_ip] => 127.0.0.1
	            [local_port] => 62971
	        )
	
	    [curl] => Array
	        (
	            [error] =>
	            [errno] => 0
	        )
	
	    [effective_url] => http://127.0.0.1:9200/test/test/1
	    [headers] => Array
	        (
	            [Content-Type] => Array
	                (
	                    [0] => application/json; charset=UTF-8
	                )
	
	            [Content-Length] => Array
	                (
	                    [0] => 96
	                )
	
	        )
	
	    [status] => 200
	    [reason] => OK
	    [body] => Array
	        (
	            [_index] => test
	            [_type] => test
	            [_id] => 1
	            [_version] => 1
	            [found] => 1
	            [_source] => Array
	                (
	                    [field] => value
	                )
	        )
	)

## Curl超时设置

通过timeout和connect\_timeout参数可以配置每个请求的Curl超时时间。这个配置主要是控制客户端的超时时间。connect\_timeout参数控制在连接阶段完成前，curl的等待时间。而timeout参数则控制整个请求完成前，最多等待多长时间。

如果超过超时时间，curl会关闭连接并返回一个致命错误。两个参数都要用<b>秒</b>作为参数。

注意：客户端超时并不意味着Elasticsearch中止请求。Elasticsearch会继续执行请求直到请求完成。在慢查询或是bulk请求下，操作会在后台继续执行，对客户端来说这些动作是隐蔽的。如果客户端在超时后立即断开连接，然后又立刻发送另外一个请求。由于客户端没有处理服务端回压（这里国内翻译成背压，但是[知乎](https://www.zhihu.com/question/49618581?from=profile_question_card)有文章指出这个翻译不够精准，会造成程序员难以理解，所以这里翻译成回压）的机制，这有可能会造成服务端过载。遇到这种情况，你会发现线程池队列会慢慢变大，当队列超出负荷，Elasticsearch会发送EsRejectedExecutionException的异常。

	$client = ClientBuilder::create()->build();
	
	$params = [
	    'index' => 'test',
	    'type' => 'test',
	    'id' => 1,
	    'client' => [
	        'timeout' => 10,        // ten second timeout
	        'connect_timeout' => 10
	    ]
	];
	$response = $client->get($params);

## 开启Future模式

客户端支持异步方式批量发送请求。通过client选项的future参数可以开启（HTTP handler要支持异步模式）：

	$client = ClientBuilder::create()->build();
	
	$params = [
	    'index' => 'test',
	    'type' => 'test',
	    'id' => 1,
	    'client' => [
	        'future' => 'lazy'
	    ]
	];
	$future = $client->get($params);
	$results = $future->wait();       // resolve the future

Future模式有两个参数可选：true 或 lazy。关于异步执行方法以及如何处理返回结果的详情，请到[Future模式](https://www.elastic.co/guide/en/elasticsearch/client/php-api/6.0/_future_mode.html)中查看。

## SSL加密

在创建客户端时，一般需要指定SSL配置，因为通常所有的请求都需要加密（查询[安全](https://www.elastic.co/guide/en/elasticsearch/client/php-api/6.0/_security.html)一章获取更多详情）。然而，在每个请求中配置SSL加密也是有可能的。例如，如果你需要在某个特定的请求中使用自签名证书，你可以通过在client选项中配置verify参数：

	$client = ClientBuilder::create()->build();

	$params = [
	    'index' => 'test',
	    'type' => 'test',
	    'id' => 1,
	    'client' => [
	        'verify' => 'path/to/cacert.pem'      //Use a self-signed certificate
	    ]
	];
	$result = $client->get($params);