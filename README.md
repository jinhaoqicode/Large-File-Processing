# Large-File-Processing
## 问题：
有一个 100GB 的文件，里面内容是文本，
要求：

- 找出第一个不重复的词
- 只允许扫一遍原文件
- 尽量少的 IO
- 内存限制 16G
- 英文单词，每行一个字符串 （长度范围从 0-100）。


## 思路：
1. 100G字符串，0-100字节随机，最后换行占两个字节
2. 每行是一个byte数组，长度1-100不等（不加上换行符），一个字节8位，所以共有2^800种组合
3. 所以整个文件不可能直接存到内存中，最坏情况，100G中，每个字符串都不同，第一个字符串就是要寻找的目标字符串
4. 100G 数据最坏情况下有多少行？ 决定用 int 还是 long 表示字符串出现频率和字符串第一次出现的位置，假设每行一个字符，行数100 * 1024 * 1024 * 1024 / 3 >2^31-1，超出int范围，若都是相同字符串，则字符串频率也会超出，java中 long最大为2^63 -1 > 100 * 1024 * 1024 *1024, 所以用long统计足够。
5. 按照最坏情况，设要把大文件拆分成 x 份，每份文件中要记录每行的字符串内容以及在源文件中第一次出现的位置，需要一个long数据转化成字符串 根据最大值 1024 * 1024 * 1024 * 00/3=35791394133.33333， 需要11个字节，再加一个分隔符占一个字节，总共需要12个字节，此文件读到内存中要把原来的String类型的统计字符串位置索引的内容转化成long，每行最多扩大8（long字节数）-1（字符串位置索引最小字节）=7个字节，（其实扩大不了这么多，因为有的是从12个字节减小到8个字节）
6. 源文件切割份数计算方式如下图 
![image](https://img.alicdn.com/imgextra/i1/1860006657/O1CN01uuJcNJ1z2x9ajtLgd_!!1860006657.png)
7. 接下来就是如何切割使得尽量均匀达到我们设置的内存，最重要的一点是相同字符串要在同一个文件中，这是保证分布运算的关键，所以就要用到Hash函数，相同字符串的hash函数值是相同的（我用的java自带的计算String的hashcode）。但是由于2^800 是一个很大的种类数，还是存在极特殊情况使得小文件分布不均匀，遍历文件对一次分割文件变大的文件按照所占内存大小重新分割。
8. 寻找被切割后的尽量少的文件数是为了尽量减少IO
9. 切割完成后的对每个文件进行处理的算法就比较简单了，读文件把其存到内存中，统计每个字符串其出现频率和第一次出现的位置。每个文件保存一个结果，即频率为1且最早出现的字符串信息，以后遍历的每个文件中若有频率为1且更早索引位置的，将原有结果替换。若文本中无结果，返回字符串"全文无非重复字符串"
10. 维护一张hashmap在读取的时候统计词频，在内存范围内，若有词超过两个，就不读入小文件，控制哈希表在14G范围内，多了就不增加写入小文件。
11. 维护一张bitmap，对每个字符串构建hash函数，14G*8=112G的数值范围已经确保bitmap足够大，100G字符串平均长度50，只有2G的种类数，112G种对比2G种，不同字符串hash冲突的概率极小，极大概率保证字符串hash值不冲突，【然后从尾到头读文件】，字符串计算hash值，查bitmap表，若为0，则置为1，加入候选解，若为1，则删除候选解。再为0变1，则替换候选解。这种效率很高，查找解的速度就是磁盘读取速度。（但是有错误概率，因为没有维护候选解，从头到尾只有一个候选解字符串，另外就是只能从大概率上保证不同字符串的hash值不冲突）
12. 终极方案，11和12同时进行，同时维护一张bitmap和一张hashmap，hashmap可以作为bitmap的候选解hash表。

## 可改进的地方
1. 算法可以优化查询速度，维护一颗树或者堆
2. 读写内容时，buffer内存效率值也可以改进，目前根据经验设置为1M
3. 考虑双线程进行读写操作，一边读一遍处理数据，这个提升了改进buffer的空间，也能提升整个的查询效率。

## 使用和运行
新建Project将3个java文件拷入即可，记得修改首行包名

主程序函数入口：FindFisrtX.main

创建文本测试用例main函数入口：FileIO.main

