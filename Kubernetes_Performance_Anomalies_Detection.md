## INTRODUCTION

Containers are really lightweight. That makes them super flexible and fast. However, they are designed to be short-lived and fragile. I know it seems odd to talk about system components that are designed to not be particularly resilient, but there’s a good reason for it - Instead of making each small computing component of a system bullet-proof, you can actually make the whole system a lot more stable by assuming each compute unit is going to fail and designing your overall process to handle it. 

This is probably the thing people have the hardest time with when they make the jump from VM-backed instances to containers. You just can’t have the same expectation for isolation or resiliency with a container as you do for a full-fledged virtual machine.  In the real world, individual containers fail a lot more than individual virtual machines. To compensate for this, containers have to be run in managed clusters that are heavily scheduled and orchestrated. The environment has to detect a container failure and be prepared to replace it immediately.  

When you have a system that is built out of a few monolithic parts, it’s possible to infer the state of the system as a whole from a few vantage points. But, if you’ve got a high entropy system such as container environment,  every data point contains a lot of information and it becomes harder to identify root causes for bottlenecks and predict the state of the system.

Enterprises use big data platforms to gather data points which is fine up to a point as you absolutely need some place to put all of this information, unfortunately for many enterprises they believe that’s the end of the story – we’ve got all the data and can access the data, job done. Of course all you’ve really done is assemble data into a big haystack, now you need to start looking for the needle. And this is where AIOps comes into play.

The basic premise here is that there are patterns and events which disrupt the normal end-to-end behavior of the system – but we still need to figure out what the cause of disruptions are to fix whatever is ailing the system. Because of the complexity and high entropy of the system, seeing those patterns and being able to analyze those patterns simply exceeds the capabilities of human operators. Yes, there may be a mathematical curve which describes what’s going on under the hood, but it is so complex human beings are not able to come up with the equation to make sense of that curve and hence it is very difficult for them to figure out how to deal with it.

AIOps enables enterprises to work with the data that is being collected in large databases and see that a curve exists, and then come up with the equation that describes the curve. AIOps processes data and then has the capacity to see patterns and provide an analytical solution that human operators can use to solve problems.

## What is a Container
most of you know about but I just want to make sure we are on the same page. so what is a container? There's a high-level approach where container is described as lightweight virtual machine. That is not accurate and it puts you in the wrong mindset - It feels like a VM, you could get shell or SSH into it, you can have your own process and when you do PS stop H stop you only see your own processes, you do ifconfig and you only see your local network interface, you can install packages in it you can run services,  so it really feels like VM but it's also feels more like chroot on steroids because it's not a VM it's just a bunch of normal processes running on a normal kernel and if you are on a machine that has docker or another container runtime installed and you do PS you will see all the processes inside all the containers so it's more transparent than VM. You can't do stuff like having a different kernel for your container or having a different OS because it's only one kernel and then you put little work between the processes each process is living is in nice little world where it can only see its own environment and not the rest of the machine. So how is it implemented? For you to understand how containers works, I used my curiosity and I started to look (using grep) in the Linux kernel source code for Lxe, I could not find any single reference to it in the whole Linux kernel. Then I looked for "containers", it returned tons of things but those containers are not the containers that I was looking for (those containers are like ACPI containers). So at some point I'm like okay do container really exist? And so after digging a little bit more I realized that I was looking in the wrong place, containers are not in the kernel, what is in the kernel however is those cgroups and namespaces.

## Methodology: How to identify bottlenecks
Goal is to identify bottlenecks in container environment, whether it's in the host or container using system metrics or in the  application code. I'm going to focus on how this works on linux using 4.9 a recent version of linux I'll include some docker specifics.

there's four things to understand about performance analysis first 
there's one kernel that can simplify some things so one kernel - where I can compare all the metrics between the containers 
there's two perspectives whether I'm looking at it from the host or whether I'm looking  at it from a container guest 
and there's namespaces and cgroups which are summarized earlier.

