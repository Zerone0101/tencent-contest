运行环境: ubuntu 14.04, 256g以上的内存, 1T以上的磁盘，nvidia 1080 ti, python2.7
Python环境: sklearn, lightgbm, tensorflow-gpu, xgboost
文件说明:
    src文件夹包含converters文件夹
    converters文件夹包含如下：
    （1）pre-csv.py 将原始的数据文件进行拼表
    （2）combine_and_shuffle.py 将初赛数据与复赛数据拼一起，随机打乱
    （3）pre-dnn.py 生成dnn的特征数据文件
    （4）pre-gbdt.py 生成gbdt的特征数据文件
    （5）lgb_analyze_importance.py 利用lgb分析特征重要性
    （6）ensemble.py 对多个结果做加权组合
    （7）norm_ensemble.py 对多个结果进行归一化后再做加权组合

    data文件夹主要用于存在数据

    models文件夹用于存放模型，文件夹包含模型如下：
    （1）lgb.py 训练lgb模型
    （2）tf_NFM.py 定义nffm模型
    （3）run_tf_nfm.py 将全部数据一次加载进内存来训练nffm模型
    （4）run_tf_nfm_by_part.py 将数据分批次加载进入内存来训练nffm模型
    （5）xgb.py 训练xgb模型

    utils文件夹用于存放参数文件和辅助函数类：
    （1）args.py args2.py args3.py donal_args.py nzc_args.py 都是参数文件，用于用不同的参数来训练模型
    （2）tencent_data_func.py 定义了大量的特征工程及其他数据操作相关的辅助函数，由于迭代太多，里面有点杂乱

    gbdt_run.sh文件可生成gbdt的特征数据文件，训练lgb模型。值得注意的是，gbdt的特征数据文件的生成需要耗费大量的时间，
    本人使用了5台服务器跑了接近4天才将特征完全完成生成完，而lgb模型对我的成绩提升其实很少，lgb的成绩为0.760左右，
    只为我的最终成绩带来了万分位的提升。如果机器不足，不建议生成gbdt的特征。而且gbdt的特征占用了大概500g的磁盘空间。

    run.sh文件可生成dnn的特征数据文件，并训练lgb模型。值得注意的是，该文件只会跑一次dnn模型。实际上，dnn模型一般一轮收敛，
    十分吃数据的分布，在同样的参数下，使用打乱样本顺序，训练多次结果取平均都可以获得4-5个千分位的提升。而我的dnn模型的单次成绩为
    0.770~0.771之间，跑5-10次，取平均分数可以接近b榜的0.774-0.775之间。将参数和对正样本加权等改动，跑多次结果取平均
    ，可以到达0.775以上。

运行说明：
    将原始数据放进data文件，运行run.sh文件即可。
    值得注意的是，初赛的数据要在前面加 "chusai_"的前缀标志。向量特征（比如interest1等）的特征生成如果使用一台服务器，可能需要两天左右。
    dnn模型的训练时间大概是2.5-3个小时之间。
    lgb模型训练时间大概为10个小时左右。
    xgb的训练时间大概是30多个小时。
    如果是xgb的loss function设置为pair rank，可能需要2天多的时间。


特征说明：
    dnn的特征主要包括6类特征：
    （1）原始onehot特征，比如aid，age，gender等。
    （2）向量特征，比如interest1，interest2，topic1，kw1等
    （3）向量长度统计特征：interest1，interest2，interest5的长度统计。
    （4）uid类的统计特征，uid的出现次数，uid的正样本次数，以及uid与ad特征的组合出现次数，组合正样本次数。
    （5）uid的序列特征，比如uid=1时，总共出现了5次，序列为[-1,1,-1,-1,-1]，
        则第一次出现时，特征为【】
        第二次出现时，特征为【-1】
        第三次出现时，特征为【-1，1】
        第四次出现时，特征为【-1，1，-1】
        第五次出现时，特征为【-1，1，-1，-1】
    （6）组合特征：age与aid的组合，gender与aid的组合，interest1与aid的组合，interest2与aid的组合。
    值得注意的是，上面是总的类别介绍，但是实际上在模型运行中，本人构造了大量的特征，做了一个大的特征集合，每次训练的是小的特征集合，最后融合。
    只是效果提升似乎也没有多明显。所以这里只选取了最优成绩的特征工程。

    gbdt的特征主要包括：
    （1）原始特征的转化率，如果是向量特征的话，则取转化率最大的作为表示
    （2）原始特征的组合转化率，包括用户内部的特征组合，用户与广告的特征组合
    （3）原始特征的出现次数，如果是向量特征的话，则取出现次数最大的作为表示
    （4）原始特征的组合出现次数，包括用户内部的特征组合出现次数，用户与广告的特征组合出现次数
    （6）uid的出现次数和转化率，uid与广告特征的组合出现次数和转化率
    值得注意的是，gbdt的特征会进行用一个小的数据集进行遍历，对特征进行排序，最后分别取top200，top400的特征集合进行全集的训练。
    在top200的情况下，线上的成绩为759左右。
    top400的情况下，需要将数据分成两部分训练取平均，成绩为761左右。

模型说明：
    本人的最优模型为去年冠军的nffm模型，但是设计了一些过滤交叉的条件，比如广告内部的特征不做交叉。本人也尝试过多种其他模型的方案，
    但是单模型都比不过nffm模型。但本人在组合时尝试使用了其他dnn模型进行训练的结果来组合，由于没有控制变量对比（组合时同时加入了nffm不同参数
    的训练结果，以及nffm对正样本加权后的训练结果），不知道组合后的效果的提升是多少，甚至有可能是降低的也说不定。