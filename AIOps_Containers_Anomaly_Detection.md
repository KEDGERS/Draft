## AIOps: Spice-up Containers Monitoring with Machine Learning

### BACKGROUND

If you are reading this paper, you have probably heard of containers. But if you haven’t, you can think of containers as easily configured, lightweight VMs that start up fast, often in under one second. They are designed to be short-lived and fragile, I know it seems odd to talk about system components that are designed to not be particularly resilient, but there’s a good reason for it - Instead of making each small computing component of a system bullet-proof, you can actually make the whole system a lot more stable by assuming each compute unit is going to fail and designing your overall process to handle it. 

Containers address several important operational problems, that is why they are taking the infrastructure world by storm. **But there is a problem:** containers come and go so frequently, and change so rapidly, that they can be an order of magnitude more difficult to monitor and operate than physical or virtual hosts.

There are patterns and events which disrupt the normal end-to-end behavior of the system, but we still need to figure out what the cause of disruptions are to fix whatever is ailing the system. Because of the complexity and high entropy of containers, seeing those patterns and being able to analyze them simply exceeds the capabilities of human operators. Yes, there may be a mathematical curve which describes what’s going on under the hood, but it is so complex that human beings are not able to come up with the equation to make sense of that curve and hence it is very difficult for them to figure out how to deal with it.

In this paper we describe how Artificial Intelligence for IT Operations (AIOps) enables enterprises to work with the telemetry data that is being collected from Containerized applications and see if that curve exists, and then come up with the equation that describes the curve.

### Massive Operational Complexity for Containerized Applications
If we are talking about containers nowadays, most people tend to think of the big blue whale or the white steering wheel on the blue background.
<p align="center"> <img src="https://miro.medium.com/max/805/1*72WozZ6G_vsox0PFNgWW8g.png"> </p>

