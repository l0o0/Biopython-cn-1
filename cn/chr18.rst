第18章  Cookbook – 用它做一些很酷的事情
================================================

Biopython目前有两个“cookbook”示例集合——本章（本章包含在教程中许多年，并渐渐成长），和
在我们的维基上的由用户贡献的集合： ```http://biopython.org/wiki/Category:Cookbook``
 <http://biopython.org/wiki/Category:Cookbook>`__ 。

我们在试着鼓励Biopython用户贡献他们自己的示例到维基上。除了能帮助社区之外，分享像这样的
示例的一个直接的好处是，你也能从其他Biopython用户和开发者中获得一些关于代码的反馈——这或许
能帮助你改进自己的Python代码。

长期来说，我们可能最终会将这一章所有的示例都转移到维基上，或者本教程其他的地方。

18.1  操作序列文件
---------------------------------

这部分将展示更多使用第 \ `5 <#chapter:Bio.SeqIO>`__ 章所描述的 ``Bio.SeqIO`` 模块来进行序列
输入/输出操作的例子。

18.1.1  过滤文件中的序列
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

通常你会拥有一个包含许多序列的大文件（例如，FASTA基因文件，或者FASTQ或SFF读长文件），和一个
单独的包含你所感兴趣的序列子集合的序列ID列表，而你希望为这个子集合创建一个新的序列文件。

让我们假设这个ID列表在一个简单的文本文件中，作为每一行的第一个词。这可能是一个表格文件，其中
第一列是ID。尝试下面的代码：

.. code:: verbatim

    from Bio import SeqIO
    input_file = "big_file.sff"
    id_file = "short_list.txt"
    output_file = "short_list.sff"
    wanted = set(line.rstrip("\n").split(None,1)[0] for line in open(id_file))
    print "Found %i unique identifiers in %s" % (len(wanted), id_file)
    records = (r for r in SeqIO.parse(input_file, "sff") if r.id in wanted)
    count = SeqIO.write(records, output_file, "sff")
    print "Saved %i records from %s to %s" % (count, input_file, output_file)
    if count < len(wanted):
        print "Warning %i IDs not found in %s" % (len(wanted)-count, input_file)

注意，我们使用Python的 ``set`` 类型而不是 ``list``，这会使得检测元素的成员与否更快。

18.1.2  生成随机化基因组
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

假设你在检视一个基因组序列，寻找一些序列特征——或许是极端局部GC含量偏差，或者可能的限制性酶切
位点。一旦你使你的Python代码在真实的基因组上运行后，试着在相同基因组的随机化版本上运行相同的
查询来进行统计分析或许是明智的选择（毕竟，任何你发现的“特性”都可能只是偶然事件）。

在这个讨论中，我们将使用来自 *Yersinia pestis biovar Microtus* 的pPCP1质粒的GenBank文件。该文
件包含在Biopython单元测试的GenBank文件夹中，或者你可以从我们的网站上得到，
```NC_005816.gb`` <http://biopython.org/SRC/biopython/Tests/GenBank/NC_005816.gb>`__.
该文件一个且仅有一个记录，所以我们能用 ``Bio.SeqIO.read()`` 函数把它当做 ``SeqRecord`` 读入：

.. code:: verbatim

    >>> from Bio import SeqIO
    >>> original_rec = SeqIO.read("NC_005816.gb","genbank")

那么，我们怎么生成一个原始序列重排后的版本能？我会使用Python内置的 ``random`` 模块来做这个，特别
是 ``random.shuffle`` 函数——但是这个只作用于Python列表。我们的序列是一个 ``Seq`` 对象，所以为了重排
它，我们需要将它转换为一个列表：

.. code:: verbatim

    >>> import random
    >>> nuc_list = list(original_rec.seq)
    >>> random.shuffle(nuc_list) #acts in situ!

现在，为了使用 ``Bio.SeqIO`` 输出重排的序列，我们需要使用重排后的列表重新创建包含一个新的 ``Seq``
的新的 ``SeqRecord``。要实现这个，我们需要将核苷酸（单字母字符串）列表转换为长字符串——标准的Python方式
是使用字符串对象的join方法来实现。

.. code:: verbatim

    >>> from Bio.Seq import Seq
    >>> from Bio.SeqRecord import SeqRecord
    >>> shuffled_rec = SeqRecord(Seq("".join(nuc_list), original_rec.seq.alphabet),
    ...                          id="Shuffled", description="Based on %s" % original_rec.id)

让我们将所有这些片段放到一起来组成一个完整的Python脚本，这个脚本将生成一个包含30个原始序列的随机重排版本的
单独的FASTA文件。

第一个版本只是使用一个大的for循环，并一个一个的输出记录（使用章节 \ `5.5.4 <#sec:Bio.SeqIO-and-StringIO>`__ 
描述的``SeqRecord`` 的格式化方法）：

.. code:: verbatim

    import random
    from Bio.Seq import Seq
    from Bio.SeqRecord import SeqRecord
    from Bio import SeqIO

    original_rec = SeqIO.read("NC_005816.gb","genbank")

    handle = open("shuffled.fasta", "w")
    for i in range(30):
        nuc_list = list(original_rec.seq)
        random.shuffle(nuc_list)
        shuffled_rec = SeqRecord(Seq("".join(nuc_list), original_rec.seq.alphabet), \
                                 id="Shuffled%i" % (i+1), \
                                 description="Based on %s" % original_rec.id)
        handle.write(shuffled_rec.format("fasta"))
    handle.close()

我个人更喜欢下面的版本，使用一个函数来重排记录，和一个生成表达式，而非for循环：

.. code:: verbatim

    import random
    from Bio.Seq import Seq
    from Bio.SeqRecord import SeqRecord
    from Bio import SeqIO

    def make_shuffle_record(record, new_id):
        nuc_list = list(record.seq)
        random.shuffle(nuc_list)
        return SeqRecord(Seq("".join(nuc_list), record.seq.alphabet), \
               id=new_id, description="Based on %s" % original_rec.id)
       
    original_rec = SeqIO.read("NC_005816.gb","genbank")
    shuffled_recs = (make_shuffle_record(original_rec, "Shuffled%i" % (i+1)) \
                     for i in range(30))
    handle = open("shuffled.fasta", "w")
    SeqIO.write(shuffled_recs, handle, "fasta")
    handle.close()

18.1.3  翻译CDS条目为FASTA文件
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

假设你有一个包含某个物种的CDS条目的输入文件，你想生成一个新的包含它们的蛋白序列的FASTA文件。也就是，从原始文件中取出
每一个核苷酸序列，并翻译它。回到章节 \ `3.9 <#sec:translation>`__ 我们看到了怎么使用 ``Seq`` 对象的 ``translate`` 方法，
和可选的 ``cds`` 参数来使得不同的起始密码子能正确翻译。

就像章节 \ `5.5.3 <#sec:SeqIO-reverse-complement>`__ 反向互补例子中展示的那样，我们能用 ``Bio.SeqIO`` 将这个结合起来。
关键是对每一个核苷酸 ``SeqRecord`` ，我们需要创建一个蛋白 ``SeqRecord`` —— 并注意对它命名。

你能编写自己的函数来做这个事情，为你的序列选择合适的蛋白标识和恰当的遗传密码。在这个例子中，我们仅使用默认的密码表，给
标识加一个前缀。

.. code:: verbatim

    from Bio.SeqRecord import SeqRecord
    def make_protein_record(nuc_record):
        """Returns a new SeqRecord with the translated sequence (default table)."""
        return SeqRecord(seq = nuc_record.seq.translate(cds=True), \
                         id = "trans_" + nuc_record.id, \
                         description = "translation of CDS, using default table")

我们接着能用这个函数将输入的核苷酸记录转换为蛋白记录，以备输出。一个优雅且内存高效的方式是使用一个生成表达式实现：

.. code:: verbatim

    from Bio import SeqIO
    proteins = (make_protein_record(nuc_rec) for nuc_rec in \
                SeqIO.parse("coding_sequences.fasta", "fasta"))
    SeqIO.write(proteins, "translations.fasta", "fasta")

这适用于任何全编码序列的FASTA文件。如果你使用部分编码序列，你可能更喜欢在上面的例子中使用 ``nuc_record.seq.translate(to_stop=True)``
因为它不会检查起始密码的有效性，等等。

18.1.4  将FASTA文件中的序列变为大写
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

通常你会从合作者那里得到FASTA文件的数据，有时候这些序列可能是大小写混合的。在某些情况下，这些可能是有意为之的（例如，
小写的作为低质量的区域），但通常不太重要。你可能希望编辑这个文件以使所有的序列都变得一致（如，都为大写），你可以使用
``SeqRecord`` 对象的 ``upper()`` 方法轻易的实现（Biopython 1.55中引入）：

