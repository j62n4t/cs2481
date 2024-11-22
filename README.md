java c
CS 4321/5321 Project 2
This project counts for 24% of your grade, of which 6% is for the checkpoint and 18% for the main submission.
1    Goals and important points
In Project 1, you developed a lot of functionality. However, the priority was end-to-end evaluation of SQL rather than efficiency. So, you used line-at-a-time I/O. You also used na¨ıve implementations for join
(TNLJ, i.e., tuple-nested loop join) and for sort (in-memory sort). Your sort implementation was
particularly problematic because it kept unbounded state – the amount of memory required by the operator depended on the size of the input rather than being bounded by a constant.
Now, you will address the above shortcomings:
• you will move from using line-at-a-time I/O to faster page-at-a-time I/O
• you will refactor your code to support different physical implementations for each relational algebra operator (e.g. different join implementations)
• you will implement Block Nested Loop Join (BNLJ), External Sort and Sort Merge Join (SMJ)
• you will ensure that you have at least one implementation for each operator that does not keep unbounded state
• you will do some performance benchmarking of your join implementations against each other
You will still support the same subset of SQL as for Project 1, and you should still construct the join tree in the same manner as in Project 1, following the query’s FROM clause.
This is a challenging Project, including significant refactoring and nontrivial algorithms to implement. You have a substantial time window to do it, but this window also includes the 4320 prelim and Spring Break.   Thus you need to budget your time wisely.
To ensure you get started early, we are requiring a checkpoint submission. The checkpoint requirements are described in Section 5. Note that completing the checkpoint does not mean you have completed half the
project - there is much more work to do after the checkpoint.
2    Input and output formats
2.1    Top-level inputs and outputs to your project
As in Project 1, we will run your code from the command line by exporting a runnable JAR and typing: java  -jar  db practicum team name 2.jar  inputdir  outputdir  tempdir
This means your top-level class has to accept inputdir, outputdir and tempdir as command-line arguments and handle them appropriately. inputdir is the directory where relevant inputs to your program will be found.  This directory will contain:
•  a queries .sql file as in Project 1.
•  a db subdirectory. This will contain a data subdirectory and and a schema .txt file. The data
directory contains files for the relations, with one file per relation as in Project 1, except that the files are now in a binary format described in Section 2.2. The schema .txt file is as in Project 1.
•  a file plan builder   config .txt which is the configuration file for the PhysicalPlanBuilder as described in Section 2.3.
outputdir is the directory where your program should write output. You may assume this directory will exist. As in Project 1, after running your code, the answer to the ith query (starting at 1) should be found in file outputdir/queryi.
tempdir is the temporary directory where your external sort operators should write their “scratch” files, a process described more fully in Section 3.4. Again, you may assume this directory exists.
2.2    Binary File Format
This describes the file format for input and output data in Project 2.
The motivation for moving to a new file format is that doing line-at-a-time file I/O is inefficient. See
http://www.idryman.org/blog/2013/09/28/java-fast-io-using-java-nio-api/ for an interesting
performance comparison of various ways to do I/O in Java. In this Project, you will use Java NIO to do
page-at-a-time I/O. A “page” for our purposes is a buffer, specifically a ByteBuffer. We will read the file in fixed-sized “chunks” . This means we need a file format where we can read data in fixed-sized chunks
rather than in variable-sized lines.  (Lines are variable-sized in that each relation potentially has tuples of a different size/length.)
Your textbook in Chapters 9.5-9.7 contains an extensive discussion of file formats for databases. In this project, we can work with a fairly simple format as we are not supporting updates.
Every relation, whether it’s a table in the database or a query answer, is stored in a single data file. Each   data file is a sequence of pages.  Every page is 4096 bytes in size. Thus, the first 4096 bytes of the file are    the first page, the next 4096 bytes are the second page, and so on. This also means that every data file is a multiple of 4096 bytes in size - there are no “half pages.”
Every page contains two pieces of metadata. The first is the number of attributes of the tuples stored on  the page - we assume all tuples stored on the page have the same number of attributes.  The second piece of metadata is the number of tuples on the page; this is particularly useful information in case the page is not full. The two pieces of metadata, as well as the tuples themselves, are stored as (four-byte) integers.    You may assume all integer values will fall into the int domain (32 bits). You may also assume that all
tuples are sufficiently small that you can fit at least one tuple per page.
The overall layout of metadata and data on a page is best illustrated by example.  Suppose we have a
relation which contains just two four-attribute tuples:  (10, 20, 30, 40) and (50, 60, 70, 80). Then it fits in a single page, set out as in the image below (each “cell” represents a four-byte integer). Any remaining
empty space at the end of any page should be filled with zeroes.There are no “half-tuples” possible on a page. If the last tuple does not fit on the page (e.g. the tuple has  3 attributes so we need 12 bytes but the page only has 10 bytes left) it will be placed on the next page. If a page does not have space for any more tuples, it is full.  Any relation that requires multiple pages is stored  so that all pages except possibly the last one are full.Your program must accept data in the binary format and produce outputs (answers) in the binary format. In the sample input and output we have provided, we give you each file in both binary   format and in human-readable Project-2-style. format as a courtesy to aid with debugging, but we will be testing with binary format files only.
When working with binary files, be aware that you can open, view and manipulate such files in programs called hex editors (or binary editors). This could be handy for debugging.
2.3    Format of the configuration file for PhysicalPlanBuilder
This section explains the format for the configuration file for your PhysicalPlanBuilder. If this is your
first time reading the document, we recommend skipping it and returning to it once you have read Section  3 (and actually understand what the PhysicalPlanBuilder is!!). We have placed this section here because once you are familiar with the project, it will be easier to have all format related info in one place.
You may assume for this project that the PhysicalPlanBuilder will use the same join implementation for every logical join operator in the logical plan, and the same sort implementation for every logical sort
operator in the logical plan.  That is, you do not need the ability to generate physical plans that have a mix of TNLJ, SMJ and BNLJ physical operators, or a mix of BNLJ operators each with a different number of   buffer pages, etc.
The configuration file has two lines; the first specfies the join method, the second the sort method. The
join method is either 0 for TNLJ, 1 for BNLJ, or 2 for SMJ. If the join method is BNLJ, there is a second integer on the same line specifying the number of buffer pages to be used. The sort method is either 0 for in-memory sort, or 1 for external sort. If the sort method is external sort, there is a second integer on the same line specifying the number of buffer pages to be used in the sort. This will always be at least 3.
For example the file: 0
0
Specifies that you want TNLJ and in-memory sort, i.e. the Project 1 na¨ıve implementation. The file 1  5
1  4
Specifies you want BNLJ with 5-pages outer relation buffers, and external sort with 4-page sort buffers.
3    Implementation instructions
First, some general remarks to consider as you begin the implementation.
You will begin with substantial refactoring of your Project 1 code; save snapshots of your code at
intermediate stages in case you mess up (you can do this by creating different branches using git). After each refactoring, be sure to test your code to verify you haven’t broken any “old” functionality.
In this Project, your codebase will expand considerably. Do not skimp on comments and on setting up a   good debugging infrastructure as you go; it will be particularly important to stay on top of your codebase and communicate with your partner(s) about APIs.
You will be working with bigger relations than in Project 1, so if you are relying on console output for
debugging, you may want to move to a setup where your debugging statements get logged to a file instead. This avoids the problem where you have so many things getting output to the console that the first lines
get truncated. One simple way to do file-based logging is to create a Logger class that uses the singleton pattern, that way every component in your code can call the logger as needed.
You should consider writing yourself a random data generator to test your code. This doesn’t have to befancy; it can create tuples by generating each attribute as a random integer in a specified range. The range
you allow for attributes will dictate how many tuples “match” in joins and selections, so you should
probably have that as a configurable setting in your data generator. For example, if you generate two
relations with 5000 tuples each, and each attribute has values in the range 0 to 100, probably a lot of
tuples will match if you try to join these two relations. If you generate the attribute values in the range from 0 to 10000, far fewer tuples will match.
Finally, be aware that once you are done with the two refactoring steps (Sections 3.1 and 3.2), the
remaining three tasks can be completed in parallel if you are looking to split up work between partners.
BNLJ and external sort can definitely be developed independently; SMJ requires a sorting implementation,
but you can use your in-memory sort from Project 1 until your external sort is ready. Also, completing
Sections 3.1 and 3.2 constitutes the requirements for the project checkpoint. All of these are great reasons to complete this portion of the implementation ASAP.
3.1    Refactor to use the new file format
Your first task is to refactor your Project 1 code to use the new binary file format for input and output. You are required to use Java NIO for this. If you are unfamiliar with Java NIO, start by reading this
tutorial: http://www.ibm.com/developerworks/java/tutorials/j-nio/j-nio .html and looking at the documentation. We recommend using ByteBuffers. They provide handy getInt and putInt methods
that relieve you of the need to write code for serializing an integer into bytes.
We recommend doing this refactoring by setting up a layer of abstraction so that most of your code does not need to know about file I/O, or even what file format is being used. For example, you can create
TupleReader and TupleWriter interfaces. TupleReader has a method to read the next tuple (and
probably some bookkeeping methods such as close, reset etc). TupleWriter has a method to write a tuple. Then the rest of your code can just create TupleReaders/TupleWriters as needed; for example, your file
scan operator can create a TupleReader and grab tuples from that instead of from the file directly. The
logic for getting tuples from a page or writing them to a page is then encapsulated in concrete implem代 写CS 4321/5321 Project 2Java
代做程序编程语言entations of TupleReader and TupleWriter.
For the new file format, the TupleReader you implement will read the file one “page” at a time – i.e. it
will have a running ByteBuffer which it will fill using read calls to an appropriate FileChannel.  Then it can extract specific tuples from the page as needed when someone requests the next tuple. The writer’s
behavior. will be symmetric - it can buffer the tuples in memory until it has a full page and then flush (write) the page.
It is a good idea to implement TupleReaders and TupleWriters for both the new binary format and the
Project 1 human-readable format. That also allows you to write some helpful utilities for debugging, such as a converter between the two formats.
Be aware of a couple of quirks of Java NIO we discovered last semester:
First, some students reported strange behavior. when using the relative get/put methods for ByteBuffer, i.e. getInt() and putInt(int  value). If you encounter issues, try the absolute methods getInt(int
index) and putInt(int  index,  int  value), which explicitly specify the desired location in the buffer.
Second, be aware that Buffer.clear() does not do what you might think it does.  The officialdocumentation at http://docs.oracle.com/javase/8/docs/api/java/nio/Buffer.html#clear-- , tells us the surprising fact that:  “This method does not actually erase the data in the buffer, but it is named as if it did because it will most often be used in situations in which that might as well be the case.”
The above affects you because it could mess up your padding with zeroes at the end of each page.
Specifically, if you reuse the same ByteBuffer for each page when writing, even if you clear() it between  pages, this does not refill the buffer with zeroes. If your second-to-last page is full and your last page is not full, and you only do a clear() between pages, the “empty” final portion of the last page you write will
actually contain some “garbage” values that are left over from the second-to-last page. The moral of the story is that you have to do something more than clear() to ensure correct padding with zeroes.
Once you have sorted out your I/O, this is a great time to write a random data generator.
You may also want to write a sorting utility that takes in a file (in either format), sorts the tuples in
memory using Collections .sort and writes out a sorted file. This will be very handy once you start
implementing different join algorithms - when run on the same data, they may output tuples in different ordrers. If you have a sorting utility, you can sort the outputs afterwards and compare the files to make  sure your SMJ is actually outputting the same results as your BNLJ.
Finally, this is as good a time as any to introduce timing functionality into your code. When your top-level class has a query plan and is ready to call dump() to write the results, call System.currentTimeMillis()  before and after the dump() to get a measure of the elapsed time. You will need this functionality for the    performance benchmarking (Section 3.6) and it is handy to implement it now. Generate yourself some
relations with, say, 5000 or 10000 tuples and see how long it takes to run your queries, as a ballpark figure.
3.2    Refactor to use both logical and physical query plans
Your Project 1 code generated the query plan directly from the Statement objects that JSqlParser
produced. You will now refactor this into a two-stage process: the generation of a logical  query plan from    the Statement, and the generation of a physical  query plan from the logical query plan. The logical query  plan will just be a relational algebra tree; the physical query plan will contain concrete implementations for each operator, such as SMJ or BNLJ.
Thus, your overall workflow in your top-level class will be:
• read a query from a file and parse it using JSqlParser
•  convert the Statement you obtained into a logical query plan
•  convert the logical query plan into a physical query plan
• evaluate, i.e., call dump() on the physical query plan, including timing the evaluation
This refactoring is necessary to separate two different concerns:
• building a relational algebra tree that represents the query at a mathematical level, and
• translating the relational algebra tree into some code that will actually run and spit out tuples.
This separation makes it easier to add functionality that pertains to only one of these - for example, in
Project 5 you will be optimizing the logical query plan.  If you are pushing selections/projections past joins, a lot of that can and should be done before you choose a concrete join implementation.
This separation also allows you flexibility in modifying or enhancing the logical plan as you build the
physical plan. Notably, if you want to use SMJ, you can insert sort operators on both inputs to the join while constructing the physical query plan. That way the join operator itself only has to perform. the
merge. Of course if you are not  using SMJ, it doesn’t make sense to insert these extra sort operators, so you want to defer the decision until you know which join implementation you are using.
To refactor, start by creating separate packages with classes for logical and physical operators. Your old
Project 1 operators will basically become your physical operators, and you will add a new package with
logical operators. A logical operator does not need to store a lot of info; depending on the type of operator, it may need to know things such as the join condition, selection condition, sort order, etc. Basically, think  about writing your query in relational algebra by hand as you have done in 4320 homeworks; if theinformation appears in your relational algebra translation, it probably needs to be in the logical operators. On the other hand, the logical operators do not need getNextTuple() or reset() methods since they will never actually “run” . That logic definitely belongs in physical operators.
Once you have both physical and logical operators, you need code to translate a logical plan to a physical plan. Fortunately, you have extensive experience with the visitor pattern, so this should not be difficult
conceptually. Write a visitor class that recursively walks your logical plan and builds up a corresponding
physical plan. In the remainder of these instructions we will call this visitor the PhysicalPlanBuilder,
but you may name it what you like.  For now the PhysicalPlanBuilder code will not be very  “clever”, but this will soon change. For example, once you have a BNLJ implementation available, you can set your
PhysicalPlanBuilder to convert the logical join operator into either a tuple-nested loop join or BLNJ
physical operator. Once you have a SMJ implementation, your PhysicalPlanBuilder will actually insert physical sort operators as well as a physical SMJ operator.
3.3    Implement BNLJAfter substantial refactoring, you are now ready to implement a new join algorithm:  block-nested loop join.
3.3.1    Implement the BNLJ physical operator
You need to implement a new physical operator to compute joins using the BNLJ algorithm. This is
described at a high level in your textbook, pp. 455-456. The basic idea is to read the outer relation one block at a time into a buffer, and execute the following logic:
procedure BNLJ(outer R, inner S) for each block B of R do
for each tuple s in S do
for each tuple r in B do
if r and s satisfy join condition then
add new tuple formed from r and s to result
Of course this is the logic to compute the entire result, whereas in the iterator model you need to output   one tuple at a time. You will therefore have to restructure the above triple-nested-loop structure for the    getNextTuple() method, and your operator will need to keep some state between invocations so it knows where to resume. You already had to something like this for the tuple nested loop join, but it will be a
little more involved here due to three nested loops.
You will notice the description in your textbook talks about building a hashtable for each block of the outer. This is a good refinement for equijoins, but requires careful handling if the join condition is not
equality. Yes, most real-world joins are equijoins, but your BNLJ algorithm should support all join
conditions as specified in the Project 1 description. Thus if you choose to implement a hashtable, you need to do something to handle non-equality join conditions as well. Alternately you may omit the hashtableand iterate over the entire buffer for each tuple of the inner relation as suggested by the above pseudocode.As regards implementing the buffer, we are going to “cheat” a bit. Our buffer pages will not directly  correspond to the file pages from Section 2.2. This is because we are not building a full-fledged buffer manager, so there is no reason to have uniform. treatment for the buffers in your
TupleReader/TupleWriter and for the buffers in the BNLJ. The buffer in your BNLJ can just be a data structure that contains Tuples.
However, the size of this buffer should be configurable and this should be something that can be passed in into the operator’s constructor. To maintain some residual consistency across the implementation, we
require the ability to set the size of the buffer in pages, where a page is 4096 bytes. Your BNLJ constructor can calculate how many tuples of the outer relation will fit in a page, assuming each tuple has size 4* the    number of attributes it contains.  (Note the discrepancy with respect to the page format from Section 2.2
where eight bytes are used up for metadata.) The BNLJ operator can then internally use a Tuple buffer with the appropriate maximum size in tuples.
Note that your textbook talks about reserving two of the pages for input and output of the inner relation; we do not need to worry about those here because these two buffers are maintained in the TupleReader
and TupleWriter. Thus, the buffer size parameter for your BNLJ should just be the number of “pages” to devote to each block of the outer relation.
To read a block of the outer relation, the BNLJ operator should repeatedly call getNextTuple() on theouter child until the buffer is filled. You may wonder if this means that we are regressing to tuple-at-a-time
I/O. However, note that if the outer child is a base table and you are using a page-at-a-time TupleReader, you are actually doing page-at-a-time file I/O even though it “looks like” the BNLJ is pulling the tuples
one at a time. Of course if the outer child is not a base table, but another operator, the BNLJ needs to pull tuples one at a time anyway.
3.3.2    Integrate your BNLJ operator with the rest of your code
Your PhysicalPlanBuilder should set the number of pages to be used for the BNLJ buffer when it creates a BNLJ physical operator. This means you need a way to specify:
• which physical implementation should be used for the logical join operator (TNLJ or BNLJ)
•  if the physical implementation desired is BNLJ, how many buffer pages to use.
You should specify the join algorithm desired and the number of buffer pages in a  configuration file. When your PhysicalPlanBuilder is constructed, it should read this configuration file and set appropriate fields  internally so it can do the desired thing during plan construction. The format for the configuration file is    described in Section 2.3.Note that the above implies you should hang on to your Project 1 TNLJ implementation. You will be  comparing your other join implementations against it, both to ascertain correctness and to benchmark running times as explained in Section 3.6.In fact, this is a good time to run some queries and compare the execution times with BNLJ versus TNLJ, for various buffer sizes in the BNLJ. The results of the queries should obviously be the same, although they may be ordered differently.



         
加QQ：99515681  WX：codinghelp  Email: 99515681@qq.com
