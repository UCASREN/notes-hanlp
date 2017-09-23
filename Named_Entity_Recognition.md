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

// 输出：
[签约/vi, 仪式/n, 前/f, ，/w, 秦光荣/nr, 、/w, 李纪恒/nr, 、/w, 仇和/nr, 等/udeng, 一同/d, 会见/v, 了/ule, 参加/v, 签约/vi, 的/ude1, 企业家/nnt, 。/w]
[王国强/nr, 、/w, 高峰/n, 、/w, 汪洋/n, 、/w, 张朝阳/nr, 光着头/l, 、/w, 韩寒/nr, 、/w, 小/a, 四/m]
[张浩/nr, 和/cc, 胡健康/nr, 复员/v, 回家/vi, 了/ule]
[王总/nr, 和/cc, 小丽/nr, 结婚/vi, 了/ule]
[编剧/nnt, 邵钧林/nr, 和/cc, 稽道青/nr, 说/v]
[这里/rzs, 有/vyou, 关天培/nr, 的/ude1, 有关/vn, 事迹/n]
[龚学平/nr, 等/udeng, 领导/n, ,/w, 邓颖超/nr, 生前/t]
```


**说明：**
1. 目前分词器已默认开启中国人名识别，不必手动开启。
2. 有一定误命中率，解决办法： 比如误命中`关键年` ，则可以通过在 ```data/dictionary/person/nr.txt``` 加入一条`关键年A 1` 来排除关键年作为人名的可能性，也可以将`关键年`作为新词登记到自定义词典中。

### 1.2 算法过程：
下面以“馆内陈列周恩来和邓颖超生前使用过的物品。”为例说明算法过程：

* 第一步：分词 
    * 配置初始化（包括：执行字符正规化（繁体->简体，全角->半角，大写->小写）、多线程并行分词）：` CharTable.normalization(charArray)`、`config.threadNumber > 1 && charArray.length > 10000`。这里字符串不需要正规化，并且其长度小于10000，也不需要进行多线程并行分词。
    * 生成一元词网
        * 1.生成词网：`WordNet wordNetAll = new WordNet(sentence)`，词网结果如下
        ```java
        0:[ ]
        1:[]
        2:[]
        3:[]
        4:[]
        5:[]
        6:[]
        7:[]
        8:[]
        9:[]
        10:[]
        11:[]
        12:[]
        13:[]
        14:[]
        15:[]
        16:[]
        17:[]
        18:[]
        19:[]
        20:[]
        21:[ ]
        ```
        * 2.生成词图：`GenerateWordNet(wordNetAll)`使用`offset`方法（即邻接链表存储句子的所有可能单词，链表i的表头是句子的第i个字，后面是词是与后面的字组成的词，并能在核心词典中索引到，其中词典使用`双数组Trie树`，即前缀树）进行存储，其结果如下：
        ```java
        0:[ ]
        1:[馆, 馆内]
        2:[内]
        3:[陈, 陈列]
        4:[列]
        5:[周, 周恩来]
        6:[恩]
        7:[来]
        8:[和]
        9:[邓]
        10:[颖]
        11:[超, 超生]
        12:[生, 生前]
        13:[前]
        14:[使, 使用]
        15:[用]
        16:[过]
        17:[的]
        18:[物, 物品]
        19:[品]
        20:[。]
        21:[ ]
        ```
        * 3.原子分词，保证图连通：可能句子中的字（词）不在词典中（比如说数字123），这就需要进行原子分词，以保证词图的连通。由于该句子所有字均能在核心词典中找到，词图结果未发生改变，如上。
    * HMM-Viterbi分词（动态规划）
        * 原理：上述22个状态，首尾是开始、结束状态。从1号结点（[馆，馆内]）开始，递推地更新经过此结点并到达下一结点的最短路径花费（类似Dijkstra算法）。递推到第22个状态时，就算出了从0号结点（开始结点）到22号结点（终结点）的最短路径花费。
        * Backtracing求解最短路径：由于每个结果都记录了到达此结点最短路径的前序结点，因此只需要从终节点（22号）开始往回溯，便可找到最短路径。
* 第二步：加载用户词典
    * 使用索引分词，并将用户词语收集到全词图中：`combineByCustomDictionary(vertexList, wordNetAll)`
    * 不使用索引分词，使用用户词典合并粗分结果：`combineByCustomDictionary(vertexList)`

## 参考资料
1. [实战HMM-Viterbi角色标注中国人名识别](http://www.hankcs.com/nlp/chinese-name-recognition-in-actual-hmm-viterbi-role-labeling.html)
2. [基于角色标注的中国人名自动识别研究](https://www.google.com.sg/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&cad=rja&uact=8&ved=0ahUKEwjyi-uPwLjWAhWDXLwKHWfpCJsQFggoMAA&url=http%3A%2F%2Fnlp.ict.ac.cn%2FAdmin%2Fkindeditor%2Fattached%2Ffile%2F20130508%2F20130508094537_92322.pdf&usg=AFQjCNGMErZ912s0it5IX_gs7gInsmWhNA)
