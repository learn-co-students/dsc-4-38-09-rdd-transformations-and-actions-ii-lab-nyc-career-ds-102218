
# Machine Learning with Spark

## Introduction

You've now explored how to perform operations on Spark RDDs for simple Map-Reduce tasks. Luckily, there are far more advanced use cases for spark, and many of the are found in the ml library, which we are going to explore in this lesson.


## Objectives
* Describe the use case for Machine Learning with Spark
* Load data with Spark DataFrames
* Train a machine learning model with Spark


## A Tale of Two Libraries

If you look at the pyspark documentation, you'll notice that there are two different libraries for machine learning [mllib](https://spark.apache.org/docs/latest/api/python/pyspark.mllib.html) and [ml](https://spark.apache.org/docs/latest/api/python/pyspark.ml.html). These libraries are extremely similar to one another, the only difference being that the mllib library is built upon the RDDs you just practiced using; whereas, the ml library is built on higher level Spark DataFrames, which has methods and attributes very similar to pandas. It's important to note that these libraries are much younger than pandas and many of the kinks are still being worked out. 

## Spark DataFrames

In the previous lessons, you've been introduced to SparkContext as the primary way to connect with a Spark Application. Here, we will be using SparkSession, which is from the [sql](https://spark.apache.org/docs/latest/api/python/pyspark.sql.html) component of pyspark. Let's go through the process of manipulating some data here. For this example, we're going to be using the [Forest Fire dataset](https://archive.ics.uci.edu/ml/datasets/Forest+Fires) from UCI, which contains data about the area burned by wildfires in the Northeast region of Portugal in relation to numerous other factors.



```python
from pyspark import SparkContext
from pyspark.sql import SparkSession
```


```python
spark = SparkSession.builder.master("local").appName("machine learning").getOrCreate()
```


```python
spark_df = spark.read.csv('./forestfires.csv',header='true',inferSchema='true')
```


```python
## observing the datatype of df
type(spark_df)
```




    pyspark.sql.dataframe.DataFrame



You'll notice that some of the methods are extremely similar or the same as those found within Pandas:



```python
spark_df.head()
```




    Row(X=7, Y=5, month='mar', day='fri', FFMC=86.2, DMC=26.2, DC=94.3, ISI=5.1, temp=8.2, RH=51, wind=6.7, rain=0.0, area=0.0)




```python
spark_df.columns
```




    ['X',
     'Y',
     'month',
     'day',
     'FFMC',
     'DMC',
     'DC',
     'ISI',
     'temp',
     'RH',
     'wind',
     'rain',
     'area']



Selecting columns is the same


```python
spark_df[['month','day','rain']].head()
```




    Row(month='mar', day='fri', rain=0.0)



But others not so much...


```python
spark_df.info()
```


    ---------------------------------------------------------------------------

    AttributeError                            Traceback (most recent call last)

    <ipython-input-37-3951e8b2005f> in <module>()
    ----> 1 spark_df.info()
    

    ~/anaconda3/lib/python3.6/site-packages/pyspark/sql/dataframe.py in __getattr__(self, name)
       1180         if name not in self.columns:
       1181             raise AttributeError(
    -> 1182                 "'%s' object has no attribute '%s'" % (self.__class__.__name__, name))
       1183         jc = self._jdf.apply(name)
       1184         return Column(jc)


    AttributeError: 'DataFrame' object has no attribute 'info'



```python
## this is better
spark_df.describe()
```




    DataFrame[summary: string, X: string, Y: string, month: string, day: string, FFMC: string, DMC: string, DC: string, ISI: string, temp: string, RH: string, wind: string, rain: string, area: string]



## Let's try some aggregations with our DataFrame


```python
spark_df_months = spark_df.groupBy('month').agg({'area':'mean'})
spark_df_months
```




    DataFrame[month: string, avg(area): double]



Notice how the grouped DataFrame is not returned when you call the aggregation method. Remember, this is still Spark! The transformations and actions are kept separate so that it is easier to manage large quantities of data. You can perform the transformation by making a `collect` method call.