.. code:: verbatim

    from Bio import SeqIO
    records = (rec.upper() for rec in SeqIO.parse("mixed.fas", "fasta"))
    count = SeqIO.write(records, "upper.fas", "fasta")
    print "Converted %i records to upper case" % count

这是怎么工作的呢？第一行只是导入 ``Bio.SeqIO`` 模块。第二行是最有趣的——这是一个Python生成器表达式，它提供从输入文件（ ``mixed.fas`` ）解析后
的每个记录的大写版本。第三行中，我们把这个生成器表达式传给 ``Bio.SeqIO.write()`` 函数，它保存新的大写的记录到我们的输出文件（ ``upper.fas`` ）。

我们使用生成器（而不是一个列表或列表解析式）的原因是，前者表示每次仅有一个记录保存在内存中。当你在处理包含成千上万的条目的大文件时，这可能非常重要。

18.1.5  对序列文件排序
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

假设你想对一个序列文件按序列长度排序（例如，一个序列拼接的重叠群(contig)集合），而且你工作的文件格式是像FASTA或FASTQ这样 ``Bio.SeqIO`` 能读写（和索引）
的格式。

如果文件足够小，你能将它都一次读入内存为一个 ``SeqRecord`` 对象列表，对列表进行排序，并保存它：

.. code:: verbatim

    from Bio import SeqIO
    records = list(SeqIO.parse("ls_orchid.fasta","fasta"))
    records.sort(cmp=lambda x,y: cmp(len(x),len(y)))
    SeqIO.write(records, "sorted_orchids.fasta", "fasta")

唯一需要聪明点的是指明一个比较函数来说明怎样对记录进行排序（这里我们按长度对他们排序）。如果你希望最长的记录在第一个，你可以交换比对，或者使用reverse参数：

.. code:: verbatim

    from Bio import SeqIO
    records = list(SeqIO.parse("ls_orchid.fasta","fasta"))
    records.sort(cmp=lambda x,y: cmp(len(y),len(x)))
    SeqIO.write(records, "sorted_orchids.fasta", "fasta")

现在这个非常直接——但是如果你的文件非常大，你不能像这样把它整个加载到内存中应该怎么办呢？例如，你可能有一些二代测序的读长要根据长度排序。这可以通过 ``Bio.SeqIO.index()``
函数解决。

.. code:: verbatim

    from Bio import SeqIO
    #Get the lengths and ids, and sort on length         
    len_and_ids = sorted((len(rec), rec.id) for rec in \
                         SeqIO.parse("ls_orchid.fasta","fasta"))
    ids = reversed([id for (length, id) in len_and_ids])
    del len_and_ids #free this memory
    record_index = SeqIO.index("ls_orchid.fasta", "fasta")
    records = (record_index[id] for id in ids)
    SeqIO.write(records, "sorted.fasta", "fasta")

首先我们使用 ``Bio.SeqIO.parse()`` 扫描一次整个文件，在一个tuple列表中记下所有记录的标识和他们的长度。接着我们对这个列表进行排序使得他们按长度顺序进行排列，并舍弃这些长度。
使用这个排列后的标识列表， ``Bio.SeqIO.index()`` 允许我们一个一个获取这些记录，我们把它们传给 ``Bio.SeqIO.write()`` 输出。

这些例子都使用 ``Bio.SeqIO`` 来解析记录为 ``SeqRecord`` 对象，并通过 ``Bio.SeqIO.write()`` 输出。当你想排序的文件格式 ``Bio.SeqIO.write()`` 不支持应该怎么办呢？如纯文本的
SwissProt格式。这里有一个额外的解决方法，使用在 Biopython 1.54 (see Section \ `5.4.2.2 <#sec:seqio-index-getraw>`__) 的 ``Bio.SeqIO.index()`` 中添加的 ``get_raw()`` 方法。

.. code:: verbatim

    from Bio import SeqIO
    #Get the lengths and ids, and sort on length         
    len_and_ids = sorted((len(rec), rec.id) for rec in \
                         SeqIO.parse("ls_orchid.fasta","fasta"))
    ids = reversed([id for (length, id) in len_and_ids])
    del len_and_ids #free this memory
    record_index = SeqIO.index("ls_orchid.fasta", "fasta")
    handle = open("sorted.fasta", "w")
    for id in ids:
        handle.write(record_index.get_raw(id))
    handle.close()

作为一个回报，因为它不将数据第二次解析为 ``SeqRecord`` 对象，它会更快。

18.1.6  FASTQ文件的简单质量过滤
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

FASTQ文件格式在Sanger被引入，目前被广泛用来存储核苷酸序列读长和它们的质量分数。
FASTQ文件（和相关的QUAL文件）是单字母注释（per-letter-annotation）的最好的例子，
因为序列中每一个核苷酸都有一个相对应的质量分数。任何单字母注释都以list、tuple或
string被保存在 ``SeqRecord`` 的 ``letter_annotations`` 字典中（包含和序列长度
相同个数的元素）。

一个常见的工作是输入一个大的测序读长集合，并根据它们的质量分数过滤他们（或修剪他们）。
下面的例子非常简单，然而足以展示处理 ``SeqRecord`` 对象中质量数据的基本用法。我们所有要
做的事情是读入一个FASTQ文件，过滤并取出那些PHRED质量分数在某个阈值（这里为20）以上的记录。

在这个例子中，我们将使用从ENA序列读长存档下载的真实数据，
```ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR020/SRR020192/SRR020192.fastq.gz`` <ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR020/SRR020192/SRR020192.fastq.gz>`__
(2MB) ，解压后为19MB的文件 ``SRR020192.fastq`` 。这些是由Roche 454 GS FLX生成的感染加利福利亚海狮的病毒单端数据（参见 ```http://www.ebi.ac.uk/ena/data/view/SRS004476`` <http://www.ebi.ac.uk/ena/data/view/SRS004476>`__
的详细信息）。

首先，让我们来统计读长数：

.. code:: verbatim

    from Bio import SeqIO
    count = 0
    for rec in SeqIO.parse("SRR020192.fastq", "fastq"):
        count += 1
    print "%i reads" % count

现在让我们做一个简单的过滤，PHRED质量不小于20：

.. code:: verbatim

    from Bio import SeqIO
    good_reads = (rec for rec in \
                  SeqIO.parse("SRR020192.fastq", "fastq") \
                  if min(rec.letter_annotations["phred_quality"]) >= 20)
    count = SeqIO.write(good_reads, "good_quality.fastq", "fastq")
    print "Saved %i reads" % count

这只取出了41892条读长中的14580条。一个更有意义的做法是根据质量来裁剪读长，但是这里只是作为一个例子。

FASTQ文件可以包含上百万的记录，所以最好避免一次全部加载它们到内存。这个例子使用一个生成器表达式，这意味着
每次只有一个 ``SeqRecord`` 被创建 —— 避免内存限制。

18.1.7  切除引物序列
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在这个例子中，我们将假设 ``GATGACGGTGT`` 是一个我们要在FASTQ格式的读长数据中
找的5’端的引物序列。同上面的例子一样，我们将使用从ENA下载的 ``SRR020192.fastq``
文件（ ```ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR020/SRR020192/SRR020192.fastq.gz`` <ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR020/SRR020192/SRR020192.fastq.gz>`__ ）。
相同的方式同样适用于任何其他支持的格式（例如FASTA文件）。

这个代码使用 ``Bio.SeqIO`` 和一个生成器表达式（避免一次加载所有的序列到内存中），
以及 ``Seq`` 对象的 ``startswith`` 方法来检查读长是否以引物序列开始：

.. code:: verbatim

    from Bio import SeqIO
    primer_reads = (rec for rec in \
                    SeqIO.parse("SRR020192.fastq", "fastq") \
                    if rec.seq.startswith("GATGACGGTGT"))
    count = SeqIO.write(primer_reads, "with_primer.fastq", "fastq")
    print "Saved %i reads" % count

这将从 ``SRR014849.fastq`` 找到13819条读长记录，并保存为一个新的FASTQ文件，``with_primer.fastq``。

现在，假设你希望创建一个包含这些读长，但去除了所有引物序列的FASTQ文件。只需要很小的修改，我们能对 ``SeqRecord``
进行切片（参见章节 \ `4.6 <#sec:SeqRecord-slicing>`__ ）以移除前11个字母（我们的引物长度）：

