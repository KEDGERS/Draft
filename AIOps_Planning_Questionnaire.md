## Getting started with AIOps-powered solutions (Questions Sheet)

### Preface:
    
   According to Gartner, IT Operations (IT Ops) are in the midst of a revolution and the forces of digital business transformation are necessitating a change to traditional IT management techniques. Consequently, we are seeing a significant change in current IT Ops procedures and a restructuring in how the IT ecosystems are managed. Gartner’s term that captures the spirit of these changes is **Artificial Intelligence for IT Operations**, or **AIOps**. 
   
   Diving into an AIOps journey without knowing what you’re trying to achieve is a recipe for disaster. In the last couple of years I’ve had the opportunity to work with companies to help them define their AIOps strategy, from problem framing to end-to-end implementation in production. We worked together to enable AIOps by deploying Machine Learning (ML) models to boost efficiency of their IT operations, develop ML-powered features, and build new solutions.

   In the process, we approached the deployment of AIOps from every angle: Business requirements, Architecture and Data Infrastructure, Program Management and Return On Investment (ROI). A set of questions are provided in this paper to help guide the process towards successful AIOps implementation, allowing you to increasingly make data-driven decisions and rally people behind data rather than arbitrary process decisions. 

After you go through the questionnaire, you should have a clearer path forward to drive your AIOps implementation.

### AIOps Planning Questionnaire:

The questionnaire addresses the key criteria's you should evaluate, and there are 5 buckets that you should cover:

#### Business Requirements:

   *These questions should be answered by the business side, goal is to identify opportunities where AIOps helps make the business more efficient and cheaper to operate.*  

* What processes are key to your IT Operations? Could any of them be (further) optimized?
* How much time/money would you save from optimizing one process (workflow) over another?
* Succeeding in Artificial Intelligence (AI) requires a commitment to DevOps and agile experimentation. How patient the organization is about the ongoing AIOps initiative?

*Picking one IT process to optimize, alongside building strong data infrastructure, will allow you to gradually optimize more IT processes and perhaps even lead to the development of a product feature. For the select IT Processes:*

 * Can you qualitatively (plain English) describe the problem you are trying to solve? What you’d like the machine-learned (ML) model to deliver?
 * Have you documented the success and failure metrics for the solution as a whole as well as the machine-learned (ML) model? Are these metrics measurable?
 * How do you plan on using the ML-model output in your solution?
 * Where in your process will you require humans in the loop?
 
#### Architecture and Data Infrastructure: 

   *It comes down to documenting the existing ecosystem, the different data sources and how does the solution accommodate these different things. These questions should be answered by Technical Architects.*
   
  * Have you invested in monitoring (measuring) everything (full-stack), from the network, machine, application and people’s level?
  * Where does the monitoring data come from? Do you have access to it? If not, how do you request access to the collected the data? 
  * Is the collected data well-curated to train the AIOps system?
  * How would you solve the problem if you did not use AIOps?
  * Can you leverage the expertise of your employees or other stakeholders in the “labeling” of your monitoring datasets, which will eventually result in a better solution altogether?
  * Do you have an understanding of the behavior of the system in normal condition (Steady State)?
  * AIOps is going to find patterns in large amount of data, are you really thinking through those patterns and how to process those at scale?
  * Do you plan on leveraging historical information or real time (Streaming) data?
  * Do you have well-defined objectives for:
        
        - Time to detect?
        - Time for notification? And escalation?
        - Time to public notification?
        - Time for graceful degradation to kick-in?
        - Time for self-healing?
        - Time to recovery — partial and full?
        - Time to all-clear and stable?
        
  * Are you documenting postmortems for IT incidents and outages?
  * Do you have well-defined guidelines for resiliency (chaos) engineering?

#### Expectations and Time to Value:

   *It is critical to have some realistic expectations about when to start to show a Return On Investment (ROI), or identify opportunities for quick wins. This is particularly important if the organization is early in the adoption of ML. We are seeing a lot of healthy skepticism around what ML can actually achieve, and if you are champing these implementations it is important to actually show that the technology really works, and eliminate some of the skepticism that is lurking in the back of the minds of a lot of decision makers and executive sponsors. These questions should be answered by Technical Architects.*

  * Can you achieve quick wins (rapid ROI) to gain credibility? 
  * What is your overall time to value for deploying the AIOps features? and can you implement them incrementally? 
  * How much training data do you need to actually start to see really value out of the AIOps solution?

#### Compliance and Interpretability:

   *When dealing with mission critical IT Operations data, and some of the most conservative decision-makers I've ever encountered - It is crucial for the success of the project to give confidence in the output that are coming from the AIOps system. In an AIOps project, the implementation process can be impacted by the Machine Learning (ML) models. The main difference in ML models is that they either prioritize explaining the world to people (Open Box approach) or predicting outcomes (Black Box approach). In many cases, even if the black box approach is easier to implement, it's better to take an open box approach so you earn trust, have level of transparency, and can have control on the solution the same way that the organization controls other aspects of the IT estate. These questions should be answered by Technical Architects.*

  * How important for you is to have that control about what machine learning algorithm is doing?
  * Do you have requirements to audit the AIOps system to remain in compliance?


#### Total Cost of Ownership (TCO) :

   *You would think some of these questions are obvious, but we've run into some projects that have gone side ways because they were not considered early enough in the process. In some cases those teams may feel introducing AIOps is threatening their jobs - What is that perception is going to have on the rollout and the effectiveness of the solution? So making sure the real benefits are well aligned to the pain points. These questions should be answered by the business side.*
   
  * Are the teams that are experiencing these IT operations pain points well aligned with the benefits that the AIOps solution can deliver?   
  * Does achieving the ROI from the AIOps solution requires your team to make major changes to the processes and tools they have today (The fewer the changes, the greater the chances for overall success)?
  
### Footnote

   There are many natural ways to approach a large AIOps project in phases. The more agile you could be in your approach, the more likely it's going to be successfully adopted and achieve the outcome you are shooting for. **Thing big, start small (and simple), learn and iterate...**

