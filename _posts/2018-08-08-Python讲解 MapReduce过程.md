---
layout: post
title:  "Python讲解MapReduce过程"
categories: 大数据
tags:  hadoop MapReduce
author: fantastic_liar
---
* content
{:toc}
### 用Python讲解MapReduce

使用python写map.py和reduce.py两个脚本，详细讲解mapreduce整个流程。（本地运行、hadoop集群上利用stream-jar运行）





map.py代码
```    
import sys

for line in sys.stdin:
    word_list=line.strip().split(" ")
    for word in word_list:
        print(word+"\t1")
```

reduce.py代码
```
import sys

current_word=None
sum=0

for line in sys.stdin:
    tmp=line.split("\t")
    if len(tmp)<2:
        continue
    key=tmp[0]
    value=tmp[1]
    if current_word==None:
        current_word=key
    if current_word!=key:
        print(current_word+"\t"+str(sum))
        sum=0
        current_word=key
    sum+=int(value)
```

准备一个简单的单词库hello.txt
```
hello world
fantastic liar love
ubuntu list centos deepin
```
本地模拟一个MapReduce

`cat hello.txt | python map.py | sort -k 1 | python reduce.py`

整个的流程是读取hello.txt 然后针对每一行运行map.py代码进行map操作，然后对收集到的结果进行排序，最后进行reduce阶段，整合单词出现的次数

实际的hadoop的运行流程与以上类似，不过更加复杂：
1. 首先读取的文件是存储在HDFS上的，每个数据块大小默认64M，因此可能存在数据在存储时被切分错误。所以首先需要对读取的文件进行split和record操作。split默认按"\n"进行划分，record保证每行数据是完整的，该操作由框架完成。
2. 针对每个Record进行map操作，得到<k,v> pairs。
3. 根据reduce的个数进行partion操作，进行分区，partition规则可以自定义（按照地理位置、按照手机归属地、按照Hash值对reduce个数取模...）
4. 如果数据超出了内存容量的80%，则进行spill操作，将数据写到磁盘。
5. 将数据分发到对应的reduce服务器上，大数据量会对IO性能有影响，因此这里可以进行压缩，在reduce服务器端进行解压缩操作，能够减小传输的数据量，提升IO性能。
6. reduce根据得到的数据进行reduce操作，并将结果进行存储。

整个过程采用hadoop-streaming.jar进行任务提交，可以通过各种参数进行制定，这里只写一个最简单的版本：
```
hadoop jar hadoop-streaming.jar \
-D mapred.job.name=python_mapred_job \
-input /input/hello.txt \
-output /output \
-mapper map.py \
-reducer reduce.py \
-file map.py \
-file reduce.py
```
-file 是为了将map.py和reduce.py代码分发到各个服务器上进行执行

**********************************************************************
粗浅的理解，如果中间有什么理解错误的话，请多多指教！！！