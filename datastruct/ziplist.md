## ziplist的实现

` ziplist的使用很普遍，在实际的场景中用的地方都有：quicklist实现和sortset的实现中，在quicklist的实现中每一个quicknode都是一个ziplist，在sortset中，当数据量比较小的时候，使用功能的就是ziplist，当数据量上升触及上限配置的时候会转为skiplist+dict实现，具体实现参考相应章节`

	首先ziplist没有单独的数据结构，整体ziplist的表示方式如下：
	
	<zlbytes><zltail><zllen><entry><entry><zlend>
	
	zlbytes : 表示ziplist的整体大小（用于判断是否需要resize） 
	zltail  : ziplist最后一个entry的偏移量
	zllen   : entry的个数
	entry   : 具体的单个元素
	zlend   : 单独的一个字节表示ziplist结束标记
	
	以上结构的优势可以看见，对一个ziplist执行pop或者push效率都是O(1)的复杂度，但是插入或者是删除中间数据会有些稍微复杂的逻辑。
	
	-------
	
	如此整体的ziplist的结构就清晰了，现在分析entry的结构
	
	<prevlen><selflen><data>
	
	prevlen : 前一个元素的长度
	selflen : 本元素的长度
	data    : 具体数据
	
	prelen的情况:
     1、第一字节 < 254，直接一字节表示长度 
     	最大为：11111100 => 253
     
     2、第一字节为：254，则相邻4bytes表示长度
     
     ------
     
	selflen的情况:
	我们观察第一字节的情况
	
     1、00xxxxxx  1byte 因为00做了标记，还剩下 2^6   => 63,  所以表示长度<=63的字符串
     
     2、01xxxxxx  2bytes 因为01做了标记，还剩下 2^14 =>16383，所以表示长度<=16383的字符串
     
     3、10xxxxxx  5bytes 因为10做了标记，还剩下38位，但是使用的时候只用了相邻的四字节，所以为 2^32，表示长度> 16383的字符串
     
     4、11000000  int16   2bytes
     5、11010000  int32   4bytes
     6、11100000  int64   8bytes
     7、11110000  24bits  3bytes
     8、11111110  8bits   1bytes
     9、1111xxxx   4bits   1 - 13 (0001 - 1101)
     10、11111111 表示结尾 255
     