Let’s put these thoughts aside and ask ourselves: What are containers in detail? If we look at the corresponding documentation of Kubernetes we only find explanations about [“Why to use containers?“](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/#why-containers) and lots of [references to Docker](https://kubernetes.io/docs/concepts/containers/images/). Docker itself explains containers as [“a standard unit of software“](https://www.docker.com/resources/what-container). Their explanations provide a general overview but do not reveal much of the underlying “magic“.

Eventually, people tend to imagine containers as lightweight virtual machines (VMs) that start up fast, which technically does not come close to the real world. A container is a lightweight virtual runtime, its primary purpose is to provide software isolation. A significant architectural shift toward containers is underway, and as with any architectural shift, that means new operational challenges. The well-understood challenges include orchestration, networking, and configuration—in fact there are many active software projects addressing these issues. However, the significant operational challenge of monitoring containers is much less well-understood. Most of the existing  monitoring solutions cover the traditional stack:

-   Application performance monitoring instruments your custom code to identify and pinpoint bottlenecks or errors

-   Infrastructure monitoring collects metrics about the host, such as CPU load and available memory

When you add containers to your stack, your world gets much, much more complex. In fact, it gets so complex that existing monitoring tools simply can’t explain your system, due to the following challenges:

- Containers come and go so frequently, and change so rapidly, that they can be more difficult to monitor and understand than physical or virtual hosts.

- Within containerized environments you might need  to monitor 150 metrics per Operating System, and for each container let's assume you collect 50 metrics plus another 50 metrics reported by an off-the-shelf component running in the container. (This is a conservative number, as we see customers collecting many more). In that case we would add 100 new metrics per container, Assuming the host runs 10 containers, the number of metrics we will collect is:
OS + (Containers per host * (Container + Off-the-shelf)) = 100 + (10 * (50 + 50)) = **1100 metrics per host**
With close to 1000 unique series being emitted, it is difficult to know which metrics to pay attention to.

If you are not addressing these challenges, you are left with two choices:

- Treat containers as hosts that come and go every few minutes. In this case your life is miserable because the monitoring system always thinks half of your infrastructure is on fire.

- Don’t track containers at all. You see what happens in the operating system and the app, but everything in the middle is a gap. In this case, you should expect a very painful ride if you are unable to identify performance bottlenecks at the Container layer of the stack. 

Instead, we need a new approach where we re-center monitoring around **proactively detecting anomalies for containerized applications** to determine how **performance bottlenecks** on the containers layer of the stack will ripple to the rest of the stack. 

###  Designing an Operating Model for Detecting   Performance Anomalies
Containers pose interesting challenges for performance monitoring and analysis, requiring new analysis methodologies and tooling. Resource-oriented analysis, as is common with systems performance tools, must now account for both hard limits and soft limits, as implemented using cgroups. A reverse diagnosis methodology can be applied to identify whether a container is resource constrained, and by which hard or soft resource. The interaction between the host and containers can also be examined, and noisy neighbors identified or exonerated. This section will walk you through our approach to identify bottlenecks in the host or container configuration, and how to dig deeper into container internals.  

First, Let's walk-through some anti-patterns to start with before diving into best practices of the reverse diagnosis approach:  

#### Performance Analysis: Anti-patterns:
These are the most common anti-patterns we observed as we work with customers throughout their Containers performance analysis journey:

**Streetlight method:** This comes from a parable about a drunk man who's looking for his keys under a streetlight and a police officer found him and asked "what are you doing?" and he said "I've lost my keys, I'm looking for them". The police officer asked "Did you lose them under the streetlight?" and the drunk says "No but that's where the light is best". We see this quite often in performance analysis, where people tune things at random until the problem goes away. You might end up going around in circles and you miss things because there are blind spots.

**Blame someone-else method:** That is about something that you are not responsible for, and you hypothesize that the problem must be a component owned by different team. We've seen this many times where people managing the network are blamed. e.g. either the network must retransmits or there's something wrong with BGP.

**Traffic light method:** Traffic lights are really easy to interpret, red is bad and green is good. Some people like to create these dashboards where they put colors on everything, colors are good for objective metrics such as errors. But performance analysis often times rely on subjective metrics like IOPS and latency, which might be good for one person who's running a chat server online and might be different for someone who's running a high frequency trading application.
 
 #### Performance Analysis: Reverse diagnosis method
 
Now that we have an understanding of anti-patterns, let's talk about the methodology we came up with to identify whether a container is resource constrained, enabling analyzing and tuning containers to be as fast and efficient as possible. The methodology is about enumerating the possible outcomes and then working backwards to identify which metrics are needed to diagnose one of these outcomes. 

Containers can rightly be classified as a type of mini-host. Just like a regular host, it runs work on behalf of resident software, and that work uses CPU, memory, I/O, and network resources. However, containers run inside cgroups which don’t report the exact same metrics you might expect from a host. Let's look at the case of identifying CPU performance bottleneck to illustrate the reverse diagnosis methodology using real world example.  

The key CPU resource metrics exposed in most container platforms are the following

| Name | Description | 
| ------------- | ------------- | 
| User CPU | Percent of time that CPU is under direct control of processes | 
| System CPU | Percent of time that CPU is executing system calls on behalf of processes |
| Throttling (count) | Number of CPU throttling enforcements for a container |
| Throttling (time) | Total time that a container's CPU usage was throttled |

An increase in CPU throttling time would be  identified due to the following root causes: 

 - Physical CPU is throttled 
 - Cap throttled 
 - Share throttled (Assuming physical CPU limited as well)
 - Not throttled
   
We came up with the following operating model to graphically present the root cause identification process for CPU performance bottleneck:
<p align="center"> <img src="https://github.com/c-Kedge/tmp/blob/master/CPU_Container.png?raw=true"> </p>

Basically, a walkthrough the operating model would look like the following:

 1. If throttling time is increasing, and the cap is throttled - then take that off the operating table straight away.  The metric tells us if the CPU cap has been hit, we know we're caps throttled. 
 2. If not, if non-voluntary context switches are not increasing for that container, it means we're not getting kicked off CPU and if we're not getting kicked off CPU we're probably not getting throttled so that would tend to put us into the Not throttled outcome unless you have some other theory and you need to dig into the kernel and debug further. 
 3. Next, if you have idle CPU but you're getting kicked off CPU then something interesting is happening like interrupts and it needs further digging. 
 4. If you don't have idle CPU and you're getting kicked off CPU, and if other tenants are not idle then you're going to have share contention
 5. Last, if they are the only tenant on this system then your physical CPU throttled.

A similar process should be followed for I/O, networking and memory: Recommendation is to start with the final outcomes (root causes) and then work backwards to come up with a differential diagnosis, so you can then identify the metrics related to one of those possible root causes. The process should be modeled as an operating map, showing a wizard that tells possible outcomes related to deviation for specific metrics.

### Designing the Dataset for Anomaly Detection

The method for anomaly detection presented in this paper is based on the fundamental principle of organizing all the containers in the system into multiple domains by centering  data partitioning on tags (labels). Since the same components across the system should behave similarly, doing the same tasks and running the same software, you want to group them together. e.g. components that are responsible for routing HTTP requests might have higher CPU usage and lower input/output operations per second (IOPS), whereas processes transferring huge amounts of data to or from a Container would have higher read operations per second. A set of performance metrics for each container is also collected at the time, and then each domain is going to be examined in order to find any outliers.

For each container in a specific domain, a set of attributes and performance metrics are collected. 

The attribute set for Container includes resource configuration and tags (Labels). The attribute set can be formalized by the following vector:  

![attributes.png](https://github.com/c-Kedge/tmp/blob/master/attributes.png?raw=true)

Where Ri represents an environment attribute, r is the number of attributes. 

The metrics set represents a X ∈ Rn where n is the number of metrics. The metrics set can be formalized as: 

![metrics.png](https://github.com/c-Kedge/tmp/blob/master/metrics.png?raw=true)

Where Xi is a real number representing the particular metric value for a particular container, and n is the number of metrics.

The matrix (defined below) of Containers in a certain monitoring domain is the important dataset of anomaly detection algorithms. Assume that an observed Container has n performance metrics, Xi, i = 1, …, n. Each metric can be considered as a random variable. These n metrics constitute a random vector, X. All the sample values of Xi (i = 1, …, n) in a point-in-time constitute a sample of X (denoted as x). Further, assume that totally l samples of all Containers in a monitoring domain are obtained in a certain time period. These l samples constitute an n-by-l original sample matrix, Xn×l, where each column (xi) represents a sample of all metrics of a container in a point-in-time. A domain performance metrics set is a set of sets where each column is a vector of a particular Containers metrics values. It can be formalized by the following matrix: 

![domain.png](https://github.com/c-Kedge/tmp/blob/master/domain.png?raw=true)
  
Where n is a number of metrics per Container and l is a number of Container in a domain. 

Let T be a training sample set representing the samples of all the Containers in a monitoring domain for a certain time period, 

![training.png](https://github.com/c-Kedge/tmp/blob/master/training.png?raw=true)

Where xi∈Rn is the input vector (or instance), yi is the output (or the label of xi), (xi, yi) is called a sample point, l is the number of samples. 

For the problem of detecting performance anomalies on containers in a certain monitoring domain, we defined the following classifier:

1) Binary classification: the task is to determine whether the state of a container represented by a sample is normal or abnormal, then

![Binary.png](https://github.com/c-Kedge/tmp/blob/master/Binary.png?raw=true)

When Yi = +1, Xi is called a positive sample; while when Yi = -1, Xi is called a negative sample. The goal is to find a real function g(x) in Rn, 
y = f(x) = sgn(g(x)), 
Such that f(x) derives the value of y for any sample x, where sgn() is the sign function. 

2) Multiclass classification: the task is to not only determine whether the state of a Container is normal or abnormal, but also determine the type of anomaly, then 

![multi.png](https://github.com/c-Kedge/tmp/blob/master/multi.png?raw=true)

c is the number of states including the normal state. The goal is to find a decision function f(x) in Rn

![multi-function.png](https://github.com/c-Kedge/tmp/blob/master/multi-function.png?raw=true)

Such that the class label y of any sample x can be predicted by y = f(x)

### Designing the Anomaly Detection Strategy

Anomaly detection for Containers in a certain monitoring domain  faces the following challenges. 

1) Multiple anomaly categories. Under Cloud environment, there are many factors that may cause anomalous performance of VMs. Anomalies of VMs are diversified. Therefore, in order to further detect the types of anomalies, anomaly detection of VMs should be considered as a multi-class classification problem. However, the existing researches in literature usually only determine the states of VMs as normal or abnormal (i.e., binary classification). 

2) Imbalanced training sample sets. In general, normal samples can be easily collected. Despite frequent occurrence, anomalies are still small probability events compared with normal states. Therefore, it is not easy to collect abnormal samples. When Cloud platform is newly deployed, or a monitoring domain is newly partitioned, the training sample set only contains normal samples. After the detection framework detects abnormal states and sends to the operator for verification, abnormal samples are gradually accumulated. Therefore, a perfect anomaly detection system should be able to deal with imbalanced training sample set. 

3) Increasing number of training samples. Since Cloud platform is a real production environment, the detection framework collects sample data of VMs in real-time. In order to accurately reflect the new trend of performance or state of VMs, the detected and verified samples should be added into the training sample set. The training of anomaly detection model usually requires much time. Therefore, the adopted anomaly detection algorithm should have the ability of online learning, i.e., the detection model can be updated only according to the newly added training samples. At the same time, some selected samples should be deleted to avoid the number of training samples exceeding the capability of training sample set. 

There is no universal detection algorithm which can solve all the problems of Containers anomaly detection. Therefore, to cope with above challenges, this paper designs strategies of selecting Support Vector Machines (SVM) based anomaly detection algorithm from a set of algorithms for different situations, which are summarized as follows:

1) If there are only normal samples, One Class SVM (OCSVM) is chosen. These situations include newly deployed Cloud platforms or newly partitioned monitoring domains. Since there is no training sample set, the solution collects some samples and sends to the operator, there are only normal samples without abnormal ones in an initial period of running time. 

2) If the ratio of one kind of samples is below a certain threshold (e.g., the proportion of the number of minority class to the total number of training sample set is less than 5%), i.e., the training sample set is imbalanced, imbalanced SVM is chosen. Imbalanced SVM can effectively solve the problem of imbalanced classification, thus improving the accuracy of anomaly detection. 

3) If there are multiple anomaly categories, and the ratio of the number of each category exceeds a certain value, multi-class SVM is chosen. Along with the operation of Cloud platform, various anomaly samples are detected and sent to the operator for verification, thus gradually accumulating a training sample set which contains all kinds of anomalies. Then the solution switches to multi-class SVM. 

