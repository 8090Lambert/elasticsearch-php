# 搜索操作

呃......这个项目如果没有什么特别之处就不叫elasticsearch了（译者注：elasticsearch直译过来就是“灵活搜索”）！现在一起来聊聊客户端的搜索操作。

在命名方案规范的前提下，客户端拥有一切的查询权限，也拥有获取REST API公开的一切参数的权限。现在来看看一些示例，方便你熟悉这些语法规则。

## Match查询

以下是Match查询的标准curl格式：

	curl -XGET 'localhost:9200/my_index/my_type/_search' -d '{
	    "query" : {
	        "match" : {
	            "testField" : "abc"
	        }
	    }
	}'

而这里则是客户端构建的同样的查询：

	$params = [
	    'index' => 'my_index',
	    'type' => 'my_type',
	    'body' => [
	        'query' => [
	            'match' => [
	                'testField' => 'abc'
	            ]
	        ]
	    ]
	];
	
	$results = $client->search($params);

这里要注意PHP数组的结构与层次是怎样与curl中的JSON请求体格式相应对的。这种方式使得JSON的写法转换为PHP的写法变得十分简单。一个快速检测PHP数组是否为预期结果的方法，就是encode为JSON格式，然后进行检查：

	$params = [
	    'index' => 'my_index',
	    'type' => 'my_type',
	    'body' => [
	        'query' => [
	            'match' => [
	                'testField' => 'abc'
	            ]
	        ]
	    ]
	];
	
	print_r(json_encode($params['body']));
	
	
	{"query":{"match":{"testField":"abc"}}}

### 使用原生JSON
	
	有时使用原生JSON来进行测试会十分方便，或者用原生JSON来进行不同系统的移植也同样方便。你可以在body中用原生JSON字符串，这样客户端会进行自动检查操作：

	$json = '{
	    "query" : {
	        "match" : {
	            "testField" : "abc"
	        }
	    }
	}';
	
	$params = [
	    'index' => 'my_index',
	    'type' => 'my_type',
	    'body' => $json
	];
	
	$results = $client->search($params);

搜索结果与Elasticsearch的响应结果一致，唯一不同的是JSON格式会转换成PHP数组。处理这些数据与数组迭代一样简单：

	$params = [
	    'index' => 'my_index',
	    'type' => 'my_type',
	    'body' => [
	        'query' => [
	            'match' => [
	                'testField' => 'abc'
	            ]
	        ]
	    ]
	];
	
	$results = $client->search($params);
	
	$milliseconds = $results['took'];
	$maxScore     = $results['hits']['max_score'];
	
	$score = $results['hits']['hits'][0]['_score'];
	$doc   = $results['hits']['hits'][0]['_source'];

## Bool查询

利用客户端可以轻松构建Bool查询。例如以下查询：

	curl -XGET 'localhost:9200/my_index/my_type/_search' -d '{
	    "query" : {
	        "bool" : {
	            "must": [
	                {
	                    "match" : { "testField" : "abc" }
	                },
	                {
	                    "match" : { "testField2" : "xyz" }
	                }
	            ]
	        }
	    }
	}'

会构建为这样子（注意方括号位置）：

	$params = [
	    'index' => 'my_index',
	    'type' => 'my_type',
	    'body' => [
	        'query' => [
	            'bool' => [
	                'must' => [
	                    [ 'match' => [ 'testField' => 'abc' ] ],
	                    [ 'match' => [ 'testField2' => 'xyz' ] ],
	                ]
	            ]
	        ]
	    ]
	];
	
	$results = $client->search($params);

这里注意must语句接收的是数组。这里会转化为JSON数组，所以最后的响应结果与curl格式的响应结果一致。想了解PHP中数组和对象的转换，请查看[用PHP处理JSON数组和JSON对象](https://github.com/Mosongxing/Elasticsearch-PHP6.0/blob/master/doc/Json.md)。

## 更为复杂的示例

这里构建一个有点复杂的例子：一个bool查询包含一个filter过滤器和一个普通查询。这在elasticsearch的查询中非常普遍，所以这个例子会非常有用。

curl格式的查询：

	curl -XGET 'localhost:9200/my_index/my_type/_search' -d '{
	    "query" : {
	        "bool" : {
	            "filter" : {
	                "term" : { "my_field" : "abc" }
	            },
	            "should" : {
	                "match" : { "my_other_field" : "xyz" }
	            }
	        }
	    }
	}'

而在PHP中：

	$params = [
	    'index' => 'my_index',
	    'type' => 'my_type',
	    'body' => [
	        'query' => [
	            'bool' => [
	                'filter' => [
	                    'term' => [ 'my_field' => 'abc' ]
	                ],
	                'should' => [
	                    'match' => [ 'my_other_field' => 'xyz' ]
	                ]
	            ]
	        ]
	    ]
	];
	
	
	$results = $client->search($params);

## Scrolling（游标）查询

在用bulk时，经常要用Scrolling功能对文档进行分页处理，如输出一个用户的所有文档。这比常规的搜索要高效，因为这里不需要对文档执行性能消耗较大的排序操作。

Scrolling会保留某个时间点的索引快照数据，然后用快照数据进行分页。游标查询窗口允许持续分页操作，即使后台正在执行索引文档、更新文档和删除文档。首先，你要在发送搜索请求时增加scroll参数。然后就会返回一个文档“页数”信息，还有一个用来获取hits分页数据的scroll_id。

更多详情请查看[游标查询](https://www.elastic.co/guide/cn/elasticsearch/guide/current/scroll.html)。

以下代码更为深入的操作的示例：

	$client = ClientBuilder::create()->build();
	$params = [
	    "scroll" => "30s",          // how long between scroll requests. should be small!
	    "size" => 50,               // how many results *per shard* you want back
	    "index" => "my_index",
	    "body" => [
	        "query" => [
	            "match_all" => new \stdClass()
	        ]
	    ]
	];
	
	// Execute the search
	// The response will contain the first batch of documents
	// and a scroll_id
	$response = $client->search($params);
	
	// Now we loop until the scroll "cursors" are exhausted
	while (isset($response['hits']['hits']) && count($response['hits']['hits']) > 0) {
	
	    // **
	    // Do your work here, on the $response['hits']['hits'] array
	    // **
	
	    // When done, get the new scroll_id
	    // You must always refresh your _scroll_id!  It can change sometimes
	    $scroll_id = $response['_scroll_id'];
	
	    // Execute a Scroll request and repeat
	    $response = $client->scroll([
	            "scroll_id" => $scroll_id,  //...using our previously obtained _scroll_id
	            "scroll" => "30s"           // and the same timeout window
	        ]
	    );
	}