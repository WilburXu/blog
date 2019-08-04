# MySQL using filesort

MySQL 执行查询语句， 对于order by谓词，可能会使用filesort或者temporary。比如explain一条语句的时候，会看到Extra字段中可能会出现，using filesort和using temporary。下面我们就来探讨下两个的区别和适用场景。

. using filesort

filesort主要用于查询数据结果集的排序操作，首先MySQL会使用sort_buffer_size大小的内存进行排序，如果结果集超过了sort_buffer_size大小，会把这一个排序后的chunk转移到file上，最后使用多路归并排序完成所有数据的排序操作。

MySQL filesort有两种使用模式:

模式1: sort的item保存了所需要的所有字段，排序完成后，没有必要再回表扫描。

模式2: sort的item仅包括，待排序完成后，根据rowid查询所需要的columns。

很明显，模式1能够极大的减少回表的随机IO。

\2. using temporary

MySQL使用临时表保存临时的结构，以用于后续的处理，MySQL首先创建heap引擎的临时表，如果临时的数据过多，超过max_heap_table_size的大小，会自动把临时表转换成MyISAM引擎的表来使用。

从上面的解释上来看，filesort和temporary的使用场景的区别并不是很明显，不过，有以下的原则:

filesort只能应用在单个表上，如果有多个表的数据需要排序，那么MySQL会先使用using temporary保存临时数据，然后再在临时表上使用filesort进行排序，最后输出结果。