```python
spark_df_months.collect()
```




    [Row(month='jun', avg(area)=5.841176470588234),
     Row(month='aug', avg(area)=12.489076086956521),
     Row(month='may', avg(area)=19.24),
     Row(month='feb', avg(area)=6.275),
     Row(month='sep', avg(area)=17.942616279069753),
     Row(month='mar', avg(area)=4.356666666666667),
     Row(month='oct', avg(area)=6.638),
     Row(month='jul', avg(area)=14.3696875),
     Row(month='nov', avg(area)=0.0),
     Row(month='apr', avg(area)=8.891111111111112),
     Row(month='dec', avg(area)=13.33),
     Row(month='jan', avg(area)=0.0)]



As you can see, there seem to be larger area fires during what would be considered the summer months in Portugal. On your own, practice more aggregations and manipualtions that you might be able to perform on this dataset. Now, we'll move on to using the machine learning applications of pyspark. 

### ML

Pyspark openly admits that they used sklearn as an inspiration for their implementation of a machine learning library. As a result, many of the methods and functionalities look similar, but there are some crucial distinctions. There are four main concepts found within the ML library:

`Transformer`: An algorithm that transforms one pyspark DataFrame into another DataFrame. 

`Estimator`: An algorithm that can be fit onto a pyspark DataFrame that can then be used as a Transformer. 

`Pipeline`: A pipeline very similar to an sklearn pipeline that chains together different actions.

The reasoning behind this separation of the fitting and transforming step is because sklearn is lazily evaluated, so the 'fitting' of a model does not actually take place until the Transformation action is called.


```python
from pyspark.ml.regression import RandomForestRegressor
from pyspark.ml import feature
from pyspark.ml.feature import StringIndexer, VectorAssembler, OneHotEncoderEstimator
```


```python
spark
```





            <div>
                <p><b>SparkSession - in-memory</b></p>
                
        <div>
            <p><b>SparkContext</b></p>

            <p><a href="http://10.128.106.158:4040">Spark UI</a></p>

            <dl>
              <dt>Version</dt>
                <dd><code>v2.3.1</code></dd>
              <dt>Master</dt>
                <dd><code>local</code></dd>
              <dt>AppName</dt>
                <dd><code>machine learning</code></dd>
            </dl>
        </div>
        
            </div>
        




```python
si = StringIndexer(inputCol='month',outputCol='month_num')
model = si.fit(spark_df)
new_df = model.transform(spark_df)
```

Note the small, but critical distinction between sklearn's implementation of a transformer and pyspark's implementation. sklearn is more object oriented and spark is more functionally based programming


```python
type(si)
```




    pyspark.ml.feature.StringIndexer




```python
type(model)
```




    pyspark.ml.feature.StringIndexerModel




```python
model.labels
```




    ['aug',
     'sep',
     'mar',
     'jul',
     'feb',
     'jun',
     'oct',
     'apr',
     'dec',
     'jan',
     'may',
     'nov']




```python
new_df.head(4)
```




    [Row(X=7, Y=5, month='mar', day='fri', FFMC=86.2, DMC=26.2, DC=94.3, ISI=5.1, temp=8.2, RH=51, wind=6.7, rain=0.0, area=0.0, month_num=2.0),
     Row(X=7, Y=4, month='oct', day='tue', FFMC=90.6, DMC=35.4, DC=669.1, ISI=6.7, temp=18.0, RH=33, wind=0.9, rain=0.0, area=0.0, month_num=6.0),
     Row(X=7, Y=4, month='oct', day='sat', FFMC=90.6, DMC=43.7, DC=686.9, ISI=6.7, temp=14.6, RH=33, wind=1.3, rain=0.0, area=0.0, month_num=6.0),
     Row(X=8, Y=6, month='mar', day='fri', FFMC=91.7, DMC=33.3, DC=77.5, ISI=9.0, temp=8.3, RH=97, wind=4.0, rain=0.2, area=0.0, month_num=2.0)]



Let's go ahead and remove the day column, as there is almost certainly no correlation between day of the week and areas burned with forest fires.


