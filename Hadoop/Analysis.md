### 搜狗搜索日志分析
#### 一、数据预处理
搜狗数据的数据格式如下：
`访问时间\t用户ID\t[查询词]\t该URL在返回结果中的排名\t用户点击的顺序号\t用户点击的URL`
示例：
`20111230000005	57375476989eea12893c0c3811607bcf	奇艺高清	1	1	http://www.qiyi.com/`
其中，用户 ID 是根据用户使用浏览器访问搜索引擎时的 Cookie 信息自动赋值，即同一次使用浏览器输入的不同查询对应同一个用户 ID
##### 1. 查看数据
- 查看数据总行数
    ``` bash
    [root@hdp-node-01 Documents]# wc -l sogou.500w.utf8
    5000000 sogou.500w.utf8
    ```
    数据一共有500万行
- 截取部分数据查看
    ```
    [root@hdp-node-01 Documents]# tail -n3 sogou.500w.utf8
    20111230104333	53a3b5132bd6af7d324f3fd55d7153ba	洗碗机好用么	3	1	http://house.focus.cn/msgview/773/58374284.html
    20111230104334	966a6bf4c4ec1cc693b6e40702984235	X档案研究所TXT下载	4	2	http://www.readist.cn/book/dl024708.html
    20111230104334	ae55d5e4b4f29a1221816a121e087567	驻马店房产网	2	2	http://www.zmdfcw.com.cn/
    ```
    截取了最后3条数据，可以看到数据格式如上所述
##### 2. 数据初步处理
为便于后面的分析，我们将数据稍作处理
- 数据拓展
    将第一个长串的时间字段拆分，添加年、月、日、小时字段
- 数据过滤
    经过数据拓展后，将第二个字段（UID）或者第三个字段（搜索关键词）为空的行过滤
- 通过一个Java程序实现以上两步
    ``` Java
    import java.io.File;
    import java.io.IOException;
    import java.io.PrintWriter;
    import java.util.Scanner;

    public class split
    {
        public static void main(String[] args)
        {
            try
            {
                String filePath = args[0];
                String outPath = args[1];
                File file = new File(filePath);
                Scanner in = new Scanner(file, "UTF-8");
                PrintWriter out = new PrintWriter(outPath, "UTF-8");
                while(in.hasNextLine())
                {
                    String line = in.nextLine();
                    String date = line.substring(0, 14);
                    String year = date.substring(0, 4) + '\t';
                    String month = date.substring(4, 6) + '\t';
                    String day = date.substring(6, 8) + '\t';
                    String hour = date.substring(8, 10);
                    String newLine = line + '\t' + year + month + day + hour;
                    String[] results = newLine.split("\t");
                    if(results[1].equals("") || results[2].equals(""))
                    {
                        continue;
                    }
                    else
                    {
                        out.println(newLine);
                        out.flush();                        
                    }
                }
            }
            catch (IOException e)
            {
                e.printStackTrace();
            }
        }
    }
    ```
- 执行，结果文件命名为sogou.500w.utf8.flt
    ``` bash
    [root@hdp-node-01 Documents]# java split sogou.500w.utf8 sogou.500w.utf8.flt
    ```
- 结果文件信息
    - 行数
        ``` bash
        [root@hdp-node-01 Documents]# wc -l sogou.500w.utf8.flt
        5000000 sogou.500w.utf8.flt
        ```
        过滤之后行数为5000000，说明源数据比较干净没有问题
    - 数据格式
    ``` bash
    [root@hdp-node-01 Documents]# tail -n1 sogou.500w.utf8.flt
    20111230104334	ae55d5e4b4f29a1221816a121e087567	驻马店房产网	2	2	http://www.zmdfcw.com.cn/	2011	12	30	10
    ```
    可以看到处理后的数据格式如预期一样
##### 3. 上传数据
- 将数据加载到HDFS上
    ``` bash
    [root@hdp-node-01 Documents]# hdfs dfs -mkdir -p /sogou/20111230
    [root@hdp-node-01 Documents]# hdfs dfs -put sogou.500w.utf8 /sogou/20111230/
    [root@hdp-node-01 Documents]# hdfs dfs -mkdir -p /sogou_ext/20111230
    [root@hdp-node-01 Documents]# hdfs dfs -put sogou.500w.utf8.flt /sogou_ext/20111230
    ```

#### 二、基于Hive构建日志数据仓库
##### 1. 初始化
- 运行Hive
    ``` bash
    [root@hdp-node-01 Documents]# hive
    ```
- 新建并使用数据库
    ```
    hive> show databases;
    OK
    default
    Time taken: 0.009 seconds, Fetched: 1 row(s)
    hive> create database sogou;
    OK
    Time taken: 0.129 seconds
    hive> use sogou;
    OK
    Time taken: 0.019 seconds
    ```
##### 2. 创建分区表
- 创建扩展4个字段（年、月、日、小时）数据的外部表
    ```
    hive> create external table sogou_ext_20111230(
        > ts string,
        > uid string,
        > keyword string,
        > rank int,
        > theorder int,
        > url string,
        > year int,
        > month int,
        > day int,
        > hour int
        > )
        > comment 'This is the extend data of sogou search data'
        > row format delimited
        > fields terminated by '\t'
        > stored as textfile
        > location '/sogou_ext/20111230';
    ```
