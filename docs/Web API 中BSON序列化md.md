##Web API 中BSON序列化
前面已经讲述最常用的两种格式：Json和XmL。在本节，将介绍另外一种格式：BSON.Web API2.1开始支持BSON.
###BSON是什么
BSON是一种二进制序列化格式。"BSON"代表"二进制 JSON"，但 BSON 和 JSON 序列化了有很大不同。BSON和JSON非常相似，都是采用名称/值对来表示。但与JSON，不同时，数值数据类型存储为字节，不是字符串。
BSON是一个轻量级，易于扫描，快速编码和解码。具有如下特点：

- BSON在文件大小上和Json差不多.主要依赖于数据，BSON 有可能小于或者大于JSON. 当序列化二进制数据时，例如图片文件，比JSON更小，因为，图片文件被序列化成二进制，而不是就行base64的编码格式。
- BSON文档更易于检索。解析器，更易于扫描二进制数据，相对应String类型。String需要解码。
- 编码和解码更有效。数据存储为字节类型，而不是String.

因此，对于本地客户端，像.net App，采用BSON作为传输媒介比JSON,XML 获得更好的性能。
而对于浏览器来说，采用JSON,更方便与Javascript进行交互。

----------

幸运的是，Web API 支持两种格式，并且由客户端来选择数据交互格式。详情请参见内容协定一节，本节不做详细描述。
###服务端如何启用BSON
上面已经介绍了BSON,如何通过配置使服务端使其支持BSON,那么可以参照如下代码：
    
    public static class WebApiConfig
    {
    public static void Register(HttpConfiguration config)
    {
    config.Formatters.Add(new BsonMediaTypeFormatter());
    
    // Other Web API configuration not shown...
    }
    }

现在，如果在客户端请求的标头里定义的格式为 **application/bson**，Web API 件采用BSON格式与客户端交互。
此外，您也可以定义自己BSON 支持代码，参照代码如下：

	var bson = new BsonMediaTypeFormatter();
	bson.SupportedMediaTypes.Add(new MediaTypeHeaderValue("application/vnd.contoso"));
	config.Formatters.Add(bson);

这样定义以后，如果请求的是 application/vnd.contoso，Web API 将自动采取BSON来做数据交互媒介。
###客户端如何使用BSON
上面讲述 服务配置BSON，那么客户端如何通过配置BSON类实现服务的调用。
######从服务请求数据
Client向服务端请求数据，然后通过BSON,反序列化成实体对象。下面通过一段代码来讲述：

	static async Task RunAsync()
	{
    using (HttpClient client = new HttpClient())
    {
        client.BaseAddress = new Uri("http://localhost");

        // 设置请求Accept头 为BSON.
        client.DefaultRequestHeaders.Accept.Clear();
        client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/bson"));

        // 发送请求
        result = await client.GetAsync("api/books/1");
        result.EnsureSuccessStatusCode();

        // 用BSON 格式来反序列化返回的结果。
        MediaTypeFormatter[] formatters = new MediaTypeFormatter[] {
            new BsonMediaTypeFormatter()
        };

        var book = await result.Content.ReadAsAsync<Book>(formatters);
    }
	}

其中，设置请求头代码是：

	client.DefaultRequestHeaders.Accept.Clear();
	client.DefaultRequestHeaders.Accept.Add(new  
    MediaTypeWithQualityHeaderValue("application/bson"));
这样，服务端会自动将数据序列化成BSON格式进行响应。


默认情况下，是没有配置支持BsonMediaTypeFormatter， 所以需要在代码里自行配置一下。代码如下：

	MediaTypeFormatter[] formatters = new MediaTypeFormatter[] {
	    new BsonMediaTypeFormatter()
	};
	
	var book = await result.Content.ReadAsAsync<Book>(formatters);

######往服务发送数据
Client向服务端发送BSON数据,先是将对象序列化成BSON格式。下面通过一段代码来讲述：

	static async Task RunAsync()
	{
    using (HttpClient client = new HttpClient())
    {
        client.BaseAddress = new Uri("http://localhost:15192");

        // 设置请求Accept头 为BSON.
        client.DefaultRequestHeaders.Accept.Clear();
        client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/bson"));

        var book = new Book()
        {
            Author = "Jane Austen",
            Title = "Emma",
            Price = 9.95M,
            PublicationDate = new DateTime(1815, 1, 1)
        };

        // 定义为BSON 格式
        MediaTypeFormatter bsonFormatter = new BsonMediaTypeFormatter();

		//像服务端发送Post 请求 
        var result = await client.PostAsync("api/books", book, bsonFormatter);
        result.EnsureSuccessStatusCode();
    }
	}

以上两个Http请求基本一样，都是需要配置请求的格式，代码如下：

	MediaTypeFormatter bsonFormatter = new BsonMediaTypeFormatter();
	var result = await client.PostAsync("api/books", book, bsonFormatter);

####使用BSON注意事项
正如我们所知，BSON采用键值对方式存储数据，所以，对于一个参数的化，通常情况下，您需要在序列化之前，把所要序列化的对象包装成键值对方式。下面这个例子：
	
	public class ValuesController : ApiController
	{
	    public IHttpActionResult Get()
	    {
	        return Ok(42);
	    }
	}
在序列化之前，BSON 格式化器，会将转成成如下：

	{ "Value": 42 }
然后才会做进一步的序列化。所以当你在客户端反序列化的时候，你要处理这样的问题，有可能会给你带来异常。通常情况下建议在序列化前，包装成键值对的结构化数据。

###BSON格式，Client端和Server端数据的格式
接下来将通过一个例子说明：

	public class Book
	{
	    public int Id { get; set; }
	    public string Title { get; set; }
	    public string Author { get; set; }
	    public decimal Price { get; set; }
	    public DateTime PublicationDate { get; set; }
	}
	
	public class BooksController : ApiController
	{
	    public IHttpActionResult GetBook(int id)
	    {
	        var book = new Book()
	        {
	            Id = id,
	            Author = "Charles Dickens",
	            Title = "Great Expectations",
	            Price = 9.95M,
	            PublicationDate = new DateTime(2014, 1, 20)
	        };
	
	        return Ok(book);
	    }
	}
下图是客户端的请求头情况：





下图是客户端收到的响应数据