```python
new_df = new_df.drop('day','month')
new_df.head()
```




    Row(X=7, Y=5, FFMC=86.2, DMC=26.2, DC=94.3, ISI=5.1, temp=8.2, RH=51, wind=6.7, rain=0.0, area=0.0, month_num=2.0)



As you can see, we have created a new column called "month_num" that represents the month by a number. Now that we have performed this step, we can use Spark's version of OneHotEncoder. Let's make sure we have an accurate representation of the months.


```python
new_df.select('month_num').distinct().collect()
```




    [Row(month_num=8.0),
     Row(month_num=0.0),
     Row(month_num=7.0),
     Row(month_num=1.0),
     Row(month_num=4.0),
     Row(month_num=11.0),
     Row(month_num=3.0),
     Row(month_num=2.0),
     Row(month_num=10.0),
     Row(month_num=6.0),
     Row(month_num=5.0),
     Row(month_num=9.0)]




```python
ohe = feature.OneHotEncoderEstimator(inputCols=['month_num'],outputCols=['month_vec'])
```


```python
one_hot_encoded = ohe.fit(new_df).transform(new_df).drop('month_num')
one_hot_encoded.head()
```




    Row(X=7, Y=5, FFMC=86.2, DMC=26.2, DC=94.3, ISI=5.1, temp=8.2, RH=51, wind=6.7, rain=0.0, area=0.0, month_vec=SparseVector(11, {2: 1.0}))




```python
features = ['X',
 'Y',
 'FFMC',
 'DMC',
 'DC',
 'ISI',
 'temp',
 'RH',
 'wind',
 'rain',
 'month_vec']

target = 'area'

vector = VectorAssembler(inputCols=features,outputCol='features')
vectorized_df = vector.transform(one_hot_encoded)
```


```python
vectorized_df.head()
```




    Row(X=7, Y=5, FFMC=86.2, DMC=26.2, DC=94.3, ISI=5.1, temp=8.2, RH=51, wind=6.7, rain=0.0, area=0.0, month_vec=SparseVector(11, {2: 1.0}), features=SparseVector(21, {0: 7.0, 1: 5.0, 2: 86.2, 3: 26.2, 4: 94.3, 5: 5.1, 6: 8.2, 7: 51.0, 8: 6.7, 12: 1.0}))



Great! We now have our data in a format that seems acceptable for the last step. Now it's time for us to actually fit our model to data! Let's try and fit a Random Forest Regression model our data.


```python
rf_model = RandomForestRegressor(featuresCol='features',labelCol='area',predictionCol="prediction").fit(vectorized_df)
```


```python
predictions = rf_model.transform(vectorized_df).select("area","prediction")
```


```python
evaluator = RegressionEvaluator(predictionCol='prediction', labelCol='area')
```


```python
evaluator.evaluate(predictions,{evaluator.metricName:"r2"})
```




    0.671401928309378




```python
evaluator.evaluate(predictions,{evaluator.metricName:"mae"})
```




    13.710954049069102



### Putting it all in a Pipeline

We just performed a whole lot of transformations to our data, and we can streamline the process to make it much more efficient let's look at how we could take our previous code and combine it to form a pipeline. Let's take a look at all the Esimators we used to create this model:

* StringIndexer
* OneHotEnconderEstimator
* VectorAssembler
* RandomForestRegressor

Once we've fit our model in the Pipeline, we're then going to want to evaluate it to determine how well it performs. We can do this with:

* RegressionEvaluator


```python
from pyspark.ml.tuning import ParamGridBuilder, TrainValidationSplit, CrossValidator
from pyspark.ml.evaluation import RegressionEvaluator

```


```python
string_indexer = StringIndexer(inputCol='month',outputCol='month_num',handleInvalid='keep')
one_hot_encoder = OneHotEncoderEstimator(inputCols=['month_num'],outputCols=['month_vec'])
vector_assember = VectorAssembler(inputCols=features,outputCol='features')
random_forest = RandomForestRegressor(featuresCol='features',labelCol='area')
stages =  [string_indexer, one_hot_encoder, vector_assember,random_forest]


pipeline = Pipeline(stages=stages)
```


