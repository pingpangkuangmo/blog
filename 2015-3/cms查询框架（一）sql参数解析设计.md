#1	前言

本篇文章主要来说明下代码模块的设计。像我们这种菜鸟级别，只有平时多读读源码，多研究和探讨其中的设计才可能提升自己，写出高质量的代码。

没有最好的设计，只有更好的设计，所以在发表我自己的愚见的同时，希望小伙伴们相互探讨更好的设计，有探讨才有更大的进步。

#2	题目及分析

我们维护了一个数据中心，对外提供查询API，如何能让用户随意的添加查询条件，而不用修改后台的查询代码呢？用户如何配置查询条件，从而达到如下的sql效果呢？：

	a.name='lg' or b.age>12
	b.id in (12,34,45)
	c.updateTime>'2015-3-28' and (b.id=2 or d.age<23)
	e.age>f.age

##2.1	查询参数的传递方式

我们作为API设计者，该如何让用户方便的传递他们任意的查询需求呢？这是我们要思考的地方。

目前来看比较好的方式莫过于：用户通过json来表达他们的查询需求。

##2.2	查询的本质分析

从上面的查询来看，我们可以总结出来查询条件无非就是某个字段满足什么样的条件。这里有三个对象：

-	查询的字段 如 b.age
-	条件值	如 12
-	怎样满足条件值	如 >

##2.3	查询的配置分析

这样我们就可以清晰明了了，一个查询条件无非就是三个内容，所以可以如下配置：

	{
		"columns":"b.age",
		"oper":">",
		"value":12
	}

很显然，上面的确很麻烦，我们无非是要表达这三个内容，所以就要简化配置：

	{
		"b.age":12,
		"oper":">"
	}

还是不够简化，如何把操作符 > 也放置进去呢？如下

	{
		"b.age@>":12
	}

这样我们就可以把三个对象表达清楚了，将查询的字段和操作符合并起来作为key,并使用分隔符@分割两者，条件值作为value。这样就做到了，非常简化的查询配置。

接下来又面临一个问题，如何表达查询条件之间的and or 关系呢？即如何表达下面的内容呢？

	c.age>14 and (b.id=2 or d.age<23)

借鉴mongodb的查询方案，可以如下配置：

	{
		"c.age@>":14,
		"$or":{
			"b.id@=":2,
			"d.age@<":23
		}
	}

通过配置一个$or作为key表明里面的几个查询条件是or的关系，如果是$and表明里面的查询条件之间是and的关系，外层默认是and的关系。

同时我们再回顾下，mongodb所作出的查询设计，也是通过用户配置json形式来表达查询意图，但是我们来看下它是如何查询

	a.age>12 对应的mongodb的查询方式为：
	{
		a.age : {
			$gt : 22
		}
	}

	我们的查询方式是

	{
		"a.age@>":22
	}

虽然看似我们的更加简单，mongodb的更加繁琐，主要是mongodb认为对于一个字段，可以有多个查询条件的，为了支持更加复杂的查询，如下：

	{	
		a.age : {
			$lt :24,
			$gt : 17
		}
	}

	然而我们也可以对此进行拆分，如下，同样满足：
	
	{
		"a.age@>":17，
		"a.age@<":24，
	}

各有各的好处和缺点，我就是我，颜色不一样的烟火。哈哈。。。
	
#3	代码设计与实现

##3.1	解析器接口的设计

对题目进行分析完了之后，就要考虑如何实现这样的json配置到sql的转化。实现起来不难，最重要的是如何做出一个高扩展性的实现？

再来看下下面的例子：

	{
		"a.name@=":"lg",
		"b.age@>":12,
		"c.id@in":[12,13,14],
		"d.time@time>":"2015-3-1"
	}

其实就是针对每个自定义的操作符进行相应的处理，所以就有了解析器接口：

	public interface SqlParamsParser {

		//这里表示该解析器是否支持对应的操作符
		public boolean support(String oper);
	
		public String getParams(String key,Object value,String oper);
		
		public SqlParamsParseItemResult getParamsResult(String key,Object value,String oper);
	
	}

其中SqlParamsParseItemResult，则是把解析后的结果分别存储起来，而不是直接拼接成一个字符串，主要为了直接拼接字符串式的sql注入，它的内容如下：

	public class SqlParamsParseItemResult {
		private String key;
		private String oper;
		private Object value;
	}

上面的key oper value 则是解析后的内容。下面举例说明

以"b.age@>":12 为例，其中getParams方法中的 key就是b.age, value就是12, oper就是> 而这个方法的返回的字符串结果为：

	b.age>12

	返回的SqlParamsParseItemResult存储的内容为分别为 key=b.age ; oper=> ; value=12

以"c.id@in":[12,13,14]为例，其中getParams方法中的 key就是c.id,value就是一个List集合，oper就是in ，这个方法的返回结果为：

	c.id in (12,13,14)

	返回的SqlParamsParseItemResult存储的内容为分别为 key=c.id ; oper=> ; value=12

以"d.time@time>":"2015-3-1"为例，其中getParams方法中的 key就是c.id,value就是一个List集合，oper就是in，这个方法的返回结果为：

	unix_timestamp（d.time） > 1425139200     (2015-3-1对应的秒数)

	返回的SqlParamsParseItemResult存储的内容为分别为 key=unix_timestamp（d.time） ; oper=> ; value=1425139200

##3.2	解析器接口的抽象类及其实现

解析器有很多相同的地方，这就需要我们进行抽象，抽出共性部分，留给子类去实现不同的部分。所以有了抽象类AbstractSqlParamsParser

有哪些共性部分呢？

-	第一点： 就是support方法。每个解析器支持某几种操作符，所以判断该解析器是否支持当前的操作符的逻辑是共同的，所以如下:

		public abstract class AbstractSqlParamsParser implements SqlParamsParser{

			private String[] opers;	
			private boolean ignoreCase=true;
	
			protected void setOpers(String[] opers){
				this.opers=opers;
			}	
			protected void setIgnoreCase(boolean ignoreCase){
				this.ignoreCase=ignoreCase;
			}
			@Override
			public boolean support(String oper) {
				if(opers!=null && oper!=null){
					for(String operItem:opers){
						if(ignoreCase){
							operItem=operItem.toLowerCase();
							oper=oper.toLowerCase();
						}
						if(operItem.equals(oper)){
							return true;
						}
					}
				}
				return false;
			}
		}

	opers属性表示当前解析器所支持的所有操作符。ignoreCase表示在匹配操作符的时候是否忽略大小写。这两个属性都设置成private,然后对子类开放了protected类型的set方法，用于子类来设置这两个属性。

-	