- 创建带分区的表
    ```
    hive> create external table sogou_partition(
        > ts string,
        > uid string,
        > keyword string,
        > rank int,
        > theorder int,
        > url string
        > )
        > comment 'This is the sogou search data by partition'
        > partitioned by(
        > year int,
        > month int,
        > day int,
        > hour int
        > )
        > row format delimited
        > fields terminated by '\t'
        > stored as textfile;
    ```
- 灌入数据
    ```
    hive> set hive.exec.dynamic.partition.mode=nonstrict;
    hive> insert overwrite table sogou_partition partition(year,month,day,hour)
    > select * from sogou_ext_20111230;
    ```

#### 三、数据分析
##### 1. 条数统计
- 数据总条数
    ```
    hive> select count(*) from sogou_ext_20111230;
    ...
    OK
    5000000
    Time taken: 53.375 seconds, Fetched: 1 row(s)
    ```
- 非空查询条数
    ```
    hive> select count(*) from sogou_ext_20111230
    > where keyword is not null and keyword !='';
    ...
    OK
    5000000
    Time taken: 49.647 seconds, Fetched: 1 row(s)
    ```
    因为预先做过数据清洗，所以并没有关键字为空的查询
- 无重复总条数（根据 ts、 uid、 keyword、 url）
    ```
    hive> select count(*) from (select count(*) from sogou_ext_20111230
    > group by ts,uid,keyword,url having count(*)=1) a;
    ...
    OK
    4999272
    Time taken: 121.199 seconds, Fetched: 1 row(s)
    ```
    单条无重复的搜索记录一共有4999272条
- 独立uid总数
    ```
    hive> select count(distinct(uid)) from sogou_ext_20111230;
    ...
    OK
    1352664
    Time taken: 62.149 seconds, Fetched: 1 row(s)
    ```
##### 2. keyword分析
- 关键词长度统计
    ```
    hive> select avg(a.cnt) from
    > (select length(keyword) as cnt from sogou_ext_20111230) a;
    ...
    OK
    7.6607936
    Time taken: 49.296 seconds, Fetched: 1 row(s)
    ```
    查询关键字的平均长度为7.66个字符，这样还看不出什么规律，下面作进一步分析
- 关键词长度分析
    - 我们将对应查询长度的条数统计出来
        ```
        hive> select count(a.len), len from
        > (select length(keyword) as len from sogou_ext_20111230) a
        > group by len;
        ```
        因为关键词长度最长有500多的，显示结果比较冗长，将其导入excel做进一步处理，按照关键字字长进行排序，因为平均长度为7.66，以7为中位数选取字长1到13的部分数据，第一个字段为条数，第二个字段为字长
        ```
        8368	1
        258816	2
        318421	3
        661141	4
        487283	5
        536134	6
        476956	7
        478349	8
        462978	9
        435883	10
        231242	11
        188980	12
        131869	13
        ```
        做一个求和，字长为1到13的搜索记录一共4676420条，占到了总搜索条数的93.5%，与前面平均字长7.66是符合的，看一下分布情况
        @import "./images/1.png"
        字长为1的搜索条数太少可以忽略，基本可以得出初步结论，人们的搜索习惯**以短字长为主，平均搜索字长为7.66，字长2-13基本包括了绝大部分的搜索情况，正常情况下人们觉得这样的长度已经能够表达自己的搜索意图**
- 关键词重复情况分析
    - 独立keyword总数
        ```
        hive> select count(distinct(keyword)) from sogou_ext_20111230;
        ...
        OK
        1490588
        Time taken: 57.203 seconds, Fetched: 1 row(s)
        ```
        注意到独立的关键词只有149万多个，说明部分关键词重复搜索
    - 查询频次最高的50词
        ```
        hive> select keyword, count(*) as cnt
        > from sogou_ext_20111230 group by keyword order by cnt desc limit 50;
        ...
        百度	38441
        baidu	18312
        人体艺术	14475
        4399小游戏	11438
        qq空间	10317
        优酷	10158
        新亮剑	9654
        馆陶县县长闫宁的父亲	9127
        公安卖萌	8192
        百度一下 你就知道	7505
        百度一下	7104
        4399	7041
        魏特琳	6665
        qq网名	6149
        7k7k小游戏	5985
        黑狐	5610
        儿子与母亲不正当关系	5496
        新浪微博	5369
        李宇春体	5310
        新疆暴徒被击毙图片	4997
        hao123	4834
        123	4829
        4399洛克王国	4112
        qq头像	4085
        nba	4027
        龙门飞甲	3917
        qq个性签名	3880
        张去死	3848
        cf官网	3729
        凰图腾	3632
        快播	3423
        金陵十三钗	3349
        吞噬星空	3330
        dnf官网	3303
        武动乾坤	3232
        新亮剑全集	3210
        电影	3155
        优酷网	3115
        两次才处决美女罪犯	3106
        电影天堂	3028
        土豆网	2969
        qq分组	2940
        全国各省最低工资标准	2872
        清代姚明	2784
        youku	2783
        争产案	2755
        dnf	2686
        12306	2682
        身份证号码大全	2680
        火影忍者	2604
        ```
        关键词中有电影类别的，有动漫类别的，也有小说和新闻，如果能对所有keyword进行归类就能得出搜索引擎的用户的需求从而更好地优化服务。注意到这里面有四个高频关键词是关于百度的，也就是用搜狗搜索引擎搜索了另一个搜索引擎的地址，这部分用户很可能不是该搜索引擎的忠实用户，下面对这一情况进行分析