```python
params = ParamGridBuilder()\
.addGrid(random_forest.maxDepth, [5,10,15])\
.addGrid(random_forest.numTrees, [20,50,100])\
.build()
```

Let's take a look at the params variable we just built


```python
print('total combinations of parameters: ',len(params))
```

    total combinations of parameters:  9



```python
reg_evaluator = RegressionEvaluator(predictionCol='prediction', labelCol='area',metricName = 'mae')

cv = CrossValidator(estimator=pipeline, estimatorParamMaps=params,evaluator=reg_evaluator)
```


```python
cross_validated_model = cv.fit(spark_df)
```

Now, let's see how well the model performed! Let's take a look at the average performance for each one of our 9 models. It looks like the optimal performance was a MAE of 23.03. Note that this is worse than our original model, but that's because our original model had substantial data leakage. We didn't do a train-test-split!


```python
cross_validated_model.avgMetrics
```




    [24.283507394467403,
     23.035417293882116,
     22.516006104739546,
     25.172824616927077,
     24.076323818873483,
     23.15615371550553,
     25.249687913709245,
     24.141126374493428,
     23.229345283019505]



Let's take a look at the optimal parameters of our best performing model. The cross_validated_model variable is now saved as the best performing model from the grid search just performed. Let's look to see how well the predictions performed. As you can see, this dataset has a large number of areas of "0.0" burned. Perhaps, it would be better to investigate this problem as classification task.