.. code:: verbatim

    from Bio import SeqIO
    trimmed_primer_reads = (rec[11:] for rec in \
                            SeqIO.parse("SRR020192.fastq", "fastq") \
                            if rec.seq.startswith("GATGACGGTGT"))
    count = SeqIO.write(trimmed_primer_reads, "with_primer_trimmed.fastq", "fastq")
    print "Saved %i reads" % count

再次，这将从 ``SRR020192.fastq`` 取出13819条读长，但是这次移除了前十个字符，并保存它们为另一个
新的FASTQ文件， ``with_primer_trimmed.fastq`` 。

最后，假设你想移除部分读长中的引物并创建一个新的FASTQ文件，而其他的读长保持不变。如果我们仍然希望
使用生成器表达式，声明我们自己的修剪函数可能更加清楚：

.. code:: verbatim

    from Bio import SeqIO
    def trim_primer(record, primer):
        if record.seq.startswith(primer):
            return record[len(primer):]
        else:
            return record

    trimmed_reads = (trim_primer(record, "GATGACGGTGT") for record in \
                     SeqIO.parse("SRR020192.fastq", "fastq"))
    count = SeqIO.write(trimmed_reads, "trimmed.fastq", "fastq")
    print "Saved %i reads" % count

这需要更长的时间，因为这次输出文件包含所有41892个读长。再次，我们将使用生成器表达式来避免任何的
内存问题。你也可以使用一个生成器函数来替代生成器表达式。

.. code:: verbatim

    from Bio import SeqIO
    def trim_primers(records, primer):
        """Removes perfect primer sequences at start of reads.
        
        This is a generator function, the records argument should
        be a list or iterator returning SeqRecord objects.
        """
        len_primer = len(primer) #cache this for later
        for record in records:
            if record.seq.startswith(primer):
                yield record[len_primer:]
            else:
                yield record

    original_reads = SeqIO.parse("SRR020192.fastq", "fastq")
    trimmed_reads = trim_primers(original_reads, "GATGACGGTGT")
    count = SeqIO.write(trimmed_reads, "trimmed.fastq", "fastq") 
    print "Saved %i reads" % count

这种形式非常灵活，如果你想做一些更复杂的事情，譬如只保留部分记录 —— 像下一个例子中展示的那样。

18.1.8  切除接头序列
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

这实际上是前面例子的一个简单扩展。我们将假设 ``GATGACGGTGT`` 是某个FASTQ格式的读长数据中的一个接头序列，并再次使用
来自NCBI的 ``SRR020192.fastq`` 文件
（ ```ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR020/SRR020192/SRR020192.fastq.gz`` <ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR020/SRR020192/SRR020192.fastq.gz>`__ ）。

然而这次，我们将在读长的 *任何位置* 查找序列，不仅仅是最开始：

.. code:: verbatim

    from Bio import SeqIO

    def trim_adaptors(records, adaptor):
        """Trims perfect adaptor sequences.
        
        This is a generator function, the records argument should
        be a list or iterator returning SeqRecord objects.
        """
        len_adaptor = len(adaptor) #cache this for later
        for record in records:
            index = record.seq.find(adaptor)
            if index == -1:
                #adaptor not found, so won't trim
                yield record
            else:
                #trim off the adaptor
                yield record[index+len_adaptor:]

    original_reads = SeqIO.parse("SRR020192.fastq", "fastq")
    trimmed_reads = trim_adaptors(original_reads, "GATGACGGTGT")
    count = SeqIO.write(trimmed_reads, "trimmed.fastq", "fastq") 
    print "Saved %i reads" % count

因为我们在这个例子中使用FASTQ输入文件， ``SeqRecord`` 对象有质量分数的单字母注释（per-letter-annotation）。
通过对 ``SeqRecord`` 对象进行切片，合适的分数也保留在剪切后的记录中，所以我们也可以将它们输出到一个FASTQ文件。

和上面的例子，我们只在每个读长的开始查找引物/接头，相比，你发现有些剪切后的读长非常短（例如，如果接头序列在读长的
中部发现，而不是开始附近）。所以，让我们再加入一个最低长度要求：

.. code:: verbatim

    from Bio import SeqIO

    def trim_adaptors(records, adaptor, min_len):
        """Trims perfect adaptor sequences, checks read length.
        
        This is a generator function, the records argument should
        be a list or iterator returning SeqRecord objects.
        """
        len_adaptor = len(adaptor) #cache this for later
        for record in records:
            len_record = len(record) #cache this for later
            if len(record) < min_len:
               #Too short to keep
               continue
            index = record.seq.find(adaptor)
            if index == -1:
                #adaptor not found, so won't trim
                yield record
            elif len_record - index - len_adaptor >= min_len:
                #after trimming this will still be long enough
                yield record[index+len_adaptor:]

    original_reads = SeqIO.parse("SRR020192.fastq", "fastq")
    trimmed_reads = trim_adaptors(original_reads, "GATGACGGTGT", 100)
    count = SeqIO.write(trimmed_reads, "trimmed.fastq", "fastq") 
    print "Saved %i reads" % count

通过改变格式名称，你也可以将这个应用于FASTA文件。该代码也可以扩展为模糊匹配，而非绝对匹配（或许用一个两两比对，或者考虑
读长的质量分数），但是这会变得更慢。

18.1.9  转换FASTQ文件
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

回到章节 \ `5.5.2 <#sec:SeqIO-conversion>`__ ，我们展示了怎样使用 ``Bio.SeqIO`` 转换两个文件格式。这里，我们将更进一步探讨
在二代DNA测序中使用的FASTQ文件。对于更加详细的介绍，请参加 Cock *et al.* (2009)
[`7 <#cock2010>`__\ ] 。FASTQ文件同时存储DNA序列（作为字符串）和相应的读长质量。

PHRED分数（在大多数FASTQ文件中使用，也存在QUAL、ACE和SFF文件中）已经成为一个用来表示某个给定碱基测序错误概率（这里用
 *P*\ :sub:`*e*` 表示）的 *事实上的* 标准，使用一个简单的10为底的对数表达式：

+--------------------------------------------------------------------------+
| *Q*\ :sub:`PHRED` = − 10 × log:sub:`10` ( *P*\ :sub:`*e*` )     (18.1)   |
+--------------------------------------------------------------------------+

这意味着一个错误的读长（ *P*\ :sub:`*e*` = 1 ）得到的PHRED质量为0，而一个非常好的 *P*\ :sub:`*e*` = 0.00001 的读长得到的PHRED
质量为50。然而原始的测序数据质量比这个要高的非常稀少，通过后期处理，如读长映射和组装，质量到达90是可能的（确实，MAQ工具允许PHRED分数
包含在0到93范围内）。

FASTQ格式有潜力成为以单文件纯文本方式存储测序读长的字符和质量分数的 *事实上的* 的标准。 唯一的美中不足是，目前至少有三个FASTQ格式版本，
它们并不兼容，且难以区分...

#. 原始的Sanger FASTQ格式将PHRED质量分数和33个ASCII字符偏移进行编码。NCBI目前在它们的Short Read Archive中使用这种格式。我们在 ``Bio.SeqIO`` 中
   称之为 ``fastq`` （或 ``fastq-sanger`` ）格式。
#. Solexa（后来由Illumina收购）引入了他们自己的版本，使用Solexa质量分数和64个ASCII字符偏移进行编码。我们叫做 ``fastq-solexa`` 格式。
#. Illumina工作流1.3进一步推出了PHRED质量分数（这比较一致）的FASTQ文件，但是却以64个ASCII字符偏移编码。我们叫做 ``fastq-illumina`` 格式。

Solexa质量分数采用一种不同的对数式定义：

*Q*\ :sub:`Solexa` = − 10 × log:sub:`10` 

| ⎛
|  ⎜
|  ⎜
|  ⎝

+-----------------------+
| *P*\ :sub:`*e*`       |
+-----------------------+
+-----------------------+
| 1−\ *P*\ :sub:`*e*`   |
+-----------------------+

 

| ⎞
|  ⎟
|  ⎟
|  ⎠

    (18.2)

由于Solexa/Illumina目前在他们的工作流1.3版本中已迁移到使用PHRED分数，Solexa质量分数将逐渐淡出使用。如果你将错误估值取等号（ *P*\ :sub:`*e*` ），这两个
等式允许在两个打分系统之间进行转换 —— Biopython在 ``Bio.SeqIO.QualityIO`` 模块中有函数可以实现，该函数在使用 ``Bio.SeqIO`` 进行从Solexa/Illumina老文件格式
到标准Sanger FASTQ文件格式转换时被调用：

.. code:: verbatim

    from Bio import SeqIO
    SeqIO.convert("solexa.fastq", "fastq-solexa", "standard.fastq", "fastq")

