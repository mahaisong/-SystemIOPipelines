# -SystemIOPipelines
讲一下pipeline，以及7年前我自己写的pipeline


-----------------------之前我在编写网络程序的时候，为解决粘包、分包、拆包、合并解析等问题，会搞成：

A申请1个缓冲区--长链表【字节】形式--实时收到的、还未处理的--可能是多个包
A的游标
B申请1个缓冲区--栈【字节】形式--解析过程中，发现不够长的，暂时等待。
C申请1个缓冲区--队列【字节数组】形式--解析不出的、异常丢弃的。
D申请1个队列--已经解析【命令字符串】出来的。

然后将每个包都塞入A，形成链条。
主程序判断B中是否有残余，有就把B拼接放在A的头部， 然后一起逐个字节来解析。
按协议和字长切一把、优化解析算法，若且切下来的字节数组符合协议，就放入D。
若总长度不符合，就不切，暂时等待下次A增加时解析【再次收到新数据-下一个包】
若总长度超出协议最长，扔解析不出来，则切掉A当前第1个字节，放入C。 然后从A第2个字节开始继续向下解析。 

不断的重复、游标不停的向后移动，不断的解析。解析一截符合要求，就从抽出来。

大体是这样。 7.8年了。我也记不太清楚了。


-----------------我所解决的问题， 现在微软的类库System.IO.Pipelines提供了工具的方法。 以后上面的逻辑不再需要字节写了。
使用Pipe的方式--管道的缓冲区：
var result = await reader.ReadAsync();
var buffer = result.Buffer; 
这个Buffer是一个ReadOnlySequence<byte>对象，它是一个相当好的动态内存对象，并且相当高效。它本身由多段Memory<byte>组成，它从逻辑上也可以看成一段连续的Memory<byte>

另外，它还有一个类似游标的位置对象SequencePosition

这个缓冲区解决了"数据读不够"的问题，一次读取的不够下次可以接着读，不用缓冲区的动态分配，高效的内存管理方式带来了良好的性能，
该Pipe实现存储了在PipeWriter和PipeReader之间传递的缓冲区的链接列表

PIPe负责主动管理 当前使用了多少数据【游标】，下次接着从结束位置后读起

reader.AdvanceTo(buffer.GetPosition(4)); 
这是一个相当实用的设计，它解决了"读了就得用"的问题，不仅可以将不用的数据下次再使用，还可以实现Peek的操作，只读但不改变游标。


同时，解决了读和写分离的能力。
1.当有数据了。才建立动态缓冲区，减少了提取建立缓冲区内存的消耗。
2.解析逻辑与网络代码分离，网络获取就是网络获取， 解析逻辑就是解析逻辑。 不再绑定再一起。------当前，过去我也是采用长链表、信号量订阅消费的方式来分离的。
3.基于span、Pipe、Segment、MemoryPool的操作------------比过去临时开辟指定大小的byte数组要效率高。
4.游标管理-------不用过去那样字节封装了。
5.异步----对于消息处理、二进制格式化协议处理、文件处理、网络通信  提供了基础基类。


———————————————————————————————————————————————————结论，后面建议大家使用System.IO.Pipelines—————————————————————————————————————————————————