- 指向别的搜索引擎的关键词分析
    ```
    hive> select count(*) from sogou_ext_20111230
    > where keyword like '%baidu%' or keyword like '%百度%';
    ...
    OK
    87846
    Time taken: 52.339 seconds, Fetched: 1 row(s)
    ```
    一共有87846条关键词与百度相关的
    ```
    hive> select count(*) from sogou_ext_20111230
    > where keyword like '%google%' or keyword like '%谷歌%';
    ...
    OK
    5007
    Time taken: 49.325 seconds, Fetched: 1 row(s)
    ```
    与谷歌相关的搜索条数只有5007条
    这两部分汇总也只有9万多条，相较于总计500万条的搜索，**分流至其他搜索引擎的比例还是很小的**
##### 3. UID分析
- UID的查询次数分布
    ```
    hive> select sum(if(uids.cnt=1,1,0)), sum(if(uids.cnt=2,1,0)),
    > sum(if(uids.cnt=3,1,0)), sum(if(uids.cnt>3,1,0))
    > from (select uid, count(*) as cnt from sogou_ext_20111230 group by uid) uids;
    ...
    OK
    549148	257163	149562	396791
    Time taken: 94.289 seconds, Fetched: 1 row(s)
    ```
    查询1次，2次，3次，及3次以上的UID个数分别为549148，257163，149562，396791
- UID平均查询次数
    ```
    hive> select sum(a.cnt)/count(a.uid) from
    > (select uid, count(*) as cnt from sogou_ext_20111230 group by uid) a;
    ...
    OK
    3.6964094557111005
    Time taken: 96.7 seconds, Fetched: 1 row(s)
    ```
    平均每个uid的查询次数为3.70次
- UID总数
    ```
    hive> select count(distinct(uid)) from sogou_ext_20111230;
    ...
    OK
    1352664
    Time taken: 57.442 seconds, Fetched: 1 row(s)
    ```
    一共有1352664个UID
- 我们不妨对用户搜索次数进行统计
    ```
    hive> select count(a.uid), cnt
    > from (select uid, count(uid) as cnt from sogou_ext_20111230 group by uid) a
    > group by cnt;
    ```
    结果较长，我们对其按照uid数进行降序排序，取前12条数据，结果如下：
    ```
    count(uid)	cnt
    549148	1
    257163	2
    149562	3
    96221	4
    66167	5
    47649	6
    35337	7
    26931	8
    21013	9
    16451	10
    13385	11
    10717	12
    ```
    @import "./images/2.png"
    可以看到在2011年12月30日这一天的搜索行为，搜索次数在1次和2次的居多，搜索次数越多的用户数量逐渐递减，考虑到是一天中的搜索行为，不妨认为搜索次数在3次及以上的算做平台的依赖用户，统计一下，依赖用户的总数为546353，前面统计过不重复的UID一共有1351664个，所以**依赖用户的比例为40.42%** ，对于依赖用户，可以对其行为进行分析，提供优化的搜索结果保证其良好体验，对于非依赖用户，可以进行意见采集改善服务从而提升用户对平台的依赖性
##### 4. 用户行为分析
- 点击次数与Rank之间的关系分析
    ```
    hive> select count(*) from sogou_ext_20111230 where rank < 11;
    ...
    OK
    4999869
    Time taken: 52.317 seconds, Fetched: 1 row(s)
    ```
    可以看到一共5000000条搜索记录，**用户基本只翻看搜索引擎返回结果的前10个占比** ，这个用户行为决定了尽管搜索引擎返回的结果数目十分庞大，但真正可能被绝大部分用户所浏览的，只是排在前面的很小一部分而已。所以传统的基于整个结果集合查准率和查全率的评价方式不再适用于网络信息检索的评价，我们需要着重强调在评价指标中有关最靠前结果文档与用户查询需求的相关度的部分
    我们可以再深入一些，看一下只查看返回结果的前5个的记录占比
    ```
    hive> select count(*) from sogou_ext_20111230 where rank < 6;
    ...
    OK
    4191408
    Time taken: 53.167 seconds, Fetched: 1 row(s)
    ```
    结合上述结果，用户点击Rank10的记录条数为4999869，而点击Rank5的记录条数为4191408，占比为83.8%，那么极有可能结果排名越靠前，点击可能就越高，我们对点击Rank10的搜索记录按照结果的排名进行统计
    ```
    hive> select rank, count(*) from sogou_ext_20111230 where rank < 11 group by rank;
    ```
    对结果按排名进行排序，结果如下：
    ```
    rank	count(rank)
    1	2071720
    2	905769
    3	554258
    4	375813
    5	283848
    6	218351
    7	179380
    8	151384
    9	128344
    10	131002
    ```
    @import "./images/3.png"
    从图中可以看出，基本遵循 **排名越靠前，点击次数越多** 的情况，并且前两名的点击可能比排名靠后的高出很多，即使是第2名的点击次数也是第3名的1.63倍，而第1名的点击次数是第二名的2.29倍，是第3名的3.74倍，难怪有“竞价排名”这种机制，排名靠前会带来大量的点击量