4) When the solution stably operates for a period of time, and the number of training samples reaches a certain value (such as 30% VT), then online learning SVM is switched. Since then the solution still collects all kinds of samples in real time (part of the samples may be added to the training sample set to update the anomaly detection model). The incremental learning process updates the anomaly detection model with small cost, while the decremental learning process ensures that the training sample size will not exceed the capacity limit. 

### Designing the Anomaly Detection Solution

In order to implement environment-aware detection and improve the detection accuracy, we followed the following detection approach:

1) Collect all the Containers running environment attributes and performance metrics at the same time. 
2) Partition all the Containers into monitoring domains based on tags (Labels). 
3) In each domain, the equipped anomaly detection module detects anomalous Containers based on their performance metrics.

The solution is composed of several modules, including Data Collection, Data partitioning, Data Processing and Environment-aware Detection. The function of each module is detailed as follows:

- Data Collection is responsible for collecting the performance metric data and environment attribute of all Containers and transmitting to the upper module.

- Data Partitioning is responsible for partitioning all the Containers into several monitoring domains according to attribute set.

- Data Processing is responsible for indispensable processing including feature extraction on collected data. Before anomaly detection, feature selection is executed on the performance metric data to reduce data dimensionality.

- Environment-aware detection is responsible for detecting anomalous Containers and fault diagnosis.

<p align="center"> <img src="https://github.com/c-Kedge/tmp/blob/master/solutionDiag.png?raw=true"> </p>



### Bring all together : Implementation
*Work in Progress...*

#### Data Collection:
Prometheus at scale = Prometheus + Thanos

#### Data Partitioning:
Data Domain + Anomaly Types (Trend, Seasonality, Cyclicity, Irregurality)

#### Data Processing:


Analyzing Prometheus Data with Spark
For a better understanding of the structure of prometheus data types have a look at Prometheus Metric Types, especially the difference between Summaries and Histograms
The measurements are stored in Ceph. Let's examine what we have stored.
In [1]:
# install and load necessary packages
!pip install seaborn

import pyspark
from datetime import datetime
import seaborn as sns
import sys
import matplotlib.pyplot as plt
%matplotlib inline
import numpy as np

print('Python version ' + sys.version)
print('Spark version: ' + pyspark.__version__)

Requirement already satisfied: seaborn in /opt/app-root/lib/python3.6/site-packages (0.8.1)
Python version 3.6.3 (default, Mar 20 2018, 13:50:41) 
[GCC 4.8.5 20150623 (Red Hat 4.8.5-16)]
Spark version: 2.2.1

To Do: List Metrics and allow user to choose specific Metric

Establish Connection to Spark Cluster
set configuration so that the Spark Cluster communicates with Ceph and reads a chunk of data.
In [2]:
import string 
import random

# Set the configuration
# random string for instance name
inst = ''.join(random.choices(string.ascii_uppercase + string.digits, k=4))
AppName = inst + ' - Ceph S3 Prometheus JSON Reader'
conf = pyspark.SparkConf().setAppName(AppName).setMaster('spark://spark-cluster.dh-prod-analytics-factory.svc:7077')
print("Application Name: ", AppName)

