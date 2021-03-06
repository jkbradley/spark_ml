# Spark ML

This is a scalable, high-performance machine learning package for `Spark`.
This package is maintained independently from `MLLib` in `Spark`.
There's a chance that some of the algorithms would later get merged with `MLLib`.
The algorithms listed here are going through continuous development, maintenance and updates and thus are provided without guarantees.
Use at your own risk.

This package is provided under the Apache license 2.0.

The current version is `0.1`.

Any comments or questions should be directed to schung@alpinenow.com

## Sequoia Forest

This is a `Random Forest` implementation that's designed to scale to an arbitrarily large data-set and very large trees (e.g. trees with millions of nodes). 

### Features and Requirements

* Designed for [Spark](https://spark.apache.org/ "Spark").
* Scales to an arbitrarily large data set and very large trees.
* Supports both classification (currently supports only information gain) and regression.
* Supports both numerical and categorical features.
* Run-time performance and the data size scale with the number of executors and machines.
* The training data must be discretized prior to training. The package provides Equal-Width and Equal-Frequency binning procedures.
* Discretization supports either unsigned `Byte` (upto 256 bins) or unsigned `Short` (upto 65536 bins) for the number of bins. This could drastically reduce the memory requirement for caching.

### Compiling the Code

Clone this repository. And run `./sbt assembly` at the project directory.

### Quick Start (for YARN and Linux variants)

1. Get `Spark` version 1.0.1. A pre-built version can be downloaded from [here](https://spark.apache.org/downloads.html "SparkDownload") for some of Hadoop variants. For different Hadoop versions, you'll have to build it after cloning it from github. E.g., to build `Spark` for Apache Hadoop 2.0.5-alpha with `YARN` support, you could do the following.
 1. `git clone https://github.com/apache/spark.git`
 2. `git checkout tags/v1.0.1`
 3. `SPARK_HADOOP_VERSION=2.0.5-alpha SPARK_YARN=true sbt/sbt assembly` or `SPARK_HADOOP_VERSION=2.0.5-alpha SPARK_YARN=true sbt/sbt -Dsbt.override.build.repos=true assembly`
 4. If you want to run this against a different `Spark` version, you should modify `project/build.scala` and change versions of `spark-core` and `spark-mllib` to appropriate versions. Of course, you'll also need to build a matching version of `Spark`.
 5. Additionally, by default, this package builds against `hadoop-client` version `1.0.4`. This will have to change, for instance if you want to build this against different Hadoop versions that are not protocol-compatible with this version. Refer to this `Spark` [page](http://spark.apache.org/docs/latest/building-with-maven.html "SparkMaven") to find out about different Hadoop versions.
2. Clone this repository, and run `./sbt assembly`.
3. In order to connect to Hadoop clusters, you should have Hadoop configurations stored somewhere. E.g., if your Hadoop configurations are stored in `/home/me/hd-config`, then make sure to have the following environment variables.
 * `export HADOOP_CONF_DIR=/home/me/hd-config`
 * `export YARN_CONF_DIR=/home/me/hd-config`
4. Find the location of the `Spark` assembly jar. E.g., it might be `assembly/target/scala-2.10/spark-assembly-1.0.1-hadoop2.0.5-alpha.jar` under the `Spark` directory. Run `export SPARK_JAR=jar_location`.
5. Have some data you want to train on in HDFS. A couple of data sets are provided in this package under the `data` directory for quick testing. E.g., copy `mnist.tsv.gz` and `mnist.t.tsv.gz` to a HDFS directory (E.g. `/Datasets/`).
6. To train a classifier using `YARN`, run the following. `SPARK_DIR` should be replaced with the directory of `Spark` and `SPARK_ML_DIR` should be replaced with the directory where this package resides. 
 * `SPARK_DIR/bin/spark-submit --master yarn --deploy-mode cluster --class spark_ml.sequoia_forest.SequoiaForestRunner --name SequoiaForestRunner --driver-memory 4G --executor-memory 4G --num-executors 10 SPARK_ML_DIR/target/scala-2.10/spark_ml-assembly-0.1.jar --inputPath /Datasets/mnist.tsv.gz --outputPath /ModelOutputs/mnist --numTrees 100 --numPartitions 10 --labelIndex 780`
 * This will train a classification forest with 100 trees using the column 780 as the label and all the other columns as numeric features. The final model would be stored in `/ModelOutputs/mnist` in HDFS.
7. Check status of training through the Hadoop job tracker page. `Spark` also provides its internal progress report when you click on the job's application master link from the job tracker page.
8. Once training is finished, you can predict on a new data set using the following command.
 * `SPARK_DIR/bin/spark-submit --master yarn --deploy-mode cluster --class spark_ml.sequoia_forest.SequoiaForestPredictor --name SequoiaForestPredictor --driver-memory 4G --executor-memory 4G --num-executors 4 SPARK_ML_DIR/target/scala-2.10/spark_ml-assembly-0.1.jar --inputPath /Datasets/mnist.t.tsv.gz --forestPath /ModelOutputs/mnist --outputPath /ModelOutputs/mnistpredictions --labelIndex 780 --outputFieldIndices 780 --pauseDuration 100`
 * The above command would predict on `mnist.t.tsv.gz` using the previously trained model in `/ModelOutputs/mnist` and write predictions under `/ModelOutputs/mnistpredictions`. It'll also write the value of the column 780 (which happens to be the label in this case) along with the predicted value. In the standard output log of the driver, you should also be able to see computed accuracy since the label is given in this case.
9. Training regression requires adding an argument `--forestType Variance`. Likewise, using categorical features requires adding an argument like `--categoricalFeatureIndices 5,6`. This would mean that columns 5 and 6 are to be treated as categorical features. For other options, refer to the command line arguments described below.

### Command Line Usage

First, compile `Spark` 1.0.1 against the appropriate Hadoop version. Then submit SequoiaForest using the `spark-submit` command.

`spark-submit [spark-submit options] --class spark_ml.sequoia_forest.SequoiaForestRunner --name SequoiaForestRunner spark_ml-assembly-0.1.jar --inputPath ... --outputPath ... --numTrees ... --numPartitions ... [optional arguments]`

#### Required Arguments

* **--inputPath** : The input path (wild card allowed) in HDFS. This should point to delimited text file(s) (e.g. csv/tsv) that are to be used as a training set. All columns that are to be used either as a label or features should be numbers (i.e. string labels or features are not allowed). Prior to running the algorithm, categorical values should be converted into 0, 1, 2, ..., K-1 if there are K classes.
* **--outputPath** : The output directory in HDFS where the trained forest will be saved.
* **--numTrees** : Number of trees to train.
* **--numPartitions** : The number of partitions to divide the data into. Recommended to be the same as the number of executors passed to spark-submit.

#### Optional Arguments

Arguments that represent column indices are zero-based (starts from 0 for the first column).

* **--validationPath** : An optional validation path (wild card allowed) in HDFS. The entire validation set would be loaded in memory and used in each iteration to measure performance, so this should be reasonably small. The file should be in the exact same format as the training set.
* **--delimiter** : The delimiter for columns in training/validation text files. The default value is *"\t"*.
* **--labelIndex** : The index of the column that would be used as the label. The default value is *0*.
* **--categoricalFeatureIndices** : A comma separated indices for categorical features in training/validation data. E.g., 3,5 would mean that columns 3 and 5 are to be used as categorical features. The default is an empty set (no features are categorical).
* **--indicesToIgnore** : A comma separated indices of columns that are to be ignored (i.e., won't be used as features or the label). The default is an empty set (no columns are ignored).
* **--forestType** : Either *InfoGain* (for classification) or *Variance* (for regression). The default value is *InfoGain*.
* **--discretizationType** : Type of discretization to do on features. Either *EqualWidth* or *EqualFrequency*. The default value is *EqualFrequency*.
* **--maxNumNumericBins** : Maximum number of bins to use when discretizing numeric features. If both numeric bin count and categorical cardinality are between *2* and *256*, Byte is used to represent features, substantially reducing memory requirement for caching. Otherwise, Short is used. The maximum value is *65536*. A smaller value could substantially speed up the training process (may or may not affect accuracy, depending on your data). The default value is *256*.
* **--maxCategoricalCardinality** : Maximum allowed cardinality for categorical features. If both numeric bin count and categorical cardinality are between *2* and *256*, Byte is used to represent features, substantially reducing memory requirement for caching. Otherwise, Short is used. The maximum value is *65536*. The default value is *256*.
* **--sampleWithReplacement** : A boolean value indicating whether bagging will be performed with-replacement or without-replacement. The default value is *true*.
* **--sampleRate** : The bagging sampling rate. Should be between *0* and *1*. The default is *1*.
* **--mtry** : Number of random features to use per tree node. The default value is *-1*. *-1* means that this will be automatically determined. For classification, *sqrt(number of features)* is used. For regression, *number of features / 3* is used.
* **--minSplitSize** : The minimum number of samples that a node should see to be eligible for splitting. The default is *2* (means trees will be fully grown) for classification and *10* for regression.
* **--maxDepth** : The maximum depth of the tree to be trained. The default is *-1* (means no limit on tree depth).

#### Advanced Arguments (for runtime performance)

* **--numRowFiltersPerIter** : The higher this number is, the more distributed node splits can be performed per iteration. However, it'll also consume more memory and network bandwidth to split more nodes per iteration. The default value *-1* means that this will be automatically determined.
* **--subTreeThreshold** : The threshold on the number of samples that a node should see before the node is trained as a sub-tree locally in an executor. The larger number means that sub-tree training would start earlier. However, it'd also require more memory per executor and would result in a larger amount of data getting shuffled. The default value *-1* means that this will be automatically determined.
* **--numSubTreesPerIter** : Number of sub trees to train per iteration. It could speed up the training process if more sub-trees are trained per iteration, along with more executors. The default value *-1* means that this will be automatically determined.
* **--pauseDuration** : Time to pause before terminating after training in seconds. This is useful for some `YARN` clusters where log messages are not stored after jobs are finished. The default is *0* (no pause).

### Predictor

This is a command line tool for predicting on new data using a previously trained forest.

`spark-submit [spark-submit options] --class spark_ml.sequoia_forest.SequoiaForestPredictor --name SequoiaForestPredictor spark_ml-assembly-0.1.jar --inputPath ... --forestPath ... --outputPath ... [optional arguments]`

#### Required Arguments

Arguments that represent column indices are zero-based (starts from 0 for the first column).

* **--inputPath** : The input path (wild card allowed) in HDFS. This should point to delimited text file(s) (e.g. csv/tsv) that a previously trained forest would predict on. It should contain all the features that were previously used for training in the same order as before. The label needs not exist, unless validation is required.
* **--outputPath** : The directory where the prediction outputs would be written. The indices chosen with the `--outputFieldIndices` option would also get written along with the prediction. E.g., one can write predictions of rows along with row identifiers.
* **--forestPath** : The directory in HDFS where the trained forest resides.

#### Optional Arguments

* **--delimiter** : Delimiter string for input/output data. The default is *"\t"*.
* **--labelIndex** : If the data set contains a label, this should be set to the label index for validations. The default is -1, meaning that there's no label.
* **--outputFieldIndices** : A comma separated indices of fields that you want to include with the prediction outputs. E.g., you may want to write predictions along with row identifiers. The default output field index is 0 (first column).
* **--indicesToIgnore** : A comma separated indices of columns to be ignored (to be excluded from features). All the other columns are to be used as either a label or features, so this should be set to match the features used in training. The default is empty (no columns are ignored).
* **--pauseDuration** : Time to pause before terminating in seconds. This is useful for some **YARN** clusters where log messages are not stored after jobs are finished. The default is *10* seconds.

### APIs

* To embed the training routine within a program, one can use **SequoiaForestTrainer.discretizeAndTrain** function to train on an `RDD` of `(Double, Array[Double])` tuples - the first element is the label and the second element is an array of features. Arguments are described in the comments. This function basically discretizes the input and performs bagging before calling the **SequoiaForestTrainer.train** function. If one wants more fine grained controls, one can customize discretization and bagging and then call the **SequoiaForestTrainer.train** function.
* To read stored models from HDFS, use the **SequoiaForestReader.readForest** function.
* To predict on new features, use the **predict** function on a **SequoiaForest** object.

### Benchmarks

The following numbers compare shallow single tree performance against `MLLib`'s decision tree (as in V1.0.1) for small data sets.
The numbers exclude pre-processing times, such as caching and discretizations. The data were repartitioned to match the number of executors.
For SequoiaForest, local sub-tree training was disabled to strictly compare distributed node split performance. Data were sampled 100% without replacement and all the features were used.

* **YearPredictionMSD** regression data set. 463715 rows. 90 features, 256 bins per feature. 3 Executors spread over 3 machines in AWS with SSD.
 1. SequoiaForest single tree depth 5 : 1.8 seconds.
 2. SequoiaForest single tree depth 10 : 17.5 seconds.
 3. `MLLib Decision Tree` depth 5 : 17 seconds.
 4. `MLLib Decision Tree` depth 10 : 106 seconds.

* Binarized **MNIST** classification data set. 1 million rows. 784 features. 256 bins per feature. 10 Executors spread over 5 machines in a local cluster with magnetic drives.
 1. SequoiaForest single tree depth 6 : 7.8 seconds.
 2. `MLLib Decision Tree` depth 6 : 101 seconds.

The benchmarks for full unpruned forests will follow.

Large data set (at least hundreds of millions of rows with a thousand features) benchmarks will follow.

### Hints/Caveats

* To see training progresses and status information, look at the standard output from the driver. It'll tell you how many trees are being trained and how many nodes have been trained, as well as the memory usage, etc. Individual executor standard output may also contain some information, such as memory usages, etc.
* To fully cache data, executors should have enough combined memory.
* Both driver and executors should have large amounts of memory if you want to process a large number of trees, a large number of large sub-trees, a large number of bins and target classes, more concurrent distributed node splits per second, etc.
* Locally training sub-trees in executors requires a lot of training data shuffling. Consequently, the shuffle performance is also bound by disk/network IOs. So it's usually not a good idea to run *too many* executors on the same machine.
* Currently, a categorical column with K distinct values has to be enumerated into 0...K-1 prior to training.
* The **SequoiaForestTrainer.discretizeAndTrain** function maps the input `RDD` into a new internal `RDD` and then caches that `RDD`. So it's not necessary to cache the input `RDD` of `(Double, Array[Double])` tuples.
* When training a large number of very large trees, do not store the models in memory unless you have *a lot* of memory in the driver. Excluding validation data from the command line options would make sure that tree nodes are directly written to the disk and not stored in memory.
* For typical problems, using unsigned `Byte` should be enough for bin counts. If you have certain categorical features that have cardinality larger than 256, you'll be forced to use unsigned `Short`, doubling memory requirements.
* The larger number of bins for features would result in slower performance.
* For classification, fully growing trees usually help. For regressions, some limits on tree growth seem to help (thus different defaults for **minSplitSize**).
* Currently, categorical feature splits are dumb and sub-optimal. They are split K-ways (where K is the cardinality of the feature), regardless of the type of the forest (classification and regression).
* The current implementation of the **SequoiaForestPredictor** command line tool is naive. For a large number of very large trees, the predictor would require a lot of memory in both driver and executors in order to fit the models in memory.

### Selecting a proper limit of tree sizes.

For classification, not limiting trees seems like a good idea. For regression, having some limits through minSplitSize or maxDepth may help.

## Gradient Boosted Trees

A powerful ensemble of boosted trees. Coming Soon.