- 直接输入URL作为查询词的比例
    ```
    hive> select count(*) from sogou_ext_20111230 where keyword like '%www%';
    ...
    OK
    73979
    Time taken: 47.375 seconds, Fetched: 1 row(s)
    ```
    匹配结果，得到输入URL的搜索条数有73979，相比5000000的总条数而言只有1.48%的体量，说明 **直接用URL作为搜索关键词的情况还是少的**
    我们看一下直接输入URL的查询中，点击的结果就是用户输入的URL的网址所占的比例
    ```
    hive> select sum(if(instr(url,keyword)>0,1,0)) from
    > (select * from sogou_ext_20111230 where keyword like '%www%') a;
    ...
    OK
    27561
    Time taken: 47.382 seconds, Fetched: 1 row(s)
    ```
    占比37.26%，说明 **超过三分之一的用户提交含有URL的查询是由于没有记全网址想借助搜索引擎来找到自己想浏览的网页** ，因此搜索引擎在处理这部分查询的时候，一个可能比较理想的方式是首先把相关的完整URL地址返回给用户，这样有较大可能符合用户的查询需求
- 用户使用高级检索的比例
    看看有多少查询使用了高级搜索
    ```
    hive> select count(*) from sogou_ext_20111230
    > where keyword like '%and%' or keyword like '%or%';
    ...
    OK
    14806
    Time taken: 82.975 seconds, Fetched: 1 row(s)
    ```
    可以看到，500万条查询中只有14806条使用了高级搜索，占比约0.3%，这说明中文检索用户更多的检索方式只是简单地输入关键词，很少使用高级搜索，说明在使用检索系统的过程中，用户更看重简便，搜索引擎应该注意用户的这一倾向，产品优化也应符合用户的习惯
- 独立用户行为分析
    对于搜索相似关键词的不同用户，可能表现出不同的行为，比如搜索次数的不同，我们统计一下
    ```
    hive> select uid, count(*) from sogou_ext_20111230
    > where keyword like '%仙剑奇侠传%' group by uid;
    ```
    结果返回了514行，我们将其导出到excel做简单排序，在514位搜索了有关“仙剑奇侠传”词条的用户中，uid为bfdbc61e50c554e75a809e37cd8beb75的用户搜索了27次相关词条，而有些用户只搜索了一次词条，我们不妨做个分组
    ```
    hive> select cnt, count(a.cnt) from
    > (select uid, count(*) as cnt from sogou_ext_20111230
    > where keyword like '%仙剑奇侠传%' group by uid) a group by cnt;
    ...
    OK
    1	244
    2	119
    3	46
    4	31
    5	24
    6	14
    7	13
    8	5
    9	3
    10	4
    11	2
    12	3
    16	1
    19	1
    20	2
    24	1
    27	1
    Time taken: 106.033 seconds, Fetched: 17 row(s)
    ```
    很明显，仅搜索一次的居多，而后搜索次数越多用户越少，说明大部分用户只是稍加了解，少部分用户是感兴趣或者是粉丝
##### 5. 查询时段分析
- 统计一下一天24小时每个时段分别的查询次数
    ```
    hive> select hour, count(*) from sogou_ext_20111230 group by hour;
    ```
    将结果导出按24小时顺序进行排序，结果如下：
    ```
    hour	count
    0	90816
    1	65707
    2	45881
    3	34244
    4	27924
    5	28213
    6	32991
    7	52832
    8	165616
    9	279105
    10	315973
    11	276103
    12	274234
    13	295936
    14	306242
    15	318645
    16	317122
    17	289648
    18	295207
    19	340115
    20	353101
    21	328949
    22	270842
    23	194554
    ```
    数据不太直观，我们将其做成趋势图
    @import "./images/4.png"
    可以明显看到搜索活跃时段为9点至24点，其中有三小段高峰期，分别是11点，15点到17点，以及20点到22点，基本分别对应中午，下午，夜间的活动高峰期
    我们看一下各时段高峰期人们分别搜索什么，展示各时段搜索频次最高的10个关键词
    - 11点段
        ```
        hive> select keyword, count(*) as cnt from sogou_ext_20111230 where hour=11
        > group by keyword order by cnt desc limit 10;
        ...
        OK
        百度	2046
        baidu	962
        4399小游戏	676
        馆陶县县长闫宁的父亲	628
        人体艺术	623
        qq空间	606
        nba	478
        优酷	452
        儿子与母亲不正当关系	424
        新疆暴徒被击毙图片	404
        Time taken: 101.074 seconds, Fetched: 10 row(s)
        ```
    - 15点到17点段
        ```
        hive> select keyword, count(*) as cnt from sogou_ext_20111230 where hour>=15 and hour<=17
        > group by keyword order by cnt desc limit 10;
        ...
        OK
        百度	7383
        baidu	3385
        4399小游戏	2833
        人体艺术	2373
        qq空间	1966
        公安卖萌	1882
        4399	1712
        优酷	1673
        7k7k小游戏	1541
        魏特琳	1501
        Time taken: 110.182 seconds, Fetched: 10 row(s)
        ```
    - 20点到22点段
        ```
        hive> select keyword, count(*) as cnt from sogou_ext_20111230 where hour>=20 and hour<=22
        > group by keyword order by cnt desc limit 10;
        ...
        OK
        百度	6532
        baidu	3226
        人体艺术	3105
        公安卖萌	2487
        新亮剑	2482
        优酷	2316
        魏特琳	1925
        4399小游戏	1744
        qq空间	1602
        百度一下 你就知道	1453
        Time taken: 99.754 seconds, Fetched: 10 row(s)
        ```
    值得注意的是以上三个时段“百度”，“优酷”，“4399”，“qq空间”，“人体艺术”这几条均有出现，基本反映了高峰时段的流量走向