Instead of assuming the vendor is going to give you the best metrics, we should think of the questions we want asked, trying to figure it out what do you really want answered from the systems. Some anti-methodologies to start with so you can understand methodologies:  

**Streetlight anti-method:** This comes from a parable about a drunk man who's looking for his keys under a streetlight and a police officer found him and asked "what are you doing?" and he said "I've lost my keys, I'm looking for them". The police officer asked "Did you lose them under the streetlight?" and the drunk says "No but that's where the light is best". I see this quite often in performance analysis, where people tune things at random until the problem goes away. You might end up going around in circles and you miss things because there are blind spots.

**Blame someone-else:** That is where something that you are not responsible for, and you hypothesize that the problem must be a component owned by different team. I've seen this many times where people managing the network are blamed, either the network must be retransmits or there's something wrong with BGP.

**Traffic light:** Traffic lights are really easy to interpret, red is bad and green is good. Some people like to create these dashboards where they put colors on everything, colors are good for objective metrics such as errors. But performance analysis often times rely on subjective metrics like IOPS and latency, which might be good for one person who's running a chat server online and may be different for someone who's running a high frequency trading application.
  
Now that we have an understanding of anti-methodologies on the performance analysis space, let's talk about methodologies. 

**Differential diagnosis** 

For systems engineers these are ways to analyze an unfamiliar system or application, identify bottleneck and you can have your dashboard support the methodologies. I came up with a way to figure this out where I enumerated the possible outcomes and then I worked backwards to what metrics do I need to diagnose one of these outcomes. For example, CPU bottleneck identification might have the following outcomes: 

 - physical CPU is throttled 
 - cap throttled 
 - share throttled (Assuming physical CPU limited as well)
 - not throttled
   
I came up with the following diagram which <<>> 



if throttle time is increasing, then the cap is throttled - let's take that off the operating table straight away. It's nice and simple in that first example that's the metric that tells us if the CPU cap has been hit that was being hit we know we're caps throttled -  we're done. if not, if non-voluntary context switches are not increasing for that process or container, it means we're not getting kicked off CPU and if we're not getting kicked off CPU we're probably not getting throttled so that would tend to put us into the Not throttled outcome unless you have some other theory and you need to dig into the kernel and debug further. but I'd be expecting that to be not throttled if you are getting kicked off CPU so it got non-voluntary context switches. Then do you have idle CPU, if you have idle CPU but you're getting kicked off CPU something strange is happening like interrupts so that needs further digging. If you don't have idle CPU you're getting kicked off CPU, you're not getting throttle time that's when it's other channels idle other tenants idle or not and if other tenants are not idle then you're going to have share contention and if they are you're the only tenant on this system then your physical CPU throttled fuel, that's pretty complicated but can be figured out

if you want to do this exercise for discs, networking and memory start with the final outcomes and then try working backwards to come up with a differential diagnosis so you can then go from the metrics you have to one of those possible outcomes. it should all be in a GUI it should just be a wizard that tells me in fact they should just tell me the outcome you are cap threatened so that was host to looking at the container.

# [](https://github.com/c-Kedge/Draft/blob/master/Kubernetes_Performance_Anomalies_Detection.md#anomalies-detection-principles)ANOMALIES DETECTION PRINCIPLES

The method for anomaly detection presented in this article is based on the fundamental principle of organizing all the VMs in the system into multiple domains with several nodes in each domain based on the component each node is running. Since the same components across the system should behave similarly, doing the same tasks and running the same software, you want to group them. E.g. components that are responsible for routing HTTP requests might have higher CPU usage and lower input/output operations per second (iops), whereas components running databases would have higher read operations per second and high memory usage. Some components are only routing the requests and some of them are making calculations during data processing, so it is a wise idea to combine the same components into a domain. VMs are segregated into multiple domains based on the components they are running. This task does not require any significant amount of calculation since every VM is already organized according to the component that it is running, which requires only O(N) time complexity algorithm. A set of performance metrics for each VM is also collected at this time. In the next step each domain is being examined in order to find any outliers present in the domain.





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

