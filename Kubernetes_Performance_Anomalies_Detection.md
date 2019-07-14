## INTRODUCTION

Containers are really lightweight. That makes them super flexible and fast. However, they are designed to be short-lived and fragile. I know it seems odd to talk about system components that are designed to not be particularly resilient, but there’s a good reason for it - Instead of making each small computing component of a system bullet-proof, you can actually make the whole system a lot more stable by assuming each compute unit is going to fail and designing your overall process to handle it. 

This is probably the thing people have the hardest time with when they make the jump from VM-backed instances to containers. You just can’t have the same expectation for isolation or resiliency with a container as you do for a full-fledged virtual machine.  In the real world, individual containers fail a lot more than individual virtual machines. To compensate for this, containers have to be run in managed clusters that are heavily scheduled and orchestrated. The environment has to detect a container failure and be prepared to replace it immediately.  

When you have a system that is built out of a few monolithic parts, it’s possible to infer the state of the system as a whole from a few vantage points. But, if you’ve got a high entropy system such as container environment,  every data point contains a lot of information and it becomes harder to identify root causes for bottlenecks and predict the state of the system.

Enterprises use big data platforms to gather data points which is fine up to a point as you absolutely need some place to put all of this information, unfortunately for many enterprises they believe that’s the end of the story – we’ve got all the data and can access the data, job done. Of course all you’ve really done is assemble data into a big haystack, now you need to start looking for the needle. And this is where AIOps comes into play.

The basic premise here is that there are patterns and events which disrupt the normal end-to-end behavior of the system – but we still need to figure out what the cause of disruptions are to fix whatever is ailing the system. Because of the complexity and high entropy of the system, seeing those patterns and being able to analyze those patterns simply exceeds the capabilities of human operators. Yes, there may be a mathematical curve which describes what’s going on under the hood, but it is so complex human beings are not able to come up with the equation to make sense of that curve and hence it is very difficult for them to figure out how to deal with it.

AIOps enables enterprises to work with the data that is being collected in large databases and see that a curve exists, and then come up with the equation that describes the curve. AIOps processes data and then has the capacity to see patterns and provide an analytical solution that human operators can use to solve problems.

## What is a Container
most of you know about but I just want to make sure we are on the same page. so what is a container? There's a high-level approach where container is described as lightweight virtual machine. That is not accurate and it puts you in the wrong mindset - It feels like a VM, you could get shell or SSH into it, you can have your own process and when you do PS stop H stop you only see your own processes, you do ifconfig and you only see your local network interface, you can install packages in it you can run services,  so it really feels like VM but it's also feels more like chroot on steroids because it's not a VM it's just a bunch of normal processes running on a normal kernel and if you are on a machine that has docker or another container runtime installed and you do PS you will see all the processes inside all the containers so it's more transparent than VM. You can't do stuff like having a different kernel for your container or having a different OS because it's only one kernel and then you put little work between the processes each process is living is in nice little world where it can only see its own environment and not the rest of the machine. So how is it implemented? For you to understand how containers works, I used my curiosity and I started to look (using grep) in the Linux kernel source code for Lxe, I could not find any single reference to it in the whole Linux kernel. Then I looked for "containers", it returned tons of things but those containers are not the containers that I was looking for (those containers are like ACPI containers). So at some point I'm like okay do container really exist? And so after digging a little bit more I realized that I was looking in the wrong place, containers are not in the kernel, what is in the kernel however is those cgroups and namespaces


## KUBERNETE


# KUBERNETES PERFORMANCE - WHAT METRICS ARE IMPORTANT

From the hundred of metrics that Kuberntes exposes, it feels overwhelming to find the metrics to pay attention to. In most cloud providers you will first be concerned with the “core” resource metrics; CPU, memory, disk and network. In a Kubernetes cluster, metrics related to CPU, memory and disk are exposed by the kubetlet (via cAdvisor), while core container metrics are scoped to each container. With close to 1000 unique series being emitted from the node_exporter, it is sometimes difficult to know which metrics to pay attention to. What are the important metrics your system?

## Methodology

As with any complex topic where there is too much information, we tend to simplify the problem into relatively easy to understand abstractions. Let’s look at a few methods of abstraction around metrics. This will allow us to better frame not only the node metrics, but all of the Kubernetes metrics going forward. These abstractions are the **Four Golden Signals**, the **USE Method** and the **RED Method**.

### The Four Golden Signals
Google gave us a place to start in the excellent SRE Handbook with the notion of the “Four Golden Signals”. The four golden signals of monitoring are latency, traffic, errors, and saturation. If you can only measure four metrics of your user-facing system, focus on these four.
Latency — The time it takes to service a request
Traffic — A measure of how much demand is being placed on your system
Errors — The rate of requests that fail
Saturation — How “full” your service is.