#### 四、 MapReduce 聚类算法实现
##### 1. K-means 算法简介
- 聚类是一种无监督的学习，它将相似的对象归到同一个簇中。聚类方法几乎可以应用于所有对象，簇内的对象越相似，聚类的效果越好。这里主要介绍一种称为K-均值（K-means）聚类的算法。之所以称之为K-均值是因为它可以发现k个不同的簇，且每个簇的中心采用簇中所含值的均值计算而成。下面会逐步介绍该算法的更多细节
- 在介绍K-均值算法之前，先讨论一下簇识别（cluster identification）。簇识别给出聚类结果的含义。假定有一些数据，现在将相似数据归到一起，簇识别会告诉我们这些簇到底都是些什么。聚类与分类的最大不同在于，分类的目标事先已知，而聚类则不一样。因为其产生的结果与分类相同，而只是类别没有预先定义，聚类有时也被称为无监督分类（unsupervised classification），它分析试图将相似对象归入同一簇，将不相似对象归到不同簇
- K-均值算法的工作流程是这样的。首先，随机确定k个初始点作为质心。然后将数据集中的每个点分配到一个簇中，具体来讲，为每个点找距其最近的质心，并将其分配给该质心所对应的簇。这一步完成之后，每个簇的质心更新为该簇所有点的平均值，多次迭代直到质心稳定
##### 2. 初步思路
因为 K-means 算法更适合用于数值型数据，考虑到原本的日志中，“该URL在返回结果中的排名”和“用户点击的顺序号”这两个字段均为数值型的值，所以对这两项进行聚类，分别用x, y代表这两个字段的数值，去发现排名以及点击的顺序号之间的聚集关系
##### 3. 准备
- 新建一个Center类，存放质心，属性x，y分别表示“用户点击的URL在返回结果中的排名”，“用户点击的顺序号”
    ``` Java
    public class Center
    {
        public double x;
        public double y;
    }
    ```
- 实现Map
map过程计算比较和各个中心的距离，这里采用欧几里得距离，输出key为距离最近质心序号，value为该节点的x，y值，这里包含main方法的类名为Kmeans，类中定义了静态变量`public static Center[] centers = new Center[5];`，用来存放质心以供Map和Reduce读取。整体逻辑很简单，value就是单条数据，将其转成字符串，分割后提取关键字段的值，计算比较和各个质心的距离，并选择距离最近的质心，将质心序号作为key，该点x，y值作为value输出。需要注意的是所导入的包一律为`org.apache.hadoop`中的，不要误用了java本身的包，另外继承自Mapper的时候注意后面的数据类型与自己要使用的对应
    ``` Java
    import org.apache.hadoop.io.IntWritable;
    import org.apache.hadoop.io.LongWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.Mapper;

    import java.io.IOException;

    public class KMapper extends Mapper<LongWritable, Text, IntWritable, Text>
    {
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException
        {
            String line = value.toString();
            String[] results = line.split("\t");
            double x = Double.parseDouble(results[3]);
            double y = Double.parseDouble(results[4]);
            double min = Math.pow(x - Kmeans.centers[0].x, 2) + Math.pow(y - Kmeans.centers[0].y, 2);
            int mincenter = 0;
            for(int i = 1; i < 5; i++)
            {
                double tmp = Math.pow(x - Kmeans.centers[i].x, 2) + Math.pow(y - Kmeans.centers[i].y, 2);
                if(tmp < min)
                {
                    mincenter = i;
                    min = tmp;
                }
            }
            context.write(new IntWritable(mincenter), new Text(Double.toString(x) + " " + Double.toString(y)));
        }
    }
    ```