如果你想转换新的Illumina 1.3+ FASTQ文件，有改变的只是ASCII的偏移。因为尽管编码不同，所有的质量分数都是PHRED分数：

.. code:: verbatim

    from Bio import SeqIO
    SeqIO.convert("illumina.fastq", "fastq-illumina", "standard.fastq", "fastq")

注意，像这样使用 ``Bio.SeqIO.convert()`` 会比 ``Bio.SeqIO.parse()`` 和 ``Bio.SeqIO.write()`` 组合快得 *多* ，因为
转换FASTQ变种（包括FASTQ到FASTA的转换）的代码经过优化。

对于质量好的读长，PHRED和Solexa分数几乎相等，这意味着，因为 ``fasta-solexa`` 和 ``fastq-illumina`` 都使用64个ASCII字符偏移，它们的文件几乎相同。这个是Illumina有意
设计选择的，意味着使用老版本 ``fasta-solexa`` 格式文件的应用或许在使用新版本 ``fastq-illumina`` 格式文件时也是可以的（在好的数据上）。当然，两个版本和原始的，由Sanger、
NCBI和其他地方使用的FASTQ标准有很大不同（格式名为 ``fastq`` 或 ``fastq-sanger`` ）。

了解更多细节，请参见内置的帮助（或 `在线帮助 <http://www.biopython.org/DIST/docs/api/Bio.SeqIO.QualityIO-module.html>`__ ）：

.. code:: verbatim

    >>> from Bio.SeqIO import QualityIO
    >>> help(QualityIO)
    ...

18.1.10  转换FASTA和QUAL文件为FASTQ文件
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

FASTQ *同时* 包含序列和他们的质量信息字符串。FASTA文件 *只* 包含序列，而QUAL文件 *只* 
包含质量。因此，一个单独的FASTQ文件可以转换为或转换自 *成对的* FASTA和QUAL文件。

从FASTQ到FASTA很简单：

.. code:: verbatim

    from Bio import SeqIO
    SeqIO.convert("example.fastq", "fastq", "example.fasta", "fasta")

从FASTQ到QUAL也很简单：

.. code:: verbatim

    from Bio import SeqIO
    SeqIO.convert("example.fastq", "fastq", "example.qual", "qual")

然而，反向则有一点复杂。你可以使用 ``Bio.SeqIO.parse()`` 迭代一个 *单独* 文件中的所有记录，
但是这里我们有两个输入文件。有几个可能的策略，然而这里假设两个文件是真的完全匹配的，最内存
高效的方式是同时循环两个文件。代码有些巧妙，所以在 ``Bio.SeqIO.QualityIO`` 模块中我们提供一个函数
来实现，叫做 ``PairedFastaQualIterator``。它接受两个句柄（FASTA文件和QUAL文件）并返回一个
 ``SeqRecord`` 迭代器：

.. code:: verbatim

    from Bio.SeqIO.QualityIO import PairedFastaQualIterator
    for record in PairedFastaQualIterator(open("example.fasta"), open("example.qual")):
       print record

这个函数将检查FASTA和QUAL文件是否一致（例如，记录顺序是相同的，并且序列长度一致）。
你可以和 ``Bio.SeqIO.write()`` 函数结合使用，转换一对FASTA和QUAL文件为单独的FASTQ文件：

.. code:: verbatim

    from Bio import SeqIO
    from Bio.SeqIO.QualityIO import PairedFastaQualIterator
    handle = open("temp.fastq", "w") #w=write
    records = PairedFastaQualIterator(open("example.fasta"), open("example.qual"))
    count = SeqIO.write(records, handle, "fastq")
    handle.close()
    print "Converted %i records" % count

18.1.11  索引FASTQ文件
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

FASTQ文件通常非常大，包含上百万的读长。由于数据量的原因，你不能一次加载所有的记录到内存中。
这就是为什么上面的例子（过滤和剪切）迭代整个文件的过程中，每次只查看一个 ``SeqRecord`` 。

然而，有时候你不能使用一个大的循环或迭代器 —— 你或许需要随机获取读长。这里 ``Bio.SeqIO.index()`` 
函数将证明非常有用，它允许你使用名字获取FASTQ中的任何读长（参见章节 \ `5.4.2 <#sec:SeqIO-index>`__ ）。

再次，我们将使用来自 ENA
(```ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR020/SRR020192/SRR020192.fastq.gz`` <ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR020/SRR020192/SRR020192.fastq.gz>`__) 
的文件 ``SRR020192.fastq`` ，尽管这是一个非常小的FASTQ文件，只有不到50,000读长：

.. code:: verbatim

    >>> from Bio import SeqIO
    >>> fq_dict = SeqIO.index("SRR020192.fastq", "fastq")
    >>> len(fq_dict)
    41892
    >>> fq_dict.keys()[:4]
    ['SRR020192.38240', 'SRR020192.23181', 'SRR020192.40568', 'SRR020192.23186']
    >>> fq_dict["SRR020192.23186"].seq
    Seq('GTCCCAGTATTCGGATTTGTCTGCCAAAACAATGAAATTGACACAGTTTACAAC...CCG', SingleLetterAlphabet())

当在包含7百万读长的FASTQ文件上测试时，索引大概花费1分钟，然而获取记录几乎是瞬间。

章节 \ `18.1.5 <#sec:SeqIO-sort>`__ 的例子展示了如何使用 ``Bio.SeqIO.index()`` 函数来
进行FASTA文件排序 —— 这也可以用在FASTQ文件上。

18.1.12  转换SFF文件
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果你处理454(Roche)序列数据，你可能会接触Standard Flowgram Format (SFF)原始数据。这包括序列读长（
called bases）、质量分数和原始流信息。

一个最常见的工作是转换SFF文件为一对FASTA和QUAL文件，或者一个单独的FASTQ文件。这些操作使用 ``Bio.SeqIO.convert()`` 
函数非常平凡（参见 \ `5.5.2 <#sec:SeqIO-conversion>`__ ）：

.. code:: verbatim

    >>> from Bio import SeqIO
    >>> SeqIO.convert("E3MFGYR02_random_10_reads.sff", "sff", "reads.fasta", "fasta")
    10
    >>> SeqIO.convert("E3MFGYR02_random_10_reads.sff", "sff", "reads.qual", "qual")
    10
    >>> SeqIO.convert("E3MFGYR02_random_10_reads.sff", "sff", "reads.fastq", "fastq")
    10

注意这个转换函数返回记录的条数，在这个例子中为10。这将给你 *未裁剪* 的读长，其中先导和跟随链中低质量的
序列，或接头序列将以小写字母显示。如果你希望得到 *裁剪* 后的读长（使用SFF文件中的剪切信息），使用下面的
代码：

.. code:: verbatim

    >>> from Bio import SeqIO
    >>> SeqIO.convert("E3MFGYR02_random_10_reads.sff", "sff-trim", "trimmed.fasta", "fasta")
    10
    >>> SeqIO.convert("E3MFGYR02_random_10_reads.sff", "sff-trim", "trimmed.qual", "qual")
    10
    >>> SeqIO.convert("E3MFGYR02_random_10_reads.sff", "sff-trim", "trimmed.fastq", "fastq")
    10

如果你使用Linux，你可以向Roche请求一份“脱离仪器”的工具（通常叫做Newbler工具）。这提供了一种可选的方式，来
在命令行实现SFF到FASTA或QUAL的转换（但是FASTQ输出目前不支持）。

.. code:: verbatim

    $ sffinfo -seq -notrim E3MFGYR02_random_10_reads.sff > reads.fasta
    $ sffinfo -qual -notrim E3MFGYR02_random_10_reads.sff > reads.qual
    $ sffinfo -seq -trim E3MFGYR02_random_10_reads.sff > trimmed.fasta
    $ sffinfo -qual -trim E3MFGYR02_random_10_reads.sff > trimmed.qual

Biopython以大小写混合的方式来表示剪切位点实际上有意地模拟了Roche工具的做法。

要获得Biopython对SFF支持的更多信息，请参考内部帮助：

.. code:: verbatim

    >>> from Bio.SeqIO import SffIO
    >>> help(SffIO)
    ...

18.1.13  识别开放读码框
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在识别可能的基因中一个非常简单的第一步是寻找开放读码框（Open Reading Frame，ORF）。
这里我们的意思是寻找六个编码框中所有的没有终止密码子的长区域 —— 一个ORF只是一个不包含任何
框内终止密码子的核苷酸区域。

当然，为了发现基因，你也需要确定起始密码子、可能的启动子的位置 —— 而且在真核生物中，你也需要
关心内含子。然而，这种方法在病毒和原核生物中仍然有效。