# specify number of nodes need (1-5)
conf.set("spark.cores.max", "8")

# specify Spark executor memory (default is 1gB)
conf.set("spark.executor.memory", "4g")

# Set the Spark cluster connection
sc = pyspark.SparkContext.getOrCreate(conf) 

# Set the Hadoop configurations to access Ceph S3
import os
(ceph_key, ceph_secret, ceph_host) = (os.getenv('DH_CEPH_KEY'), os.getenv('DH_CEPH_SECRET'), os.getenv('DH_CEPH_HOST'))
ceph_key = 'DTG5R3EEWN9JBYJZH0DF'
ceph_secret = 'pdcEGFERILlkRDGrCSxdIMaZVtNCOKvYP4Gf2b2x'
ceph_host = 'http://storage-016.infra.prod.upshift.eng.rdu2.redhat.com:8080'
sc._jsc.hadoopConfiguration().set("fs.s3a.access.key", ceph_key) 
sc._jsc.hadoopConfiguration().set("fs.s3a.secret.key", ceph_secret) 
sc._jsc.hadoopConfiguration().set("fs.s3a.endpoint", ceph_host) 

#Get the SQL context
sqlContext = pyspark.SQLContext(sc)

Application Name:  M4JZ - Ceph S3 Prometheus JSON Reader

Metric variables to be specified for analysis
NOTE: these variables are dependent on the metric and how we want to filter the data.
In [3]:
# specify metric we want to analyze
metric_name = 'kubelet_docker_operations_latency_microseconds'

# choose a label
label = ""

# specify for histogram metric type
bucket_val = '0.3'

# specify for summary metric type
quantile_val = '0.99'

# specify any filtering when collected the data
# For example:
# If I want to just see data from a specific host, specify "metric.hostname='free-stg-master-03fb6'"
where_labels = ["metric.hostname='free-stg-master-03fb6'"]

Read Metric data from Ceph and detect metric type
In [4]:
jsonUrl = "s3a://DH-DEV-PROMETHEUS-BACKUP/prometheus-openshift-devops-monitor.1b7d.free-stg.openshiftapps.com/" + metric_name

try:
    jsonFile_sum = sqlContext.read.option("multiline", True).option("mode", "PERMISSIVE").json(jsonUrl + '_sum/')
    jsonFile = sqlContext.read.option("multiline", True).option("mode", "PERMISSIVE").json(jsonUrl + '_count/')
    try:
        jsonFile_bucket = sqlContext.read.option("multiline", True).option("mode", "PERMISSIVE").json(jsonUrl + '_bucket/')
        metric_type = 'histogram'
    except:
        jsonFile_quantile = sqlContext.read.option("multiline", True).option("mode", "PERMISSIVE").json(jsonUrl+'/')
        metric_type = 'summary'
except:
    jsonFile = sqlContext.read.option("multiline", True).option("mode", "PERMISSIVE").json(jsonUrl+'/')
    metric_type = 'gauge or counter'

#Display the schema of the file
print("Metric Type: ", metric_type)
print("Schema:")
jsonFile.printSchema()

Metric Type:  summary
Schema:
root
 |-- metric: struct (nullable = true)
 |    |-- __name__: string (nullable = true)
 |    |-- beta_kubernetes_io_arch: string (nullable = true)
 |    |-- beta_kubernetes_io_fluentd_ds_ready: string (nullable = true)
 |    |-- beta_kubernetes_io_instance_type: string (nullable = true)
 |    |-- beta_kubernetes_io_os: string (nullable = true)
 |    |-- clam_controller_enabled: string (nullable = true)
 |    |-- clam_server_enabled: string (nullable = true)
 |    |-- failure_domain_beta_kubernetes_io_region: string (nullable = true)
 |    |-- failure_domain_beta_kubernetes_io_zone: string (nullable = true)
 |    |-- fluentd_test: string (nullable = true)
 |    |-- hostname: string (nullable = true)
 |    |-- image_inspector_enabled: string (nullable = true)
 |    |-- instance: string (nullable = true)
 |    |-- job: string (nullable = true)
 |    |-- kubernetes_io_hostname: string (nullable = true)
 |    |-- logging_infra_fluentd: string (nullable = true)
 |    |-- node_role_kubernetes_io_compute: string (nullable = true)
 |    |-- node_role_kubernetes_io_infra: string (nullable = true)
 |    |-- node_role_kubernetes_io_master: string (nullable = true)
 |    |-- operation_type: string (nullable = true)
 |    |-- ops_node: string (nullable = true)
 |    |-- placement: string (nullable = true)
 |    |-- region: string (nullable = true)
 |    |-- type: string (nullable = true)
 |-- values: array (nullable = true)
 |    |-- element: array (containsNull = true)
 |    |    |-- element: string (containsNull = true)


UI for selecting interface
In [5]:
labels = []
for i in jsonFile.schema["metric"].jsonValue()["type"]["fields"]:
    labels.append(i["name"])

print("number of labels: ", len(labels))
print("\n===== Labels =====")
inc = 0
for i in labels:
    inc = inc+1
    print(inc, "\t", i)
    
prompt = "\n===== Select a Label (specify number from 0 to " + str(len(labels)) + "\n0 indicates no label to select\n"
label_num = int(input(prompt))
if label_num != 0:
    label = labels[label_num - 1]
print("\n===== Label Selected: ",label)

number of labels:  24

===== Labels =====
1 	 __name__
2 	 beta_kubernetes_io_arch
3 	 beta_kubernetes_io_fluentd_ds_ready
4 	 beta_kubernetes_io_instance_type
5 	 beta_kubernetes_io_os
6 	 clam_controller_enabled
7 	 clam_server_enabled
8 	 failure_domain_beta_kubernetes_io_region
9 	 failure_domain_beta_kubernetes_io_zone
10 	 fluentd_test
11 	 hostname
12 	 image_inspector_enabled
13 	 instance
14 	 job
15 	 kubernetes_io_hostname
16 	 logging_infra_fluentd
17 	 node_role_kubernetes_io_compute
18 	 node_role_kubernetes_io_infra
19 	 node_role_kubernetes_io_master
20 	 operation_type
21 	 ops_node
22 	 placement
23 	 region
24 	 type

