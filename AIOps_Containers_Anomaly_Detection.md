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
{
 "cells": [
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# Analyzing Prometheus Data with Spark\n",
    "For a better understanding of the structure of prometheus data types have a look at Prometheus Metric Types, especially the difference between Summaries and Histograms\n",
    "\n",
    "The measurements are stored in Ceph. Let's examine what we have stored."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 1,
   "metadata": {
    "scrolled": true
   },
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "Requirement already satisfied: seaborn in /opt/app-root/lib/python3.6/site-packages (0.8.1)\n",
      "Python version 3.6.3 (default, Mar 20 2018, 13:50:41) \n",
      "[GCC 4.8.5 20150623 (Red Hat 4.8.5-16)]\n",
      "Spark version: 2.2.1\n"
     ]
    }
   ],
   "source": [
    "# install and load necessary packages\n",
    "!pip install seaborn\n",
    "\n",
    "import pyspark\n",
    "from datetime import datetime\n",
    "import seaborn as sns\n",
    "import sys\n",
    "import matplotlib.pyplot as plt\n",
    "%matplotlib inline\n",
    "import numpy as np\n",
    "\n",
    "print('Python version ' + sys.version)\n",
    "print('Spark version: ' + pyspark.__version__)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## *To Do: List Metrics and allow user to choose specific Metric*"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "### Establish Connection to Spark Cluster\n",
    "\n",
    "set configuration so that the Spark Cluster communicates with Ceph and reads a chunk of data."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 2,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "Application Name:  M4JZ - Ceph S3 Prometheus JSON Reader\n"
     ]
    }
   ],
   "source": [
    "import string \n",
    "import random\n",
    "\n",
    "# Set the configuration\n",
    "# random string for instance name\n",
    "inst = ''.join(random.choices(string.ascii_uppercase + string.digits, k=4))\n",
    "AppName = inst + ' - Ceph S3 Prometheus JSON Reader'\n",
    "conf = pyspark.SparkConf().setAppName(AppName).setMaster('spark://spark-cluster.dh-prod-analytics-factory.svc:7077')\n",
    "print(\"Application Name: \", AppName)\n",
    "\n",
    "# specify number of nodes need (1-5)\n",
    "conf.set(\"spark.cores.max\", \"8\")\n",
    "\n",
    "# specify Spark executor memory (default is 1gB)\n",
    "conf.set(\"spark.executor.memory\", \"4g\")\n",
    "\n",
    "# Set the Spark cluster connection\n",
    "sc = pyspark.SparkContext.getOrCreate(conf) \n",
    "\n",
    "# Set the Hadoop configurations to access Ceph S3\n",
    "import os\n",
    "(ceph_key, ceph_secret, ceph_host) = (os.getenv('DH_CEPH_KEY'), os.getenv('DH_CEPH_SECRET'), os.getenv('DH_CEPH_HOST'))\n",
    "ceph_key = 'DTG5R3EEWN9JBYJZH0DF'\n",
    "ceph_secret = 'pdcEGFERILlkRDGrCSxdIMaZVtNCOKvYP4Gf2b2x'\n",
    "ceph_host = 'http://storage-016.infra.prod.upshift.eng.rdu2.redhat.com:8080'\n",
    "sc._jsc.hadoopConfiguration().set(\"fs.s3a.access.key\", ceph_key) \n",
    "sc._jsc.hadoopConfiguration().set(\"fs.s3a.secret.key\", ceph_secret) \n",
    "sc._jsc.hadoopConfiguration().set(\"fs.s3a.endpoint\", ceph_host) \n",
    "\n",
    "#Get the SQL context\n",
    "sqlContext = pyspark.SQLContext(sc)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "#### Metric variables to be specified for analysis\n",
    "\n",
    "NOTE: these variables are dependent on the metric and how we want to filter the data."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 3,
   "metadata": {},
   "outputs": [],
   "source": [
    "# specify metric we want to analyze\n",
    "metric_name = 'kubelet_docker_operations_latency_microseconds'\n",
    "\n",
    "# choose a label\n",
    "label = \"\"\n",
    "\n",
    "# specify for histogram metric type\n",
    "bucket_val = '0.3'\n",
    "\n",
    "# specify for summary metric type\n",
    "quantile_val = '0.99'\n",
    "\n",
    "# specify any filtering when collected the data\n",
    "# For example:\n",
    "# If I want to just see data from a specific host, specify \"metric.hostname='free-stg-master-03fb6'\"\n",
    "where_labels = [\"metric.hostname='free-stg-master-03fb6'\"]"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "#### Read Metric data from Ceph and detect metric type"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 4,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "Metric Type:  summary\n",
      "Schema:\n",
      "root\n",
      " |-- metric: struct (nullable = true)\n",
      " |    |-- __name__: string (nullable = true)\n",
      " |    |-- beta_kubernetes_io_arch: string (nullable = true)\n",
      " |    |-- beta_kubernetes_io_fluentd_ds_ready: string (nullable = true)\n",
      " |    |-- beta_kubernetes_io_instance_type: string (nullable = true)\n",
      " |    |-- beta_kubernetes_io_os: string (nullable = true)\n",
      " |    |-- clam_controller_enabled: string (nullable = true)\n",
      " |    |-- clam_server_enabled: string (nullable = true)\n",
      " |    |-- failure_domain_beta_kubernetes_io_region: string (nullable = true)\n",
      " |    |-- failure_domain_beta_kubernetes_io_zone: string (nullable = true)\n",
      " |    |-- fluentd_test: string (nullable = true)\n",
      " |    |-- hostname: string (nullable = true)\n",
      " |    |-- image_inspector_enabled: string (nullable = true)\n",
      " |    |-- instance: string (nullable = true)\n",
      " |    |-- job: string (nullable = true)\n",
      " |    |-- kubernetes_io_hostname: string (nullable = true)\n",
      " |    |-- logging_infra_fluentd: string (nullable = true)\n",
      " |    |-- node_role_kubernetes_io_compute: string (nullable = true)\n",
      " |    |-- node_role_kubernetes_io_infra: string (nullable = true)\n",
      " |    |-- node_role_kubernetes_io_master: string (nullable = true)\n",
      " |    |-- operation_type: string (nullable = true)\n",
      " |    |-- ops_node: string (nullable = true)\n",
      " |    |-- placement: string (nullable = true)\n",
      " |    |-- region: string (nullable = true)\n",
      " |    |-- type: string (nullable = true)\n",
      " |-- values: array (nullable = true)\n",
      " |    |-- element: array (containsNull = true)\n",
      " |    |    |-- element: string (containsNull = true)\n",
      "\n"
     ]
    }
   ],
   "source": [
    "jsonUrl = \"s3a://DH-DEV-PROMETHEUS-BACKUP/prometheus-openshift-devops-monitor.1b7d.free-stg.openshiftapps.com/\" + metric_name\n",
    "\n",
    "try:\n",
    "    jsonFile_sum = sqlContext.read.option(\"multiline\", True).option(\"mode\", \"PERMISSIVE\").json(jsonUrl + '_sum/')\n",
    "    jsonFile = sqlContext.read.option(\"multiline\", True).option(\"mode\", \"PERMISSIVE\").json(jsonUrl + '_count/')\n",
    "    try:\n",
    "        jsonFile_bucket = sqlContext.read.option(\"multiline\", True).option(\"mode\", \"PERMISSIVE\").json(jsonUrl + '_bucket/')\n",
    "        metric_type = 'histogram'\n",
    "    except:\n",
    "        jsonFile_quantile = sqlContext.read.option(\"multiline\", True).option(\"mode\", \"PERMISSIVE\").json(jsonUrl+'/')\n",
    "        metric_type = 'summary'\n",
    "except:\n",
    "    jsonFile = sqlContext.read.option(\"multiline\", True).option(\"mode\", \"PERMISSIVE\").json(jsonUrl+'/')\n",
    "    metric_type = 'gauge or counter'\n",
    "\n",
    "#Display the schema of the file\n",
    "print(\"Metric Type: \", metric_type)\n",
    "print(\"Schema:\")\n",
    "jsonFile.printSchema()"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## UI for selecting interface"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 5,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "number of labels:  24\n",
      "\n",
      "===== Labels =====\n",
      "1 \t __name__\n",
      "2 \t beta_kubernetes_io_arch\n",
      "3 \t beta_kubernetes_io_fluentd_ds_ready\n",
      "4 \t beta_kubernetes_io_instance_type\n",
      "5 \t beta_kubernetes_io_os\n",
      "6 \t clam_controller_enabled\n",
      "7 \t clam_server_enabled\n",
      "8 \t failure_domain_beta_kubernetes_io_region\n",
      "9 \t failure_domain_beta_kubernetes_io_zone\n",
      "10 \t fluentd_test\n",
      "11 \t hostname\n",
      "12 \t image_inspector_enabled\n",
      "13 \t instance\n",
      "14 \t job\n",
      "15 \t kubernetes_io_hostname\n",
      "16 \t logging_infra_fluentd\n",
      "17 \t node_role_kubernetes_io_compute\n",
      "18 \t node_role_kubernetes_io_infra\n",
      "19 \t node_role_kubernetes_io_master\n",
      "20 \t operation_type\n",
      "21 \t ops_node\n",
      "22 \t placement\n",
      "23 \t region\n",
      "24 \t type\n",
      "\n",
      "===== Select a Label (specify number from 0 to 24\n",
      "0 indicates no label to select\n",
      "23\n",
      "\n",
      "===== Label Selected:  region\n"
     ]
    }
   ],
   "source": [
    "labels = []\n",
    "for i in jsonFile.schema[\"metric\"].jsonValue()[\"type\"][\"fields\"]:\n",
    "    labels.append(i[\"name\"])\n",
    "\n",
    "print(\"number of labels: \", len(labels))\n",
    "print(\"\\n===== Labels =====\")\n",
    "inc = 0\n",
    "for i in labels:\n",
    "    inc = inc+1\n",
    "    print(inc, \"\\t\", i)\n",
    "    \n",
    "prompt = \"\\n===== Select a Label (specify number from 0 to \" + str(len(labels)) + \"\\n0 indicates no label to select\\n\"\n",
    "label_num = int(input(prompt))\n",
    "if label_num != 0:\n",
    "    label = labels[label_num - 1]\n",
    "print(\"\\n===== Label Selected: \",label)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "#### Preprocessing Prometheus Data\n",
    "\n",
    "Once we have chosen a metric to focus on, filter specific data into a data frame and preprocess data. "
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 6,
   "metadata": {},
   "outputs": [],
   "source": [
    "import pyspark.sql.functions as F\n",
    "from pyspark.sql.types import StringType\n",
    "from pyspark.sql.types import IntegerType\n",
    "\n",
    "# create function to convert POSIX timestamp to local date\n",
    "def convert_timestamp(t):\n",
    "    return str(datetime.fromtimestamp(int(t)))\n",
    "\n",
    "def format_df(df):\n",
    "    #reformat data by timestamp and values\n",
    "    df = df.withColumn(\"values\", F.explode(df.values))\n",
    "    \n",
    "    df = df.withColumn(\"timestamp\", F.col(\"values\").getItem(0))\n",
    "    df = df.sort(\"timestamp\", ascending=True)\n",
    "    \n",
    "    df = df.withColumn(\"values\", F.col(\"values\").getItem(1))\n",
    "\n",
    "    # drop null values\n",
    "    df = df.na.drop(subset=[\"values\"])\n",
    "    \n",
    "    # cast values to int\n",
    "    df = df.withColumn(\"values\", df.values.cast(\"int\"))\n",
    "\n",
    "    # define function to be applied to DF column\n",
    "    udf_convert_timestamp = F.udf(lambda z: convert_timestamp(z), StringType())\n",
    "\n",
    "    # convert timestamp values to datetime timestamp\n",
    "    df = df.withColumn(\"timestamp\", udf_convert_timestamp(\"timestamp\"))\n",
    "\n",
    "    # calculate log(values) for each row\n",
    "    df = df.withColumn(\"log_values\", F.log(df.values))\n",
    "    \n",
    "    return df"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "### Take data from Ceph json and parse into Spark Dataframe\n",
    "We take the data from the json file located in Ceph and query it into a Spark df. This df is then parsed using the function above."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 7,
   "metadata": {},
   "outputs": [],
   "source": [
    "def extract_from_json(json, name, select_labels, where_labels):\n",
    "    #Register the created SchemaRDD as a temporary variable\n",
    "    json.registerTempTable(name)\n",
    "    \n",
    "    #Filter the results into a data frame\n",
    "\n",
    "    query = \"SELECT values\"\n",
    "    \n",
    "    # check if select labels are specified and add query condition if appropriate\n",
    "    if len(select_labels) > 0:\n",
    "        query = query + \", \" + \", \".join(select_labels)\n",
    "        \n",
    "    query = query + \" FROM \" + name\n",
    "    \n",
    "    # check if where labels are specified and add query condition if appropriate\n",
    "    if len(where_labels) > 0:\n",
    "        query = query + \" WHERE \" + \" AND \".join(where_labels)\n",
    "\n",
    "    print(query)\n",
    "    data = sqlContext.sql(query)\n",
    "\n",
    "    # sample data to make it more manageable\n",
    "    # data = data.sample(False, fraction = 0.05, seed = 0)\n",
    "    # TODO: get rid of this hack\n",
    "    data = sqlContext.createDataFrame(data.head(1000), data.schema)\n",
    "    \n",
    "    return format_df(data)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 8,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "SELECT values, metric.region FROM kubelet_docker_operations_latency_microseconds WHERE metric.hostname='free-stg-master-03fb6'\n",
      "+------+---------+-------------------+------------------+\n",
      "|values|   region|          timestamp|        log_values|\n",
      "+------+---------+-------------------+------------------+\n",
      "| 44588|us-east-2|2018-02-10 23:59:59|10.705220043509515|\n",
      "| 77849|us-east-2|2018-02-10 23:59:59|11.262526331964487|\n",
      "|    88|us-east-2|2018-02-10 23:59:59| 4.477336814478207|\n",
      "|     2|us-east-2|2018-02-10 23:59:59|0.6931471805599453|\n",
      "|    29|us-east-2|2018-02-10 23:59:59| 3.367295829986474|\n",
      "|   417|us-east-2|2018-02-10 23:59:59|6.0330862217988015|\n",
      "|    16|us-east-2|2018-02-10 23:59:59| 2.772588722239781|\n",
      "|  1003|us-east-2|2018-02-10 23:59:59| 6.910750787961936|\n",
      "|    44|us-east-2|2018-02-10 23:59:59| 3.784189633918261|\n",
      "|    29|us-east-2|2018-02-10 23:59:59| 3.367295829986474|\n",
      "|964458|us-east-2|2018-02-10 23:59:59|13.779321564501078|\n",
      "|    16|us-east-2|2018-02-11 00:23:58| 2.772588722239781|\n",
      "| 44838|us-east-2|2018-02-11 00:23:58|10.710811273158345|\n",
      "|     2|us-east-2|2018-02-11 00:23:58|0.6931471805599453|\n",
      "|    44|us-east-2|2018-02-11 00:23:58| 3.784189633918261|\n",
      "|  1008|us-east-2|2018-02-11 00:23:58| 6.915723448631314|\n",
      "|969736|us-east-2|2018-02-11 00:23:58| 13.78477914848751|\n",
      "|    29|us-east-2|2018-02-11 00:23:58| 3.367295829986474|\n",
      "|    88|us-east-2|2018-02-11 00:23:58| 4.477336814478207|\n",
      "|   417|us-east-2|2018-02-11 00:23:58|6.0330862217988015|\n",
      "+------+---------+-------------------+------------------+\n",
      "only showing top 20 rows\n",
      "\n"
     ]
    }
   ],
   "source": [
    "if label != \"\":\n",
    "    select_labels = ['metric.' + label]\n",
    "else:\n",
    "    select_labels = []\n",
    "\n",
    "# get data and format\n",
    "data = extract_from_json(jsonFile, metric_name, select_labels, where_labels)\n",
    "\n",
    "data.count()\n",
    "data.show()"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## Calculate Sampling Rate"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 20,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "average hourly sample count:  55633.5\n",
      "+----+-----+\n",
      "|hour|count|\n",
      "+----+-----+\n",
      "|   0|55441|\n",
      "|   1|55352|\n",
      "|   2|55346|\n",
      "|   3|55331|\n",
      "|   4|55452|\n",
      "|   5|55463|\n",
      "|   6|55452|\n",
      "|   7|55463|\n",
      "|   8|55452|\n",
      "|   9|55463|\n",
      "|  10|55452|\n",
      "|  11|55463|\n",
      "|  12|55452|\n",
      "|  13|55504|\n",
      "|  14|55470|\n",
      "|  15|55691|\n",
      "|  16|55978|\n",
      "|  17|56078|\n",
      "|  18|56052|\n",
      "|  19|56068|\n",
      "+----+-----+\n",
      "only showing top 20 rows\n",
      "\n"
     ]
    }
   ],
   "source": [
    "def calculate_sample_rate(df):\n",
    "    # define function to be applied to DF column\n",
    "    udf_timestamp_hour = F.udf(lambda dt: int(datetime.strptime(dt,'%Y-%m-%d %X').hour), IntegerType())\n",
    "\n",
    "    # convert timestamp values to datetime timestamp\n",
    "\n",
    "    # new df with hourly value count\n",
    "    vals_per_hour = df.withColumn(\"hour\", udf_timestamp_hour(\"timestamp\")).groupBy(\"hour\").count()\n",
    "\n",
    "    # average density (samples/hour)\n",
    "    avg = vals_per_hour.agg(F.avg(F.col(\"count\"))).head()[0]\n",
    "    print(\"average hourly sample count: \", avg)\n",
    "    \n",
    "    # sort and display hourly count\n",
    "    vals_per_hour.sort(\"hour\").show()\n",
    "    \n",
    "calculate_sample_rate(data)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "### Calculate number of values per Label"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 28,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "+---------+-------+\n",
      "|   region|  count|\n",
      "+---------+-------+\n",
      "|us-east-2|1335204|\n",
      "+---------+-------+\n",
      "\n"
     ]
    }
   ],
   "source": [
    "def calculate_vals_per_label(df):\n",
    "    # new df with vals per label\n",
    "    df.groupBy(label).count().show()\n",
    "    \n",
    "calculate_vals_per_label(data)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 10,
   "metadata": {},
   "outputs": [],
   "source": [
    "from pyspark.sql.window import Window\n",
    "\n",
    "def get_deltas(df):\n",
    "    df_lag = df.withColumn('prev_vals',\n",
    "                        F.lag(df['values'])\n",
    "                                 .over(Window.partitionBy(\"timestamp\").orderBy(\"timestamp\")))\n",
    "\n",
    "    result = df_lag.withColumn('deltas', (df_lag['values'] - df_lag['prev_vals']))\n",
    "    result = result.drop(\"prev_vals\")\n",
    "    \n",
    "    max_delta = result.agg(F.max(F.col(\"deltas\"))).head()[0]\n",
    "    min_delta = result.agg(F.min(F.col(\"deltas\"))).head()[0]\n",
    "    mean_delta = result.agg(F.avg(F.col(\"deltas\"))).head()[0]\n",
    "    \n",
    "    return max_delta, min_delta, mean_delta, result"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 11,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "+------+--------------------------------------+-------------------+------------------+-------+\n",
      "|values|failure_domain_beta_kubernetes_io_zone|          timestamp|        log_values| deltas|\n",
      "+------+--------------------------------------+-------------------+------------------+-------+\n",
      "|    18|                            us-east-2a|2018-03-03 04:28:59|2.8903717578961645|   null|\n",
      "|     2|                            us-east-2a|2018-03-03 04:28:59|0.6931471805599453|    -16|\n",
      "|   279|                            us-east-2a|2018-03-03 04:28:59| 5.631211781821365|    277|\n",
      "|   495|                            us-east-2a|2018-03-03 04:28:59|  6.20455776256869|    216|\n",
      "|386700|                            us-east-2a|2018-03-03 04:28:59|12.865404477595389| 386205|\n",
      "| 18266|                            us-east-2a|2018-03-03 04:28:59| 9.812796687251625|-368434|\n",
      "|    11|                            us-east-2a|2018-03-03 04:28:59|2.3978952727983707| -18255|\n",
      "|     6|                            us-east-2a|2018-03-03 04:28:59| 1.791759469228055|     -5|\n",
      "|    18|                            us-east-2a|2018-03-03 04:28:59|2.8903717578961645|     12|\n",
      "|    21|                            us-east-2a|2018-03-03 04:28:59| 3.044522437723423|      3|\n",
      "| 31711|                            us-east-2a|2018-03-03 04:28:59|10.364418902828275|  31690|\n",
      "|    18|                            us-east-2a|2018-03-03 08:18:59|2.8903717578961645|   null|\n",
      "|     2|                            us-east-2a|2018-03-03 08:18:59|0.6931471805599453|    -16|\n",
      "|   279|                            us-east-2a|2018-03-03 08:18:59| 5.631211781821365|    277|\n",
      "|   541|                            us-east-2a|2018-03-03 08:18:59| 6.293419278846481|    262|\n",
      "|437178|                            us-east-2a|2018-03-03 08:18:59|12.988095713798836| 436637|\n",
      "| 20649|                            us-east-2a|2018-03-03 08:18:59| 9.935422171066474|-416529|\n",
      "|    11|                            us-east-2a|2018-03-03 08:18:59|2.3978952727983707| -20638|\n",
      "|     6|                            us-east-2a|2018-03-03 08:18:59| 1.791759469228055|     -5|\n",
      "|    18|                            us-east-2a|2018-03-03 08:18:59|2.8903717578961645|     12|\n",
      "+------+--------------------------------------+-------------------+------------------+-------+\n",
      "only showing top 20 rows\n",
      "\n",
      "Max delta: 6576011\n",
      "Min delta: -6272578\n",
      "Mean delta: 16174.62638770783\n"
     ]
    }
   ],
   "source": [
    "max_delta, min_delta, mean_delta, data = get_deltas(data)\n",
    "data.show()\n",
    "print(\"Max delta:\", max_delta)\n",
    "print(\"Min delta:\", min_delta)\n",
    "print(\"Mean delta:\", mean_delta)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 12,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "+------+--------------------------------------+-------------------+------------------+-------+--------------------+\n",
      "|values|failure_domain_beta_kubernetes_io_zone|          timestamp|        log_values| deltas|               rhmax|\n",
      "+------+--------------------------------------+-------------------+------------------+-------+--------------------+\n",
      "|    18|                            us-east-2a|2018-03-03 04:28:59|2.8903717578961645|   null|2.734607690901823...|\n",
      "|     2|                            us-east-2a|2018-03-03 04:28:59|0.6931471805599453|    -16|3.038452989890915E-7|\n",
      "|   279|                            us-east-2a|2018-03-03 04:28:59| 5.631211781821365|    277|4.238641920897826...|\n",
      "|   495|                            us-east-2a|2018-03-03 04:28:59|  6.20455776256869|    216|7.520171149980014E-5|\n",
      "|386700|                            us-east-2a|2018-03-03 04:28:59|12.865404477595389| 386205| 0.05874848855954084|\n",
      "| 18266|                            us-east-2a|2018-03-03 04:28:59| 9.812796687251625|-368434|0.002775019115667...|\n",
      "|    11|                            us-east-2a|2018-03-03 04:28:59|2.3978952727983707| -18255|1.671149144440003...|\n",
      "|     6|                            us-east-2a|2018-03-03 04:28:59| 1.791759469228055|     -5|9.115358969672745E-7|\n",
      "|    18|                            us-east-2a|2018-03-03 04:28:59|2.8903717578961645|     12|2.734607690901823...|\n",
      "|    21|                            us-east-2a|2018-03-03 04:28:59| 3.044522437723423|      3|3.190375639385460...|\n",
      "| 31711|                            us-east-2a|2018-03-03 04:28:59|10.364418902828275|  31690|0.004817619138121541|\n",
      "|    18|                            us-east-2a|2018-03-03 08:18:59|2.8903717578961645|   null|2.734607690901823...|\n",
      "|     2|                            us-east-2a|2018-03-03 08:18:59|0.6931471805599453|    -16|3.038452989890915E-7|\n",
      "|   279|                            us-east-2a|2018-03-03 08:18:59| 5.631211781821365|    277|4.238641920897826...|\n",
      "|   541|                            us-east-2a|2018-03-03 08:18:59| 6.293419278846481|    262|8.219015337654925E-5|\n",
      "|437178|                            us-east-2a|2018-03-03 08:18:59|12.988095713798836| 436637| 0.06641724006072652|\n",
      "| 20649|                            us-east-2a|2018-03-03 08:18:59| 9.935422171066474|-416529|0.003137050789412875|\n",
      "|    11|                            us-east-2a|2018-03-03 08:18:59|2.3978952727983707| -20638|1.671149144440003...|\n",
      "|     6|                            us-east-2a|2018-03-03 08:18:59| 1.791759469228055|     -5|9.115358969672745E-7|\n",
      "|    18|                            us-east-2a|2018-03-03 08:18:59|2.8903717578961645|     12|2.734607690901823...|\n",
      "+------+--------------------------------------+-------------------+------------------+-------+--------------------+\n",
      "only showing top 20 rows\n",
      "\n"
     ]
    }
   ],
   "source": [
    "# rhmax = current value / max(values)\n",
    "# if rhmax (of new point) >> 1, we can assume that the point is an anomaly\n",
    "def get_rhmax(df):\n",
    "    real_max = df.agg(F.max(F.col(\"values\"))).head()[0]\n",
    "    result = df.withColumn(\"rhmax\", df[\"values\"]/real_max)\n",
    "    return result\n",
    "\n",
    "result = get_rhmax(data)\n",
    "result.show()"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "### Check Metric Type\n",
    "Determine the metric type by checking filenames and data. For deciphering between counter and gauge, the only way to tell the difference is looking at the raw data. If the data is monotonically increasing, it is a counter, otherwise, it's a gauge. \n",
    "https://prometheus.io/docs/concepts/metric_types/\n",
    "\n",
    "**Histogram characteristics:**\n",
    "* has a _count file (check criteria)\n",
    "* has a _sum file (check criteria)\n",
    "* has a _bucket file (check criteria)\n",
    "\n",
    "**Summary characteristics:**\n",
    "* has a _count file (check criteria)\n",
    "* has a _sum file (check criteria)\n",
    "* does not have a _bucket file (check criteria)\n",
    "* has a quantile label\n",
    "\n",
    "**Counter characteristics:**\n",
    "* monotonically increasing (check criteria)\n",
    "\n",
    "**Gauge characteristics:**\n",
    "* no specific characteristics, so we assume that gauges have none of the above"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "### Decifer between gauge and counter\n",
    "If the metric type is counter, create new column that is the derivative of the values"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 10,
   "metadata": {},
   "outputs": [],
   "source": [
    "def gauge_counter_separator(df):\n",
    "    vals = np.array(df.select(\"values\").collect())\n",
    "    diff = vals - np.roll(vals, 1) # this value - previous value (should always be zero or positive for counter)\n",
    "    diff[0] = 0 # ignore first difference, there is no value before the first\n",
    "    diff[np.where(vals == 0)] = 0\n",
    "    # check if these are any negative differences, if not then metric is a counter.\n",
    "    # if counter, we must convert it to a gauge by keeping the derivatives\n",
    "    if ((diff < 0).sum() == 0):\n",
    "        metric_type = 'counter'\n",
    "    else:\n",
    "        metric_type = 'gauge'\n",
    "    return metric_type, df\n",
    "\n",
    "if metric_type == \"gauge or counter\":\n",
    "    metric_type, data = gauge_counter_separator(data)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 11,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "Metric type:  summary\n"
     ]
    }
   ],
   "source": [
    "print(\"Metric type: \", metric_type)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "### Load data based on Histogram or  Summary Metric"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 12,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "SELECT values FROM kubelet_docker_operations_latency_microseconds\n",
      "SELECT values, metric.quantile FROM kubelet_docker_operations_latency_microseconds\n",
      "+------+-------------------+------------------+\n",
      "|values|          timestamp|        log_values|\n",
      "+------+-------------------+------------------+\n",
      "| 36338|2018-05-29 23:59:59|10.500619304662386|\n",
      "| 36338|2018-05-30 00:00:59|10.500619304662386|\n",
      "| 36338|2018-05-30 00:01:59|10.500619304662386|\n",
      "| 36338|2018-05-30 00:02:59|10.500619304662386|\n",
      "| 36338|2018-05-30 00:03:59|10.500619304662386|\n",
      "| 36338|2018-05-30 00:04:59|10.500619304662386|\n",
      "| 36338|2018-05-30 00:05:59|10.500619304662386|\n",
      "| 36338|2018-05-30 00:06:59|10.500619304662386|\n",
      "| 36338|2018-05-30 00:07:59|10.500619304662386|\n",
      "| 36338|2018-05-30 00:08:59|10.500619304662386|\n",
      "| 36338|2018-05-30 00:09:59|10.500619304662386|\n",
      "| 36338|2018-05-30 00:10:59|10.500619304662386|\n",
      "| 36338|2018-05-30 00:11:59|10.500619304662386|\n",
      "| 36338|2018-05-30 00:12:59|10.500619304662386|\n",
      "| 36338|2018-05-30 00:13:59|10.500619304662386|\n",
      "| 36338|2018-05-30 00:14:59|10.500619304662386|\n",
      "| 36338|2018-05-30 00:15:59|10.500619304662386|\n",
      "| 36338|2018-05-30 00:16:59|10.500619304662386|\n",
      "| 36338|2018-05-30 00:17:59|10.500619304662386|\n",
      "| 36338|2018-05-30 00:18:59|10.500619304662386|\n",
      "+------+-------------------+------------------+\n",
      "only showing top 20 rows\n",
      "\n",
      "+------+--------+-------------------+------------------+\n",
      "|values|quantile|          timestamp|        log_values|\n",
      "+------+--------+-------------------+------------------+\n",
      "| 36762|    0.99|2018-05-30 15:00:59| 10.51221998195371|\n",
      "| 36762|    0.99|2018-05-30 15:01:59| 10.51221998195371|\n",
      "| 36762|    0.99|2018-05-30 15:02:59| 10.51221998195371|\n",
      "| 36762|    0.99|2018-05-30 15:03:59| 10.51221998195371|\n",
      "| 36762|    0.99|2018-05-30 15:04:59| 10.51221998195371|\n",
      "| 36762|    0.99|2018-05-30 15:05:59| 10.51221998195371|\n",
      "| 36762|    0.99|2018-05-30 15:06:59| 10.51221998195371|\n",
      "| 36762|    0.99|2018-05-30 15:07:59| 10.51221998195371|\n",
      "| 36762|    0.99|2018-05-30 15:08:59| 10.51221998195371|\n",
      "| 36762|    0.99|2018-05-30 15:09:59| 10.51221998195371|\n",
      "| 36762|    0.99|2018-05-30 15:10:59| 10.51221998195371|\n",
      "|261262|    0.99|2018-05-30 15:15:59|12.473279014220623|\n",
      "|261262|    0.99|2018-05-30 15:16:59|12.473279014220623|\n",
      "|261262|    0.99|2018-05-30 15:17:59|12.473279014220623|\n",
      "|261262|    0.99|2018-05-30 15:18:59|12.473279014220623|\n",
      "|261262|    0.99|2018-05-30 15:19:59|12.473279014220623|\n",
      "|261262|    0.99|2018-05-30 15:20:59|12.473279014220623|\n",
      "|261262|    0.99|2018-05-30 15:21:59|12.473279014220623|\n",
      "|261262|    0.99|2018-05-30 15:22:59|12.473279014220623|\n",
      "|261262|    0.99|2018-05-30 15:23:59|12.473279014220623|\n",
      "+------+--------+-------------------+------------------+\n",
      "only showing top 20 rows\n",
      "\n"
     ]
    }
   ],
   "source": [
    "if metric_type == 'histogram':\n",
    "    data_sum = extract_from_json(jsonFile_sum, metric_name, select_labels, where_labels)\n",
    "    \n",
    "    select_labels.append(\"metric.le\")\n",
    "    data_bucket = extract_from_json(jsonFile_bucket, metric_name, select_labels, where_labels)\n",
    "    \n",
    "    # filter by specific le value\n",
    "    data_bucket = data_bucket.filter(bucket_val)\n",
    "    \n",
    "    data_sum.show()\n",
    "    data_bucket.show()\n",
    "    \n",
    "elif metric_type == 'summary':\n",
    "    # get metric sum data\n",
    "    data_sum = extract_from_json(jsonFile_sum, metric_name, select_labels, where_labels)\n",
    "    \n",
    "    # get metric quantile data\n",
    "    select_labels.append(\"metric.quantile\")\n",
    "    data_quantile = extract_from_json(jsonFile_quantile, metric_name, select_labels, where_labels)\n",
    "    \n",
    "    # filter by specific quantile value\n",
    "    data_quantile = data_quantile.filter(data_quantile.quantile == quantile_val)\n",
    "    # get rid of NaN values once again. This is required once filtering takes place\n",
    "    data_quantile = data_quantile.na.drop(subset='values')\n",
    "    \n",
    "    data_sum.show()\n",
    "    data_quantile.show()"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "### Descriptive Stats\n",
    "The portion of statistics dedicated to summarizing a total population\n",
    "\n",
    "#### Mean \n",
    "Arithmetic average of a range of values or quantities, computed by dividing the total of all values by the number of values.\n",
    "\n",
    "#### Variance\n",
    "In the same way that the mean is used to describe the central tendency, variance is intended to describe the spread.\n",
    "The xi – μ is called the “deviation from the mean”, making the variance the squared deviation multiplied by 1 over the number of samples. This is why the square root of the variance, σ, is called the standard deviation.\n",
    "\n",
    "#### Standard Deviation\n",
    "Standard deviation (SD, also represented by the Greek letter sigma σ or the Latin letter s) is a measure that is used to quantify the amount of variation or dispersion of a set of data values.[1] A low standard deviation indicates that the data points tend to be close to the mean (also called the expected value) of the set, while a high standard deviation indicates that the data points are spread out over a wider range of values.\n",
    "\n",
    "#### Median\n",
    "\n",
    "Denotes value or quantity lying at the midpoint of a frequency distribution of observed values or quantities, such that there is an equal probability of falling above or below it. Simply put, it is the *middle* value in the list of numbers.\n",
    "The median is a better choice when the indicator can be affected by some outliers."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 13,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "\tMean(values):  67087.9063346175\n",
      "\tVariance(values):  56691431555.4375\n",
      "\tStddev(values):  238099.62527361838\n",
      "\tMedian(values):  628.0\n"
     ]
    }
   ],
   "source": [
    "def get_stats(df):\n",
    "    # calculate mean\n",
    "    mean = df.agg(F.avg(F.col(\"values\"))).head()[0]\n",
    "    \n",
    "    # calculate variance\n",
    "    var = df.agg(F.variance(F.col(\"values\"))).head()[0]\n",
    "\n",
    "    # calculate standard deviation\n",
    "    stddev = df.agg(F.stddev(F.col(\"values\"))).head()[0]\n",
    "\n",
    "    # calculate median\n",
    "    median = float(df.approxQuantile(\"values\", [0.5], 0.25)[0])\n",
    "    \n",
    "    return mean, var, stddev, median\n",
    "    \n",
    "mean, var, stddev, median = get_stats(data)\n",
    "\n",
    "print(\"\\tMean(values): \", mean)\n",
    "print(\"\\tVariance(values): \", var)\n",
    "print(\"\\tStddev(values): \", stddev)\n",
    "print(\"\\tMedian(values): \", median)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "### Histogram \n",
    "The most common representation of a distribution is a histogram, which is a graph that shows the frequency or probability of each value. Plots will be generated by operation type"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "We will use Seaborn module for this. __Kernel Density Estimation__ * will be added for smoothing.\n",
    "* In statistics, kernel density estimation (KDE) is a non-parametric way to estimate the probability density function of a random variable. Kernel density estimation is a fundamental data smoothing problem where inferences about the population are made, based on a finite data sample.\n",
    "* The kernel density estimate may be less familiar, but it can be a useful tool for plotting the shape of a distribution. Like the histogram, the KDE plots encodes the density of observations on one axis with height along the other axis:\n",
    "\n",
    "Log normalization is required because, for different operations, values seems to be in very different scales. We can use log normalization for only positive values. We filter out the values that are zero"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 14,
   "metadata": {},
   "outputs": [
    {
     "name": "stderr",
     "output_type": "stream",
     "text": [
      "/opt/app-root/lib/python3.6/site-packages/matplotlib/axes/_axes.py:6462: UserWarning: The 'normed' kwarg is deprecated, and has been replaced by the 'density' kwarg.\n",
      "  warnings.warn(\"The 'normed' kwarg is deprecated, and has been \"\n",
      "/opt/app-root/lib/python3.6/site-packages/matplotlib/figure.py:459: UserWarning: matplotlib is currently using a non-GUI backend, so cannot show the figure\n",
      "  \"matplotlib is currently using a non-GUI backend, \"\n"
     ]
    },
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "None\n"
     ]
    },
    {
     "data": {
      "image/png": "iVBORw0KGgoAAAANSUhEUgAAA4kAAAK6CAYAAACUtxniAAAABHNCSVQICAgIfAhkiAAAAAlwSFlzAAALEgAACxIB0t1+/AAAADl0RVh0U29mdHdhcmUAbWF0cGxvdGxpYiB2ZXJzaW9uIDIuMi4yLCBodHRwOi8vbWF0cGxvdGxpYi5vcmcvhp/UCwAAIABJREFUeJzs3X1w3fV9L/j3kY5lY/wY6shuYtzdwUlIMWG6XcJemDgjKrTgslyK6Uwmdy5klnRKkh3IH/y1HbdhdoabhNmEhB0eLintLb2dCWRYSjQblhpyDTekyTTtdSlzk3Vu3QiCFQPGj2BJ55z9QzpHlq1n/fTwE6/XP5LO+Z1zPhJmjt76fL+fb6XRaDQCAAAASdoWuwAAAACWDiERAACAFiERAACAFiERAACAFiERAACAFiERAACAlupiF7AYDh8+Pu+vsXHj6hw5cmreX6dIal4Yal4Yal4YZah506a1i11CqSzEe+RkyvBv6kzqnV9lqrdMtSbqnW9lqHey90edxHlSrbYvdgkzpuaFoeaFoeaFUcaaWdrK9m9KvfOrTPWWqdZEvfOtbPWeTUgEAACgRUgEAACgRUgEgCVu37596enpSXd3dx5++OEJr3vmmWfy4Q9/OP/4j//Yuu2hhx5Kd3d3enp68sILLyxEuQCU3HtycA0AlEWtVsvdd9+dRx99NJ2dndm9e3e6urpy0UUXjbnuxIkT+Q//4T/kYx/7WOu2AwcOpLe3N729venv789nPvOZPPPMM2lvL/deGQDml04iACxh+/fvz7Zt27J169Z0dHRk165d2bt37znX3XffffnsZz+blStXtm7bu3dvdu3alY6OjmzdujXbtm3L/v37F7J8AEpIJxEAlrD+/v5s3ry59XVnZ+c5Qe+f/umfcujQoXzyk5/Mt771rTGPPbOz2NnZmf7+/klfb+PG1Ys+la9sx5aod36Vqd4y1Zqod76Vrd4zCYkAUGL1ej3/7t/9u9xzzz2FPN9in+u1adPaRT+rcSbUO7/KVG+Zak3UO9/KUO9kIVZIBIAlrLOzM4cOHWp93d/fn87OztbXJ0+ezM9+9rP823/7b5Mkhw8fzu23354HHnhgyscCwHjsSQSAJWzHjh05ePBg+vr6MjAwkN7e3nR1dbXuX7t2bf72b/82zz33XJ577rlcdtlleeCBB7Jjx450dXWlt7c3AwMD6evry8GDB3PppZcu4ncDQBnoJALAElatVrNnz57cdtttqdVquemmm7J9+/bcd999ueSSS3L11VdP+Njt27fn2muvzXXXXZf29vbs2bPHZFMApiQkAsASt3PnzuzcuXPMbXfccce41/7FX/zFmK9vv/323H777fNWGwDLj+WmAAAAtAiJAAAAtAiJAAAAtAiJAAAAtAiJAAAAtAiJAAAAtAiJAAAAtAiJAAAAtAiJAAAAtAiJAAAAtAiJAAAAtAiJAAAAtAiJAAAAtAiJAAAAtAiJAAAAtAiJAAAAtAiJAAAAtAiJAAAAtFQXu4Dl6nsvHczxE++Oe98nL/vAwhYDAMvY9//htXFv934LMDs6iQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQAAALQIiQCwhO3bty89PT3p7u7Oww8/fM79f/VXf5Xrr78+N9xwQz71qU/lwIEDSZJXX301l156aW644YbccMMN2bNnz0KXDkBJVRe7AABgfLVaLXfffXceffTRdHZ2Zvfu3enq6spFF13Uuub666/Ppz71qSTJ3r17c8899+Rb3/pWkuTCCy/MU089tSi1A1BeOokAsETt378/27Zty9atW9PR0ZFdu3Zl7969Y65Zs2ZN6/N33nknlUplocsEYJnRSQSAJaq/vz+bN29ufd3Z2Zn9+/efc91f/uVf5tFHH83g4GD+/M//vHX7q6++mn/9r/911qxZkzvvvDO//du/vSB1A1BuQiIAlNynP/3pfPrTn87TTz+dBx54IF/+8pfz/ve/P88//3w2btyYl19+OZ///OfT29s7pvM4no0bV6dabV+gyse3adPaGV2/ds2qQp5nthbqdYqi3vlTploT9c63stV7JiERAJaozs7OHDp0qPV1f39/Ojs7J7x+165d+ZM/+ZMkSUdHRzo6OpIkl1xySS688ML88z//c3bs2DHpax45cmruhc/Bpk1rc/jw8Rk95viJd8e9fabPMxuzqXcxqXf+lKnWRL3zrQz1ThZi7UkEgCVqx44dOXjwYPr6+jIwMJDe3t50dXWNuebgwYOtz7///e9n27ZtSZK33nortVotSdLX15eDBw9m69atC1Y7AOWlkwgAS1S1Ws2ePXty2223pVar5aabbsr27dtz33335ZJLLsnVV1+dxx57LC+99FKq1WrWrVuXL3/5y0mSH//4x/nGN76RarWatra2fOlLX8qGDRsW+TsCoAyERABYwnbu3JmdO3eOue2OO+5off5Hf/RH4z6up6cnPT0981obAMuT5aYAAAC0CIkAAAC0CIkAAAC0CIkAAAC0CIkAAAC0CIkAAAC0CIkAAAC0CIkAAAC0CIkAAAC0TCsk7tu3Lz09Penu7s7DDz98zv0DAwO58847093dnZtvvjmvvvpq676HHnoo3d3d6enpyQsvvDDlc/b19eXmm29Od3d37rzzzgwMDCRJHn300Vx33XW5/vrrc8stt+S1115rPebJJ5/MNddck2uuuSZPPvnkzH8KAAAAJJlGSKzVarn77rvzyCOPpLe3N9/97ndz4MCBMdc8/vjjWbduXZ599tnceuutuffee5MkBw4cSG9vb3p7e/PII4/kS1/6Umq12qTPee+99+bWW2/Ns88+m3Xr1uWJJ55Iklx88cX5zne+k6effjo9PT356le/miR5++23c//99+fb3/52Hn/88dx///05evRooT8kAACA94opQ+L+/fuzbdu2bN26NR0dHdm1a1f27t075prnnnsuN954Y5Kkp6cnL730UhqNRvbu3Ztdu3alo6MjW7duzbZt27J///4Jn7PRaOSHP/xhenp6kiQ33nhj67WuuOKKnHfeeUmSyy67LIcOHUqSvPjii7nyyiuzYcOGrF+/PldeeeWYjiUAAADTN2VI7O/vz+bNm1tfd3Z2pr+//5xrtmzZkiSpVqtZu3Ztjhw5MuFjJ7r9yJEjWbduXarVapJk8+bN57xWkjzxxBP5xCc+Me36Fto7p4fS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AADKQkgsUH3ko8E1AABAWQmJBWqcPbjGclMAAKBkhMQCjZ6TOLLctBUSF60kAACAGRESC9TcetjWOgJjZLmpPYkAAEBJCIkFag6uqbQ5AgMAACgnIbFArU5ihEQAAKCchMQCje5JHP66eQSG6aYAAEBZCIkFanYSzz4CQycRAAAoCyGxQGd3Elsh0eAaAACgJITEAo1ONz37CAwhEQAAKAchsUCj002Hv26GRSERAAAoCyGxQGd3EpvLTZ2TCAAAlIWQWKBz9ySO3K6TCAAAlISQWKDWclPnJAIAACUlJBao3joCY/ijcxIBAICyERIL1OwkNjuIjsAAAADKRkgsUKPVSay0PlYqlpsCAADlMa2QuG/fvvT09KS7uzsPP/zwOfcPDAzkzjvvTHd3d26++ea8+uqrrfseeuihdHd3p6enJy+88MKUz9nX15ebb7453d3dufPOOzMwMJAk+fGPf5wbb7wxH/3oR/O9731vzOtffPHFueGGG3LDDTfkD//wD2f2EyhQq5NYGb2tva2Sen2RCgIAAJihKUNirVbL3XffnUceeSS9vb357ne/mwMHDoy55vHHH8+6devy7LPP5tZbb829996bJDlw4EB6e3vT29ubRx55JF/60pdSq9Umfc577703t956a5599tmsW7cuTzzxRJJky5Ytueeee/K7v/u759S4atWqPPXUU3nqqafy4IMPzvmHMluj001HU2JbpWK5KQAAUBpThsT9+/dn27Zt2bp1azo6OrJr167s3bt3zDXPPfdcbrzxxiRJT09PXnrppTQajezduze7du1KR0dHtm7dmm3btmX//v0TPmej0cgPf/jD9PT0JEluvPHG1mt98IMfzEc+8pG0tS3dFbJnD65JhvclWm4KAACURXWqC/r7+7N58+bW152dndm/f/8512zZsmX4CavVrF27NkeOHEl/f38+9rGPjXlsf39/koz7nEeOHMm6detSrVZb1zSvn8zp06fze7/3e6lWq/mDP/iD/M7v/M6k12/cuDrVavuUzztT6/pPDH9cuypJsnbNqlTb29IY+bxp06a1hb92UZZybRNR88JQ88JQMwCw2KYMiWXw/PPPp7OzM319fbnlllvyoQ99KBdeeOGE1x85cmpe6nj76DtJkpMnB7Kqo5rjJ95NkgwN1VufJ8nhw8fn5fXnatOmtUu2tomoeWGo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rT5mFY89iQBzJyQWZLJOYltruamQCABFGhyqp/2MkNiablr3ngswW0JiQRqt6ab2JALAQhmqjQ2Jzc9rlpsCzJqQWJDR5abn3jd6TiIAUKSzO4ltbZW0VUanngIwc0JiQeqTdhKHP1puCgDFOntPYpK0t7elZrkpwKwJiQWZvJNouSkAzIfhTuLYX2eq7RWdRIA5EBIL0mgNrtFJBICF0Gg0MlirtyaaNrW3tdmTCDAHQmJB6pN0EttGTkq08gUAilOrN9Jo5JzlptX2SobqOokAsyUkFmTy6aZjrwEA5q65pLR9vD2JOokAsyYkFqSZ/xyBAQALY3Bo/JBYbaukVm+0hsoBMDNCYkHqrT2J5943egSGNysAKMrQSLdwvE5iMhoiAZiZaYXEffv2paenJ93d3Xn44YfPuX9gYCB33nlnuru7c/PNN+fVV19t3ffQQw+lu7s7PT09eeGFF6Z8zr6+vtx8883p7u7OnXfemYGBgSTJj3/849x444356Ec/mu9973tjXv/JJ5/MNddck2uuuSZPPvnkzH4CBRmdbqqTCAALYXBkuel4exITIRFgtqYMibVaLXfffXceeeSR9Pb25rvf/W4OHDgw5prHH38869aty7PPPptbb7019957b5LkwIED6e3tTW9vbx555JF86UtfSq1Wm/Q577333tx666159tlns27dujzxxBNJki1btuSee+7J7/7u74557bfffjv3339/vv3tb+fxxx/P/fffn6NHjxbyw5mJen0anUQpEQAKM7rcdOyvM83O4sBgbcFrAlgOpgyJ+/fvz7Zt27J169Z0dHRk165d2bt375hrnnvuudx4441Jkp6enrz00ktpNBrZu3dvdu3alY6OjmzdujXbtm3L/v37J3zORqORH/7wh+np6UmS3Hjjja3X+uAHP5iPfOQjaTvrjeDFF1/MlVdemQ0bNmT9+vW58sorx3QsF0pzKem4ncToJAJA0YYm2pM4stx0QCcRYFamDIn9/f3ZvHlz6+vOzs709/efc82WLVuSJNVqNWvXrs2RI0cmfOxEtx85ciTr1q1LtVpNkmzevPmc15pNfQth8sE1wx9toAeA4ky03LR5bqJOIsDsVBe7gMWwcePqVKvthT7n6tUdw8+9YXU2bVqbHHgza9esSpI0KsNZvFptz9o1q4bvX6KWcm0TUfPCUPPCUDNM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TCIREdVCVVU8/vjjePrppzE6Oopf/vKXuHDhworb9PX14W/+5m/w+c9/fsXHE4kEfvSjH+FnP/sZnnnmGfzoRz/C0tLSZh6/IYql2qebmrdthZLTrS6RLiDgleGuYYXJZoiEjSCxlYfXxFMFhP0uSBvtZLvOQFcAqqZjJpZt0MmoFTFItEElk8jppkREZKNTp05hZGQEQ0NDcLvdOH78OE6cOLHiNoODg7j55pshiit/pb/66qu499570d7ejra2Ntx777145ZVXNvP4DVEpN60hyDAzidyV2FzpXAm5guq4oTUA0BEql5smWzNI1HUd8VTBUhlvf1cQAPsSaSUGiTbQqh1cwz2JRERUg9nZWfT29lb+OxqNYnZ2tuFf62TL001rLzdlJrG5zCDEHJbiJNcOrmlF2YKCoqJVgt1aDHQZg3om5xkk0jJON7VB9SswwCCRiIgcLRLxQ25yKWB3d2jNz0nls0V7QpXbhYKrvzE2Px8uv3EOhbzr3rdVdt3nWn8POx/D7vuqxe/PLwAAejuD6/5dr1fLbU1vX1is6b6GByMIeGWksqW6np9mPbeZ6SQAoK+79uf24E09AICFVKFp56+W0893vVY777UYJNqgphUY3JRIRERVikajmJmZqfz37OwsotFo1V/75ptvrvjau+66a8Ovi8eb25fU3R3C/Hxqzc8nyz1j6WQe8+Vfu6n06tkf837Uch/j7HwKPqm2fq2NbHTeWqz19wBg22PYed5afXjZCNy8LnHdv+u1QkFv1betx/x8CkG/G/FUwfLz08zn9tLVGABrz61aKCHglXFpItG081ejmc+vFa1w3vWCWJab2mB5uun6t+PgGiIiqsXBgwcxNjaG8fFxFItFjI6O4siRI1V97X333YdXX30VS0tLWFpawquvvor77ruvwSduPLPc1GWl3LTEctNmMstNnTbZ1BTyuZDOllpyfoQ5cCdiod9TEAQM9QQxF8+hUFTtPhq1KAaJNqglkwhwDQYREVVHlmU89thj+PrXv44HH3wQn/vc57B371788Ic/rAywOXXqFD796U/jueeew1//9V/j+PHjAID29nb82Z/9GR5++GE8/PDD+Na3voX29vZm/nVsUaoMrqk9SCxxxH9TTS5kEPK7IEvOfPsZ9Lmg6TqyBaXZR6lZJUgMWxsKNNgdhA5wXyJVsNzUBsvTTde/nRlEapoO0eZyFyIi2poOHz6Mw4cPr/jYt7/97cr/P3ToEE6ePLnq15oB4layvAKj+r5J87YlZhKbJpkpIpUtYbAn2OyjrCnoN5bQp7MlBLzVL6R3gnh54E4k6MF8Ilfz15v/LhPzaezqD9t6NmpNzryU02Iqg2s2nG5qaMUyBiIiIicoKRpkSYBYwy44ZhKbz8xQtTu01BQwyk0BIJUrNfkktYunigCWp7TWaqgcJI7PpW07E7U2Bok2WF6Bsf7txEomscEHIiIi2qKKigZXjdNXl3sS2W/VLJPzRvDhxB2Jpmszia0mnsrD65bg81grEuzvCkAQgAkGiVTGINEGy+Wm7EkkIiJqpGJJrakfEVjuX2QmsXmmWiKTaJwtlSs2+SS1i6cKlrOIAOBxSYhG/BifS7PijQAwSLRF9YNrjM+rGr/5iIiIrDAyibW9famUm7InsWkmFjIQBQFtAecGiZVMYouVmxZLKjJ5pa4gETD6ErMFpTIEh7Y3Bok2qH5wjfEnM4lERETWlBQNHlet5abG7c31GbS5dF3H1HwGPREfJIdONgWWexJbrdw0nra+/uJaQ90BAOxLJINzv1NbiFZjJlFnJpGIiMiSoqJazyQqzCQ2QyJdRLagYKAchDiVmUlMtVqQmKxv/YXp2gmnRAwSbaBXppuuf7vlTGKDD0RERLQF6bqOUkmz3pPIILEpJheMoGOgy9lBYiWT2GLlpvZlEjnhlJYxSLSBVuXgGvGaPYlERERUG0XVoANw1Vxu2hpBYknRcPpyDP/06uUtNWVyat4YWjPQ7dwdiQDg88iQRKHlBtckyj2EkZC3rvvpbPPC55EwUf73ou2NQaINqh5cU/6TPYlERES1K5aDvFoziZUVGA4NEgslFf/yxhU8e/IS3vloHol0EWevxJt9LNtMlCeb9js8kygIAoI+V8v1JMYqQWJ9mURBEDDYHcTMYhYl9u9uewwSbVB9uWk5k8ggkYiIqGbF8nRSd42ZRHd5cI1T3/j+/fMf4ZmXLkLVdBza3Yn2oBuzseyWGbQztZCBJAqIRnzNPsqGgn5Xy5WbJmwKEgGjL1HTdUwtZOu+L2pt1jZu0gpmzFf1nkSWmxIREdXMDPK22uCaM2MxtAXc+JN7hiuTW09dXMTUQhY7ekNNPl19FFXD+FwaA10ByA6ebGoK+VyYnM9A1TRIovPPCxiZREkUKoN36nFtX+LIOq+9l9+dXPXj998+UPcZyBla49XvcGbQx8E1REREjbMVy02TmSIS6SJ29IYqAeKQOWVyC/QlTs5nUFI07OgLN/soVQmWh9dkckqTT1K9RLqASMizYbKiGpxwSiYGiTaouieRKzCIiIgsq5SbyltncM3VuRQAYCi6nLXpCHvg98iYmE+3fPXR5ZkkAGBXf4sEiX43ACCVbY3hNaqmIZEuoN2GUlNgeQItJ5wSg0QbGLPWALHqTGJr/8AnIiJqBqvlprIkQhQERwaJ47PGm/GR6PLkT0EQMNgTQLGkYT6Ra9bRbDE2bQSJrVI2G2yxNRjJTAm6DnTYFCT6PDJ62n0Yn0tXZm7Q9sQg0QbVTzfl4BoiIiKrKuWmrtrfvrhk0ZGDYK7M3phJBJZLTls1o/Pyu5N4+d1JvH8pBkkUcGFyac0+NicxdyWmWmTCaSyVBwC017kj8VqDPUGkcyUk0q2RTaXGYJBog+qnmxp/as67kElEROR4VstNASNIdGQmcS4Nn0dCV9vKHXe9HX7IktDSGR1FNUohO8IeiBuVWzlEyN9amURzsqldmUQA2NlnXLB4+6O5dW9XKKn4/YdzmORexS2pqiDx5MmTOHbsGI4ePYqf/OQnN3y+WCzi0UcfxdGjR/HII49gYmKi8rkf//jHOHr0KI4dO4ZXXnllw/scHx/HI488gqNHj+LRRx9FsVhc9zEmJiZw6NAhfOELX8AXvvAFPPbYY9aeiTqYP7w3nm7KTCIREZFVZibQZSGT6HY5L0gsFFXMLGYx1BO64T2EJIno7woglS0hmWnNjE4sWYCuA11tzl99YTInhKZaJEg0dyTa1ZMIAJ++rR9ul4jn3rwKRV39e2ZhKY/R313BB2NxvHNu3rbHJufY8Kesqqp4/PHH8fTTT2N0dBS//OUvceHChRW3eeaZZxAOh/HCCy/ga1/7Gp544gkAwIULFzA6OorR0VE8/fTT+N73vgdVVde9zyeeeAJf+9rX8MILLyAcDuPnP//5uo8BAMPDw/jFL36BX/ziF3j88cdte3KqZWYGN8okilyBQUREZFnJ4nRTAHBJzgsSJ+bT0AEM9wRX/Xyrl5wuLhmlkJ3XZUmdLOQzBtekW6Tc1M4diaaQ343Dtw0glizgtTMzKz6n6zpefGcCz71+FelcCR6XhHiqgHyxdabBUnU2/Cl76tQpjIyMYGhoCG63G8ePH8eJEydW3ObFF1/EQw89BAA4duwYXnvtNei6jhMnTuD48eNwu90YGhrCyMgITp06teZ96rqO119/HceOHQMAPPTQQ5XHWusxnKDmTCKDRCIiopoVS0Ym0Vq5qeS4FRhXy8HfcHT1oS4D3QEIACZatJxvYckYunN9Ka2TLQ+ucXb21uz5/PBqHADw0dVE5WN2OHbXECRRwD+/frXyvlXXdfzDr8/j7391Di5ZxGc+NohbdkQAALOx1h6wRDeSN7rB7Owsent7K/8djUZx6tSpG27T19dn3KEsIxQKIR6PY3Z2FrfddtuKr52dnQWAVe8zHo8jHA5DluXKbczbr/UYgFFy+sUvfhHBYBCPPvoo7rzzznX/TpGIH7KFXzBrcXuM83Z2BdEd8RsfvLCIUHDlD0Vv+XbhsA/d3c6c8uXUc62HZ94cPPPm4JmJ1laqc3CNUzKJ5hv5Nz4wsjRzieyqb+69bhld7T7MxXNIZYsIldcztIrFpTxcsljp82sFlXLTFskkZgtGBs/n3fAtfU06wl7ce7AXJ9+bxtvn5vHxm3vw//12DL9+ewID3QHcc2sUAa8LLtlIgMzEshhpkQm2VB17X1FN0NPTg5deegmRSASnT5/Gt771LYyOjiIYXL10AwDi8aytZ8iW69YT8SyEayanpdL5Fbczr4DGEhnMzzvvqlp3dwjz86lmH6MmPPPm4Jk3B8/cGAxitw4zE+iycKHXLYtQVA2artuydNwOsWQBogC0rTOZcqgngPlEDqcuLuLeg32beLr6FEsqktkS+jr9G05/dxKPS4JbFlumJzGbV+B1S5AaMBjoc3eP4JVT0xh9bQzJTBH/49XL6Grz4i+/fDveu7gAAOhs80GWBMws2vvemppvw0tx0WgUMzPL9cizs7OIRqM33GZ6ehoAoCgKUqkUIpHIml+71scjkQiSySQUxbgqMjMzU3mstR7D7XYjEjFS3QcOHMDw8DAuX75s6cmwSin/0pKljcpNjT853ZSIiKh2y+WmFjKJ5eyjU7KJmqYjniqgLehZ9w3+YLkv8d0LC5t1NFsstGA/oinod7VET6Ku68jmFfhtziKaoh1+fPzmHlydTeM/v3AO4YAb//7f3L6i/1ESBXS3+7CUKSJXYF/iVrLhT9mDBw9ibGwM4+PjKBaLGB0dxZEjR1bc5siRI3j22WcBAM8//zzuueceCIKAI0eOYHR0FMViEePj4xgbG8OhQ4fWvE9BEHD33Xfj+eefBwA8++yzlcda6zFisRhU1filYT7G0NCQfc9QFczJT7K0/tPJ6aZERETWLWcSrQ2uAZwTJCazRaiajo7w+gNH2gJuhPwunL4cc8zZq2EOrWmlfkRTyOduiRUYRUWDqunwexpXGPjgPSMAAJ9Hwl9++TZEzbaqa/R1Gh+biTGbuJVs+KqSZRmPPfYYvv71r0NVVXzpS1/C3r178cMf/hAHDhzAZz7zGTz88MP4zne+g6NHj6KtrQ1PPvkkAGDv3r343Oc+hwcffBCSJOGxxx6DJBklIqvdJwB85zvfwV/8xV/gBz/4AW655RY88sgjALDmY7z11lt46qmnIMsyRFHE9773PbS3tzfkyVpLtUGi+Vmdg2uIiIhqViq3dLhdFspNy1/jlEArljT3260fRAmCgMHuIM5eieOj8TgO7OzcjOPVbTHZ2pnEwqyKYkm19FrbLNm8kblrVCYRMIYq/fuv3I6OsAd9nYFVb9PbUQ4SWXK6pVT1qjp8+DAOHz684mPf/va3K//f4/HgqaeeWvVrv/nNb+Kb3/xmVfcJAENDQ5W1F9da6zGOHTtWmYbaLCXVCPo2LjdlJpGIiMiqYp0rMIDlQLPZ4ikjiNookwgAgz0BnL0Sx3vnF1smSFxYysPnkRqa5WqUUGXCaQkdrRAk1vkcrzcR9f7bB3Drzo51v74j7IVLEplJ3GJq/ylLN1BUDbIkbNiYXelJZIxIRERUs1LJnG5qYQVGuSfRKWswzExiNfvtohE/fB4Z715YcMz6r/UspQvI5hV0hr0tNbTGZK7BcPqE0+XJps2dHiuKAno6fEhlS4gl8xt/AbUEBok2UBRtw1JT4NrBNc7/AU9EROQ0hXIWsNV7EnVdRyxZQNDnqirgFUUBB3d1YDGZx2QL7Ey8NJUE0Jr9iMDyGgyn9yXm8sb5nJCtNUtOzb2N1PoYJNqgpFYbJLLclIiIyCozk2glSHQ7aLpprqCgUFKrKjU13b6nCwDwhxaYcnp6LAYAiHbeOOSkFZjlpqlcscknWZ+ZSWxkT2K1KkHilUSTT0J2YZBoA0XVqvqFxUwiERGRdcVy5Y6VPYfmbsWiA3oS46nqS01NB3d3QhQEvOfwIFHXdbx/cREuWUR3m6/Zx7Ek5HcDgOPXYGzG4JpqRcIeuF0iM4lbSPNfVVuAoupVLTEVwEwiERGRVSVFtTS0BnBWuWkibWSo2oPVB4kBrwv7htrw0dUEljJFtAXcjTpeXebiOSws5TEcDUJswIL3zRD0tUa5abagQJYEy98T1VhvqM21REFANOLH+FwaC0s5dLXoBQJaxiDRBoqqweveuGnYvPDJGJGIiKh2RUWrlI3Wyknlpom0kUlsC9YW6N22pwsfXk3gD+fmcf8dAwDWfhN//+0D9R3SovcvLQIA+rtWX5fQCsyexJTTg8S8Ap9HdsxwoEjIg/G5NObiDBK3AgaJNlBq7ElUWW5KRERUs2JJhVu2tpLAzCQWS/YGic+9NoZUevWJjmsFakvpIkQBCPtrCxLvvKkH//03l/BPv72Mu/dH4XPAwJLrnb5s9CO2cpBYWYHh4HJTVdOQL6o1X2hoJHOAjnkRhFobexJtUFL0qoJEkT2JRERElpUUrbLKolbm15XU5mYSdV3HUrqIcMBdczlmZ5sXxz8xgkS6iGdfudSgE1pXUlR8eDWO/q5ApWSzFQUqKzCcO7gmlzd6a50w2dTkK/dGLqWd+7xR9Rgk1knXdWNwjVRFTyKnmxIREVlWVLQ6ehKNDGSp1NzBNfFUASVVQ1sN/YjXevCeEUQ7/Djx9gSuzKRsPl19zk0soVjScGCD5etOJ0sifB7Z0T2J2UJ5/UWTdyRey8xsx5lJ3BIYJNbJLB2Va5huqjOTSEREVBNd141MosVyU7dDMolTC8aew3aLZYIuWcS/e2AfdB34T8996KgLz6fL/YgHd3U2+ST1C/lcju5JrEw2dVAm0e8xvjeZSdwanPPKalFmA3xtexIbeiQiIqItx/x9a3lwjdyYnsRamUGi1UwiAOzf0YF7bo3i9TOz6In4cPNIxK7j1eX0pRjcsoh9Q22YX8o1+zg1uX4AkKbrSGWKeOkPE/jjOwabdKq1OWlHosnrliEAWGImcUtgJrFOSvmKpKuqINH400lX/YiIiFpB0QwSLWYSzYqfZmcSJ81MYp0rLL5yZC98HhlvfzSPS1NJO45Wl1gyj8mFDG4ajljO9jqJ1y1B05v/elmLk3YkmkRRQCjgrqx4odbGILFOimoEfFI1PYnmnkSmEomIiGpSLPcSWu1JNIPLUrMziYsZCAIQqjNIbAu48b/9q1shCgJePTWNNz+Yber0dHOq6YFdrd2PaPK4jddLodjcHta1ODFIBIwy6kSmAJ0JkZbHILFOzCQSERE1nllu6rIYJHrKZar5omLbmWql6zqmFrII+92QbFg0f2h3J45/cgTtQTc+vJrAr968ilyhOX+/9y9unX5EAPC4HB4kFhQIAHxupwWJHhRLxnoOam0MEutkBonVDK4RBWYSiYiIrKi33NScApltUhAFAIl0EbmCYutuu3DAjc/dM4IdfSHMJ/J46Z1JqJtcIpnNl3Dq0iKiER+ika2xRN1bziQ6NdjJ5hV4PVLNa1Qara2cIeeuxNbnrMsPLai2wTXGn4wRiYiIalNUyuWmFgfX+MqTF5uVaQOunWxqfWjNalyyiE8d6oMoCLg0lcQbH8zhyB8NVgbmNdprZ2ZRUjTcd6hv0x6z0bzlDF2uiZnntei6jmxBQcTm15EdljJGP+Jv3p1Cb6d/xefuv32gGUfa0PVDi0xOPe9mYSaxTmZPYi3lpqzTJiIiqo05ldRquakkivC6pUovVzMsTza1L5NoEgQB99waRWfYgwuTS3jxndXf+NpN13WcfG8KkijgvoN9m/KYm8Hs9Wvm62Ut6VwJmqY7rh8RWL4Y08yMPdnDea+uFrNcblrF4BqWmxIREVlSqmQSrU/O9Hvlpr55nVpsTCbRJEsi7r9jAKOvXcF/+fU5zC/l0NuxMpsTCnrxsT329Q2OzaQwPpfGH+3rrmuth9M4OUiMp4xSTmcGieUMLIPElsdMYp3M0ci1lZsySCQiIqpFvZlEwFg83sw3/ZMLGYiCgHDA1bDHCPhcOHx7PwDglfemKhezG+Xke1MAgE/ftnWyiMDyknonZsTMfj/zjE7iZ5C4ZTjv1dViFKWG6ablFRjNHFFNRETUikqVwTX1BYmThQw0Xa8Mk9ssuq5jeiGDnogPklj732GtvqnVRDv82L+jA2cux3BpKol9Q+01P1418kUFr38wi46wBwd2bo2ppiaXLEKWBGYSa2RmEp34vFFtnPfqajHLexJr6El05l5WIiIixyraUm7qgg4gX1Aq0043SzJTRCav4KbhyKY83i0jEZwdi+HslTj2DrY1ZKDMW2fnUCiquGmoHSdPTdl+/80kCAL8Xpcjgx0zSPQ5MJPIctOtg+WmdVrek7jxD9/KCgyWmxIREdXELDetJ5PYzCzHZHloTX+Xf4Nb2sPvlbGjL4yldBFTC9mGPIYZGO4ZbGvI/Teb3yujUFIr/bBOYZabBhyYSRRFAV63xCBxC3Deq6vFKDX0JJq7bBrdH0BERLTVmJlEl8U9icDym+pm9JlNVYLEwKbt3rtlJIJLU0mcvRLDQHfA1vuemE/j4mQS/V0BBH2bm5XdLGZ/XTxdRE+7c/Y/xirlps583n0eGalssdnHqNt2X43BTGKdKoNrqriyaTbb5xy6mJWIiMipbOlJbOLEyqlFI5vX32lvsLaezjYvohEfphayti83f/HtCQDAvqGtmUUEll8v8WS+ySdZKZEqwCWJdQ1xaiSfR4ai6pXvWWpNznx1tZBaBteY38x5puCJiIhqUjR/37rqG1wDNCeTeHkqCUkU0Ne5OeWmplt2GD2QZ8fitt1nOlfC707PoKvNi8GeoG336zSVINHmALte8VTBkUNrTOauxFYsOX3vwgKePXmpUrmwnTFIrJM5uKaaclNJFCAIQK7Yet80REREzVQq9yR66ig39TUpk5hIF3BlNoWbhtvrKpe1YrAniJDfhYtTSeRtev9x8r0pFBUNn/nY4KZPid1M5kWFRMo5pZO5goJMXnF0kOjk9SHryeYVvH8phlS2hOkG9fG2EgaJdVouN934h6QgCHDLEnIFXp0gIiKqRcHsSawrk2j0cGXzJVvOVK33Ly4CAA7t2vw1EaIg4OaRCDRNx4WJpbrvT1E1nHh7Ah63hE8d6rfhhM4VKPf8xVLOKTedjRvBSzjgbvJJ1taqE07PXI5BK6+pm5zPNPk0zefcyxAtwiw3lavceeSSxZb7piEiImq25Z7EelZgNCfDceqSESQe3N2cXYK7+sL4/YdzuDqbxicsfP21AzzGppOIpwq4ebgdb344a98hHch8vSRSzik3nYk5P0g0n7ecA9eHrCWbV3BuPIGAV4aq6ZhcSEPX9YasjmkVzCTWSdGqH1wDGEGiXeUeRERE20WxZE43taEncRPfvCqqhjOXY+hp96G3Y3P7EU0et4RoxI+FpTzSufqyqGevGL2NN49szr7HZvK6JQjC8l5CJ5iL5QAAYb9zg0RfC5abnrkcg6rpOLi7E/1dAeQKamWK7HbFILFOimKkpasZXAMYU9nyBZW7EomIiGpgx3TTZqzAOD+xhHxRxcHdnU3NSgxHjQEzY1PWS04XEjnMJ/IY7A44OpNlF0EQ4PfIjhpcM1MpN3Xm+gug9cpNr80i7h5oq6yL2e4lpwwS67S8J7G6H/wuWYQOoMA1GERERFWrTDdtsRUYpy4uAABua1KpqWmoPIX0Uh1BoplFNCembgd+r4yldLHSq9Zss7EsZElAwMG7KZeDxNZ4r3ttFlESBfR3BSAAmJxPN/toTcUgsU617EkErtmV2CJXV4iIiJygWFLhlsW6snFejwwBm5tJPHVxEW6XiJuG2zftMVcT8LnQGfZgci5taXBPoaTiykwabQF308pmm8HvdUHVdCQdsBxe13XMxHLoifgdPVVWEgV4XFJLvNfN5EsrsogA4HFJ6I74sJDII7+NkzoMEutkZhKrLTc1R1+3wjcOERGRU5QUre7l4aIgwOuRNy2TOJ/IYXoxi/0jHZu++mI1Q9EQNN0IXGs1Np2EpuvYPRDeVsM8zD5WJ/QlprIl5AoKohFfs4+yIZ9HaomexEtTSaiajt0DbZDE5df1QFcAOoDphe1bcsogsU6V6aZVB4nlTOI2vjJBRERUq6Kiwu2qP9Dye2RkC5uzAsMMxpo11fR6w+WS03fOzdf8tRcnkxAA7Opvs/lUzmaWKDshSDQnrdebpgAAIABJREFUm7ZCJtfnkVFStEoyxanGZlIAgM4274qPV/oSGSSSVYpq1KhXGyS6WW5KRERUs6INmUTAeNO/WZnE9y81bz/iatqCbrQF3Xj/UgwlpfqL1Yl0AQtLefR3BRy9xL0RnBQkzpaDxGgLBImVNRgOf797pRwkdoQ9Kz4eCXng88iYnM9A36bDJhkk1qlkYXAN4PxvGiIiIicplbS6diSa/B4Z+aIKVWtshkNRNZy9EsdAd+CGLEWzCIKAnf1tKJRUfDAWr/rrLk4mAQC7B8KNOppjmeWmCQdMODUnm7ZKJhHY3CFRVlyZScHrlir/ziZBEDDQHUChpGJxKd+k0zUXg8Q6qRYH12znRlgiIqJaGeWm9mQSgcZPXpyYS6OkaLhtd1dDH6dWZrlotSWnqqbh0tQS3LJYmZC6nZivl1iy+UHibHlHYitkElthDUY6V8JiMo/OsHfVPtuBLqPkdGKbrsJgkFinkqpBEoWqp0yZ/RROv7JCRETkFJqmQ1H1unYkmvybtCvx/ISxauLeg70NfZxaRTv9CAfc+MP5har6xc5cjiNXULGjLwypytaarcR8vTghkzgby8LnkRD2O3f9hcnfAmswxmaMDHnHGpn+nvKAICf82zfD9vtut5mi6FX3IwLLU1DzRQaJRERE1ShVdiTaUW5qvMHONfBibSpbxPRiFvsG29DXGWjY41ghCgI+flMP0rkSTl+ObXj7V9+fBgDs2YalpgAgiSJCfhdiTe5J1DQds/EcohF/S0yXrZSbOjiTaPYjdl7Xj2jyuiWIgoDMNk3sMEisk6JqVfcjAtf2JDr3ygoREZGTFMtDVmzNJFrYFVgtM4t4+PaBhj1GPe49ZGQ3f3tqet3bpXMlvHt+Hm0Bt2P6KpshEvQgkSo0dYBJLJmHomot0Y8IXJtJdG6AVZlsGl79tS0IAgI+GZnc5kxDdhoGiXUqqVrV/YgAKv0UTv6mISIicpJKJtGOnsTym9dGZQc0TcfFSaOH72M3dTfkMeo1Eg1hsDuAdy8sILXOkvhXTk1BUXXsHmxriexVo7SHPCiU1Ka+dzOH1rRCPyJg7EkEnP1+98pMCiG/a92JvQGvyxh05fBVHo3AILFOiqpVSkirsbwn0bnfNERERE5SKJmZRBvKTRvckzgxn0auoGJXf9iWvY6NIAgC7j3YB1XT8foHs6vepqSo+NWb4/C6Jewb3F67Ea/XETLKEZu5BmN5aI2vaWeohSSJcEmiYwc1pnMlLCzlMRINrXsBJOBt7EUlJ2OQWCdF0Wpq5K5MN3XwlRUiIiInMTOJtpSbNng0//lxo9R071B7Q+7fLp+4tReSKOC3769ecvrb0zNYyhRx/x0Djg12N0u7GSQ2cYDJTKx11l+YvB7JsTM4rswapaYjvaF1bxfwGT3MmQaWpzsVg8Q6KaoOVw09iZIoQpZEZNmTSEREVJWineWmDcwkpnMlTC5k0NXmRSS0+jAMpwgH3Di4qxNXZ9O4Wn7DbNI0Hc+9fhWyJOCBjw816YTOYf5bxpu4BmO2HCRGIy0UJLol5IuqI5fRm0NrdmwUJJqZxJwzg91GYpBYJ2NwTW1Po8/BV1aIiIicpmRruWnjpptemGiNLKLpvkN9AIDfvj+z4uO//2gOc4kc7j3Yh/ags4PdzRBxSCaxLeCuTA1tBV63DF0HCiXn9fOZQ2uqzSQ2ctCVUzFIrFOtg2sAwOeWHd3IS0RE5CTFRpSbFux/0zc5n4YoCBtmJ5zi0O5OhPwuvHZmprIzUdd1/PNrVyAIwJ/cPdzkEzpDJNjcnsSSomExmW+ZoTUmr9u4qOPExMiVmSSCPteak01NZiYxvQ17ElvncoQDqZoGXUdNg2sAY3fMUmbtaWJERES0rBIk2tAbt7wCw943fYqqIZYqoDPsrcwfcDpZEnHP/l688Ptx/N/PnsbNIxGIAnB1Lo27bulpqdLGRoqEjECiWUHifCIHXQd6W2RojclbviDjtOE1mXwJ84k8bt3ZseHUXrPyoN41GC+/O7nqx+936JocgEFiXRTFqLG2Um5aKKnQNB2iuH1HShMR0cZOnjyJ73//+9A0DY888gi+8Y1vrPh8sVjEd7/7XZw5cwbt7e148sknMTg4iImJCTz44IPYuXMnAOC2227D448/3oy/Qt2K5XJTO4Ivj1uCIAAZmyt6YskCdB3oarF9gp+5cxDvnJvDuxcW8O6FhcrHH7xnpImnchafR4LHLWExmW/K41f6EVs2k+isINHsRxyJbpzxd8kiPC6pYYOunIxBYh1K5dIMuYbBNQAq9eT5olK5QkFERHQ9VVXx+OOP46c//Smi0SgefvhhHDlyBHv27Knc5plnnkE4HMYLL7yA0dFRPPHEE/jBD34AABgeHsYvfvGLZh3fNnZONxUFAX6PbHtP4sKSsaKgq9352Z7rsxoPfmIEqWwJPe0+XJ5OoTPswXAVb6C3C0EQMNQTxMXJJeQKyqb3BU4uZAC01tAaAPCZQaLDWqyqHVpjCvhkJDNF6Lq+rfaFtkY9hEOZ9fu1Xtn0uhu7o4mIiLaGU6dOYWRkBENDQ3C73Th+/DhOnDix4jYvvvgiHnroIQDAsWPH8NprrzlymmA9KtNNbRhcAxglp3b/Dl5IGFmmVsskAkYQFA64cc+tvfi3n92LB+5iL+L1bhpqh64DFyeXNv2xz16JAwD2DLTWvkrz/a7jMonlab7D1QaJXhcUVXfkAJ5GYiaxDkr5l5Yk1l5uCgB5rsEgIqJ1zM7Oore3t/Lf0WgUp06duuE2fX3GlEpZlhEKhRCPG28qJyYm8MUvfhHBYBCPPvoo7rzzzg0fMxLxQ7YpGLOqu3vlmzd3OXPT3RW44XMAEAquHpitdlsACAc9mJxLr/n5mlxYRCjoRSxVgMctob9neTn3Wve/1nk3S63Pl1PPu1m6u0O489Y+jL52BROxHP747rVfN7a8pq6RKyg4P7GE3YNt2L2js/Jxu56TRj63Zkyl6sbj2PHc2HEf07EcfB4Z+/d0QxCEDZ+D9rAX43NpoHzbWs5g3rbW7zknYJBYB0UzrtS6ZGvlpswkEhFRo/T09OCll15CJBLB6dOn8a1vfQujo6MIBoPrfl08nt2kE66uuzuE+fmVe/tiCaOUM5cp3PA5AEilV+8VW+22AOCWROSLKqZnlmqeK7Dq48TSSGaK6O8KIJ1ZHm6y1uOvdd7NEAp613z8Z174cJNPs7H1zrtZ5udT6A66IQB498NZ/Mmdg6vebrXXbr3evbAARdVw81D7ivu24zlp9HOrKsb73FSmgFQ6X/dzY8fzW1JUTM6lsWsgjIWFtHG+DZ4Dd7mtbC6WgdclVn2Ga89b68+ozbJekMpy0zqYmcTaB9cs9yQSERGtJRqNYmZmeYfd7OwsotHoDbeZnp4GACiKglQqhUgkArfbjUgkAgA4cOAAhoeHcfny5c07vI1KSnlPog3TTYHlNRh2raNq5VJTqo7fK2OoJ4hL06lKj+xmOHMpBgA4sLNj0x7TLh6XMSQq56DKuamFLDRdx1DP+hfLrmWuwchss+E1DBLrsDy4ptY9icYvOSd90xARkfMcPHgQY2NjGB8fR7FYxOjoKI4cObLiNkeOHMGzzz4LAHj++edxzz33QBAExGIxqKrxe2Z8fBxjY2MYGhra9L+DHZZ7Eu152+Lz2lvRs7BkBInd7QwSt7J9Q+1QVA2Xp5Ob9pinLy/C65awu8X6EQGj19XrlhyRFHn53Um8/O4kXvj9OAAgl1cqH9tIwGfPGoxWw3LTOlgdXOOz+QomERFtTbIs47HHHsPXv/51qKqKL33pS9i7dy9++MMf4sCBA/jMZz6Dhx9+GN/5zndw9OhRtLW14cknnwQAvPXWW3jqqacgyzJEUcT3vvc9tLe3N/lvZI25AsOO6abAcibRrrH28+Vy2M425082Jev2DbXj129P4PxEAvuGGv+9NJfIYTaewx17u2wpi24Gr1tG2kHBlbnrMhLyVP01AXNX4jbLJDJIrIPVclNzuWjOAVdWiIjI2Q4fPozDhw+v+Ni3v/3tyv/3eDx46qmnbvi6Y8eO4dixYw0/32ZYyhQBACG/25b783vtCxJ1XcfiUh4hv6uyF462pr3lwPCj8QSOf6Lxj3fm0iKA1iw1NXndEuKpAlTVGZNBzSCxvYYg0eeRIArMJFINSqoxuKbmPYksNyUiIqpaPFmAzyPbtp/OzAzYUW6aSBdQVDQMdAfqvi9ytraAG9GIDxcnl6BpOkSxsTvz3i/3I6bzparKIp3IvHDihDUYuq4jniog5HfVVAUoCAL8Xte2yyS2Zu7aISrlphYH17DclIiIaGOxVAEd4eqv/G9kudy0/szAbMyYBtvVzlLT7WDfUDtyBdVYidBAiqrh7NU4Qn6XbRn0ZnDSrsRcQUWhpNZUamoKeGXkCgpUbWvtoF0PM4l1MINEyep0UwaJREREKzz32tiKcfElRUOuoKAjZN/gDjsH15hBYjcnm24L+4ba8cqpaZwbT2CkymXsVlycXEKhqGJHAx9jM3jN3eAWWqyuz55eu7Lj/tsHar6/Sqlp0EKQ6HMB8ZwtF5ZaBTOJdah7cI0DrqoQERE5Wab8pszK1f+12Dm4Zi6WhSgIiNiY6STnMvsSz00kGvo4ZqnpQFdrlzGbmUQntFjFU0aAaTWTCGyv4TUMEuugWOxJ9FZ6ErfPC42IiMgKM5CztdzUpkxiSVGxkMijI+yBJPIt1XbQ3eZFe9CN8+MJ6HpjSg91XcfpS4uQJQHRDn9DHmOz+NzWM4l2MzOJVn6WbMc1GPyJVoeSxemmsiTCLYsMEomIiDZgXrlvRCYxV2dWYHwuA03X0clS021DEATsG2pHMlvCTLnU2G5vnJ3F1bk09u/osG03aLM4aXBNPFWALAkIlgO+WmzHNRjsSayD1cE1gFFyynJTIiKi9Zk9QB3h2gOxtSZC3rM/CqD+N3xTCxkA9gaw5Hz7d3TgzbNz+PE/ncHh2/shCEZFmdkzZ6VfzhRPFfD3z5+DxyXhq5/diw+uxO06dlM4ZXCNqmlYyhTRGfZW/r1qEfCVy02ZSaRqmEGibOEqj9cjM5NIRES0ATOQ67AxEPO4JEiigGyhvjd8U4tGkNgWbN3pk1S7ew/24ubhdlydTePsmH1BnK7r+H/++SyyBQVfObIHPZHWLjUF6htcY6eldBG6bv2CDjOJVBOr5aaAUaMdT+Y3viEREdE2lm3A4BpBEODzyHUPrpkuZxLbAswkbmWrZaQP7u7EldkU3j43j852L6I2BHQv/2ESZy7HcGBXBw7f3l/3/TmBLImQJaHpg2vMfkSrA6Zcsgi3S6wM0toOmEmsg9XBNYBRblpUtEo2koiIiG6UyStwy2KlbM0ufq9c9+CaqcUMfB650ndF24fPI+PTtxmB3Ml3p+qqDtN1HWcux/DfXrqAgFfGn37uFkslkU7ldctNLzetBIl1XGwKeF3I5EoNG1jkNMwk1qFSbmqxJxEwarSDPsbqREREq8nmlcr4eTv5PTIS5TeOVhRLxmTTvhZfUUDWRTv8uGNvF945t4Dn3riKg7u7MNBVfUYxX1Twxgez+PXbE5icN7LS/8uDt2y5HlevW8JiMg9d15sW/FaCRAs7Ek0Br4x4qoBMXrE0/KbVMEisQ12Da65Zg7EdXmhERES1KioqSopWGT9vJ7/XqOgpKZqlCZIzsSx0rD9QZ63BObR13LqzA9mCgnNXl/C796chCMDvTs/A55EhiwIkSYSm6SiUVOSLKjwuEalsCalsCYWSkV2TRAE7ekO4ZUcE2YKy5V43Xo8MXTdWzpi9fZstniog4JXhdlnP+ps/hxKpwrZ4784gsQ71DK4xM4kcXkNERLQ6s2fQXFlhJ/PNajpXspS5qUw2tTB1lbYOQRBw1y1R3La7C1OxLE5fXKxkBVcjSwJCfjeiER9Cfhd29rfhj+8YwHsXFzbx1JvLLMdOZopNCRKXMkXkiyoGu+vL+ps/h2KpAgZ7gnYczdEYJNahpFjvSfQySCQiIlpXJmf8jmxEJrEn4gMAzMWz1oLERWNH3lYrDSRrPG4Jh/Z0Y0c0iFxBhaIacydUTYcoCPC4JXhcEj77scEt1W9YDd81QWJf5+aXZ1+cXAIAdLX76roff7nsPZ7aHoMnGSTWoZ5y08oiX+5KJCIiWpW5oqIRmURzGuV0LIubhiM1f7052bQj7IWm8nc5GQRBqAQTa31+uzGHTqWytU8GPTeewLnxBI7eOYSQxcc/P5EAAPTUGSSaWdB4Hb3MrYQTU+pQ357E5Z5EIiIiutFyJtH+ILG3wwgSZ2NZS19vTjZdLyAgouX3vMlssaavyxcVvP3hPGLJAi5NJS0//vmJJQgC0NVeX2m4+b0eY5BIGynVM920fFUlzyCRiIhoVeaKCr/H/nLT3k4zSMzV/LWKqmEunkN/l39bZoaIanFtT2ItTl1crLzXvlAuGa1VoaTiykwKnWGvpffr11ouN2WQSBtQFDNItLYnEWC5KRER0VoyOaM8rRGZxKDPhYBXxoyFTOJsPAdV09HfhP4qolZjpdx0Np7FR1cTCPpcGOgOIJ4qYD5R+wWdsekkVE2v9CDXQ5ZEuF0ig0TamKLpEARAEq1MN2W5KRER0XqyBQVul1h3BmAtvR1+zCdylfaRapn9iM0YwkHUasz3vLVkEv/xN5eg68Af7evCvqF2AMCHY7GaH/vchJGB7K6zH9EU8Lo4uIY2piiapaE1wHK5KYNEIiKi1WVzCoL+xo3M7+3w4+JUEotLeUQ7ql+CPrVoBIn9XQGU9EadjrairbYDsRpulwQB1fckXppK4q0P59DZ5sVIbwi6bpSsnrsax8FdHZDE6iv4LpSDRDsyiYBRchpPFZArKJWqwK2KmcQ6KKpm+erm8p5ElpsSERFdr1hSUVK1hg6GMQPDWktOzR2J/Z3VB5ZE25W5AiRZRbmprut45qULAICP3dQNQRAgigJ29YeRL6qYmEtX/biapuPC5BKiEZ9tAZ05aXk7lJwySKxDSdUtTTYFWG5KRES0nmy+PNm0gUFir8UgcXoxC7dLREdbfdMSibYLr1tCqopy0/mlPD4aT+DWnR2V708A2D3QBqC2ATaTCxnkCgr2DLbVfuA1BLbR8BoGiXUwyk2tTTUzm3jzRQaJRERE18uUg0S/t3HlplELazA0Tcf0YhZ9nQGInGxKVBWvW0a2oGzY/3tlJgUAuHVHx4qPR0Ie9ER8mJrPVC4gbeRCeT/i3sF2CydenfnzKLYN+hIZJNahnnJTUTRS7yw3JSIiulE2X55s2shy03KfUi2ZxIUlY9ANS02JqlftGgwzSByJBm/43M0jHdABXJqqLpt4vtyPuNfGTOJ2WoPBILEOiqpZLjcFAJ9bYrkpERHRKpYziY0LEt0uCZ1hD2bj1Y/Wn1owAsr+Lk42JaqW2RMYT68fXF2ZNYLE4d7QDZ/bO9wOUQAuT6eqeszzE0sI+lwrylbrxSCRqlJSNcgW1l+YfB4ZOZabEhER3WC5J7Fx5aaAUXIaTxWqbv8wJ5ty/QVR9doCbgDLQ59Wo+s6rsyk0N3uXfX73uuW0d9l7Exc734AIJbMYzGZx97BNgg2loWb52KQSOtSVR2ybP2F5/PILDclIiJaRaZcbtrITCKwPLxmNlZdNtHckchMIlH12kJGkDi9sHZpdyxZQDpXwkj0xiyiaUdfGADw5tnZdR9vudTUvn5EAHDJInweCbEkg0Rag6brUDXd8p5EwCg3VVQNJaW2Jb5ERERbXTavwOOSLPf+V6syvCZeXV/i5ZkUPC4J3e2cbEpUrfagB4AxcXQtZqnpyCqlpqahniAkUcCbZ+eg62svKT17JQ4Atk42NUVCXsQ5uIbWopQDu3p+efnKKWuzOZ+IiIiMsrNMvtTwLCJQ2xqMbL6EqYUMdvaFINXRbkK03XhcEtqCbkwtrL3ncGxm4yDRJYsY7AliJpbF+Bo7E9O5El7/YAadYQ929q19X1ZFQh5k8goKpa1dDcifcBaZI3zrCRIj5asqsW1Q10xERFStoqJBUfWGTjY1RWsIEi9NJQE0JjtBtNUNdAWwmCysObTxqjm0Zp1yUwDYUQ4i31ij5PTlP0yiWNLw2TuHGnIxJxIy3r8ntvj7dwaJFpVUI8Vdz3TTzvIS3sWlrZ+yJiIiqtbFcj9R5yYsq+8KeyFLQlW7Es1F3rv7GSQS1aq/POzJHP50vSszKXSEPQj73evez0B3AF63hDc/uLHktKRoOPH2BHweCZ++rd+eg1+nI7Q9kjyNv0S3RZnlpi6p9sE1L787CQCYiRnfJG+cnUW6XHJ6/+0DNp2QiIio9RRLKt6/FINLFnHzSKThjyeKAnoifszEctB1fd1JiBfNIHGAQSJRrfq7y0HiQmbFhZaX351ENq9gKVPEUE+w8j55LbIk4o693XjtzAwuTSVXfD++8cEsljJF/Mldw5W1G3YzM4lW+xKz+RJ8HtnWqauNwEyiRWa5qVRHuak5RjeTY08iERERALx3fgGFkopbd3bA45I25TGjER9yBQXJ7Nq/jzVNx6XpJHo7/Aj6GruWg2grGuhaDhKvF0saAVdn2FPVfd29vwfAypJTXdfx/FtXIYkCPnvnYL3HXVMkZFQ4WFmDsZQu4r//5hJeO7P+dFYnYJBokRkk1jPdNFD+JZPOc1ciERFROlfCu+fm4HVLuGUTsoim5TUYa5ecTi1kkCuo2D0Q3qxjEW0pZpC42oTTxXKQ2BGursR8/44OBLwy3jo7V7m/M5djmJzP4OO39FR9P1bUU256dS4FXQcuTCxtuOux2VhuapFi9iTWESR6XCJkSWAmkYiICMC/vHEFRUXDnTd3w1VHz3+trp1wum9o9b1qF6ZYakpUD7/Xhfage9XgaLG8d7Da4E6WRNx7sA+/emsc/+HpNzDQHYCmGe/Nj3182L5DryJSznbGLexKnJw3/u6CALx2egb/+tO74HU7Mxxz5qlaQMmcbipbrycWBAEBn4tBIhERbXtL6QJO/H4CAZ8LN60RqDWKOeF0vSv7Zj/iHg6tIbJsoCuAM2Nx5ArKip7BWDIPn0eqae3Nw/fvxs6+MN48O4v3Ly1CUXX0dvhxeSaJyzPJRhwfAOD3yHC7xJrLTQtFFfPxHLrbvejt8OP9SzH8428u4atH9zXopPVhkGjR8uCa+q50Br0uLKWLKCoq3PLm9F4QERE5za/eGkdR0fCJQ/119ftbMRINweOS8M65eXzlyJ5VB0pcnEzC55HQXy6ZI6La9ZWDxKnF5eE1uYKCbF7BQHdt31uyJOLu/VHcvT+KbL6ED8bimEvkGnHsFQRBQCTkrXlwzdRCBjqAge4gbt0RwZXZNE68PYG7bok6cq0OexItsmNPIgAEfEacnsmxL5GIiLavdy8swOPa3F5Ek8ct4WM3dWNhKY/z5fUb10rnSpiJZbGrLwxRdPZEQiInqwyvmV/O2sfKZZuddfQR+r0u3HlzT02ZyHp0hDxIZksolZNG1ZiYTwMABrsDkCQRnzwQBQD89F/OVkplnYRBokUl24JETjglIqLtLZbMY3oxi5uG2zc9i2j6xIFeAMDvTs/c8Dmz1FSSRLz87uSK/xFR9Qa6ggBWDq9ZHlpT3WRTJzDXYCTS1ZWcarqOqYUs/B658rU9ET8+fksPphezmF5jd2QzMUi0yBxcU29jfdBrTjhlkEhERNvTmbEYAODWHR1NO8MtwxG0B91468M5lBR1xeculofWdLf7mnE0oi2jv2tl/2+hpOLc1QREUWip76/lXYnVBYkLiTwKJRUD3YEV5ezmoKzL0yn7D1kn9iRaZPYkSlJ9ZSfLmUSWmxIR0fZ05rIRJO7f2YGZhLUF1bVYKwM40B3EmcsxvHdhEXfe3FP5+IUJM0hs3Fh9ou3A73UhEvJgqpw5e/7Nq8gWFBzY1bFikI3TLa/BqO7n1WS51PT6vsudfcZKnbGZJO471GfjCevHTKJFduxJBIBgpSeRmUQiItp+NF3HB2NxtAfd6O/0N/Usu/qNN2zXlpyqmobL0ym0Bd1wuzhgjqhe/Z1+xJIFTC9m8M+vX4HXLeHgrs5mH6smkZBxwajaTOLEfAaiIKCvc2WQONgdhCQKjswkMki0yK7BNT6PDFEwmuKJiIi2m/HZNNK5Em7d2bHqVNHNFAl5MNwTxPuXFpHKFqGoGv7h1+dRKKnoaaFSOCIn6y/3Jf74n86gWNJw+96uTd2Lagez3HRxaeNMYiyZRzxVQLTDd8Pf0yWLGOwJYnwuVYktnKK1/kUcpFTuSaw3SBQEAX6vCxn2JBIR0TbkhH7Ea33yQC9UTcdL70ziyZ+9hxffmcRAdwCHdrdWpoPIqcySy6uzaQx2Bxy5/mEjvR1+CIKRIdzIqUuLAIys4Wp29oWhqHpl+qlTtE7xr8NUyk3l+q96BnwyZmM5qJqzriAQERE1WqUf0SFB4t37o/hvL13A/3j1MgDgjr1d+Prn9+ONs7NNPhnR1nDtrtGvfGYv5i3sNmz2ZGGPW0JfZwBXZ1PQdB3iOlUQpy8ZP+PW2gO5szeEl2EMr9nRG27Aaa1hJtEic3BNvZlEYHnCKYfXEBHRdlIsqTg/sYThniDCAXezjwMAaAt6cNvuLgDA5z+5A9/61wdbaqAGkdMNdAXg88j4o33djqkgsGIkGkS+qGI+vnaQq+s6zo0n4PfKCPldq97GHF5zeTrZkHNaxZ96Ftm1JxG4ZsIpS06JiGgbOTeRgKJq2L/TWW8Uv/75/Yin8hhYozyMiKzzeWT8X//7J+B1t/YgqJFoCK+dmcWV2RSiHasP3ZqJZZHOlbCjL7Rmz3Vflx9ul4gxhwWJzCRaZNfgGmA5SEwzk0hERNuIWWrqtGyC3yszQCRqoKDPZct76GYa6Q0BAMZm1p5Mem48AQCIRtYefCWJIoajIUzNvCAgAAAgAElEQVQuZFAoqWvebrO19r9OEymKObim/p5ErsEgIqLt6MzlOFyyiL0tOLiCiLa3oR4jSLyybpBo7Fjtiay/3mdnbxi6Dlyddc4qDAaJFimaObjGhkxipSeRQSIREW0PyUwRE/Np7Bts4/5BImo5fq+MnogPV2dT0HV91ducn0gg4JXRHly/53pnnxFwOmlfIoNEi+wcXBMoZxLT7EkkIqJtwizDumk40uSTEBFZMxINIZNXVt2XuJDIYWEpj72D7RvugDWH1zipL5FBokV2Dq6RRBE+j8TppkREtG2cmzCCxH1D7U0+CRGRNWZf4pVVykTPlPcj7h3auJy+J+KD3yM7asIpg0SLFNVIK9tRbgoYJaeZfAmatnq6moiIaCs5P74EWRIqZVZERK1mJLp2kPjBZSNI3De48YUwQRCwoy+E2XgOWYdUFjJItGh5umn9g2sAY8qTrgOJdMGW+yMiInKqXEHB1bkUdvSF4ZLZj0hErWk4akxBvjKTvuFzH1yOwS2LlWzjRiolp+sMwtlMDBItKtnYkwgsr8FYTN5Y00xERLSVXJxagq5Xd4WdiMipQn43OsMeXJlJrhhek8mXcGUmiV394apjhR295vAaZ5ScMki0yMwkSqI9mURzeM1qja9ERERbiTkWfl8VvTpERE42HA0hmS0hkS5WPnZ+wrgQtreGC2F7Boyfhx9dTdh+RisYJFqkqDpkSdxwWlG1gl5mEomIaHs4P56AgOU3RURErWq14TXnx2sfzNUW9GCgK4BzE4lKxWIzMUi0SFE1uGR7AkQACPqNIHFmMWvbfRIRETmNomq4NJ3EQHcQ/vIFUiKiVmUOr7l6TS/huYkERFHA7oFwTfd1y0gExZKGS1NLtp7RCgaJFimqZls/IgCEA264ZBEXJpv/oiAiImqUsZkUSorGUlMi2hLMTKI5cObqbApj0ynsGmiD1y3XdF/7d3QAAD4Yi9t7SAtqOzlVlBR7g0RRENDd7sPUQgZLmSLaAm7b7puIiMgprJRhERE5VXvQg7agG+fGE3jsP76JiXlj0uk9B3prvq+bhtshCgI+uBLDQ9hl91FrwkyiRYqqwWVjkAgYizQB4MKEMxpWiYiI7HauHCTWMtCBiMjJdvaGkS0omF7M4I69XfizLx7Aw0f21Xw/Po+Mnf0hXJ5KIVdQGnDS6jGTaJGi6vB7GxMknp9Ywsdu6rH1vomIiJpN03VcmFxCV5sXkZCn2cchIrLFv/3sXnz85h4c3N2JYHmtndUNCLeMdODiZBIfXU3g9r1ddh6zJswkWlRSNciSfYNrAKCrzQtJFHB+g0zimbEYfvbiBcwlcrY+PhERUSNNLWSQySssNSWiLaW73YdPHOitBIj1uHVHBADwwZVY3fdVD2YSLVJs7kkEAFkSsaPPSDEXiio8bqnyuZffnQQAaJqOfzx5Cdm8guffuopd/WH8r5/fj56I39azEBER2Y39iERE69vV3wa3LOJsk4fXMJNoga7rUDXd9iARMHo0NF1fc/Tt1bk0snkFvZ1+hP1uXJxM4v/4yRv47fvTtp+FiIjITu+cmwcA3MQgkYhoVS5ZxN6hdkwuZLCULjTtHAwSLVBUHQDgsrncFAD2Dhojwc9PrB4kfnjFuKpw9y1R/E/37cCnbuuDxy3iv544j2y+ZPt5iIiI7DAxl8aZsThuHm5HtIPVL0REa9lfKTltXjaRQaIFiqoBQEMyiXsGzCDxxr7ExaU85uI59HcF0BZ0QxQE7OwL4/Of2IFMXsEvX7ti+3mIiIjs8Ku3xgEAD9w13OSTEBE52/4RY19iM0tOGSRaUDKDRNn+py/kd6Ov048LU0momrbic2fLVxNuGYms+Phn7xxEZ9iDX/9+HAscZkNERA6zlC7g9Q9m0Nvhx6Hdnc0+DhGRow1Fgwh4ZZwZi0HX9aacgUGiBYpiBG9270k07R1sR6GoYmIuU/lYrqBgbDqFcMCN/q6VZTouWcKXDu+Gour4+W8uNuRMREREVp14ZxKKquOBjw9BFOxv1SAi2kpEQcBte7oQTxXwm/emmnOGpjxqi5PKwaHP25jhsMt9icslp+fGE9B0HTcPt0NY5RfsXfuj2NkXwptn53BxjaE3REREm61QUvHyHyYR9LnwiQO9zT4OEVFL+NLh3fB5ZPzsxQtYXMpv+uNzBYYFbQE3/uLLt2G4J9iQ+99bnvp2fmIJh28fwEfjcXx0NQGXLGJ3uWfxeqIg4CtH9uL//M/v4O+fP4evHNmDfUPtEC0u8qxHOlfC+Fwa43NpLCRySOdKSOVKSOdK0DQduq5D142lytdbL6OuV26jQ9P08tcLUFXN+JgO21LykihAkkSIggCfR0J70IP2kAcdIQ929IWxZ6DNll04RERb3e9OzyCdK+Hzn9wBj0va+AuIiAiRkAf/5sge/PRfPsR/eu5D/MWXb1s1UdQoDBItOrircT0V3W1etAXdeO/CAr791CvIF1UAwKHdnXCt0gdp7lAEgJ19IVyeTuFv/+EPCPlduGNvFzpCXgiiAFEABEHAqi8vAQgFvchlixBFAZIkQBZFyJIAWRIhiYLxwix/saJoKCoqioqGVKaI+UQec4kc5uJZJNLFVf9ekihAFAUYd2PcUa2vdfP2giBAFIz7g65DEARI4vL91kOHEawqqgZN05HMFjExn7nhdn2dfuzqD2OoJ4ShniAGugMIeGVIIhP0RESAcdHwV29ehSwJ+MwfDTT7OERELeW+Q31468M5nL4cw6vvT+NTh/o37bGrChJPnjyJ73//+9A0DY888gi+8Y1vrPh8sVjEd7/7XZw5cwbt7e148sknMTg4CAD48Y9/jJ///OcQRRF/9Vd/hU996lPr3uf4+Dj+8i//EolEArfeeiv+9m//Fm6329JjtCpBEHBwVydePTWN7nYvPn2b8YKIdvg2/Np7D/Vhz2AbrsykcHU2jZPvbe7+xIBXxkB3AJGgB5GwB2G/G163BI9basg02FDQi1S68Sn4kqIhm1eQzpUwn8hhLpHDfCKH6cUsgJkVt5VEAR6XBJcsQpZEyLIItyyiq82L3g4/dg93IOKXMRwNMqCkupQUFbPxHGZjOcwlslhKF6GoGhRVNzLsDX58AYDP50ahUIIgCHBJIlwu4/XucUsIel0I+lwI+l3oDHvRHvKwH82iRvwebiRd1/Hm2Tn8l1+fQypbwh/fMYC2oKfhj0tEznVtUoOqIwgC/uc/uRn/4T++gf964gJ297ehvyuwKY+9YZCoqioef/xx/PSnP0U0GsXDDz+MI0eOYM+ePZXbPPPMMwiHw3jhhRcwOjqKJ554Aj/4wQ9w4cIFjI6OYnR0FLOzs/jTP/1TPP/88wCw5n0+8cQT+NrXvobjx4/jsccew89//nN89atfrfkxJKm1S1r+3QP78K8+uQOdbV4IglD1N5YoCOjrDKCvM4C79uuIJfMoKRp03ciO6au9bSx/yOt1IZMtVko3Nc0o61TNEtHlm0ISBcjlkkyPS0TI70bA54LUhPLWzeCSRbQF3WgLujHQbXxzmlnGeKqAeLKApUwRJVWDomhQVA2qpiNfVKDmdSiqhvG5tHFnb1wFAPg8EvYNtuOm4Qh6Ij50hr2IhD3we2SIosA309vctWXZ2bxilGxni5hL5HB5KolLU0lMzGdWLdt2Kpcsoqfdh56Ir/xzyo++zgCCPhlejwxf+WLSZpbTtIJG/B5u5O/IZLaIn46exXsXF+GWRXz5j/fg6McHG/Z4duMbWSJyks42L758ZA/+3+c+wl89/QZ294dx1y1R3LU/iraAu2GPu2GQeOrUKYyMjGBoaAgAcPz4cZw4cWLFL6cXX3wRf/7nfw4AOHbsGB5//HHouo4TJ07g+PHjcLvdGBoawsjICE6dOgUAq97n7t278frrr+Pv/u7vAAAPPfQQfvSjH+GrX/1qzY9xxx132Pg0bR47fzmJgoCuto2zj/9/e3ceF1XV/wH8M2wugAsqaIWlPgHuaE8sAiKrJjuoaS/UwHJ5SlREWdw3RERNMRcsHzLpKZWtJFuEcAO0EiWTfBSVpQRREBnAGWbm+/uDZ84PlFlQcdDO+w9fzp177/neM+eec+695x7kntVTuReFlpag6V1Fg04Y0E/5ukSEB2Ip7teLIZYQyipqUV5Vj4tFd3Gx6K7iNASCNg/J5Z4/7DpPAOB/79aquvTT1hKgV/em8meor4duXXWh31m3abi4fFh3OxceIoJ+104Q1on+d0Erf4pJaJTKIBJLIWqU4oFY2vRu8v8ucv+8U4f8q3cU7lfwv3/0dLTxL79h7Tq8/3nQHu1we7aR2ef/xMWiu7Do3wPvvmUB455dVW/EcRzHKeQ48iV00tVGzm+3cLm4GkV/3cexs8XY+qF9u6Wp8iKxoqICffv+/2xkJiYm7EKv+Tr9+jX1knV0dGBoaIjq6mpUVFRg5MiRLbatqKgAgFb3WV1djW7dukFHR4etI1//cdJQpE8fQ1WH/cQmPIM0OI7juBdfe7XDijxpGznLbwRm+Y147O0fp/2c7Gbx2OlxHMe1F3l9+jTqKG/jbvAe9/oT70dd/IUojuM4juM4juM4jlF5kWhiYoLy8v+fmKOiogImJiaPrHPrVtMEKRKJBLW1tejZs6fCbRUt79mzJ+7fvw+JRAIAKC8vZ2m1NQ2O4ziOexG0RzvMcRzHccqovEgcPnw4bt68idLSUojFYmRkZMDZ2bnFOs7OzkhNTQUAfP/997CxsYFAIICzszMyMjIgFotRWlqKmzdvYsSIEQr3KRAIYG1tzSa3SU1NZWm1NQ2O4ziOexG0RzvMcRzHccoISI2/Pn7ixAlER0dDKpUiICAA8+bNw/bt2zFs2DC4uLhAJBJhyZIlKCwsRPfu3bFt2zb2gv3u3buRnJwMbW1tREVFwdHRUeE+gaY/gbFo0SLU1NRg8ODBiIuLg56e3mOlwXEcx3EvgvZohzmO4zhOEbUuEjmO4ziO4ziO47i/Bz5xDcdxHMdxHMdxHMfwi0SO4ziO4ziO4ziO4ReJ7eDkyZMYP3483NzckJCQ0O7p3bp1C9OnT8fEiRPh4eGBzz77DAAQHx8PBwcH+Pj4wMfHBydOnGDb7N27F25ubhg/fjxOnTqlMvbS0lJMnjwZbm5uWLhwIcRiMQBALBZj4cKFcHNzw+TJk1FWVqZ23M7OzvDy8oKPjw/8/f0BAPfu3UNQUBDc3d0RFBSEmpoaAE1/tHv9+vVwc3ODl5cXfv/9d7af1NRUuLu7w93dnU3cAACXLl2Cl5cX3NzcsH79eshHVitKQ5Xr16+zvPTx8cHo0aORmJjY4fI5MjIStra28PT0ZMs0ma/K0lAW86ZNmzBhwgR4eXnhgw8+wP379wEAZWVlGDFiBMvvlStXtktsio5fWcyaLguK0lAW88KFC1m8zs7O8PHx6VD5zP29POv280koans7MqlUCl9fX8yZM0fToah0//59hISEYMKECXjrrbeQn5+v6ZCUSkxMhIeHBzw9PREaGgqRSKTpkFpoS9+gI2hLv6AjaC1euf3798Pc3BxVVVUaiOwJEPdUSSQScnFxoZKSEhKJROTl5UVXr15t1zQrKiro0qVLRERUW1tL7u7udPXqVdqxYwd98sknj6x/9epV8vLyIpFIRCUlJeTi4kISiURp7CEhIXT06FEiIlqxYgUlJSUREdHBgwdpxYoVRER09OhRWrBggdpxOzk50d27d1ss27RpE+3du5eIiPbu3UuxsbFERJSdnU2zZs0imUxG+fn5NGnSJCIiqq6uJmdnZ6qurqZ79+6Rs7Mz3bt3j4iIAgICKD8/n2QyGc2aNYuys7OVptEWEomExowZQ2VlZR0un8+dO0eXLl0iDw+PDpGvitJQFfOpU6eosbGRiIhiY2PZ/kpLS1us19zTik3Z8SuLWZNlQVEaqmJubuPGjRQfH9+h8pn7+9BE+/kkFLW9Hdn+/fspNDSUZs+erelQVFq6dCkdOnSIiIhEIhHV1NRoOCLFysvLycnJiRoaGoioqf5OTk7WcFQttaVv0BG0pV/QEShqX//66y8KDg6mcePGPdLn7ej4k8SnrKCgAK+++ipMTU2hp6cHDw8PZGZmtmuaxsbGGDp0KADAwMAAAwcOREVFhcL1MzMz4eHhAT09PZiamuLVV19FQUGBwtiJCHl5eRg/fjwAwM/Pjx1TVlYW/Pz8AADjx49Hbm4ue6rwODIzM+Hr6wsA8PX1xfHjx1ssFwgEsLS0xP3793H79m2cPn0adnZ26NGjB7p37w47OzucOnUKt2/fhlAohKWlJQQCAXx9fVnMitJoi9zcXJiamuLll19WeiyayOc333wT3bt37zD5qigNVTHb29tDR0cHAGBpadnib7215mnGpuj4VcWsyLMoC4rSUDdmIsKxY8davQuqyXzm/j400X4+iba2vZpWXl6O7OxsTJo0SdOhqFRbW4uff/6Zxaqnp4du3bppOCrlpFIpHjx4AIlEggcPHsDY2FjTIbXQlr5BR/A0+gXPkqL2dePGjViyZAkEAoEGonoy/CLxKauoqEDfvn3ZZxMTk2faaJSVlaGwsBAjR44EACQlJcHLywuRkZFsGIGiGBUtr66uRrdu3diJ2bdvX3ZMFRUV6NevHwBAR0cHhoaGqK6uVjveWbNmwd/fH1999RUA4O7du6xi7dOnD+7evdtqzPIY1D2W5jErSqMtMjIyWnSmO3o+azJflW2jruTkZIwdO5Z9Lisrg6+vLwIDA/HLL7+oTOdpHb86NFUWnrTu+eWXX9CrVy+89tprbFlHzmfuxfM8l4eH296OKDo6GkuWLIGWVsfv+pWVlcHIyAiRkZHw9fXFsmXLUF9fr+mwFDIxMUFwcDCcnJxgb28PAwMD2NvbazoslZ5Gf0hTHu4XdETHjx+HsbExLCwsNB3KY+n4NQWntrq6OoSEhCAqKgoGBgaYNm0afvzxR6Snp8PY2BgxMTGaDrGF//znP0hNTcW+ffuQlJSEn3/+ucX3AoGg3e+8PE4aYrEYWVlZmDBhAgB0+Hx+WEfNV0V2794NbW1teHt7A2i6e//TTz8hLS0NERERWLx4MYRCoUZie9jzVhaaO3r0aIsbHx05nzmuI3m47e2IfvrpJxgZGWHYsGGaDkUtEokEly9fxrRp05CWloYuXbp06HdUa2pqkJmZiczMTJw6dQoNDQ1IT0/XdFht8jzV2Q/3CzqihoYG7N27FwsWLNB0KI+NXyQ+ZSYmJi0ef1dUVMDExKTd021sbERISAi8vLzg7u4OAOjduze0tbWhpaWFyZMn47ffflMao6LlPXv2xP379yGRSAA0DVmRH5OJiQlu3boFoKlSr62tRc+ePdWKWb6PXr16wc3NDQUFBejVqxcbinj79m0YGRm1GrM8BnWPpXnMitJQ18mTJzF06FD07t0bQMfPZ2XH/CzyVdk2qqSkpCA7OxtxcXGs8dLT02PHPmzYMPTv3x83btx4qrE97nmsybLwJHWPRCLBjz/+iIkTJ7JlHTmfuRfT81geWmt7O6Lz588jKysLzs7OCA0NRV5eHsLCwjQdlkJ9+/ZF37592ZPZCRMm4PLlyxqOSrGcnBy88sorMDIygq6uLtzd3Tv8RDvAk/eHNKG1fkFHVFJSgrKyMjYpXHl5Ofz9/VFZWanp0NTGLxKfsuHDh+PmzZsoLS2FWCxGRkYGnJ2d2zVNIsKyZcswcOBABAUFseXN3/s6fvw4Xn/9dQBNs4pmZGRALBajtLQUN2/exIgRIxTGLhAIYG1tje+//x5A02yE8mNydnZmMxJ+//33sLGxUeukra+vZ08l6uvrcebMGbz++utwdnZGWloaACAtLQ0uLi4snbS0NBARLly4AENDQxgbG8Pe3h6nT59GTU0NampqcPr0adjb28PY2BgGBga4cOECiKjVfT2chroyMjLg4eHxXOSznCbzVVEaqpw8eRKffPIJdu/ejS5durDlVVVVkEqlAMDy1dTU9KnGpuj4VdFkWVCUhjpycnIwcODAFkP9OnI+cy8mTbSfT0JR29sRLV68GCdPnkRWVha2bt0KGxsbxMXFaToshfr06YO+ffvi+vXrAJrmARg0aJCGo1LspZdewsWLF9HQ0AAi6vDxyj1pf+hZU9Qv6IjMzc2Rm5uLrKwsZGVloW/fvkhJSUGfPn00HZr6nvFEOX8L2dnZ5O7uTi4uLrRr1652T+/nn38mMzMz8vT0JG9vb/L29qbs7GwKCwsjT09P8vT0pDlz5lBFRQXbZteuXeTi4kLu7u5sZkJlsZeUlFBAQAC5urrS/PnzSSQSERHRgwcPaP78+eTq6koBAQFUUlKiVswlJSXk5eVFXl5eNHHiRJZWVVUVzZgxg9zc3GjmzJlUXV1NREQymYxWr15NLi4u5OnpSQUFBWxfhw8fJldXV3J1daUjR46w5QUFBeTh4UEuLi60Zs0akslkStNQR11dHVlZWdH9+/fZso6Wz4sWLSI7OzsaMmQIOTg40KFDhzSar8rSUBazq6srjR07lpVp+Yye3333HU2cOJG8vb3J19eXMjMz2yU2RcevLGZNlwVFaSiLmYgoPDycvvjiixbrdpR85v5ennX7+SQUtb0dXV5e3nMxu+nly5fJz8+PPD09ad68eR1+5uPt27fT+PHjycPDg8LCwlid3VG0pW/QEbSlX9ARKGpf5Vqb0b+jExA9wVSUHMdxHMdxHMdx3AuFDzflOI7jOI7jOI7jGH6RyHEcx3Ecx3EcxzH8IpHjOI7jOI7jOI5j+EUix3Ecx3Ecx3Ecx/CLRI7jOI7jOI7jOI7hF4kvCHNzc9TV1bVpm+nTp+Onn35qc1rOzs7473//q3K9xMRE3L17t837f5xjkSsrK4O1tfVjbfs8SElJwY0bN9jnzMxMbNq06Zmlf/bsWfj7+6tcr7CwEN9+++0ziOjp++2337B48WJNh9GuXvTzhGsfT1I3t+b48eMoKCh4avtTxxdffIEJEybA19eX/a1eTVJ2LkZERODgwYNt2l9KSgpCQkLaHEd8fLxabcnZs2dx+vTpNu//cY6lucftrzwPWmsvfXx88ODBg2cWg7r9uvj4eIjF4mcQ0dP3/vvvo6SkRNNhtKunfZ7wi0Su3Rw4cOCxLhI1TSKRdNi0U1NTcfPmTfbZxcUF4eHh7RxV2xUWFuK7777TdBiPZfjw4diyZUubttFkmeG455Wqi0SpVPrU0/z8888RGxuLtLQ0GBgYqL1de8TyPDp37hzOnDmj6TDaTCaTQVN/8U1V+9Bae5meno7OnTu3Z1iPZefOnWhsbNR0GI9l37596N+/v9rr83Yd0NF0ANzTJZPJEBMTgzt37iAmJgazZs1CcHAwnJycADTdZWj+OScnBx9//DFqamrw1ltvITQ0FABw+/ZtrF+/Hn/99RdEIhE8PDwwd+7cR9JTtN7u3btx+/ZthISEoFOnTtiyZQv+8Y9/tBrzDz/8gK1bt6JTp05wd3dv8d3JkyexdetWSKVSGBkZYe3atXj11VcBAEeOHMGBAwcAALq6uti7d2+LbcViMZYuXYq+ffsiPDwclZWVCo/J2dkZEydORF5eHszMzBAdHd1qrMXFxVi5ciWqqqqgo6ODRYsWYezYsQCa7rJ/8MEHyMzMxIMHDxAaGorx48cDAC5evIi4uDh2Fz4kJATjxo1DWVkZAgIC4O/vj7y8PEyZMgWvvfYaPvroI4hEIkilUsydOxceHh5ITk7GpUuXsH79enz00UcIDw9HeXk5srOzsWPHDgBAQkICvv76awBNFzvLly+Hvr4+4uPjcePGDdTW1qK0tBT9+/fH9u3b0aVLFxw/fhzbt2+HlpYWpFIpVqxYodZTJolEgjlz5qC6uhoikQgjRozAmjVrUFdXhx07dkAoFMLHxwdvvvkmli9frjIPpk6dihMnTqChoQEbNmzAP//5TwDATz/9hPj4eEgkEmhpaSEmJganT5/Gn3/+iVWrVgEA7ty5A29vb2RmZqJLly6txmtubo6FCxfi+PHjuHfvHtavX4+cnBycOnUKEokE27dvx6BBg3D27Fls2rQJKSkpCtO3sLCAubk5PvzwQ2RnZ8PBwQHz589HXFwcTp06BQBwcHBAWFgYtLW18dVXXyExMRF6enqQyWT46KOPMGjQIFy/fh3R0dGorq5GY2MjZs6ciYCAAABAfn4+YmNjWX4tXboU9vb2KCgowIYNG1BfX4+uXbti2bJlGDFihMp8TEpKQmJiIgwMDODo6Mjy5e7du1i8eDG7oWNra4uoqCiVvz/396aoHALAwYMHceDAARgaGsLR0RFJSUk4e/Zsi+1PnTqFrKws5OTk4PDhwwgKCkK/fv2wfv16DBs2DJcvX8bChQshFApx4MAB1jENDw+Hra0tgKZ628fHBzk5OaisrERwcDACAwMhk8mwdu1a5OXlQU9PD127dsWXX36JhQsXorS0FEuXLsXQoUOxZcsWpKWl4dNPPwUA9O/fH2vXrkWvXr2QkpKCr7/+Gvr6+iguLsbmzZsRHR2NoUOHoqCgAH/++SdmzJgBExMTHDx4ELdv38aSJUvw1ltvAVBc5wOKz0Vl8vLysGHDBmzZsgWXLl1qUe+npKS0+FxbW4u5c+eipKQEvXv3xubNm2FiYgKgqY344YcfIJVKYWJignXr1qFPnz6PpNfaelVVVfjyyy8hk8mQk5MDDw8PzJ49u9V4KyoqsHTpUlRWVuLll1+Gltb/P5O4c+cOVq1axZ7qzJo1C76+vgCAoqIibNiwAZWVlQCA4OBg+Pn5tdh3RkYG9u/fj48//hh9+/ZVeEzx8fG4evUqhEIh/vrrL3z11Vfo3r37I7FKpVKFdXdERAR0dHRw7do1VFdX480338TKlSuhp6cHoVCIjRs34sqVKxCJRLC2tkZkZCS0tbUxffp0WFhY4OLFi+jeveoMGwQAABS9SURBVDt2797dpvbS3Nwc58+fh76+/mPV+U9Sr+/fvx8ZGRmQSqXo1KkTVq9ejcGDB2PNmjUAgKlTp0JLSwuff/45tLS0lObBsGHDcOHCBdy+fRtvvfUWwsLCWPlYv349u+nt6ekJX19fBAQEIDMzE506dQIA1v/x8vJqNdaIiAjo6enh5s2bKC0thZubG5ycnBAfH4/y8nLMnDkTM2fOBNBUX+zZswdmZmatpj9nzhxERERAW1sbN27cQF1dHdLT0xX2Q69fv47IyEg0NDRAJpPBz88Ps2bNglgsxrZt2/Dzzz9DLBbD3Nwcq1evhr6+PmpraxEdHY1Lly5BIBDgn//8J1auXIm6ujqsX78ev/32G4CmJ8nvv/8+ACjNx2vXriEyMhL19fUwMzODSCRiebNz504cPXoUnTp1gkAgwIEDB9CtWze1ygBD3AvBzMyM7t69S/Pnz6eYmBiSyWRERBQYGEhZWVlsveafAwMDKSgoiBobG0koFJKnpyf77t1336Vz584REZFIJKJp06bR6dOniYjIycmJrly50qb1FKmsrCQrKysqKioiIqKEhAQyMzMjoVBId+7cIWtra7p69SoRER06dIgmTZpERER5eXnk6upKt2/fJiIioVBIDx48oNLSUrKysqLq6moKDAykzz77jKWlKtZVq1apzOdJkybRoUOHiIjo6tWrZGVlRXfv3mW/QXx8PBERFRUVkZWVFd25c4dqamrIx8eHKioqiIiooqKCHBwcqKamhkpLS8nMzIwyMjJYGvfu3SOJRMLyx8HBge7du8d+s+a/Z3JyMs2fP5+IiLKzs8nDw4Nqa2tJJpPRkiVLKDY2loiIduzYQW5ublRTU0MymYyCgoLoq6++IiIiLy8vOn/+PBERSSQSqq2tVXj8eXl55OfnR0REMpmMqqqq2P+XLFlCX3zxxSNxEZFaeSA/rvT0dHr77beJiOj69es0ZswYunHjBhE1/W61tbVUXV1NY8aMIaFQSEREO3fupA0bNij97czMzOjgwYNERPTtt9+SpaUlSzMhIYEWL178yDEqSl++v71797L9JyUl0cyZM0kkEpFIJKIZM2ZQUlISERGNHj2aHbtIJKL6+npqbGwkPz8/unbtGhER1dbWkru7O127do0d36+//sp+l3v37pFIJCJHR0fKyckhIqIzZ86Qo6MjiUQipflYWFhIdnZ2VFlZSUREq1atIisrKyIi+ve//00rVqxgxyEvaxz3MHndrKwcFhYWkr29PasX161bx8raw8LDw+nzzz9nn/Py8sjCwoLVR0REVVVVrD0rKioiBwcH9p2TkxPFxMQQEVFpaSlZWlqSUCik33//nSZMmEBSqZSIWpbp5u3SlStXyM7Ojp2b27ZtowULFhBRUx1maWlJxcXFbNvAwEBasGABSaVSKi8vpxEjRtDWrVuJiOjixYssNmX1nbJzUVH+pKenk7+/P5WXl7PYmtevzT8nJyfT8OHDWZsaHx/PvktLS6Ply5ezfElKSqLQ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\n",
      "text/plain": [
       "<Figure size 1080x864 with 2 Axes>"
      ]
     },
     "metadata": {},
     "output_type": "display_data"
    }
   ],
   "source": [
    "sns.set(color_codes = True)\n",
    "\n",
    "def plot_hist(df, axis_label):\n",
    "    vals = np.array(df.select(\"values\").collect())\n",
    "    \n",
    "    if np.count_nonzero(vals) == 0:\n",
    "        return \"Error: All values are zero\"\n",
    "    \n",
    "    # log normalization for postiive numbers only\n",
    "    log_vals = np.log(vals[vals != 0])\n",
    "\n",
    "    # plot both the distribution and the normalized distribution\n",
    "    fig, ax = plt.subplots(nrows=1, ncols=2, figsize=(15,12))\n",
    "    sns.distplot(vals, kde=True, ax=ax[0], axlabel=axis_label)\n",
    "    sns.distplot(log_vals, kde=True, ax=ax[1], axlabel = \"log transformed \"+ axis_label)\n",
    "    fig.show()\n",
    "\n",
    "plot_hist(data, metric_name)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "#### Box-Whisker\n",
    "Box plots may also have lines extending vertically from the boxes (whiskers) indicating variability outside the upper and lower quartiles, hence the terms box-and-whisker plot and box-and-whisker diagram. __Outliers__ may be plotted as individual points."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 15,
   "metadata": {},
   "outputs": [],
   "source": [
    "# df is the Spark dataframe\n",
    "# filter label is x-axis, ie. the metric label which we want to categorize the data by\n",
    "def plot_box_whisker(df, filter_label):\n",
    "    plt.figure(figsize=(20,15))\n",
    "    df = df.withColumn(\"log_values\", F.log(df.log_values))\n",
    "    ax = sns.boxplot(x=filter_label, y=\"log_values\", hue=filter_label, data=df.toPandas())  # RUN PLOT   \n",
    "\n",
    "    plt.show()\n",
    "    plt.clf()\n",
    "    plt.close()\n",
    "\n",
    "if label != \"\":\n",
    "    plot_box_whisker(data, label)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "### Finding trend in time series, if there any \n",
    "Trend means, if over time values have increasing or decreasing pattern. Before doing anamoly detection, we have to remove trend or will have to use trend tolerable models, to handle the false positives"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 16,
   "metadata": {},
   "outputs": [
    {
     "data": {
      "text/plain": [
       "Text(0,0.5,'value')"
      ]
     },
     "execution_count": 16,
     "metadata": {},
     "output_type": "execute_result"
    },
    {
     "data": {
      "image/png": "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//79ueCCC0hMTOTdd9/1+7Fub529Ma7tzXMuGzZsYPv27Sxfvtz9H3V3jt/5vkftOXz4MOXl5YwdOxa73U5FRQV33XUXR48e9etxVoB5wIgRIygtLaWsrIympiacTid2u93j/RpjmDdvHkOGDGHixInudrvdTl5eHgB5eXkkJCSc1m6MYd++fYSFhWG1WomLi6OoqIja2lpqa2spKioiLi4Oq9VKaGgo+/btwxhz1mWd2Ud7HnroIQoLCykoKOB///d/uemmm/jd737HjTfeyObNm4GTZzmdGju73e4+02nz5s3cdNNNWCwW7HY7TqeTpqYmysrKKC0t5brrrmvzfbBYLG320RFRUVFER0fz2WefAbB7925++tOf+u04A1x++eW8//77fPvttxhj2L17N1deeaXfj3V76+yNcW2rj3MpLCxk9erVPPfcc/Tt2/e0demu8Tvf96g9V199Nbt376agoICCggKio6PZsGEDUVFRfj3OOgvRQ7Zv324SExNNQkKCefbZZ73S5969e81VV11lxowZY+68805z5513mu3bt5uvv/7a/PKXvzQOh8P893//t6murjbGGNPa2moWLFhgEhISzJgxY0xJSYl7WevWrTO33nqrufXWW8369evd7SUlJSY5OdkkJCSYJ554wrS2thpjTJt9dNSePXvcZyEePnzYjBs3ztx6661m+vTpprGx0RhjzPHjx8306dPNrbfeasaNG2cOHz7snv/ZZ581CQkJJjEx0X3GkzFtvw9t9dFRH374oUlLSzNjxowxDzzwgKmpqfH7cV6xYoVJSkoyycnJ5uGHH3afpeZPYz1z5kwTGxtrfvazn5mbb77ZrF271qfj2l4f7dV86623mvj4ePf38NTZgt05fp15j9qr+ftuueUW91mI/jLOZ6PngYmISEDSLkQREQlICjAREQlICjAREQlICjAREQlICjAREQlICjCRbvLMM8+47xS+YsUK3njjDY/3WVdXxx/+8AeP9yPij3QavUg3ufrqq3n33Xe55JJLvNZneXk548aN4+233/ZanyL+ItjXBYj0BE888QQA//Vf/0WfPn0YOHAg//Ef/8E999zDM888w2effUZDQwOlpaUMGzaMKVOmkJWVxZdffonD4eCRRx4BTt5eZ9GiRXz55Zc0NjaSnJzM/fffT2trK5mZmezZs4eQkBAuvvhiXnvtNTIzM6mvr2fs2LH07duX1157jTVr1uB0OmlpaeHCCy9kwYIFXHvttcDJkM3IyGDbtm3U1NSwaNEidu3axc6dO2lubmbFihX89Kc/5e2332bx4sVcc801HDhwgL59+5KVlcWVV17pszEW+YEOXe4sIud01VVXmYaGBmOMMY888oh5+eWXjTHGPP3008bhcJi6ujrT3NxsUlJSzKRJk0xjY6P55ptvzE033WQOHTpkjDFmwoQJpri42Bhz8jljd999tykqKjIHDhwwt912m2lpaTHGGPdzl8rKyswNN9xwWh2n7qBgjDFvvfWWSU9PP63GV155xRhjzBtvvGFiYmJMQUGBMcaYnJwc89BDDxljTt4Z5aqrrjJvv/22McaYDRs2mLS0tO4bLJFuoC0wES+Ii4sjLCwMOLkVdM011xASEkJISAg/+clPOHz4MFarleLiYveTcOHkkwU+/fRT0tLSaG5uZt68edx4443ccsstbfa1f/9+Vq1aRW1tLRaLhdLS0tN+f/vttwO4nx13alnDhw9n69at7ukGDx7MDTfcAMDYsWN57LHHaGho+METDkR8RQEm4gUXXnih++egoKAfvG5paaG1tRWLxcL69eu54IILfrAMp9PJ22+/za5du1i+fPlpj3A/pampiQcffJBXXnmFYcOG4XK53M+iOrOWPn36EBIS4m7v06eP+67nIoFAZyGKdJNLLrmEhoaGTs8fGhrKv/3bv532bLEjR45w9OhRvv76a7799ltuvvlmHn74YcLCwigrKyM0NNT9lGU4GWDNzc3uBx3++c9/7nQ9hw8f5p133gHgb3/7G1dddZW2vsSvaAtMpJtMmjSJX/7yl1x00UUMHDiwU8tYvnw5S5cuJSUlBTgZiosXL+b48eM89thjNDc309LSQnx8PDExMfTp04eUlBRSUlIIDw/ntddeY8aMGYwfP56IiAj3o+g746qrrmLdunUsWLCAiy66iGXLlnV6WSKeoNPoReQH3n77bZ588kk2bNjg61JE2qRdiCIiEpC0BSYiIgFJW2AiIhKQFGAiIhKQFGAiIhKQFGAiIhKQFGAiIhKQ/j/sA2KOCnuuAwAAAABJRU5ErkJggg==\n",
      "text/plain": [
       "<Figure size 432x288 with 1 Axes>"
      ]
     },
     "metadata": {},
     "output_type": "display_data"
    }
   ],
   "source": [
    "#TODO: downsampling\n",
    "data = data.sort(\"timestamp\", ascending=True)\n",
    "vals = np.array(data.select(\"values\").collect())\n",
    "plt.plot(vals)\n",
    "plt.xlabel(\"timestamp\")\n",
    "plt.ylabel(\"value\")"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 17,
   "metadata": {},
   "outputs": [],
   "source": [
    "# be sure to stop the Spark Session to conserve resources\n",
    "sc.stop()"
   ]
  }
 ],
 "metadata": {
  "kernelspec": {
   "display_name": "Python 3",
   "language": "python",
   "name": "python3"
  },
  "language_info": {
   "codemirror_mode": {
    "name": "ipython",
    "version": 3
   },
   "file_extension": ".py",
   "mimetype": "text/x-python",
   "name": "python",
   "nbconvert_exporter": "python",
   "pygments_lexer": "ipython3",
   "version": "3.6.3"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 2
}


#### Environment-aware Detect
Jupyter