### The USE Method
Brendan Gregg does an excellent job explaining how to think about Utilization, Saturation and Errors when looking at the resources in your system. He gives the following definitions:
Resource: all physical server functional components (CPUs, disks, busses, …)
Utilization: the average time that the resource was busy servicing work
Saturation: the degree to which the resource has extra work which it can’t service, often queued
Errors: the count of error events
He suggests, but does not prescribe, exactly which metrics that represent utilization, saturation and errors in the context of a Unix system. The rest of this paper we will apply the USE method to resources in your Kubernetes nodes.

While the USE method is targeted at resources, actual physical things with hard limits, it is an incomplete picture when it comes to the software running on those resources. That’s where the RED method comes in.

### The RED Method
Tom Wilkie coined the RED method as:
Rate: The number of requests per second.
Errors: The number of those requests that are failing.
Duration: The amount of time those requests take.

On the surface the RED method seems very similar to the USE method and the Four Golden Signals. When do you apply USE vs RED? The USE method is for resources and the RED method is for my services. Now we have frame of reference on how to apply these simplifying abstractions to the the metrics in Kubernetes systems.

## Kubernetes Node Metrics
Now we have a couple of methods to help select which metrics to pay attention to. Is this thing a resource or an application? The nodes in your cluster have resources. The most important resources your nodes provide in a Kubernetes cluster are CPU, Memory, Network and Disk. Let’s apply the USE method to all of these.

### CPU Utilization, Saturation, and Errors

### Memory Utilization, Saturation and Errors

### Disk Utilization, Saturation, and Errors

### Network Utilization, Saturation and Errors

The node_exporter project exposes a wealth of information about the resources on the nodes in your Kubernetes cluster. Viewing these resources through the lens of utilization, saturation and errors gives you starting point for investigation of resource constraints and capacity planning.

# ANOMALIES DETECTION PRINCIPLES
The method for anomaly detection presented in this article is based on the fundamental principle of organizing all the VMs in the system into multiple domains with several nodes in each domain based on the component each node is running. Since the same components across the system should behave similarly, doing the same tasks and running the same software, you want to group them. E.g. components that are responsible for routing HTTP requests might have higher CPU usage and lower input/output operations per second (iops), whereas components running databases would have higher read operations per second and high memory usage. Some components are only routing the requests and some of them are making calculations during data processing, so it is a wise idea to combine the same components into a domain. VMs are segregated into multiple domains based on the components they are running. This task does not require any significant amount of calculation since every VM is already organized according to the component that it is running, which requires only O(N) time complexity algorithm. A set of performance metrics for each VM is also collected at this time. In the next step each domain is being examined in order to find any outliers  present in the domain.

For each VM, a set of metrics are collected. This list may be extended by any other significant metrics. This set represents a X ∈ Rn where n is the number of metrics. The metrics set can be formalized as: 

<<Include equition>>

Where xi is a real number representing the particular metric value for a particular VM. A domain performance metrics set is a set of sets where each column is a vector of a particular VM's metrics values. It can be formalized by the following matrix Y: 

<<Include equition>>
  
Where n is a number of metrics per VM and m is a number of VMs in a domain. 

Let T be a training sample set representing the samples of all the VMs in a monitoring domain for a certain time period, 
<<Include equition>>
  
Where xi∈Rn is the input vector (or instance), yi is the output (or the label of xi), (xi, yi) is called a sample point, l is the number of samples. 

1) Binary classification: the task is to determine whether the state of a VM represented by a sample is normal or abnormal, then 
<<Include equition>>

When yi = +1, xi is called a positive sample; while when yi = -1, xi is called a negative sample. The goal is to find a real function g(x) in Rn, 
y = f(x) = sgn(g(x)), 
Such that f(x) derives the value of y for any sample x, where sgn() is the sign function. 

2) Multiclass classification: the task is to not only determine whether the state of a VM is normal or abnormal, but also determine the type of anomaly, then 
yi∈ = { 1, 1, ..., c }, i = 1, 2, …, l, 

c is the number of states including the normal state. The goal is to find a decision function f(x) in Rn, y = f(x) : Rn→, 

In order to implement environment-aware detection and improve the detection accuracy, we followed the following detection approach
1) Collect all the VM's running environment attributes and performance metrics at the same time. 
2) Partition all the VMs in Cloud platform into several monitoring domains based on similarity in running environment by using clustering algorithms, which makes VMs in a same monitoring domain have similar running environment. 
3) In each domain, the equipped anomaly detection algorithm detects anomalous VMs based on their performance metrics.

# CHANLLENGES AND EQUIPPED ALGORITHMS

# SOLUTION PROTOTYPING AND ANALYSIS

# SUMMARY