- 实现Reduce
测试时选用了5个质心，所以map后有5个key，每个key对应一个reduce任务。如果内容更多，我们只有5个reduce任务显然是不够的，可以考虑在map后本地先用combine预处理，得到第一次的处理结果，再将处理结果传送给reduce任务。Reduce部分很简单，将归到一个质心下的点求平均值，重新计算聚类之后的新质心。在编写程序的时候，主要是将第一步传来的序列化对象处理，转化为字符串后切出两个值，然后累加，最后除以点的总数即可，额外添加一步输出，这样运行后能在终端里看到结果。需要注意的是所导入的包一律为`org.apache.hadoop`中的，不要误用了java本身的包，另外继承自Reducer的时候注意后面的数据类型与自己要使用的对应
    ``` Java
    import org.apache.hadoop.io.IntWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.Reducer;

    import java.io.IOException;

    public class KReducer extends Reducer<IntWritable,Text,IntWritable,Text>
    {
        protected void reduce(IntWritable key, Iterable<Text> values, Context context) throws IOException, InterruptedException
        {
            int n = key.get();
            double sumx = 0, sumy = 0;
            int num = 0;
            double sx = Kmeans.centers[n].x;
            double sy = Kmeans.centers[n].y;
            for(Text value:values)
            {
                String[] results = value.toString().split(" ");
                double xx = Double.parseDouble(results[0]);
                double yy = Double.parseDouble(results[1]);
                sumx+=xx;
                sumy+=yy;
                num++;
            }
            Kmeans.centers[n].x = (sumx + sx) / num;
            Kmeans.centers[n].y = (sumy + sy) / num;
            System.out.println(Kmeans.centers[n].x + " " + Kmeans.centers[n].y + " " + num);
            context.write(key, new Text(Double.toString(Kmeans.centers[n].x) + " " + Double.toString(Kmeans.centers[n].y)));
        }
    }
    ```
- 主类部分
主类部分主要是通过随机数随机指定质心，然后新建配置文件管理配置信息，并从配置文件中获得job实例，通过job管理跟踪task，job中需要设置运行类，mapper类，reducer类，以及设置输入的key和value以及输出的key和value。同时要设置读取的文件路径，输出路径设置的时候需要加一步判断，如果输出文件夹已存在，需要删除，否则报错，最后需要有状态检测，退出代码1执行失败，0则执行成功
    ``` Java
    import org.apache.hadoop.conf.Configuration;
    import org.apache.hadoop.fs.FileSystem;
    import org.apache.hadoop.fs.Path;
    import org.apache.hadoop.io.IntWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.Job;
    import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
    import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

    import java.util.Random;

    public class Kmeans
    {
        public static Center[] centers = new Center[5];

        public static void main(String[] args) throws Exception
        {
            Random r = new Random();
            for(int i = 0; i < 5; i++)
            {
                centers[i] = new Center();
                centers[i].x = r.nextDouble() * 10 + 1;
                centers[i].y = r.nextDouble() * 10 + 1;
            }
            Configuration conf = new Configuration();
            Job job = Job.getInstance(conf);
            job.setJarByClass(Kmeans.class);
            job.setMapperClass(KMapper.class);
            job.setReducerClass(KReducer.class);

            job.setMapOutputKeyClass(IntWritable.class);
            job.setMapOutputValueClass(Text.class);

            job.setOutputKeyClass(IntWritable.class);
            job.setMapOutputValueClass(Text.class);

            FileInputFormat.setInputPaths(job, "hdfs://hdp-node-01:9000/sogou/20111230/sogou.500w.utf8");
            Path path = new Path("hdfs://hdp-node-01:9000/tmp");
            FileSystem fileSystem = path.getFileSystem(conf);
            if(fileSystem.exists(path))
            {
                fileSystem.delete(path, true);
            }
            FileOutputFormat.setOutputPath(job, path);
            boolean res = job.waitForCompletion(true); //true print information about job
            System.exit(res?0:1);
        }
    }
    ```
- 测试运行
以上的程序只进行了一次迭代，主要是为了测试程序是否有问题，我们部署好之后试运行了几次，下面分别是第一次和第二次和第三次的结果
    - 第一次
        ```
        7.214117473929565 2.4315226861140307 951357
        9.360172817169504 7.277625792293848 21757
        1.8827826682487936 1.2392867854399146 3976780
        1.3203408688732374 4.9712221908843865 12630
        3.7877700461140975 4.36423574524811 37476
        ```
    - 第二次
        ```
        1.7436213338872386 1.3429076287862443 3774748
        6.265940886722267 1.6770515136450537 1053757
        6.447816350076548 7.265226537649535 4141
        8.960245798483784 4.3523384268338265 159702
        9.24197910563815 8.467830488572691 7652
        ```
    - 第三次
        ```
        8.9426467334044 2.603087106180421 406271
        6.115037646797433 4.019061694806084 184760
        3.906391449751902 2.1579232169074123 1105911
        9.86301455301736 8.744859518223945 4651
        1.6929229846050575 1.0305068740487302 3298407
        ```