===== Select a Label (specify number from 0 to 24
0 indicates no label to select
23

===== Label Selected:  region

Preprocessing Prometheus Data
Once we have chosen a metric to focus on, filter specific data into a data frame and preprocess data.
In [6]:
import pyspark.sql.functions as F
from pyspark.sql.types import StringType
from pyspark.sql.types import IntegerType

# create function to convert POSIX timestamp to local date
def convert_timestamp(t):
    return str(datetime.fromtimestamp(int(t)))

def format_df(df):
    #reformat data by timestamp and values
    df = df.withColumn("values", F.explode(df.values))
    
    df = df.withColumn("timestamp", F.col("values").getItem(0))
    df = df.sort("timestamp", ascending=True)
    
    df = df.withColumn("values", F.col("values").getItem(1))

    # drop null values
    df = df.na.drop(subset=["values"])
    
    # cast values to int
    df = df.withColumn("values", df.values.cast("int"))

    # define function to be applied to DF column
    udf_convert_timestamp = F.udf(lambda z: convert_timestamp(z), StringType())

    # convert timestamp values to datetime timestamp
    df = df.withColumn("timestamp", udf_convert_timestamp("timestamp"))

    # calculate log(values) for each row
    df = df.withColumn("log_values", F.log(df.values))
    
    return df

Take data from Ceph json and parse into Spark Dataframe
We take the data from the json file located in Ceph and query it into a Spark df. This df is then parsed using the function above.
In [7]:
def extract_from_json(json, name, select_labels, where_labels):
    #Register the created SchemaRDD as a temporary variable
    json.registerTempTable(name)
    
    #Filter the results into a data frame

    query = "SELECT values"
    
    # check if select labels are specified and add query condition if appropriate
    if len(select_labels) > 0:
        query = query + ", " + ", ".join(select_labels)
        
    query = query + " FROM " + name
    
    # check if where labels are specified and add query condition if appropriate
    if len(where_labels) > 0:
        query = query + " WHERE " + " AND ".join(where_labels)

    print(query)
    data = sqlContext.sql(query)

    # sample data to make it more manageable
    # data = data.sample(False, fraction = 0.05, seed = 0)
    # TODO: get rid of this hack
    data = sqlContext.createDataFrame(data.head(1000), data.schema)
    
    return format_df(data)
In [8]:
if label != "":
    select_labels = ['metric.' + label]
else:
    select_labels = []

# get data and format
data = extract_from_json(jsonFile, metric_name, select_labels, where_labels)

data.count()
data.show()

SELECT values, metric.region FROM kubelet_docker_operations_latency_microseconds WHERE metric.hostname='free-stg-master-03fb6'
+------+---------+-------------------+------------------+
|values|   region|          timestamp|        log_values|
+------+---------+-------------------+------------------+
| 44588|us-east-2|2018-02-10 23:59:59|10.705220043509515|
| 77849|us-east-2|2018-02-10 23:59:59|11.262526331964487|
|    88|us-east-2|2018-02-10 23:59:59| 4.477336814478207|
|     2|us-east-2|2018-02-10 23:59:59|0.6931471805599453|
|    29|us-east-2|2018-02-10 23:59:59| 3.367295829986474|
|   417|us-east-2|2018-02-10 23:59:59|6.0330862217988015|
|    16|us-east-2|2018-02-10 23:59:59| 2.772588722239781|
|  1003|us-east-2|2018-02-10 23:59:59| 6.910750787961936|
|    44|us-east-2|2018-02-10 23:59:59| 3.784189633918261|
|    29|us-east-2|2018-02-10 23:59:59| 3.367295829986474|
|964458|us-east-2|2018-02-10 23:59:59|13.779321564501078|
|    16|us-east-2|2018-02-11 00:23:58| 2.772588722239781|
| 44838|us-east-2|2018-02-11 00:23:58|10.710811273158345|
|     2|us-east-2|2018-02-11 00:23:58|0.6931471805599453|
|    44|us-east-2|2018-02-11 00:23:58| 3.784189633918261|
|  1008|us-east-2|2018-02-11 00:23:58| 6.915723448631314|
|969736|us-east-2|2018-02-11 00:23:58| 13.78477914848751|
|    29|us-east-2|2018-02-11 00:23:58| 3.367295829986474|
|    88|us-east-2|2018-02-11 00:23:58| 4.477336814478207|
|   417|us-east-2|2018-02-11 00:23:58|6.0330862217988015|
+------+---------+-------------------+------------------+
only showing top 20 rows


Calculate Sampling Rate
In [20]:
def calculate_sample_rate(df):
    # define function to be applied to DF column
    udf_timestamp_hour = F.udf(lambda dt: int(datetime.strptime(dt,'%Y-%m-%d %X').hour), IntegerType())

    # convert timestamp values to datetime timestamp

    # new df with hourly value count
    vals_per_hour = df.withColumn("hour", udf_timestamp_hour("timestamp")).groupBy("hour").count()

    # average density (samples/hour)
    avg = vals_per_hour.agg(F.avg(F.col("count"))).head()[0]
    print("average hourly sample count: ", avg)
    
    # sort and display hourly count
    vals_per_hour.sort("hour").show()
    
calculate_sample_rate(data)

average hourly sample count:  55633.5
+----+-----+
|hour|count|
+----+-----+
|   0|55441|
|   1|55352|
|   2|55346|
|   3|55331|
|   4|55452|
|   5|55463|
|   6|55452|
|   7|55463|
|   8|55452|
|   9|55463|
|  10|55452|
|  11|55463|
|  12|55452|
|  13|55504|
|  14|55470|
|  15|55691|
|  16|55978|
|  17|56078|
|  18|56052|
|  19|56068|
+----+-----+
only showing top 20 rows


Calculate number of values per Label
In [28]:
def calculate_vals_per_label(df):
    # new df with vals per label
    df.groupBy(label).count().show()
    
calculate_vals_per_label(data)

+---------+-------+
|   region|  count|
+---------+-------+
|us-east-2|1335204|
+---------+-------+

In [10]:
from pyspark.sql.window import Window

def get_deltas(df):
    df_lag = df.withColumn('prev_vals',
                        F.lag(df['values'])
                                 .over(Window.partitionBy("timestamp").orderBy("timestamp")))

    result = df_lag.withColumn('deltas', (df_lag['values'] - df_lag['prev_vals']))
    result = result.drop("prev_vals")
    
    max_delta = result.agg(F.max(F.col("deltas"))).head()[0]
    min_delta = result.agg(F.min(F.col("deltas"))).head()[0]
    mean_delta = result.agg(F.avg(F.col("deltas"))).head()[0]
    
    return max_delta, min_delta, mean_delta, result
In [11]:
max_delta, min_delta, mean_delta, data = get_deltas(data)
data.show()
print("Max delta:", max_delta)
print("Min delta:", min_delta)
print("Mean delta:", mean_delta)

+------+--------------------------------------+-------------------+------------------+-------+
|values|failure_domain_beta_kubernetes_io_zone|          timestamp|        log_values| deltas|
+------+--------------------------------------+-------------------+------------------+-------+
|    18|                            us-east-2a|2018-03-03 04:28:59|2.8903717578961645|   null|
|     2|                            us-east-2a|2018-03-03 04:28:59|0.6931471805599453|    -16|
|   279|                            us-east-2a|2018-03-03 04:28:59| 5.631211781821365|    277|
|   495|                            us-east-2a|2018-03-03 04:28:59|  6.20455776256869|    216|
|386700|                            us-east-2a|2018-03-03 04:28:59|12.865404477595389| 386205|
| 18266|                            us-east-2a|2018-03-03 04:28:59| 9.812796687251625|-368434|
|    11|                            us-east-2a|2018-03-03 04:28:59|2.3978952727983707| -18255|
|     6|                            us-east-2a|2018-03-03 04:28:59| 1.791759469228055|     -5|
|    18|                            us-east-2a|2018-03-03 04:28:59|2.8903717578961645|     12|
|    21|                            us-east-2a|2018-03-03 04:28:59| 3.044522437723423|      3|
| 31711|                            us-east-2a|2018-03-03 04:28:59|10.364418902828275|  31690|
|    18|                            us-east-2a|2018-03-03 08:18:59|2.8903717578961645|   null|
|     2|                            us-east-2a|2018-03-03 08:18:59|0.6931471805599453|    -16|
|   279|                            us-east-2a|2018-03-03 08:18:59| 5.631211781821365|    277|
|   541|                            us-east-2a|2018-03-03 08:18:59| 6.293419278846481|    262|
|437178|                            us-east-2a|2018-03-03 08:18:59|12.988095713798836| 436637|
| 20649|                            us-east-2a|2018-03-03 08:18:59| 9.935422171066474|-416529|
|    11|                            us-east-2a|2018-03-03 08:18:59|2.3978952727983707| -20638|
|     6|                            us-east-2a|2018-03-03 08:18:59| 1.791759469228055|     -5|
|    18|                            us-east-2a|2018-03-03 08:18:59|2.8903717578961645|     12|
+------+--------------------------------------+-------------------+------------------+-------+
only showing top 20 rows

Max delta: 6576011
Min delta: -6272578
Mean delta: 16174.62638770783
In [12]:
# rhmax = current value / max(values)
# if rhmax (of new point) >> 1, we can assume that the point is an anomaly
def get_rhmax(df):
    real_max = df.agg(F.max(F.col("values"))).head()[0]
    result = df.withColumn("rhmax", df["values"]/real_max)
    return result

result = get_rhmax(data)
result.show()

+------+--------------------------------------+-------------------+------------------+-------+--------------------+
|values|failure_domain_beta_kubernetes_io_zone|          timestamp|        log_values| deltas|               rhmax|
+------+--------------------------------------+-------------------+------------------+-------+--------------------+
|    18|                            us-east-2a|2018-03-03 04:28:59|2.8903717578961645|   null|2.734607690901823...|
|     2|                            us-east-2a|2018-03-03 04:28:59|0.6931471805599453|    -16|3.038452989890915E-7|
|   279|                            us-east-2a|2018-03-03 04:28:59| 5.631211781821365|    277|4.238641920897826...|
|   495|                            us-east-2a|2018-03-03 04:28:59|  6.20455776256869|    216|7.520171149980014E-5|
|386700|                            us-east-2a|2018-03-03 04:28:59|12.865404477595389| 386205| 0.05874848855954084|
| 18266|                            us-east-2a|2018-03-03 04:28:59| 9.812796687251625|-368434|0.002775019115667...|
|    11|                            us-east-2a|2018-03-03 04:28:59|2.3978952727983707| -18255|1.671149144440003...|
|     6|                            us-east-2a|2018-03-03 04:28:59| 1.791759469228055|     -5|9.115358969672745E-7|
|    18|                            us-east-2a|2018-03-03 04:28:59|2.8903717578961645|     12|2.734607690901823...|
|    21|                            us-east-2a|2018-03-03 04:28:59| 3.044522437723423|      3|3.190375639385460...|
| 31711|                            us-east-2a|2018-03-03 04:28:59|10.364418902828275|  31690|0.004817619138121541|
|    18|                            us-east-2a|2018-03-03 08:18:59|2.8903717578961645|   null|2.734607690901823...|
|     2|                            us-east-2a|2018-03-03 08:18:59|0.6931471805599453|    -16|3.038452989890915E-7|
|   279|                            us-east-2a|2018-03-03 08:18:59| 5.631211781821365|    277|4.238641920897826...|
|   541|                            us-east-2a|2018-03-03 08:18:59| 6.293419278846481|    262|8.219015337654925E-5|
|437178|                            us-east-2a|2018-03-03 08:18:59|12.988095713798836| 436637| 0.06641724006072652|
| 20649|                            us-east-2a|2018-03-03 08:18:59| 9.935422171066474|-416529|0.003137050789412875|
|    11|                            us-east-2a|2018-03-03 08:18:59|2.3978952727983707| -20638|1.671149144440003...|
|     6|                            us-east-2a|2018-03-03 08:18:59| 1.791759469228055|     -5|9.115358969672745E-7|
|    18|                            us-east-2a|2018-03-03 08:18:59|2.8903717578961645|     12|2.734607690901823...|
+------+--------------------------------------+-------------------+------------------+-------+--------------------+
only showing top 20 rows


Check Metric Type
Determine the metric type by checking filenames and data. For deciphering between counter and gauge, the only way to tell the difference is looking at the raw data. If the data is monotonically increasing, it is a counter, otherwise, it's a gauge. https://prometheus.io/docs/concepts/metric_types/
Histogram characteristics:
has a _count file (check criteria)
has a _sum file (check criteria)
has a _bucket file (check criteria)
Summary characteristics:
has a _count file (check criteria)
has a _sum file (check criteria)
does not have a _bucket file (check criteria)
has a quantile label
Counter characteristics:
monotonically increasing (check criteria)
Gauge characteristics:
no specific characteristics, so we assume that gauges have none of the above

Decifer between gauge and counter
If the metric type is counter, create new column that is the derivative of the values
In [10]:
def gauge_counter_separator(df):
    vals = np.array(df.select("values").collect())
    diff = vals - np.roll(vals, 1) # this value - previous value (should always be zero or positive for counter)
    diff[0] = 0 # ignore first difference, there is no value before the first
    diff[np.where(vals == 0)] = 0
    # check if these are any negative differences, if not then metric is a counter.
    # if counter, we must convert it to a gauge by keeping the derivatives
    if ((diff < 0).sum() == 0):
        metric_type = 'counter'
    else:
        metric_type = 'gauge'
    return metric_type, df

if metric_type == "gauge or counter":
    metric_type, data = gauge_counter_separator(data)
In [11]:
print("Metric type: ", metric_type)

Metric type:  summary

Load data based on Histogram or Summary Metric
In [12]:
if metric_type == 'histogram':
    data_sum = extract_from_json(jsonFile_sum, metric_name, select_labels, where_labels)
    
    select_labels.append("metric.le")
    data_bucket = extract_from_json(jsonFile_bucket, metric_name, select_labels, where_labels)
    
    # filter by specific le value
    data_bucket = data_bucket.filter(bucket_val)
    
    data_sum.show()
    data_bucket.show()
    
elif metric_type == 'summary':
    # get metric sum data
    data_sum = extract_from_json(jsonFile_sum, metric_name, select_labels, where_labels)
    
    # get metric quantile data
    select_labels.append("metric.quantile")
    data_quantile = extract_from_json(jsonFile_quantile, metric_name, select_labels, where_labels)
    
    # filter by specific quantile value
    data_quantile = data_quantile.filter(data_quantile.quantile == quantile_val)
    # get rid of NaN values once again. This is required once filtering takes place
    data_quantile = data_quantile.na.drop(subset='values')
    
    data_sum.show()
    data_quantile.show()

SELECT values FROM kubelet_docker_operations_latency_microseconds
SELECT values, metric.quantile FROM kubelet_docker_operations_latency_microseconds
+------+-------------------+------------------+
|values|          timestamp|        log_values|
+------+-------------------+------------------+
| 36338|2018-05-29 23:59:59|10.500619304662386|
| 36338|2018-05-30 00:00:59|10.500619304662386|
| 36338|2018-05-30 00:01:59|10.500619304662386|
| 36338|2018-05-30 00:02:59|10.500619304662386|
| 36338|2018-05-30 00:03:59|10.500619304662386|
| 36338|2018-05-30 00:04:59|10.500619304662386|
| 36338|2018-05-30 00:05:59|10.500619304662386|
| 36338|2018-05-30 00:06:59|10.500619304662386|
| 36338|2018-05-30 00:07:59|10.500619304662386|
| 36338|2018-05-30 00:08:59|10.500619304662386|
| 36338|2018-05-30 00:09:59|10.500619304662386|
| 36338|2018-05-30 00:10:59|10.500619304662386|
| 36338|2018-05-30 00:11:59|10.500619304662386|
| 36338|2018-05-30 00:12:59|10.500619304662386|
| 36338|2018-05-30 00:13:59|10.500619304662386|
| 36338|2018-05-30 00:14:59|10.500619304662386|
| 36338|2018-05-30 00:15:59|10.500619304662386|
| 36338|2018-05-30 00:16:59|10.500619304662386|
| 36338|2018-05-30 00:17:59|10.500619304662386|
| 36338|2018-05-30 00:18:59|10.500619304662386|
+------+-------------------+------------------+
only showing top 20 rows

+------+--------+-------------------+------------------+
|values|quantile|          timestamp|        log_values|
+------+--------+-------------------+------------------+
| 36762|    0.99|2018-05-30 15:00:59| 10.51221998195371|
| 36762|    0.99|2018-05-30 15:01:59| 10.51221998195371|
| 36762|    0.99|2018-05-30 15:02:59| 10.51221998195371|
| 36762|    0.99|2018-05-30 15:03:59| 10.51221998195371|
| 36762|    0.99|2018-05-30 15:04:59| 10.51221998195371|
| 36762|    0.99|2018-05-30 15:05:59| 10.51221998195371|
| 36762|    0.99|2018-05-30 15:06:59| 10.51221998195371|
| 36762|    0.99|2018-05-30 15:07:59| 10.51221998195371|
| 36762|    0.99|2018-05-30 15:08:59| 10.51221998195371|
| 36762|    0.99|2018-05-30 15:09:59| 10.51221998195371|
| 36762|    0.99|2018-05-30 15:10:59| 10.51221998195371|
|261262|    0.99|2018-05-30 15:15:59|12.473279014220623|
|261262|    0.99|2018-05-30 15:16:59|12.473279014220623|
|261262|    0.99|2018-05-30 15:17:59|12.473279014220623|
|261262|    0.99|2018-05-30 15:18:59|12.473279014220623|
|261262|    0.99|2018-05-30 15:19:59|12.473279014220623|
|261262|    0.99|2018-05-30 15:20:59|12.473279014220623|
|261262|    0.99|2018-05-30 15:21:59|12.473279014220623|
|261262|    0.99|2018-05-30 15:22:59|12.473279014220623|
|261262|    0.99|2018-05-30 15:23:59|12.473279014220623|
+------+--------+-------------------+------------------+
only showing top 20 rows


Descriptive Stats
The portion of statistics dedicated to summarizing a total population
Mean
Arithmetic average of a range of values or quantities, computed by dividing the total of all values by the number of values.
Variance
In the same way that the mean is used to describe the central tendency, variance is intended to describe the spread. The xi – μ is called the “deviation from the mean”, making the variance the squared deviation multiplied by 1 over the number of samples. This is why the square root of the variance, σ, is called the standard deviation.
Standard Deviation
Standard deviation (SD, also represented by the Greek letter sigma σ or the Latin letter s) is a measure that is used to quantify the amount of variation or dispersion of a set of data values.[1] A low standard deviation indicates that the data points tend to be close to the mean (also called the expected value) of the set, while a high standard deviation indicates that the data points are spread out over a wider range of values.
Median
Denotes value or quantity lying at the midpoint of a frequency distribution of observed values or quantities, such that there is an equal probability of falling above or below it. Simply put, it is the middle value in the list of numbers. The median is a better choice when the indicator can be affected by some outliers.
In [13]:
def get_stats(df):
    # calculate mean
    mean = df.agg(F.avg(F.col("values"))).head()[0]
    
    # calculate variance
    var = df.agg(F.variance(F.col("values"))).head()[0]

    # calculate standard deviation
    stddev = df.agg(F.stddev(F.col("values"))).head()[0]

    # calculate median
    median = float(df.approxQuantile("values", [0.5], 0.25)[0])
    
    return mean, var, stddev, median
    
mean, var, stddev, median = get_stats(data)

print("\tMean(values): ", mean)
print("\tVariance(values): ", var)
print("\tStddev(values): ", stddev)
print("\tMedian(values): ", median)

	Mean(values):  67087.9063346175
	Variance(values):  56691431555.4375
	Stddev(values):  238099.62527361838
	Median(values):  628.0

Histogram
The most common representation of a distribution is a histogram, which is a graph that shows the frequency or probability of each value. Plots will be generated by operation type

We will use Seaborn module for this. Kernel Density Estimation * will be added for smoothing.
In statistics, kernel density estimation (KDE) is a non-parametric way to estimate the probability density function of a random variable. Kernel density estimation is a fundamental data smoothing problem where inferences about the population are made, based on a finite data sample.
The kernel density estimate may be less familiar, but it can be a useful tool for plotting the shape of a distribution. Like the histogram, the KDE plots encodes the density of observations on one axis with height along the other axis:
Log normalization is required because, for different operations, values seems to be in very different scales. We can use log normalization for only positive values. We filter out the values that are zero
In [14]:
sns.set(color_codes = True)

def plot_hist(df, axis_label):
    vals = np.array(df.select("values").collect())
    
    if np.count_nonzero(vals) == 0:
        return "Error: All values are zero"
    
    # log normalization for postiive numbers only
    log_vals = np.log(vals[vals != 0])

    # plot both the distribution and the normalized distribution
    fig, ax = plt.subplots(nrows=1, ncols=2, figsize=(15,12))
    sns.distplot(vals, kde=True, ax=ax[0], axlabel=axis_label)
    sns.distplot(log_vals, kde=True, ax=ax[1], axlabel = "log transformed "+ axis_label)
    fig.show()

plot_hist(data, metric_name)

/opt/app-root/lib/python3.6/site-packages/matplotlib/axes/_axes.py:6462: UserWarning: The 'normed' kwarg is deprecated, and has been replaced by the 'density' kwarg.
  warnings.warn("The 'normed' kwarg is deprecated, and has been "
/opt/app-root/lib/python3.6/site-packages/matplotlib/figure.py:459: UserWarning: matplotlib is currently using a non-GUI backend, so cannot show the figure
  "matplotlib is currently using a non-GUI backend, "

None

 

Box-Whisker
Box plots may also have lines extending vertically from the boxes (whiskers) indicating variability outside the upper and lower quartiles, hence the terms box-and-whisker plot and box-and-whisker diagram. Outliers may be plotted as individual points.
In [15]:
# df is the Spark dataframe
# filter label is x-axis, ie. the metric label which we want to categorize the data by
def plot_box_whisker(df, filter_label):
    plt.figure(figsize=(20,15))
    df = df.withColumn("log_values", F.log(df.log_values))
    ax = sns.boxplot(x=filter_label, y="log_values", hue=filter_label, data=df.toPandas())  # RUN PLOT   

    plt.show()
    plt.clf()
    plt.close()

if label != "":
    plot_box_whisker(data, label)

Finding trend in time series, if there any
Trend means, if over time values have increasing or decreasing pattern. Before doing anamoly detection, we have to remove trend or will have to use trend tolerable models, to handle the false positives
In [16]:
#TODO: downsampling
data = data.sort("timestamp", ascending=True)
vals = np.array(data.select("values").collect())
plt.plot(vals)
plt.xlabel("timestamp")
plt.ylabel("value")
Out[16]:
Text(0,0.5,'value')

 
In [17]:
# be sure to stop the Spark Session to conserve resources
sc.stop()

#### Environment-aware Detect
Jupyter