```python
predictions = cross_validated_model.transform(spark_df)
predictions.select('prediction','area').show(300)
```

    +------------------+-------+
    |        prediction|   area|
    +------------------+-------+
    | 6.586531997403031|    0.0|
    | 4.919740844889813|    0.0|
    | 5.547914541367747|    0.0|
    | 6.893712206015393|    0.0|
    | 5.179697933378074|    0.0|
    | 9.218160801134067|    0.0|
    | 22.01227935091017|    0.0|
    | 7.357427417579902|    0.0|
    | 8.955984236069005|    0.0|
    |21.322545727623893|    0.0|
    | 6.731756502788954|    0.0|
    |18.223391492201202|    0.0|
    | 7.025265763112522|    0.0|
    | 10.43514507469672|    0.0|
    |62.953059671916726|    0.0|
    | 8.678236550869634|    0.0|
    | 4.961681338171639|    0.0|
    |  7.88099846133502|    0.0|
    | 4.502810492724764|    0.0|
    |  5.14657175330601|    0.0|
    | 10.93142862217377|    0.0|
    | 4.615792936762086|    0.0|
    |  5.65826956897323|    0.0|
    | 9.938450792664792|    0.0|
    | 7.678003823745614|    0.0|
    | 6.572181564331163|    0.0|
    | 8.052658244970239|    0.0|
    | 9.852608941724712|    0.0|
    |17.723862819276402|    0.0|
    |11.038881533381282|    0.0|
    |  5.82439771599864|    0.0|
    |6.9467990511095365|    0.0|
    |10.346031226787973|    0.0|
    |4.9645115032881595|    0.0|
    | 5.036718177165964|    0.0|
    |6.8405683496226946|    0.0|
    | 6.682718221448678|    0.0|
    | 8.136453366150993|    0.0|
    |13.678554722212425|    0.0|
    |4.5239261393014285|    0.0|
    | 19.83349830210394|    0.0|
    | 4.862932077723048|    0.0|
    | 4.459039575383673|    0.0|
    | 4.969754792298946|    0.0|
    | 6.897603880953402|    0.0|
    |  34.2145367122954|    0.0|
    | 7.390984957793332|    0.0|
    | 7.141222990874708|    0.0|
    | 4.341435126294971|    0.0|
    | 5.682197120813108|    0.0|
    |10.230944389879607|    0.0|
    | 4.428702598417711|    0.0|
    | 4.761558738307957|    0.0|
    | 4.761558738307957|    0.0|
    |4.4654096567109125|    0.0|
    | 10.85611282669806|    0.0|
    | 6.145279028133171|    0.0|
    | 6.406113751011762|    0.0|
    |  6.14468946216614|    0.0|
    | 8.076263097777574|    0.0|
    | 5.257420085801288|    0.0|
    | 5.887683275227865|    0.0|
    | 5.498341470574046|    0.0|
    | 3.627920319200929|    0.0|
    | 4.765243977952476|    0.0|
    | 7.885422151528384|    0.0|
    | 8.493717507066105|    0.0|
    | 6.091252879721221|    0.0|
    | 6.759180750772922|    0.0|
    | 4.986833085674115|    0.0|
    | 5.218679709560246|    0.0|
    | 4.345297892280104|    0.0|
    | 4.741494188424767|    0.0|
    | 8.586000336466954|    0.0|
    | 8.063529690106163|    0.0|
    | 8.923132068767787|    0.0|
    | 9.075890898531675|    0.0|
    | 9.029397768612986|    0.0|
    | 5.346044055542701|    0.0|
    |12.271738282845236|    0.0|
    |11.818183771954505|    0.0|
    | 9.957728449128789|    0.0|
    | 6.878849157817688|    0.0|
    | 6.895320816040911|    0.0|
    | 7.993688777376615|    0.0|
    |14.187333974283318|    0.0|
    |18.639056727531635|    0.0|
    | 21.09652108322387|    0.0|
    |17.219869466787856|    0.0|
    |5.1825217565614095|    0.0|
    | 5.357615003073127|    0.0|
    | 6.311057054401755|    0.0|
    |11.825503811444305|    0.0|
    | 16.36579649991413|    0.0|
    | 8.872849734596686|    0.0|
    | 5.472386298803346|    0.0|
    | 4.905919980283968|    0.0|
    | 4.971946577615097|    0.0|
    |5.5289382263673295|    0.0|
    | 6.468160414993256|    0.0|
    | 6.468160414993256|    0.0|
    |   5.1667250300838|    0.0|
    |  4.20669056915504|    0.0|
    |57.568908018703134|    0.0|
    | 5.230330334793721|    0.0|
    |5.2365757003956706|    0.0|
    | 5.180829097078061|    0.0|
    | 4.429083667605773|    0.0|
    |5.1180122959122585|    0.0|
    | 5.484348680906115|    0.0|
    |5.3265870458480995|    0.0|
    | 4.030460671075029|    0.0|
    |6.2433325681989675|    0.0|
    | 4.533692403528835|    0.0|
    |5.2548491590504485|    0.0|
    | 4.885662851181793|    0.0|
    | 4.755407967387757|    0.0|
    | 5.142605691603236|    0.0|
    | 4.827821784121938|    0.0|
    |4.1992672094361305|    0.0|
    | 5.023973141124753|    0.0|
    | 5.582271629035316|    0.0|
    | 8.494039851495835|    0.0|
    | 5.906819711747906|    0.0|
    | 5.079170816529416|    0.0|
    | 8.508873679798855|    0.0|
    | 4.544400078023479|    0.0|
    |10.717473878938764|    0.0|
    | 6.658341233540407|    0.0|
    |  6.02766058364641|    0.0|
    | 4.912032370099339|    0.0|
    |4.4216923557367585|    0.0|
    | 4.573571893580359|    0.0|
    | 6.201285759935665|    0.0|
    |4.0643835853036325|    0.0|
    | 6.093965973821651|    0.0|
    | 8.339668155003249|    0.0|
    | 7.714150846953341|    0.0|
    |17.640955869724657|   0.36|
    | 19.06649715914383|   0.43|
    |10.142589769972673|   0.47|
    | 4.038056041651417|   0.55|
    |17.117233157223676|   0.61|
    |  5.75865781578045|   0.71|
    | 7.419393802651569|   0.77|
    |  35.0665659903334|    0.9|
    | 4.908185320602965|   0.95|
    |13.293125897993718|   0.96|
    | 4.690606641750777|   1.07|
    |16.884075030894646|   1.12|
    | 9.624059982735831|   1.19|
    |13.913484377413178|   1.36|
    |10.447320325670772|   1.43|
    |5.1002194247082215|   1.46|
    |21.908667658545973|   1.46|
    | 6.980353070692324|   1.56|
    |12.943993668842838|   1.61|
    |5.5346657403656225|   1.63|
    |  4.05386613864729|   1.64|
    | 8.052658244970239|   1.69|
    |5.2263534324622425|   1.75|
    |  5.99557365964709|    1.9|
    | 8.063639021678533|   1.94|
    |61.310282713820385|   1.95|
    | 7.243181470891568|   2.01|
    | 5.677850199579921|   2.14|
    | 4.322277067733869|   2.29|
    | 6.059649685484659|   2.51|
    |12.452876006461656|   2.53|
    | 7.120897062022606|   2.55|
    | 8.075391750785663|   2.57|
    | 7.454789009548103|   2.69|
    | 7.656736831523963|   2.74|
    | 10.47563948232531|   3.07|
    | 4.481281666476903|    3.5|
    | 4.797025602026793|   4.53|
    | 6.451864678546468|   4.61|
    | 4.435973785122204|   4.69|
    | 6.418463326588235|   4.88|
    | 9.369030658718557|   5.23|
    |11.246024694439924|   5.33|
    | 9.379691972569908|   5.44|
    | 4.908551606991628|   6.38|
    | 8.878684641439989|   6.83|
    |11.465630687262495|   6.96|
    | 8.538844131656234|   7.04|
    | 11.94345198039145|   7.19|
    | 16.73821444261459|    7.3|
    |4.5462204654059715|    7.4|
    | 6.521282363581571|   8.24|
    | 6.419773553705057|   8.31|
    | 8.128829371229937|   8.68|
    | 5.939990068867677|   8.71|
    |14.573822461130758|   9.41|
    | 5.939990068867677|  10.01|
    | 5.817167228218083|  10.02|
    | 6.451864678546468|  10.93|
    | 11.36394193548643|  11.06|
    | 8.273156661323553|  11.24|
    | 9.400673982110339|  11.32|
    |17.435525281607134|  11.53|
    | 4.838667290507194|   12.1|
    | 7.215967145423657|  13.05|
    |11.542837817565838|   13.7|
    | 5.350661137603522|  13.99|
    | 8.622161652723998|  14.57|
    |7.5155772903584515|  15.45|
    | 12.73085706022988|   17.2|
    |12.525733198308941|  19.23|
    |  17.2006300060638|  23.41|
    | 8.451874098188352|  24.23|
    |10.819825662745295|   26.0|
    | 6.785682151475937|  26.13|
    | 7.880175103864316|  27.35|
    | 5.862730626021231|  28.66|
    | 5.862730626021231|  28.66|
    | 8.393256123218528|  29.48|
    | 8.814478711533448|  30.32|
    |11.659499648033554|  31.72|
    |16.727429887388574|  31.86|
    |  8.95368632496906|  32.07|
    | 9.073500261631352|  35.88|
    | 6.229587250625675|  36.85|
    |23.904701119644155|  37.02|
    | 8.498884537462231|  37.71|
    | 12.69369032191087|  48.55|
    |11.142013266142133|  49.37|
    |15.667337312812109|   58.3|
    | 41.56966311751706|   64.1|
    |19.713740704967496|   71.3|
    |30.678259552021963|  88.49|
    |  37.2424751896204|  95.18|
    |12.306928571474389| 103.39|
    | 49.90430692299485| 105.66|
    |105.65749086015862| 154.88|
    | 42.77178988289069| 196.48|
    | 81.30366032339744| 200.94|
    | 74.62566188419719| 212.88|
    | 570.0039527411222|1090.84|
    | 5.859645741324642|    0.0|
    | 5.373963308282531|    0.0|
    | 6.447314418021751|    0.0|
    | 5.424223886092537|  10.13|
    | 11.79870295080085|    0.0|
    | 6.735686471692832|   2.87|
    | 8.980666010572296|   0.76|
    | 6.521980622790253|   0.09|
    | 4.257872676465614|   0.75|
    | 7.558061474008494|    0.0|
    | 5.549786970946245|   2.47|
    | 29.85420738190639|   0.68|
    |  5.94350435273435|   0.24|
    | 5.285138453697332|   0.21|
    | 5.114322672047228|   1.52|
    | 8.636190177022476|  10.34|
    |13.587809606948044|    0.0|
    |  9.96273742260744|   8.02|
    | 4.312291050211501|   0.68|
    | 5.212458133388273|    0.0|
    | 5.489783719776476|   1.38|
    | 5.883948102954414|   8.85|
    | 4.937299638895994|    3.3|
    | 4.154095754485762|   4.25|
    | 6.923981513893794|   1.56|
    | 5.372046148715911|   6.54|
    |  5.20096332898115|   0.79|
    | 5.821419665593251|   0.17|
    |5.8977064774063415|    0.0|
    | 5.068152483768956|    0.0|
    | 5.349794625500795|    4.4|
    | 6.823283896355089|   0.52|
    |10.762462562336916|   9.27|
    | 5.062707139161695|   3.09|
    | 11.44207959054006|   8.98|
    |14.986076615364173|  11.19|
    | 6.599182244901718|   5.38|
    | 10.96090789330834|  17.85|
    | 9.857884212730083|  10.73|
    | 10.96090789330834|  22.03|
    | 10.96090789330834|   9.77|
    | 6.673010022679497|   9.27|
    |12.297190159712953|  24.77|
    |  5.63459280515827|    0.0|
    | 5.156529454912147|    1.1|
    | 7.363034090012201|  24.24|
    |7.0363811848664675|    0.0|
    |11.300757163195472|    0.0|
    | 8.205921160940848|    0.0|
    | 7.767448731904718|    0.0|
    |7.2447286933898765|    0.0|
    | 7.541697115306537|    0.0|
    |16.834717175855513|    8.0|
    | 4.946291217869888|   2.64|
    | 54.89473597667571|  86.45|
    |10.870213444938095|   6.57|
    | 7.202947293033188|    0.0|
    |4.8344348207306105|    0.9|
    | 7.056051111947495|    0.0|
    | 16.21362601535279|    0.0|
    | 5.565047039715807|    0.0|
    +------------------+-------+
    only showing top 300 rows
    


