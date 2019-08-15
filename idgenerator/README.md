## id-md5-search

一个身份证MD5查询系统，输入MD5值，查找对应身份证号码。

### 步骤

1. 生成数据
2. 排序数据
3. 加载索引(使用redis | 或者读入内存)
4. 查询

### 设计思路

1. 存储最近100年所有有可能出现的身份证数据（md5+身份证号）
  * 预估数据量为 6000 * 100 * 365 * 999 = 218781000000(6000 为区号)
2. 以什么方式存储
  * 如果以字符串存储，身份证18位，md5 32 位，每条数据50个字节，大约10T数据
  * 如果以数字方式存储，身份证号可以使用一个int64存储，md5值可以使用两个uint64存储，每条数据24个字节，大约5T
3. 存在什么地方
  * 数据库（数据太多，为了查询性能需要分库分表-分表按md5值顺序存储-需要先排序
  * 存文件（以数字形式存储比以字符串存储占用空间小，优先使用数字形式存储
4. 数据生成后需要将内容按照md5值排序
  * 排序方式拆分成, 生成时按区号生成，有大约6000文件，每个文件800M
  * 先把这6000文件排序
    * 800M文件拆分成100个8M文件，内部使用快速排序排序后写入临时文件
	* 分别取100 文件的一条数据，放入小顶堆，取堆顶元素
	* 堆顶元素写入新文件，然后再从堆顶元素对应的文件读取一条新的数据
	* 依次执行，生成新的排序文件
  * 把6000排序好序的文件使用堆排序生成6000 新的排序文件,文件名为md5范围
5. 查询时首先判断范围找到指定文件，读入文件使用二分查找查找数据
  * 一次读入800M数据效率较低，可以参考数据库把文件内按指定大小划分，记录范围
  * 比如文件内每1000 条数据统计一个范围，把这个范围作为索引加载到内存或者存储到redis(大约需要6G内存)
  * 查询时先根据索引找到文件以及文件的大概位置，读取那一块的数据，使用二分查找查找记录