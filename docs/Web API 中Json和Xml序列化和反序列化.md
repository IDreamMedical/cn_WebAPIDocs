##第二节：Web API 中Json和Xml序列化和反序列化
前一节，我们讲述了自定义媒体类型格式。那么到底CLR对象通过什么方式转成Http消息呢？同时Http消息转成CLR对象呢？那么本节的主题就是讲述Web API 中Json和Xml序列化和反序列化;

在 ASP.NET Web API，媒体类型格式化程序是一个具有如下特点的对象。该对象必须具有如下：



-  从Http消息体转成CRL对象。
-  CRL对象转成Http消息。

ASP.NET Web API 默认提供两种支持格式：Json和Xml, Web API 框架初始化时提供两种管道，
客户端可以创建其他任何一种格式的Http请求。具体方法就是添加Accept标头设置请求格式；下面具体介绍两种格式。

##Json格式
**JsonMediaTypeFormatter**提供**Json**格式的序列化工作，**JsonMediaTypeFormatter**默认采用第3方开源库（Json.NET ）。如果你足够自信的化， 你可以通过**DataContractJsonSerializer**来取代 Json.NET的序列化工作。具体配置方法如下：

	var json = GlobalConfiguration.Configuration.Formatters.JsonFormatter;
	json.UseDataContractJsonSerializer = true;

####那些被序列化
默认情况下，采用**Json.NET**来序列化， 除了被标志为**JsonIgnore**的属性或字段，其他所有的Public字段都会被序列化。例如：

	public class Product
	{
	    public string Name { get; set; }

	    public decimal Price { get; set; }

	    [JsonIgnore]
	    public int ProductCode { get; set; } // 忽略掉，不序列化；
	}

而采用 **DataContractJsonSerializer**就有很大不同了，通常只有标志为**DataContract **，并且成员被**DataMember**标志后才会被序列化，注意，这里不区别属性是否是**Public**和**Private**
####只读属性序列化
通常情况下，只读属性也会序列化；

####日期格式序列化


####索引序列化


####大小写序列化

####匿名类型和弱类型序列化





##Xml格式
