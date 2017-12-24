---
title: Hadoop FileInputFormat相关
date: 2017-01-02 19:22:28
tags: hadoop
categories:
- hadoop
---


#版本:
hadoop1. mapred.xxx  : 使用JobConf
hadoop2. mapreduce.lib.xxx   :  使用taskAttemptContext

#FileInputFormat

FileInputFormat:
>直接通过文件大小\split大小计算偏移量,生成InputSplit(FileSplit).
然后把InputSplit传给RecordReader.

NLineInputFormat: 
> 将多行作为一个Split.
具体多少行由job参数控制:
`mapreduce.input.lineinputformat.linespermap`
新老api都一样.
把文件传给LineReader,找出N行的偏移量,生成InputSplit,
再把InputSplit传给RecordReader,RecordReader再传给LineReader.(二次解析)

CombineFileInputFormat:
> Map的时候临时合并小文件.减少Split的数量(减少Map的数量)
从多个Block(文件)获取数据,生成InputSplit. 相关参数:
```
public static final String SPLIT_MINSIZE_PERNODE = 
    "mapreduce.input.fileinputformat.split.minsize.per.node";
  public static final String SPLIT_MINSIZE_PERRACK = 
    "mapreduce.input.fileinputformat.split.minsize.per.rack";
```


上述的都是Key为偏移量,Value为具体内容的.还有一个不是这样的:
KeyValueTextInputFormat:
> 生成Split的时候按大小;
分割Recode的时候按换行.
特殊之处在于KeyValueLineRecordReader.
默认的K,V分隔符为制表符`\t`.若一行中没有的话,就Key为整行,Value为空字符串"". 
控制分隔符的参数:
`mapreduce.input.keyvaluelinerecordreader.key.value.separator`

#RecordReader
##LineRecordReader
```
比较健全,支持不同字符集(因为直接接受的是byte数组,而且对utf-8有bom格式有处理);
支持压缩.

由于inputSplit是按大小分割的,可能出现某一行跨split的情况.
源码中的处理逻辑是将跨行line的算作前一个split.
实现上: (建议直接看源码)
注意只能跨split,不能跨文件. 
因为同一个文件的不同split,可以用同一个流读,因此是可以实现的.
1. 除了第一个split,每个split丢弃第一行.
        (因为默认已经被前一个split消费了)
2. 每个split处理时,
while(没到split末尾){
  读一行;// 这个过程中可能用流读到了下一个split.
}

```

