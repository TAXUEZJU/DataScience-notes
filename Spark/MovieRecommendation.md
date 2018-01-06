### 使用Spark构建电影推荐服务
本文参考以下博客：
[Building a Movie Recommendation Service with Apache Spark & Flask - Part 1](https://www.codementor.io/jadianes/building-a-recommender-with-apache-spark-python-example-app-part1-du1083qbw)
本文将使用MovieLens的数据集，借助Spark的ALS实现通过协同过滤构建电影推荐服务。文章主要有两部分，前一部分主要关于如何将电影和评分数据转换成Spark RDDs；后一部分则是关于如何构建推荐服务。
#### 获取并处理数据
为了使用Spark构建电影推荐服务，需要有尽可能预先处理的模型数据。这里将加载和解析数据集，得到的RDD以待后面的使用。
##### 下载
我们可以下载GroupLens Research从MovieLens网站整理收集的评分数据集，最新的数据集获取地址在[这里](https://grouplens.org/datasets/movielens/latest/)
- 数据集有以下两种规模：
    Small: 100,000 ratings and 1,300 tag applications applied to 9,000 movies by 700 users. Last updated 10/2016.
    Full: 26,000,000 ratings and 750,000 tag applications applied to 45,000 movies by 270,000 users. Includes tag genome data with 12 million relevance scores across 1,100 tags. Last updated 8/2017.
    ``` Python
    small_dataset_url = 'http://files.grouplens.org/datasets/movielens/ml-latest-small.zip'
    complete_dataset_url = 'http://files.grouplens.org/datasets/movielens/ml-latest.zip'
    ```
- 定义下载位置
    ``` Python
    import os
    datasets_path = os.path.join('..', 'datasets')

    complete_dataset_path = os.path.join(datasets_path, 'ml-latest.zip')
    small_dataset_path = os.path.join(datasets_path, 'ml-latest-small.zip')
    ```
- 下载
    ``` Python
    import urllib

    small_f = urllib.urlretrieve(small_dataset_url, small_dataset_path)
    complete_f = urllib.urlretrieve(complete_dataset_url, complete_dataset_path)
    ```
- 解压到独立文件夹
    ``` Python
    import zipfile

    with zipfile.ZipFile(small_dataset_path, "r") as z:
        z.extractall(datasets_path)

    with zipfile.ZipFile(complete_dataset_path, "r") as z:
        z.extractall(datasets_path)
    ```
##### 加载解析数据集
评分数据集（`ratings.csv`）中每一行数据的格式为：
`userId,movieId,rating,timestamp`
电影数据集（`movies.csv`）中每一行数据的格式为：
`movieId,title,genres`，其中`genres`的格式为`Genre1|Genre2|Genre3...`
标签文件（`tags.csv`）的格式为：
`userId,movieId,tag,timestamp`
最后，链接文件（`links.csv`）的格式为：
`movieId,imdbId,tmdbId`

由于这些文件的格式统一且简单，可以用Python的`split()`方法解析。解析电影和评分数据会产生两个RDDs
- 对评分数据中的每一行，我们创建一个元组`(UserID, MovieID, Rating)`，时间戳不使用
- 对电影数据中的每一行，我们创建一个元组`(MovieID, Title)`，类别这里不使用

随后加载原始数据，需要过滤掉标头
``` Python
small_ratings_file = os.path.join(datasets_path, 'ml-latest-small', 'ratings.csv')

small_ratings_raw_data = sc.textFile(small_ratings_file)
small_ratings_raw_data_header = small_ratings_raw_data.take(1)[0]
```

将原始数据解析生成新的RDD
``` Python
small_ratings_data = small_ratings_raw_data.filter(lambda line: line!=small_ratings_raw_data_header)\
    .map(lambda line: line.split(",")).map(lambda tokens: (tokens[0],tokens[1],tokens[2])).cache()
```

对`movies.csv`进行类似的操作
``` Python
small_movies_file = os.path.join(datasets_path, 'ml-latest-small', 'movies.csv')

small_movies_raw_data = sc.textFile(small_movies_file)
small_movies_raw_data_header = small_movies_raw_data.take(1)[0]

small_movies_data = small_movies_raw_data.filter(lambda line: line!=small_movies_raw_data_header)\
    .map(lambda line: line.split(",")).map(lambda tokens: (tokens[0],tokens[1])).cache()

small_movies_data.take(3)
```

#### 协同过滤
##### 使用small规模的数据集作为ALS的参数
- 首先将数据切分成训练，验证，测试数据集
    ``` Python
    training_RDD, validation_RDD, test_RDD = small_ratings_data.randomSplit([6, 2, 2], seed=0L)
    validation_for_predict_RDD = validation_RDD.map(lambda x: (x[0], x[1]))
    test_for_predict_RDD = test_RDD.map(lambda x: (x[0], x[1]))
    ```
- 开始训练
    ``` Python
    from pyspark.mllib.recommendation import ALS
    import math

    seed = 5L
    iterations = 10
    regularization_parameter = 0.1
    ranks = [4, 8, 12]
    errors = [0, 0, 0]
    err = 0
    tolerance = 0.02

    min_error = float('inf')
    best_rank = -1
    best_iteration = -1
    for rank in ranks:
        model = ALS.train(training_RDD, rank, seed=seed, iterations=iterations,
                          lambda_=regularization_parameter)
        predictions = model.predictAll(validation_for_predict_RDD).map(lambda r: ((r[0], r[1]), r[2]))
        rates_and_preds = validation_RDD.map(lambda r: ((int(r[0]), int(r[1])), float(r[2]))).join(predictions)
        error = math.sqrt(rates_and_preds.map(lambda r: (r[1][0] - r[1][1])**2).mean())
        errors[err] = error
        err += 1
        print 'For rank %s the RMSE is %s' % (rank, error)
        if error < min_error:
            min_error = error
            best_rank = rank

    print 'The best model was trained with rank %s' % best_rank
    ```
