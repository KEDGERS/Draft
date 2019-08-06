## AIOps: Spice-up Containerized Apps Monitoring with Machine Learning

### BACKGROUND

If you are reading this paper, you have probably heard of containers. But if you haven’t, you can think of containers as easily configured, lightweight VMs that start up fast, often in under one second. They are designed to be short-lived and fragile, I know it seems odd to talk about system components that are designed to not be particularly resilient, but there’s a good reason for it - Instead of making each small computing component of a system bullet-proof, you can actually make the whole system a lot more stable by assuming each compute unit is going to fail and designing your overall process to handle it. 

Containers address several important operational problems, that is why they are taking the infrastructure world by storm. **But there is a problem:** containers come and go so frequently, and change so rapidly, that they can be an order of magnitude more difficult to monitor and operate than physical or virtual hosts.

There are patterns and events which disrupt the normal end-to-end behavior of a containerized app, but we still need to figure out what the causes of disruption are to fix whatever is ailing the system. Because of the complexity and high entropy of containers, seeing those patterns and being able to analyze them simply exceeds the capabilities of human operators. Yes, there may be a mathematical curve which describes what’s going on under the hood, but it is so complex that human beings are not able to come up with the equation to make sense of that curve and hence it is very difficult for them to figure out how to deal with it.

In this paper we describe how Artificial Intelligence for IT Operations (AIOps) enables enterprises to work with performance metrics being collected from Containerized environment and see if that curve exists, and then come up with the solution that addresses the curve.

### Massive Operational Complexity for Containerized Applications

If we are talking about containers nowadays, most people tend to think of the big blue whale or the white steering wheel on the blue background.
<p align="center"> <img src="https://miro.medium.com/max/805/1*72WozZ6G_vsox0PFNgWW8g.png"> </p>

Let’s put these thoughts aside and ask ourselves: What are containers in detail? If we look at the corresponding documentation of Kubernetes we only find explanations about [“Why to use containers?“](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/#why-containers) and lots of [references to Docker](https://kubernetes.io/docs/concepts/containers/images/). Docker itself explains containers as [“a standard unit of software“](https://www.docker.com/resources/what-container). Their explanations provide a general overview but do not reveal much of the underlying “magic“.

Eventually, people tend to imagine containers as lightweight virtual machines (VMs) that start up fast, which technically does not come close to the real world. A container is a lightweight **virtual runtime**, its primary purpose is to provide software isolation. A significant architectural shift toward containers is underway, and this architectural shift comes with new operational challenges. The well-understood challenges include orchestration, networking, and configuration — In fact there are many active software projects addressing these issues. However, the significant operational challenge of monitoring containers is much less well-understood. Most of the existing  monitoring solutions cover the traditional stack:

-   Application performance monitoring instruments your custom code to identify and pinpoint bottlenecks or errors

-   Infrastructure monitoring collects metrics about the host, such as CPU load and available memory

When you add containers to your stack, your world gets much, much more complex. In fact, it gets so complex that existing monitoring tools simply can’t explain your system, due to the following challenges:

- Containers come and go so frequently, and change so rapidly, that they can be more difficult to monitor and understand than physical or virtual hosts.

- Within containerized environments you might need  to monitor 150 metrics per Operating System, and for each container let's assume you collect 50 metrics, plus another 50 metrics reported by an off-the-shelf component running in the container. (This is a conservative number, as we see customers collecting many more). In that case we would add 100 new metrics per container, Assuming the host runs 10 containers, the number of metrics we will collect is:
OS + (Containers per host * (Container + Off-the-shelf)) = 100 + (10 * (50 + 50)) = **1100 metrics per host**
With close to 1000 unique series being emitted, it is difficult to know which metrics to pay attention to.

If you are not addressing these challenges, you are left with two choices:

- Treat containers as hosts that come and go every few minutes. In this case your life is miserable because the monitoring system always thinks half of your infrastructure is on fire.

- Don’t track containers at all. You see what happens in the operating system and the app, but everything in the middle is a gap. In this case, you should expect a very painful ride if you are unable to identify performance bottlenecks at the Container layer of the stack. 

Instead, we need a new approach where we re-center monitoring around **proactively detecting anomalies for containerized applications** to determine how **performance bottlenecks** on the containers layer of the stack will ripple to the rest of the stack. 

###  Designing the Operating Model
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

### Designing the Dataset

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

### Designing the Strategy for Anomaly Detection

Anomaly detection for Containers in a certain monitoring domain  faces the following challenges. 

1) Multiple anomaly categories. Under Cloud environment, there are many factors that may cause anomalous performance of Containers. Anomalies of Containers are diversified. Therefore, in order to further detect the types of anomalies, anomaly detection should be considered as a multi-class classification problem. 

2) Imbalanced training sample sets. In general, normal samples can be easily collected. Despite frequent occurrence, anomalies are still small probability events compared with normal states. Therefore, it is not easy to collect abnormal samples. When Cloud platform is newly deployed, or a monitoring domain is newly partitioned, the training sample set only contains normal samples. After the detection framework detects abnormal states and sends to the operator for verification, abnormal samples are gradually accumulated. Therefore, a perfect anomaly detection system should be able to deal with imbalanced training sample set. 

3) Increasing number of training samples. Since Cloud platform is a real production environment, the detection framework collects sample data in real-time. In order to accurately reflect the new trend of performance, the detected and verified samples should be added into the training sample set. The training of anomaly detection model usually requires much time. Therefore, the adopted anomaly detection algorithm should have the ability of online learning, i.e., the detection model can be updated only according to the newly added training samples. At the same time, some selected samples should be deleted to avoid the number of training samples exceeding the capability of training sample set. 

There is no universal detection algorithm which can solve all these challenges. Therefore, to cope with the above challenges, this paper designs strategies of selecting Support Vector Machines (SVM) based anomaly detection algorithm from a set of algorithms for different situations, which are summarized as follows:

1) If there are only normal samples, One Class SVM (OCSVM) is chosen. These situations include newly deployed Cloud platforms or newly partitioned monitoring domains. There are only normal samples without abnormal ones in an initial period of running time. 

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
Spark + Custom code: Data Domain 
#### Data Processing:
Spark + Custom Code: Anomaly Types (Trend, Seasonality, Cyclicity, Irregurality)

#### Environment-aware Detect
Jupyter

