# 命名实体识别
> `Hanlp`的NER部分主要涉及的算法：HMM-Viterbi、N-最短路径。先总体说一下命名实体识别的算法过程：
>- 先是分词，使用的算法有：`标准分词(基于HMM-Viterbi分词实现)`、`NLP分词`、`索引分词`、`N-最短路径分词`、`CRF分词`、`极速词典分词`、`繁体分词`
>- 然后，
>- 最后，

>下面，我们分别具体介绍其中国人名、音译人名、地名、机构名以及日本人名的识别过程。

## 1. 中国人名识别
> 可自动识别中国人名，其标注为：`nr`：

### 1.1 调用方法
```java
String[] testCase = new String[]{
    "签约仪式前，秦光荣、李纪恒、仇和等一同会见了参加签约的企业家。",
    "王国强、高峰、汪洋、张朝阳光着头、韩寒、小四",
    "张浩和胡健康复员回家了",
    "王总和小丽结婚了",
    "编剧邵钧林和稽道青说",
    "这里有关天培的有关事迹",
    "龚学平等领导,邓颖超生前",
    };
Segment segment = HanLP.newSegment().enableNameRecognize(true);
for (String sentence : testCase)
{
    List<Term> termList = segment.seg(sentence);
    System.out.println(termList);
}   
```
**说明：**
1. 目前分词器已默认开启中国人名识别，不必手动开启。
2. 有一定误命中率，解决办法： 比如误命中`关键年` ，则可以通过在 ```data/dictionary/person/nr.txt``` 加入一条`关键年A 1` 来排除关键年作为人名的可能性，也可以将`关键年`作为新词登记到自定义词典中。

### 1.2 算法过程：
* 第一步：分词 
    * 配置初始化（包括：执行字符正规化（繁体->简体，全角->半角，大写->小写）、多线程并行分词）
    * HMM-Viterbi分词
        * 生成词网：
        * 生成词图1：使用`offset`方法（即邻接链表存储句子的所有可能单词，链表i的表头是句子的第i个字，后面是词是与后面的字组成的词，并能在核心词典中索引到。）
        * 生成词图2：使用`双数组Trie树`，即前缀树存储核心词典
    * 23
* 第二步：

## 参考资料
1. [实战HMM-Viterbi角色标注中国人名识别](http://www.hankcs.com/nlp/chinese-name-recognition-in-actual-hmm-viterbi-role-labeling.html)