Now let's go ahead and take a look at the feature importances of our Random Forest model. In order to do this, we need to unroll our pipeline to access the Random Forest Model. Let's start by first checking out the "bestModel" attribute of our cross_validated_model.


```python
type(cross_validated_model.bestModel)
```




    pyspark.ml.pipeline.PipelineModel



So ml is treating the entire pipeline as the best performing model, let's see if we can go deeper into the pipeline to access the Random Forest model within it. Up above, we put the Random Forest Model as the final "stage" in the stages variable list. Let's look at the stages attribute of the bestModel.


```python
cross_validated_model.bestModel.stages
```




    [StringIndexer_47e4acf62f704a21366f,
     OneHotEncoderEstimator_43a7a11529ff9df60e6b,
     VectorAssembler_41f781ef51b853baa063,
     RandomForestRegressionModel (uid=RandomForestRegressor_4a2488f1d756e41813da) with 100 trees]



Perfect! There's the RandomForestRegressionModel, represented by the last item in the stages list. Now, we should be able to access all the attributes of the Random Forest Regressor.


```python
optimal_rf_model = cross_validated_model.bestModel.stages[3]
```


```python
optimal_rf_model.fe
```




    Param(parent='RandomForestRegressor_4a2488f1d756e41813da', name='featuresCol', doc='features column name')




```python
optimal_rf_model.featureImportances
```




    SparseVector(22, {0: 0.099, 1: 0.0777, 2: 0.0028, 3: 0.0318, 4: 0.0, 5: 0.006, 6: 0.0, 7: 0.0006, 8: 0.0, 9: 0.0002, 10: 0.0005, 12: 0.0006, 14: 0.1209, 15: 0.0913, 16: 0.1111, 17: 0.0713, 18: 0.1247, 19: 0.1423, 20: 0.119, 21: 0.0001})



## Summary

Hopefully by now you have seen the power of pyspark and its pipelines. With the use of a pipeline, you could train a huge number of models simultaneously, saving you a substantial amount of time and effort. Up next, you will have a chance to build an ml pipeline of your own with a classification problem!
