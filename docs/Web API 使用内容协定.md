##Web API 使用内容协定

HTTP 规范 (RFC 2616) 定义了内容协商作为"HTTP处理流程中，当有多种可选择的响应形式，自动选择最佳的响应返回给调用端"。决定HTTP内容协商的主要机制是这些请求标头 ︰

1. **Accept**：可接受的媒体格式，例如：“application/json,” “application/xml,” 	"application/vnd.example+xml"。
2. **Accept-Charset**: 可接受的字符编码格式，例如：UTF-8 or ISO 8859-1。
3. **Accept-Encoding**: 定义编码格式，  例如：gzip.
4. **Accept-Language**：采取的语言格式，例如：“en-us”.

当然，服务端也可能根据请求头的其他部分，如果请求头里包括 **X-Requested-With**，服务端会将将请求视为AJAX请求，因此默认为Json格式，即使没有**Accept**标头。

####序列化
考虑下面一段代码：

	public Product GetProduct(int id)
	{
	    var item = _products.FirstOrDefault(p => p.ID == id);
	    if (item == null)
	    {
	        throw new HttpResponseException(HttpStatusCode.NotFound);
	    }
	    return item; 
	}

请求头如下：

	GET http://localhost.:21069/api/products/1 HTTP/1.1
	Host: localhost.:21069
	Accept: application/json, text/javascript, */*; q=0.01 //请求头可以是JSON, Javascript, or 任意类型(*/*)

响应情况可能如下:

	HTTP/1.1 200 OK
	Content-Type: application/json; charset=utf-8 // 返回的是Json 格式。
	Content-Length: 57
	Connection: Close
	
	{"Id":1,"Name":"Gizmo","Category":"Widgets","Price":1.99}

上面一段代码将资源看作做CLR类型， 然后序列化到Http消息体中。

----------

控制器也可以返回 **HttpResponseMessage**，若要指定一个用于响应正文的 CLR 对象，请调用 **CreateResponse** 扩展方法 ︰

	
	public HttpResponseMessage GetProduct(int id)
	{
	    var item = _products.FirstOrDefault(p => p.ID == id);
	    if (item == null)
	    {
	        throw new HttpResponseException(HttpStatusCode.NotFound);
	    }
	    return Request.CreateResponse(HttpStatusCode.OK, product);
	}

通过**HttpResponseMessage**， 你可以更多的控制返回的消息（包括状态码，标头等等）。
序列化的资源的对象被称为媒体格式化程序。媒体格式化程序从 **MediaTypeFormatter** 类派生。Web API 为 XML 和 JSON，提供媒体格式化程序，您可以创建自定义格式化程序来支持其他媒体类型。有关编写自定义格式化程序的信息，请参阅媒体格式化程序一节。

####内容协定工作流程
首先，该管道从 **HttpConfiguration** 对象中获取 **IContentNegotiator** 服务。它还从 **HttpConfiguration.Formatters** 集合中获取媒体格式化程序的列表。

接下来，管道调用 **IContentNegotiatior.Negotiate**，依次通许如下步骤︰

1. 要序列化的对象类型
2. 媒体格式化程序的集合
3. HTTP 请求

**Negotiate**方法返回两部分信息 ︰

- 要使用的格式化程序
- 媒体类型的响应


如果发现没有格式化程序，协商方法将返回 null，并且客户端收到 HTTP 406 错误 （不接受媒体格式类型）。
下面的一段代码将自动完成以上工作：

	public HttpResponseMessage GetProduct(int id)
	{
    var product = new Product() 
        { Id = id, Name = "Gizmo", Category = "Widgets", Price = 1.99M };

    IContentNegotiator negotiator = this.Configuration.Services.GetContentNegotiator();

    ContentNegotiationResult result = negotiator.Negotiate(
        typeof(Product), this.Request, this.Configuration.Formatters);
    if (result == null)
    {
        var response = new HttpResponseMessage(HttpStatusCode.NotAcceptable);
        throw new HttpResponseException(response));
    }

    return new HttpResponseMessage()
    {
        Content = new ObjectContent<Product>(
            product,		        // 序列化的实体 
            result.Formatter,           // 格式化器列表
            result.MediaType.MediaType  //  MIME类型
        )
    };
	}
####默认的内容协定
**DefaultContentNegotiator** 类提供 **IContentNegotiator** 的默认实现。它使用几个标准来选择格式化程序。

首先，格式化程序必须能够序列化类型。通过调用 **MediaTypeFormatter.CanWriteType** 来验证是否是可序列化的类型了。

接下来，内容协定器查找每个格式化程序和评估是否匹配HTTP 请求。为了匹配HTTP请求，内容协定器需要在格式化程序做两件事︰

1. **SupportedMediaTypes**集合，它包含受支持的媒体类型的列表。内容协定器会尝试匹配此该请求**Accept**标头的列表。注意，**Accept**标头在请求的范围内。例如，"文本"是匹配 **text/*** 和** */***。
2. **MediaTypeMappings**集合，它包含一个**MediaTypeMapping**对象列表。**MediaTypeMapping**类提供通用的方法来匹配和HTTP请求媒体类型。例如，它可以映射到一个自定义的具有特定媒体类型的HTTP标头。
如果找到多个匹配，将返回成功。例如:

	`Accept: application/json, application/xml; q=0.9, */*; q=0.1`


在此示例中，**application/json** 已经找到，所以它在不再查找 **application/xml**。

如果没有找到匹配的，内容协定器尝试匹配请求体的任何媒体类型。例如，如果请求包含 JSON 数据，内容协定器查找 JSON 格式化程序。如果仍没有匹配项，内容协定器只是拿第一的可序列化类型格式化程序来序列化。
格式化程序选定后，内容协定器选择最佳的字符编码。在格式化程序查找**SupportedEncodings**属性，并将它与请求中接受字符集头匹配（如果有）。