##XMLRecordReader
原理仿照LineRecordReader,但减免了压缩和字符集的处理.
```
(源码的逻辑更为严密,这里尽量简单叙述)
定义startTag和endTag. 
返回startTag->endTag之间的数据. 
丢弃endTag->startTag之间的数据.
若没有endTag,返回startTag到文件尾的数据.

仿LineRecordReader,将跨split的record划分到前一个split中.
实现:
while(没到文件末尾){
  找startTag: 沿途遇到的byte=>不存;
}
1. 找不到或找到的startTag完全在下一个split,返回false. (实际代码有优化)
2. 存一下startTag,继续:
while(没到文件末尾){
  找endTag: 沿途遇到的byte=>存;
}
1. 找不到endTag,返回到文件尾的数据.
2. 找到了endTag,返回到此为止的数据.

```
源码如下:
```
package com.fenbi.ape.hive.serde.fileformat;

import org.apache.hadoop.fs.FSDataInputStream;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.DataOutputBuffer;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapred.*;
import org.apache.hadoop.mapred.FileSplit;
import org.apache.hadoop.mapred.TextInputFormat;
import org.apache.hadoop.mapreduce.lib.input.*;

import java.io.IOException;

/**
 * Created by xiaoyue26 on 17/9/14.
 * <p>
 * 不接受压缩. 要修改为接受压缩的话,需要学LineRecordReader再修改.
 * 只支持utf-8
 */
public class XmlInputFormat extends TextInputFormat {

    public static final String START_TAG_KEY = "xmlinput.start";
    public static final String END_TAG_KEY = "xmlinput.end";

    @Override
    public RecordReader<LongWritable, Text> getRecordReader(InputSplit inputSplit,
                                                            JobConf jobConf,
                                                            Reporter reporter) throws IOException {
        //new org.apache.hadoop.mapreduce.lib.input.LineRecordReader();
        return new XmlRecordReader((FileSplit) inputSplit, jobConf);
    }

    /**
     * XMLRecordReader class to read through a given xml document to output xml
     * blocks as records as specified by the start tag and end tag
     */
    public static class XmlRecordReader implements
            RecordReader<LongWritable, Text> {
        private final byte[] startTag;
        private final byte[] endTag;
        private final long start;
        private final long end;
        private final FSDataInputStream fsin;
        private final DataOutputBuffer buffer = new DataOutputBuffer();

        public XmlRecordReader(FileSplit split, JobConf jobConf) throws IOException {
            //不支持通配符,正则
            startTag = jobConf.get(START_TAG_KEY).getBytes("utf-8");
            endTag = jobConf.get(END_TAG_KEY).getBytes("utf-8");

            // open the file and seek to the start of the split
            start = split.getStart();
            end = start + split.getLength();
            Path file = split.getPath();
            FileSystem fs = file.getFileSystem(jobConf);
            fsin = fs.open(split.getPath());
            //先定位到文件此次的开头
            fsin.seek(start);
        }

        // 捞出 偏移量key,文本value
        @Override
        public boolean next(LongWritable key, Text value) throws IOException {
            if (fsin.getPos() < end) {
                if (readUntilMatch(startTag, false)) {
                    try {
                        //存一下startTag
                        buffer.write(startTag);
                        if (readUntilMatch(endTag, true)) {
                            key.set(fsin.getPos());
                            value.set(buffer.getData(), 0, buffer.getLength());
                            return true;
                        }
                    } finally {
                        buffer.reset();
                    }
                }
            }
            return false;
        }

        @Override
        public LongWritable createKey() {
            return new LongWritable();
        }

        @Override
        public Text createValue() {
            return new Text();
        }

        @Override
        public long getPos() throws IOException {
            return fsin.getPos();
        }

        @Override
        public synchronized void close() throws IOException {
            if (fsin != null) {
                fsin.close();
            }

        }

        @Override
        public float getProgress() throws IOException {
            if (start == end) {
                return 0.0f;
            }
            return Math.min(1.0f, (fsin.getPos() - start) / (float) (end - start));
        }

        /**
         * withinBlock=True  : 遇到的byte都存入buffer,包括match数组(找endTag)
         * withinBlock=False : 不存入buffer(找startTag)
         * <p>
         * return True   : 匹配上了
         * return False  : 文件结束都没匹配上
         * </p>
         * split是按splitSize划分的.
         */
        private boolean readUntilMatch(byte[] match, boolean withinBlock) throws IOException {
            int i = 0;
            while (true) {
                int b = fsin.read();
                // end of file:
                if (b == -1) return false;
                // save to buffer:
                if (withinBlock) buffer.write(b);

                // check if we're matching:
                if (b == match[i]) {
                    i++;
                    // 匹配完毕:
                    if (i >= match.length) return true;
                } else {
                    // 回溯,从头再来
                    i = 0;
                }
                // see if we've passed the stop point:
                /*
                 * 前面检查了有没超过文件尾,这里检查有没有超过split尾
                 * 情形1: 如果在找startTag(withinBlock=false)
                 *     , 而且一个都没匹配上, 而且已经到达split尾, [start,end)
                 *       返回false.
                 * (万里缉凶):
                 * 情形2: 如果在找endTag(withinBlock=True)
                 *      ,即使已经到split末尾,也允许跨split的查找.
                 *       尽量返回true.
                 * 情形3: 如果i!=0,即使已经到split末尾,也允许跨split的查找,以保证startTag或者endTag不被分割.
                 *       尽量返回true.
                 *
                **/

                if (!withinBlock && i == 0 && fsin.getPos() >= end) {
                    return false;
                }
            }
        }
    }
}
```