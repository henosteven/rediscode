## zipmap的实现

`现在的redis代码中使用的比较少了，具体原因需要要分析分析`

> zipmap主要使用在内存高效使用的hashtable上，举个例子
> "foo" => "bar", "hello" => "world" 这样的数据使用zipmap表示为

	<zmlen><len>"foo"<len><free>"bar"<len>"hello"<len><free>"world"
	\x02\x03foo\x03\x00bar\x05hello\x05\x00world\xff


这样看的时候就很清晰了

* zmlen : 表示zipmap的数据个数，使用一字节，当zipmap的长度 >= 254的时候，需要遍历整个zipmap才能获取长度

* len   : 表示邻近字符串的长度，如果第一字节小于254则直接表示字符串的长度，如果等于254，则使用紧邻的后4个字节表示长度，一共需要五个字节

* free  : 表示value多余的空间，例如，之前foo => bar, 但是现在foo => b , 那么就会有2个字节的多余空间。free只占用一字节，redis的策略是，当free太大的时候，需要重新resize，让内存占用尽可能小