注意输出的五条结果，最后一个值是该类中点的个数，加起来应为数据总数5000000，如果不是说明程序有问题。我们初步观察下跑出来的结果（因为只有一次迭代，不具有显著参考意义），可以发现按照规模来看，实际有效的聚类数量应该在3到4之间，k设置为5明显不是很合适，所以在后面的正式程序中，会分别考虑k为4和3的情况，并进行多次迭代
##### 4. 正式实现
思路很清晰，主类中设置全局变量k，作为设定的质心个数，设置两个static数组，`public static Center[] incenters = new Center[k];`，
`public static Center[] outcenters = new Center[k];`，分别保存一次迭代前的质心和一次迭代后的质心，同时新增一个static的boolean数组，用来标识每一个质心是否已收敛，主类中设置循环，只有当所有质心均收敛，才停止运行，否则让incenters的值等于outcenters的值，继续跑一遍迭代，Map中除了变量名外基本没有更改，Reduce中除了修改变量名之外，需要让outcenters存储一次迭代后的值，并分别判断该质心是否已收敛（这里是用绝对值小于0.1作为标准的）
- Kmeans.java
    ``` java
    import org.apache.hadoop.conf.Configuration;
    import org.apache.hadoop.fs.FileSystem;
    import org.apache.hadoop.fs.Path;
    import org.apache.hadoop.io.IntWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.Job;
    import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
    import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

    import java.util.Random;

    public class Kmeans
    {
        public static int k = 3;
        public static Center[] incenters = new Center[k];
        public static Center[] outcenters = new Center[k];
        public static boolean[] flag = new boolean[k];
        public static int count = 0;

        public static void main(String[] args) throws Exception
        {
            Random r = new Random();
            for(int i = 0; i < k; i++)
            {
                incenters[i] = new Center();
                outcenters[i] = new Center();
                incenters[i].x = r.nextDouble() * 10 + 1;
                incenters[i].y = r.nextDouble() * 10 + 1;
            }
            while(true)
            {
                count++;
                for(int i = 0; i < k; i++)
                {
                    flag[i] = false;
                }
                Configuration conf = new Configuration();
                Job job = Job.getInstance(conf);
                job.setJarByClass(Kmeans.class);
                job.setMapperClass(KMapper.class);
                job.setReducerClass(KReducer.class);

                job.setMapOutputKeyClass(IntWritable.class);
                job.setMapOutputValueClass(Text.class);

                job.setOutputKeyClass(IntWritable.class);
                job.setMapOutputValueClass(Text.class);

                FileInputFormat.setInputPaths(job, "hdfs://hdp-node-01:9000/sogou/20111230/sogou.500w.utf8");
                Path path = new Path("hdfs://hdp-node-01:9000/tmp");
                FileSystem fileSystem = path.getFileSystem(conf);
                if(fileSystem.exists(path))
                {
                    fileSystem.delete(path, true);
                }
                FileOutputFormat.setOutputPath(job, path);
                boolean res = job.waitForCompletion(true); //true print information about job
                boolean finish = true;
                for(boolean b:flag)
                {
                    finish = finish && b;
                }
                if(finish)
                {
                    System.exit(res?0:1);
                }
                else
                {
                    for(int i = 0; i < k; i++)
                    {
                        incenters[i].x = outcenters[i].x;
                        incenters[i].y = outcenters[i].y;
                    }
                }
            }
        }
    }
    ```
- KMapper.java
    ``` java
    import org.apache.hadoop.io.IntWritable;
    import org.apache.hadoop.io.LongWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.Mapper;

    import java.io.IOException;

    public class KMapper extends Mapper<LongWritable, Text, IntWritable, Text>
    {
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException
        {
            String line = value.toString();
            String[] results = line.split("\t");
            double x = Double.parseDouble(results[3]);
            double y = Double.parseDouble(results[4]);
            double min = Math.pow(x - Kmeans.incenters[0].x, 2) + Math.pow(y - Kmeans.incenters[0].y, 2);
            int mincenter = 0;
            for(int i = 1; i < Kmeans.k; i++)
            {
                double tmp = Math.pow(x - Kmeans.incenters[i].x, 2) + Math.pow(y - Kmeans.incenters[i].y, 2);
                if(tmp < min)
                {
                    mincenter = i;
                    min = tmp;
                }
            }
            context.write(new IntWritable(mincenter), new Text(Double.toString(x) + " " + Double.toString(y)));
        }
    }
    ```
- KReducer.java
    ``` java
    import org.apache.hadoop.io.IntWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.Reducer;

    import java.io.IOException;

    public class KReducer extends Reducer<IntWritable,Text,IntWritable,Text>
    {
        protected void reduce(IntWritable key, Iterable<Text> values, Context context) throws IOException, InterruptedException
        {
            int n = key.get();
            double sumx = 0, sumy = 0;
            int num = 0;
            double sx = Kmeans.incenters[n].x;
            double sy = Kmeans.incenters[n].y;
            for(Text value:values)
            {
                String[] results = value.toString().split(" ");
                double xx = Double.parseDouble(results[0]);
                double yy = Double.parseDouble(results[1]);
                sumx+=xx;
                sumy+=yy;
                num++;
            }
            Kmeans.outcenters[n].x = (sumx + sx) / num;
            Kmeans.outcenters[n].y = (sumy + sy) / num;
            System.out.println("count:" + Kmeans.count + " " + Kmeans.outcenters[n].x + " " + Kmeans.outcenters[n].y + " " + num);
            if(Math.abs(Kmeans.outcenters[n].x - Kmeans.incenters[n].x) < 0.1 && Math.abs(Kmeans.outcenters[n].y - Kmeans.incenters[n].y) < 0.1)
            {
                Kmeans.flag[n] = true;
            }
            context.write(key, new Text(Double.toString(Kmeans.outcenters[n].x) + " " + Double.toString(Kmeans.outcenters[n].y)));
        }
    }
    ```
