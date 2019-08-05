# Questionnaire - How to get started with building Artificial Intelligence for IT Operations (AIOps) based solutions

## Preface:
    
   According to Gartner, IT Operations (IT Ops) are in the midst of a revolution. The forces of digital business transformation are necessitating a change to traditional IT management techniques. Consequently, we are seeing a significant change in current IT Ops procedures and a restructuring in how we manage our IT ecosystems. And Gartner’s term that captures the spirit of these changes is Artificial Intelligence for IT Operations, or *AIOps*. Diving into AIOps journey without knowing what you’re trying to achieve is a recipe for disaster. In the last couple of years I’ve had the opportunity to work with companies to help them define their AIOps strategy, from problem framing to end-to-end implementation in production. We worked together to enable *AIOps* by deploying *Machine Learning (ML) models* to boost efficiency of their IT operations, develop ML-powered features, and build new solutions.

In the process, we approached the deployment of ML from every angle: Business requirements, technological implementation, compliance, pricing/monetization. A set of questions are provided in this document to help guide the process towards successfull AIOps implementation. The questionnaire will allow you to increasingly make data-driven decisions and to rally people behind data rather than arbitrary process decisions. After you go through the questionnaire, you should have a clear path forward to drive your AIOps implementation.

## AIOps Planning Questionnaire:

The questionnaire addresses the key criterias you should evaluate and there are mainly 4 buckets that you should cover:

### General requirements:

   These questions should be answered by the business side. Goal is to identify opportunities where AIOps helps make the business more efficient and cheaper to operate.  

* What processes are key to your IT Operations? Could any of them be (further) optimized?
* How much time/money would you save from optimizing one process (workflow) over another?

Picking one IT process to optimize, alongside building strong data infrastructure, will allow you to gradually optimize more IT processes and perhaps even lead to the development of a product feature.

 * Can you qualitatively describe the problem you are trying to solve? What you’d like the machine-learned (ML) model to deliver?
 * Have you documented the success and failure metrics for the solution as a whole as well as the machine-learned (ML) model? Are these metrics measurable?
 * How do you plan on using the ML-model output in your solution?
 * Can you leverage the expertise of your employees or other stakeholders in the “labeling” of your datasets, which will eventually result in a better product offering altogether?
 * Where in your process will you require humans in the loop?
 * How do you plan on training people to maintain them? 

### Data Infrastructure and existing Tool-belt: 

   It comes down to documenting the existing ecosystem, the different datasources and how does the solution accomodate these different things. These questions should be answered by Technical Architects.
   
  * How would you solve the problem if you did not use ML? What would be the outcome?
  * Where does the training data come from? 
  * How can you ensure that all your critical datasources align to the desired outcome that the machine learning algorithms powered solution offers you? How could you leverage this data to develop a better or more customized offering? 
  * Do you plan on leveraging historical information or real time (streaming) data?

### Capabilities and Time to Value:

   It is critical to have some realistic expectations about when to start to show a Return On Investment (ROI), or identify opportunities for quick wins. This is particualarly important if the organization is early in the adoption of ML. We are seeing a lot of healthy skepticism around what ML can actually achieve, and if you are champing these implementations it is important to actually show that the technology really works, and eliminate some of the skepticism that is lurking in the back of the minds of a lot of decision makers and executive sponsors. These questions should be answered by Technical Architects.

  * Can you achieve quick wins (rapid ROI) to gain credibility? 
  * What is your overall time to value for deploying these ML capabilities? and can you implement them incrementally? 
  * How much training data do you need to actually start to see really value out of the ML solution?

### Compliance and Interpretability:

   When dealing with mission critical IT Operations data and some of the most concervative IT leaders I've ever encountered, it's critical to give confidence in the output and the decisions that are coming from ML. In many cases even if the black box approach is easier, it's better to take an open box approach so the team will actually trust, have level of transparency, and can have control on the solution the same way that they control other aspects of the IT estate. These questions should be answered by Technical Architects.

  * Are you dealing with "open box" approach for machine learning or more of "black box" approach? 
  * How important for you is to have that control about what machine learning algorithm is doing?
  * Do you have the ability to audit the system to remain in compliance?


### Total Cost of Ownership (TCO):

   You would think some of these questions are obvious, but we've run into some projects that have gone side ways because they were not considered early enough in the process. In some cases those teams may feel introducing ML is threatning their jobs, what is that perception is going to have on rollout and the effectiveness of the solution, or they are going to have different perception, this will help them do their jobs more effectively, a force multiplier, and enabler and allows them to do less busy work. So making sure the real benefits of ML are well aligned to the painpoints of your teams within the domain that the data can provide for you is really important aspect. These questions should be answered by Technical Architects.
   
  * Are the teams that are experiencing these IT operations pain points are well aligned with the benefits that the ML-based solution can deliver? 
  * The ML model is going to find patterns in large amount of data, are you really thinking through those patterns and how to be able to process those at scale? Are these aligned to the teams that are working through those IT processes today? 
  * Does achieving the ROI from ML requires your team to make major changes to the processes and tools they have today (The fewer the changes, the greater the chances for overall success)?
  
## Footnote

  There are many natural ways to approach a large AIOps projects in phases. The more agile you could be in your approach, the more likely it's going to be successfully adopted and achieve the outcome you are shooting for. Thing big, start small, learn and iterate...