为了展示怎样用Biopython实现这个目的，我们首先需要一个序列来查找。作为例子，我们再次使用细菌的
质粒 —— 然而，这次我们将以没有任何基因标记的纯文本FASTA文件开始：
```NC_005816.fna`` <http://biopython.org/SRC/biopython/Tests/GenBank/NC_005816.fna>`__ 。
这是一个细菌序列，所以我们需要使用NCBI密码子表11（参见章节 \ `3.9 <#sec:translation>`__ 关于翻
译的介绍）。

.. code:: verbatim

    >>> from Bio import SeqIO 
    >>> record = SeqIO.read("NC_005816.fna","fasta")
    >>> table = 11
    >>> min_pro_len = 100

这里有一个巧妙的技巧，使用 ``Seq`` 对象的 ``split`` 方法获得一个包含六个读码框中所有可能的ORF翻译的
列表：

.. code:: verbatim

    >>> for strand, nuc in [(+1, record.seq), (-1, record.seq.reverse_complement())]:
    ...     for frame in range(3):
    ...         length = 3 * ((len(record)-frame) // 3) #Multiple of three
    ...         for pro in nuc[frame:frame+length].translate(table).split("*"):
    ...             if len(pro) >= min_pro_len:
    ...                 print "%s...%s - length %i, strand %i, frame %i" \
    ...                       % (pro[:30], pro[-3:], len(pro), strand, frame)
    GCLMKKSSIVATIITILSGSANAASSQLIP...YRF - length 315, strand 1, frame 0
    KSGELRQTPPASSTLHLRLILQRSGVMMEL...NPE - length 285, strand 1, frame 1
    GLNCSFFSICNWKFIDYINRLFQIIYLCKN...YYH - length 176, strand 1, frame 1
    VKKILYIKALFLCTVIKLRRFIFSVNNMKF...DLP - length 165, strand 1, frame 1
    NQIQGVICSPDSGEFMVTFETVMEIKILHK...GVA - length 355, strand 1, frame 2
    RRKEHVSKKRRPQKRPRRRRFFHRLRPPDE...PTR - length 128, strand 1, frame 2
    TGKQNSCQMSAIWQLRQNTATKTRQNRARI...AIK - length 100, strand 1, frame 2
    QGSGYAFPHASILSGIAMSHFYFLVLHAVK...CSD - length 114, strand -1, frame 0
    IYSTSEHTGEQVMRTLDEVIASRSPESQTR...FHV - length 111, strand -1, frame 0
    WGKLQVIGLSMWMVLFSQRFDDWLNEQEDA...ESK - length 125, strand -1, frame 1
    RGIFMSDTMVVNGSGGVPAFLFSGSTLSSY...LLK - length 361, strand -1, frame 1
    WDVKTVTGVLHHPFHLTFSLCPEGATQSGR...VKR - length 111, strand -1, frame 1
    LSHTVTDFTDQMAQVGLCQCVNVFLDEVTG...KAA - length 107, strand -1, frame 2
    RALTGLSAPGIRSQTSCDRLRELRYVPVSL...PLQ - length 119, strand -1, frame 2

注意，这里我们从 *每条* 序列的5’末（起始）端开始计算读码框。对 *正向* 链一直从5’末（起始）端开始计算有时更加容易。

你可以轻易编辑上面的循环代码，来创建一个待选蛋白列表，或者将它转换为一个列表解析。现在，这个代码所不能做的一个事情
是记录蛋白的位置信息。

你可以用几种方式来处理。例如，下面的代码以蛋白计数的方式记录位置信息，并通过乘以三倍来转换为父序列，接着调整编码框
和链：

.. code:: verbatim

    from Bio import SeqIO 
    record = SeqIO.read("NC_005816.gb","genbank")
    table = 11
    min_pro_len = 100

    def find_orfs_with_trans(seq, trans_table, min_protein_length):
        answer = []
        seq_len = len(seq)
        for strand, nuc in [(+1, seq), (-1, seq.reverse_complement())]:
            for frame in range(3):
                trans = str(nuc[frame:].translate(trans_table))
                trans_len = len(trans)
                aa_start = 0
                aa_end = 0
                while aa_start < trans_len:
                    aa_end = trans.find("*", aa_start)
                    if aa_end == -1:
                        aa_end = trans_len
                    if aa_end-aa_start >= min_protein_length:
                        if strand == 1:
                            start = frame+aa_start*3
                            end = min(seq_len,frame+aa_end*3+3)
                        else:
                            start = seq_len-frame-aa_end*3-3
                            end = seq_len-frame-aa_start*3                        
                        answer.append((start, end, strand,
                                       trans[aa_start:aa_end]))
                    aa_start = aa_end+1
        answer.sort()
        return answer

    orf_list = find_orfs_with_trans(record.seq, table, min_pro_len)
    for start, end, strand, pro in orf_list:
        print "%s...%s - length %i, strand %i, %i:%i" \
              % (pro[:30], pro[-3:], len(pro), strand, start, end)

输出是：

.. code:: verbatim

    NQIQGVICSPDSGEFMVTFETVMEIKILHK...GVA - length 355, strand 1, 41:1109
    WDVKTVTGVLHHPFHLTFSLCPEGATQSGR...VKR - length 111, strand -1, 491:827
    KSGELRQTPPASSTLHLRLILQRSGVMMEL...NPE - length 285, strand 1, 1030:1888
    RALTGLSAPGIRSQTSCDRLRELRYVPVSL...PLQ - length 119, strand -1, 2830:3190
    RRKEHVSKKRRPQKRPRRRRFFHRLRPPDE...PTR - length 128, strand 1, 3470:3857
    GLNCSFFSICNWKFIDYINRLFQIIYLCKN...YYH - length 176, strand 1, 4249:4780
    RGIFMSDTMVVNGSGGVPAFLFSGSTLSSY...LLK - length 361, strand -1, 4814:5900
    VKKILYIKALFLCTVIKLRRFIFSVNNMKF...DLP - length 165, strand 1, 5923:6421
    LSHTVTDFTDQMAQVGLCQCVNVFLDEVTG...KAA - length 107, strand -1, 5974:6298
    GCLMKKSSIVATIITILSGSANAASSQLIP...YRF - length 315, strand 1, 6654:7602
    IYSTSEHTGEQVMRTLDEVIASRSPESQTR...FHV - length 111, strand -1, 7788:8124
    WGKLQVIGLSMWMVLFSQRFDDWLNEQEDA...ESK - length 125, strand -1, 8087:8465
    TGKQNSCQMSAIWQLRQNTATKTRQNRARI...AIK - length 100, strand 1, 8741:9044
    QGSGYAFPHASILSGIAMSHFYFLVLHAVK...CSD - length 114, strand -1, 9264:9609

如果你注释掉排序语句，那么蛋白序列将和之前显示的顺序一样，所以你能确定这是在做
相同的事情。这里，我们可以按位置进行排序，使得和GenBank文件中的实际注释相比对
较更加容易（就像章节 \ `17.1.9 <#sec:gd_nice_example>`__ 中显示的那样）。

然而，如果你想要的只是所有开放读码框的位置，这翻译每一个可能的密码子将是很浪费
时间的，包括转换和查找反向互补链。那么，你所要做的所有事情是查找可能的终止密码子
(和他们反向互补)。使用正则表达式是一个很显然的方式（参见Python中的 ``re`` 模块）。
这是描述查找字符串的一种非常强大的方式（然而非常复杂），也被许多编程语言和命令行
工具，如 ``grep``，所支持。你能找到关于这个话题的整本的书籍！

18.2  序列解析与简单作图
----------------------------------------

这一部分展示更多使用第 \ `5 <#chapter:Bio.SeqIO>`__ 章介绍的 ``Bio.SeqIO`` 模块进行序列解析的例子，
以及Python类库matplotlib中 ``pylab`` 的作图接口（参见 `matplotlib 主页的教程 <http://matplotlib.sourceforge.net/>`__ ）。
注意，跟随这些例子，你需要安装matplotlib - 但是没有它，你也可以尝试数据的解析的内容。

18.2.1  序列长度柱状图
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

许多时候，你可能想要将某个数据集中的序列长度分布可视化 —— 例如，基因组组装项目中的contig的大小范围。
在这个例子中，我们将再次使用我们的兰花FASTA文件 ```ls_orchid.fasta`` <http://biopython.org/DIST/docs/tutorial/examples/ls_orchid.fasta>`__ ，
它只包含94条序列。

首先，我们使用 ``Bio.SeqIO`` 来解析这个FASTA文件，并创建一个序列长度的列表。你可以用一个for循环来实现，
然而我觉得列表解析（list comprehension）更赏心悦目：

.. code:: verbatim

    >>> from Bio import SeqIO
    >>> sizes = [len(rec) for rec in SeqIO.parse("ls_orchid.fasta", "fasta")]
    >>> len(sizes), min(sizes), max(sizes)
    (94, 572, 789)
    >>> sizes
    [740, 753, 748, 744, 733, 718, 730, 704, 740, 709, 700, 726, ..., 592]

现在我们得到了所有基因的长度（以整数列表的形式），我们可以用matplotlib的柱状图功能来显示它。

.. code:: verbatim

    from Bio import SeqIO
    sizes = [len(rec) for rec in SeqIO.parse("ls_orchid.fasta", "fasta")]

    import pylab
    pylab.hist(sizes, bins=20)
    pylab.title("%i orchid sequences\nLengths %i to %i" \
                % (len(sizes),min(sizes),max(sizes)))
    pylab.xlabel("Sequence length (bp)")
    pylab.ylabel("Count")
    pylab.show()

这将弹出一个包含如下图形的新的窗口：

|image26|

注意，这些兰花序列的长度大多数大约在740bp左右，这里有可能有两个显著的序列分类，其中包含一个更短的序列子集。

*提示：* 除了使用 ``pylab.show()`` 在窗口中显示图像以外，你也可以使用 ``pylab.savefig(...)`` 来保存图像到
文件中（例如PNG或PDF文件）。

18.2.2  序列GC%含量作图
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

核酸序列另一个容易计算的量值是GC%。例如，你可能想要查看一个细菌基因组中所有基因的GC%含量，并研究任何离群值
来确定可能最近通过基因水平转移而获得的基因。同样，对于这个例子，我们再次使用我们的兰花FASTA文件
```ls_orchid.fasta`` <http://biopython.org/DIST/docs/tutorial/examples/ls_orchid.fasta>`__ 。

首先，我们使用 ``Bio.SeqIO`` 解析这个FASTA文件并创建一个GC百分含量的列表。再次，你可以使用for循环，但我更喜
欢这样：

.. code:: verbatim

    from Bio import SeqIO
    from Bio.SeqUtils import GC

    gc_values = sorted(GC(rec.seq) for rec in SeqIO.parse("ls_orchid.fasta", "fasta"))

读取完每个序列并计算了GC百分比，我们接着将它们按升序排列。现在，我们用这个浮点数列表采用matplotlib作图：

.. code:: verbatim

    import pylab
    pylab.plot(gc_values)
    pylab.title("%i orchid sequences\nGC%% %0.1f to %0.1f" \
                % (len(gc_values),min(gc_values),max(gc_values)))
    pylab.xlabel("Genes")
    pylab.ylabel("GC%")
    pylab.show()

像之前的例子一样，这将弹出一个新的窗口包含如下图形：

|image27|

如果你使用的是一个物种中的所有基因集，你可能得到一个比这个更加平滑的图。

18.2.3  核苷酸点线图
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

点线图是可视化比较两条核苷酸序列的相似性的一种方式。采用一个滑动窗来相互比较较短的子序列，通常使用一个不匹配阈值。
这里为了简单起见，我们将只查找完全匹配（如下图黑色所示）。

我们需要两条序列开始。为了论证，我们只取兰花FASTA文件中的前两条序列。
```ls_orchid.fasta`` <http://biopython.org/DIST/docs/tutorial/examples/ls_orchid.fasta>`__:

.. code:: verbatim

    from Bio import SeqIO
    handle = open("ls_orchid.fasta")
    record_iterator = SeqIO.parse(handle, "fasta")
    rec_one = record_iterator.next()
    rec_two = record_iterator.next()
    handle.close()

我们将展示两种方式。首先，一个简单幼稚的实现，它将所有滑动窗大小的子序列相互比较，并生产一个相似性矩阵。
你可以创建一个矩阵或数组对象，而在这儿，我们只用一个用嵌套的列表解析生成的布尔值列表的列表。

.. code:: verbatim

    window = 7
    seq_one = str(rec_one.seq).upper()
    seq_two = str(rec_two.seq).upper()
    data = [[(seq_one[i:i+window] <> seq_two[j:j+window]) \
            for j in range(len(seq_one)-window)] \
           for i in range(len(seq_two)-window)]

注意，我们在这里并 *没有* 检查反向的互补匹配。现在我们将使用matplotlib的 ``pylab.imshow()`` 函数来显示这个数据，
首先请求灰度模式，以保证这是在黑白颜色下完成的：

.. code:: verbatim

    import pylab
    pylab.gray()
    pylab.imshow(data)
    pylab.xlabel("%s (length %i bp)" % (rec_one.id, len(rec_one)))
    pylab.ylabel("%s (length %i bp)" % (rec_two.id, len(rec_two)))
    pylab.title("Dot plot using window size %i\n(allowing no mis-matches)" % window)
    pylab.show()

这将弹出一个新的窗口，包含类似这样的图形：

|image28|

可能如您所料，这两条序列非常相似，图中部分滑动窗大小的线沿着对角线匹配。这里没有能够说明是倒装或其他有趣事件的偏离对角线的匹配。

上面的代码在小的例子中工作得很好，但是应用到大的序列时，这里有两个问题，将在后面讨论。首先，这个暴力的所有对所有的比对方式非常慢。
作为替代，我们将创建一个词典来映射所有滑动窗大小的子序列的位置，然后取两者的交集来获得两条序列中都发现的子序列。这将占用更多的内存，
然而速度 *更* 快。另外， ``pylab.imshow()`` 函数受限于它能显示的矩阵的大小。作为替代，我们将使用 ``pylab.scatter()`` 函数。

我们从创建，从滑动窗大小的子序列到其位置的字典映射，开始：

.. code:: verbatim

    window = 7
    dict_one = {}
    dict_two = {}
    for (seq, section_dict) in [(str(rec_one.seq).upper(), dict_one),
                                (str(rec_two.seq).upper(), dict_two)]:
        for i in range(len(seq)-window):
            section = seq[i:i+window]
            try:
                section_dict[section].append(i)
            except KeyError:
                section_dict[section] = [i]
    #Now find any sub-sequences found in both sequences
    #(Python 2.3 would require slightly different code here)
    matches = set(dict_one).intersection(dict_two)
    print "%i unique matches" % len(matches)

为了使用 ``pylab.scatter()`` 函数，我们需要分开的 *x* 和 *y* 轴的列表：

.. code:: verbatim

    #Create lists of x and y co-ordinates for scatter plot
    x = []
    y = []
    for section in matches:
        for i in dict_one[section]:
            for j in dict_two[section]:
                x.append(i)
                y.append(j)

现在我们能以散点图的形式画出优化后的点线图：

.. code:: verbatim

    import pylab
    pylab.cla() #clear any prior graph
    pylab.gray()
    pylab.scatter(x,y)
    pylab.xlim(0, len(rec_one)-window)
    pylab.ylim(0, len(rec_two)-window)
    pylab.xlabel("%s (length %i bp)" % (rec_one.id, len(rec_one)))
    pylab.ylabel("%s (length %i bp)" % (rec_two.id, len(rec_two)))
    pylab.title("Dot plot using window size %i\n(allowing no mis-matches)" % window)
    pylab.show()

这将弹出一个新的窗口，包含类似如下图形：

|image29|

我个人认为第二个图更加易读！再次注意，我们在这里 *没有* 检查反向互补匹配 —— 你可以扩展这个例子
来实现它，或许可以以一种颜色显示正向匹配，另一种显示反向匹配。

18.2.4  绘制序列读长数据的质量图
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果你在处理二代测序数据，你可能希望绘制数据的质量图。这里使用两个包含双端读长的FASTQ文件作为例子，
``SRR001666_1.fastq`` 为正向读长， ``SRR001666_2.fastq`` 为反向读长。它们从ENA序列读长档案FTP中下载
（ ```ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR001/SRR001666/SRR001666_1.fastq.gz`` <ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR001/SRR001666/SRR001666_1.fastq.gz>`__
和
```ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR001/SRR001666/SRR001666_2.fastq.gz`` <ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR001/SRR001666/SRR001666_2.fastq.gz>`__ ），
且来自 *E. coli* —— 参见 ```http://www.ebi.ac.uk/ena/data/view/SRR001666`` <http://www.ebi.ac.uk/ena/data/view/SRR001666>`__ 的详细介绍。
在下面的代码中， ``pylab.subplot(...)`` 函数被用来在两个子图中展示正向和反向的质量。这里也有少量的代码来保证仅仅展示前50个读长的质量。

.. code:: verbatim

    import pylab
    from Bio import SeqIO
    for subfigure in [1,2]:
        filename = "SRR001666_%i.fastq" % subfigure
        pylab.subplot(1, 2, subfigure)
        for i,record in enumerate(SeqIO.parse(filename, "fastq")):
            if i >= 50 : break #trick!
            pylab.plot(record.letter_annotations["phred_quality"])
        pylab.ylim(0,45)
        pylab.ylabel("PHRED quality score")
        pylab.xlabel("Position")
    pylab.savefig("SRR001666.png")
    print "Done"

你应该注意到，这里我们使用了 ``Bio.SeqIO`` 的格式名称 ``fastq`` ，因为NCBI使用标准Sanger FASTQ和PHRED分数的存储这些读长。然而，你可能从读长的长度中猜到，这些数据来自
Illumina Genome Analyzer，而且可能最初是以Solexa/Illumina FASTQ两种格式变种中的一种存在。

这个例子使用 ``pylab.savefig(...)`` 函数，而不是``pylab.show(...)`` ，然而就像前面提到的一样，它们两者都非常有用。下面是得到的结果：

|image30|

18.3  处理序列比对
-----------------------------

这部分可以看做是第 \ `6 <#chapter:Bio.AlignIO>`__ 章的继续。

18.3.1  计算摘要信息
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

一旦你有一个比对，你很可能希望找出关于它的一些信息。我们尽力将这些功能分离到
单独的能作用于比对对象的类中，而不是将所有的能生成比对信息的函数都放入比对对象
本身。

准备计算比对对象的摘要信息非常快捷。假设我们已经得到了一个比对对象 ``alignment`` ，
例如使用在第 \ `6 <#chapter:Bio.AlignIO>`__ 章介绍的 ``Bio.AlignIO.read(...)`` 读入。
我们计算对象的摘要信息所要做的所有事情是：

.. code:: verbatim

    from Bio.Align import AlignInfo
    summary_align = AlignInfo.SummaryInfo(alignment)

``summary_align`` 对象非常有用，它将帮你做以下巧妙的事情：

#. 计算一个快速一致序列 – 参见章节 \ `18.3.2 <#sec:consensus>`__
#. 获取一个针对该比对的位点特异性打分矩阵 – 参见章节 \ `18.3.3 <#sec:pssm>`__
#. 计算比对的信息量 – 参见章节 \ `18.3.4 <#sec:getting_info_content>`__
#. 生成该比对中的替换信息 – 章节 \ `18.4 <#sec:sub_matrix>`__ 详细描述了使用该方法生成
一个替换矩阵

18.3.2  计算一个快速一致序列
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在章节 \ `18.3.1 <#sec:summary_info>`__ 中描述的 ``SummaryInfo`` 对象提供了一个可以计算
比对的快速一致序列的功能。假设我们有一个 ``SummaryInfo`` 对象，叫做 ``summary_align``，我们能通过
下面的方法计算一个一致序列：

.. code:: verbatim

    consensus = summary_align.dumb_consensus()

就行名字显示的那样，这是一个非常简单的一致序列计算器，它将只是在一致序列中累加每个位点的所有残基，如果
最普遍的值大于某个阈值时，这个最普遍的残基将被添加到一致序列。如果它没有到达这个阈值，将添加一个“不确定字符”。
最终返回的一致序列对象是一个Seq对象，它的字母表是从组成一致序列所有序列的字母表中推断出来的。所以使用
``print consensus`` 将给出如下信息：

.. code:: verbatim

    consensus Seq('TATACATNAAAGNAGGGGGATGCGGATAAATGGAAAGGCGAAAGAAAGAAAAAAATGAAT
    ...', IUPACAmbiguousDNA())

你可以通过传入可选参数来调整 ``dumb_consensus`` 的工作方式：

 **the threshold**
    这是用来设定某个残基在某个位点出现频率超过哪个阈值，才将其添加到一致序列。默认为0.7（即70%）。
**the ambiguous character**
    作为一致序列中的不确定字符。默认为’N’。
**the consensus alphabet**
    作为一致序列的字母表。如果没有提供，我们将从比对序列的字母表基础上推断该字母表。

18.3.3  位点特异性打分矩阵
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

位点特异性打分矩阵（Position specific score matrices，PSSMs）以另一种和一致序列不同的方式总结比对信息，或许
对不同的工作有用。总的来说，PSSM是一个计数矩阵。对于比对中的每一列，每一个字母都被计数并加和。这些加和值将
和一个放在左轴的代表序列相对一起显示出来。这个序列可能是一致序列，但也可以是比对中的任何序列。例如，对于比对，

.. code:: verbatim

    GTATC
    AT--C
    CTGTC

它的PSSM是：

.. code:: verbatim

          G A T C
        G 1 1 0 1
        T 0 0 3 0
        A 1 1 0 0
        T 0 0 2 0
        C 0 0 0 3

假设我们有一个比对对象叫做 ``c_align`` ，为了获得PSSM和旁边的一致序列，我们首先得到一个摘要对象，并计算一致序列：

.. code:: verbatim

    summary_align = AlignInfo.SummaryInfo(c_align)
    consensus = summary_align.dumb_consensus()

现在，我们想创建PSSM，但是在计算中忽略任何 ``N`` 不确定残基：

.. code:: verbatim

    my_pssm = summary_align.pos_specific_score_matrix(consensus,
                                                      chars_to_ignore = ['N'])

关于此的需要说明的两点是：

#. 为了维持字母表的严格性，你可以只在PSSM的顶部包含在比对对象中出现的字母。空白缺口字符并不包含在PSSM的顶轴中。
#. 传入并显示在左侧轴的序列可以不必为一致序列。例如，你如果想要在轴上显示比对中的第二条序列，你需要做：

   .. code:: verbatim

       second_seq = alignment.get_seq_by_num(1)
       my_pssm = summary_align.pos_specific_score_matrix(second_seq
                                                         chars_to_ignore = ['N'])

上面的命令返回一个 ``PSSM`` 对象。像上面我们展示的一样打印PSSM，我们只需要简单的 ``print my_pssm``，结果如下：

.. code:: verbatim

        A   C   G   T
    T  0.0 0.0 0.0 7.0
    A  7.0 0.0 0.0 0.0
    T  0.0 0.0 0.0 7.0
    A  7.0 0.0 0.0 0.0
    C  0.0 7.0 0.0 0.0
    A  7.0 0.0 0.0 0.0
    T  0.0 0.0 0.0 7.0
    T  1.0 0.0 0.0 6.0
    ...

你可以用 ``your_pssm[sequence_number][residue_count_name]`` 获得任何PSSM的元素。例如，获取上面PSSM中第二个元素的
‘A’残基的计数，你可以：

.. code:: verbatim

    >>> print my_pssm[1]["A"]
    7.0

PSSM类的结构有望使得获取元素和打印漂亮的矩阵都很方便。

18.3.4  信息量
~~~~~~~~~~~~~~~~~~~~~~~~~~~

一个潜在而有用的衡量进化保守性的尺度是序列的信息量。

一个有用的面向于分子生物学家的分子信息论的介绍可以在这里找到：
```http://www.lecb.ncifcrf.gov/~toms/paper/primer/`` <http://www.lecb.ncifcrf.gov/~toms/paper/primer/>`__ 。
对于我们的目地，我们将查看一致序列或其部分的信息量。我们使用下面的公式计算多序列比对中某个特定的列的信息量：

*IC*\ :sub:`*j*` = 

+-------------------+
| *N*\ :sub:`*a*`   |
+-------------------+
| ∑                 |
+-------------------+
| *i*\ =1           |
+-------------------+

 *P*\ :sub:`*ij*` *log*

| ⎛
|  ⎜
|  ⎜
|  ⎝

+--------------------+
| *P*\ :sub:`*ij*`   |
+--------------------+
+--------------------+
| *Q*\ :sub:`*i*`    |
+--------------------+

| ⎞
|  ⎟
|  ⎟
|  ⎠

其中：

-  *IC*\ :sub:`*j*` – 比对中第 *j* 列的信息量。
-  *N*\ :sub:`*a*` – 字母表中字母的个数。
-  *P*\ :sub:`*ij*` – 第 *j* 列的某个特定字母 *i* 的频率（即，如果G在比对列中6次有3次出现，则为0.5）
-  *Q*\ :sub:`*i*` – 字母 *i* 的期望频率。这是一个可选参数，由用户自行决定使用。默认情况下，它被自动赋值为
   0.05 = 1/20，若为蛋白字母表；或0.25 = 1/4 ，若为核酸字母表。这是在没有先验分布假设的情况下计算信息量。而在假设
   先验分布或使用非标准字母表时，你需要提供 *Q*\ :sub:`*i*` 的值。

好了，现在我们知道Biopython计算了什么信息量，让我们看看怎么对部分比对区域计算它。

首先，我们需要使用我们的比对来获得一个比对摘要对象，我们假设它叫做 ``summary_align`` （参见章节
 \ `18.3.1 <#sec:summary_info>`__ 以获取怎样得到它的操作说明）。一旦我们得到这个对象，计算某个区域的信息量就像下面一样简单：

.. code:: verbatim

    info_content = summary_align.information_content(5, 30,
                                                     chars_to_ignore = ['N'])

哇哦，这比上面的公式看起来要简单多了！变量 ``info_content`` 现在含有一个浮点数来表示指定区域（比对中的5到30）的信息量。
我们在计算信息量时特意忽略了不确定残基’N’，因为这个值没有包括在我们的字母表中（因而我们不必要关心它！）。

像上面提到的一样，我们同样能通过提供期望频率计算相对信息量：

.. code:: verbatim

    expect_freq = {
        'A' : .3,
        'G' : .2,
        'T' : .3,
        'C' : .2}

期望值不能以原始的字典传入，而需要作为 ``SubsMat.FreqTable`` 对象传入（参见章节
 \ `20.2.2 <#sec:freq_table>`__ 以获得关于FreqTables的更多信息）。FreqTable对象
 提供了一个关联字典和字母表的标准，和Biopython中Seq类的工作方式类似。

要从频率字典创建一个FreqTable对象，你只需要：

.. code:: verbatim

    from Bio.Alphabet import IUPAC
    from Bio.SubsMat import FreqTable

    e_freq_table = FreqTable.FreqTable(expect_freq, FreqTable.FREQ,
                                       IUPAC.unambiguous_dna)

现在我们得到了它，计算我们比对区域的相对信息量就像下面一样简单：

.. code:: verbatim

    info_content = summary_align.information_content(5, 30,
                                                     e_freq_table = e_freq_table,
                                                     chars_to_ignore = ['N'])

现在，``info_content`` 将包含与期望频率相关的该区域的相对信息量。

返回值是按上面的公式以2为对数底计算的。你可以通过传入 ``log_base`` 参数来改变成你想
要的底数：

.. code:: verbatim

    info_content = summary_align.information_content(5, 30, log_base = 10,
                                                     chars_to_ignore = ['N'])

好了，现在你已经知道怎么计算信息量了。如果你想要在实际的生命科学问题中应用它，最好找一些关于
信息量的文献钻研，以了解它是怎样用的。希望你的钻研不会出现在编码这个函数时所犯的错误。

18.4  替换矩阵
---------------------------

替换矩阵是每天的生物信息学工作中的极端重要的一部分。它们提供决定两个不同的
残基有多少相互替换的可能性的得分规则。这在序列比较中必不可少。Durbin等的
“Biological Biological Sequence Analysis” 一书中提供了对替换矩阵以及它们的用法的
非常好的介绍。一些非常有名的替换矩阵是PAM和BLOSUM系列矩阵。

Biopython提供了大量的常见替换矩阵，也提供了创建你自己的替换矩阵的功能。

18.4.1  使用常见替换矩阵
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

18.4.2  从序列比对创建你自己的替换矩阵
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

使用替换矩阵类能轻易做出的一个非常酷的事情，是从序列比对创建出你自己的替换矩阵。
实际中，通常是使用蛋白比对来做。在这个例子中，我们将首先得到一个Biopython比对对象，
然后得到一个摘要对象来计算关于这个比对的相关信息。文件 `protein.aln <examples/protein.aln>`__ 
（也可在 `这里 <http://biopython.org/DIST/docs/tutorial/examples/protein.aln>`__ 获取）包含
Clustalw比对输出。

.. code:: verbatim

    >>> from Bio import AlignIO
    >>> from Bio import Alphabet
    >>> from Bio.Alphabet import IUPAC
    >>> from Bio.Align import AlignInfo
    >>> filename = "protein.aln"
    >>> alpha = Alphabet.Gapped(IUPAC.protein)
    >>> c_align = AlignIO.read(filename, "clustal", alphabet=alpha)
    >>> summary_align = AlignInfo.SummaryInfo(c_align)

章节 \ `6.4.1 <#sec:align_clustal>`__
和 \ `18.3.1 <#sec:summary_info>`__ 包含关于此类做法的更多信息。

现在我们得到了我们的 ``summary_align`` 对象，我们想使用它来找出不同的残基相互替换的次数。
为了使例子更已读，我们将只关注那些有极性电荷侧链的氨基酸。幸运的是，这能在生成一个替代字典
时很轻松的实现，通过传入所有需要被忽略的字符。这样我们将能创建一个只包含带电荷的极性氨基酸的
替代字典：

.. code:: verbatim

    >>> replace_info = summary_align.replacement_dictionary(["G", "A", "V", "L", "I",
    ...                                                      "M", "P", "F", "W", "S",
    ...                                                      "T", "N", "Q", "Y", "C"])

这个关于氨基酸替代的信息以python字典的形式展示出来将会像如下的样子（顺序可能有所差异）：

.. code:: verbatim

    {('R', 'R'): 2079.0, ('R', 'H'): 17.0, ('R', 'K'): 103.0, ('R', 'E'): 2.0,
    ('R', 'D'): 2.0, ('H', 'R'): 0, ('D', 'H'): 15.0, ('K', 'K'): 3218.0,
    ('K', 'H'): 24.0, ('H', 'K'): 8.0, ('E', 'H'): 15.0, ('H', 'H'): 1235.0,
    ('H', 'E'): 18.0, ('H', 'D'): 0, ('K', 'D'): 0, ('K', 'E'): 9.0,
    ('D', 'R'): 48.0, ('E', 'R'): 2.0, ('D', 'K'): 1.0, ('E', 'K'): 45.0,
    ('K', 'R'): 130.0, ('E', 'D'): 241.0, ('E', 'E'): 3305.0,
    ('D', 'E'): 270.0, ('D', 'D'): 2360.0}

这个信息提供了我们所接收的替换次数，或者说我们期望的不同的事情相互替换有多频繁。事实证明，
令人惊奇的是，这就是我们继续创建替代矩阵所需要的全部信息。首先，我们使用替代字典信息创建
一个“接受替换矩阵”（Accepted Replacement Matrix，ARM）：

.. code:: verbatim

    >>> from Bio import SubsMat
    >>> my_arm = SubsMat.SeqMat(replace_info)

使用这个“接受替换矩阵”，我们能继续创建我们的对数奇数矩阵（即一个标准类型的替换举证）：

.. code:: verbatim

    >>> my_lom = SubsMat.make_log_odds_matrix(my_arm)

你创建的这个对数奇数矩阵可以用以下参数进行自定义：

-  ``exp_freq_table`` – 你可以传入一个每个字母的期望频率的表格。如果提供，在计算期望替换时，
   这将替代传入的“接收替换矩阵”。
-  ``logbase`` - 用来创建对数奇数矩阵的对数底数。默认为10。
-  ``factor`` - 用来乘以每个矩阵元素的因数。默认为10，这样通常可以使得矩阵的数据容易处理。
-  ``round_digit`` - 矩阵中四舍五入所取的小数位数，默认为0（即没有小数）。

一旦你获得了你的对数奇数矩阵，你可以使用函数 ``print_mat`` 很漂亮的显示出来。使用我们创建的
矩阵可以得到：

.. code:: verbatim

    >>> my_lom.print_mat()
    D   2
    E  -1   1
    H  -5  -4   3
    K -10  -5  -4   1
    R  -4  -8  -4  -2   2
       D   E   H   K   R

很好。我们现在得到了自己的替换矩阵！

18.5  BioSQL – 存储序列到关系数据库中
---------------------------------------------------------

`BioSQL <http://www.biosql.org/>`__ 是 `OBF <http://open-bio.org/>`__ 多个项目
（BioPerl、 BioJava等）为了支持共享的存储序列数据的数据库架构而共同努力的结果。
理论上，你可以用BioPerl加载GenBank文件到数据库中，然后用Biopython从数据库中提取
出来为一个包含Feature的Record对象 —— 并获得或多或少和直接用 ``Bio.SeqIO`` 
（第 `5 <#chapter:Bio.SeqIO>`__ 章）加载GenBank文件为SeqRecord相同的东西。

Biopython中BioSQL模块的文档目前放在
```http://biopython.org/wiki/BioSQL`` <http://biopython.org/wiki/BioSQL>`__
，是我们维基页面的一部分。