- `k = 3`时的运行结果
    ```
    count:1 9.330568537848727 7.2883297584927895 21903
    count:1 4.5134409571408 1.7462600047707584 2630245
    count:1 1.1231791997089249 1.2239109037465625 2347852
    count:2 9.003357732695832 4.249588491516997 207679
    count:2 4.936153395496585 1.8111865064949184 1815343
    count:2 1.3040943275963746 1.1608433867871726 2976978
    count:3 8.643376314862662 3.149515482663674 448798
    count:3 5.038404996574 1.928561981004139 1093580
    count:3 1.539743587961416 1.1868689408048037 3457622
    count:4 8.638887127136964 2.894119188529561 486234
    count:4 4.912689972584228 2.0028641251211776 1056144
    count:4 1.5397436561149798 1.1868689483318133 3457622
    count:5 8.489841585779402 2.8109350243322027 535060
    count:5 4.810422168719511 2.0053115659128307 1007613
    count:5 1.5397043842667055 1.1863729369159899 3457327
    count:6 8.338324899505043 2.626763343610904 595636
    count:6 4.67036853937298 2.069615025929891 947037
    count:6 1.5397043842553464 1.186372936772523 3457327
    count:7 8.338363878059832 2.626676382048798 595629
    count:7 4.6681856091230625 2.072951326387883 947811
    count:7 1.5396089000926887 1.1852781917203628 3456560
    ```
    最终数据聚合于点(8.34,2.63)，(4.67,2.07)，(1.54,1.19)
- `k = 4`时的运行结果
    ```
    k = 4
    count:1 8.970585055507796 9.105676213893169 3429
    count:1 1.614347467766237 8.48876217945685 460
    count:1 7.011441895564772 2.354176619380053 1087648
    count:1 1.8051595390822657 1.2871192613617601 3908463
    count:2 8.87743223556124 6.896210737937779 34781
    count:2 2.334761418898725 5.784896747651664 11634
    count:2 6.8871051924960724 2.272155150936749 1081954
    count:2 1.788761585274924 1.255482582694286 3871631
    count:3 8.66143410962328 5.619378371933299 97299
    count:3 3.3307048654328417 4.536882328843173 65310
    count:3 6.875257649701752 2.0277448263210407 983941
    count:3 1.7874267964451556 1.2425336401101823 3853450
    count:4 8.82644530678811 4.880429218766998 152415
    count:4 3.704764300776321 3.6603003718385843 217520
    count:4 6.795208695522376 1.8482693793956522 895732
    count:4 1.7339259212895037 1.1864743295613005 3734333
    count:5 8.853772732467293 4.2150509472702975 225732
    count:5 3.845033228507199 3.075970963632213 401004
    count:5 6.705268680851961 1.684489029078676 774842
    count:5 1.6609804892049684 1.1494261057970214 3598422
    count:6 8.787408244978677 3.8444359554933283 286183
    count:6 3.9000143364617643 2.331115379541546 847143
    count:6 6.783711826424661 1.4849235772591354 609184
    count:6 1.4617637079409267 1.1194779260799284 3257490
    count:7 8.845297814341023 3.5441725843632064 336325
    count:7 4.025077480462755 2.06713138188083 908939
    count:7 6.922327604648137 1.7569833088283002 472410
    count:7 1.4652001238645118 1.1349793163378428 3282326
    count:8 8.84529798646492 3.544171691585771 336325
    count:8 3.763011315761597 1.7898032544436921 1223168
    count:8 6.922327898070753 1.7569838847257866 472410
    count:8 1.30218772001054 1.1505817144720394 2968097
    count:9 8.841362544993842 3.547745981714008 336668
    count:9 3.7633944453444044 1.786958398339191 1222504
    count:9 6.922327898071375 1.7569838847270056 472410
    count:9 1.3021549869956723 1.1512142665156035 2968418
    ```
    最终数据聚合于点(8.84,3.55)(3.76,1.79)(6.92,1.76)(1.30,1.51)
- 对结果的分析
    - `k = 3`时，数据聚合于点(8.34,2.63)，(4.67,2.07)，(1.54,1.19)，数据规模分别为595629，947811，3456560，按行为解释即有约69.1%的用户聚集在点击搜索结果的第一页，点击第一条或第二条返回结果，约19.0%的用户聚集在点击搜索结果的第二页，在第4个或第五个返回结果之间，约11.9%的用户聚集在点击搜索结果的2到3页，在第8到第9条结果之间，那可以预见的是，按照这个分布，点击第3页以后的情况很少很少了，搜索引擎只有在前3页或者说前2页提供用户最想要的搜索结果才能尽可能满足更多的用户
    - `k = 4`时，有两个簇规模比较明显，另两个簇离得比较远，感觉参考意义不大，两个规模明显的簇得质心分别是(3.76,1.79)和(1.30,1.51)，其实看不出太大的意义，相较而言，`k = 3`时的数据更有解释意义
    - 感觉这样两个字段k值越小，解释意义越大，所以补充又跑了一下`k = 2`的情况，这里只附上最后结果。数据最终聚集于质心(6.94,2.42)，(1.79,1.27)，基本能反映上面的分析情况
