#Questionnaire - Building Artificial Intelligence for IT Operations (AIOps) solutions

Overview:

According to Gartner, IT Operations (IT Ops) are in the midst of a revolution. The forces of digital business transformation are necessitating a change to traditional IT management techniques. Consequently, we are seeing a significant change in current IT Ops procedures and a restructuring in how we manage our IT ecosystems. And Gartner’s term that captures the spirit of these changes is Artificial Intelligence for IT Operations, or AIOps. Diving into AIOps journey without knowing what you’re trying to achieve is a recipe for disaster. In the last couple of years I’ve had the opportunity to work with companies to help them define their AIOps strategy, from problem framing to end-to-end implementation in production. We worked together to enable AIOps by deploying Machine Learning (ML) models to boost efficiency of their IT operations, develop ML-powered features, and build new solutions.

In the process, we approached the deployment of ML from every angle: Business requirements, technological implementation, compliance, pricing/monetization. A set of questions are provided in this document to help guide the process towards successfull AIOps implementation. The questionnaire will allow you to increasingly make data-driven decisions and to rally people behind data rather than arbitrary process decisions. After you go through the questionnaire, you should have a clear path forward to drive your initial AIOps implementation.

The questionnaire addresses the key criterias you should evaluate and there are mainly 4 buckets that you should cover:


---Business requirements:
These questions should be answered by the business side, goal is to identify opportunities where AIOps helps make the business more efficient and cheaper to operate.  

What processes are key to your IT Operations? Could any of them be (further) optimized?
How much time/money would you save from optimizing one workflow over another? 
Can you qualitatively describe the problem you are trying to solve? What you’d like the machine-learned (ML) model to deliver?
How do you plan on using the ML-model output in your solution?
Have you documented the success and failure metrics for the solution as a whole as well as the machine-learned (ML) model? Are these metrics measurable?
How do you plan on training people to maintain them? 
Where in your process will you require humans in the loop?
Can you leverage the expertise of your employees or other stakeholders in the “labeling” of your datasets, which will eventually result in a better product offering altogether?

Picking one IT workflow to optimize, alongside building strong data infrastructure, will allow you to gradually optimize more IT processes and perhaps even lead to the development of a product feature.

---Data Infrastructure and existing Tool-belt: check min 25 on the video

It comes down to documenting your existing ecosystem, the different datasource that you have today and how does the solution you are building accomodate these different things:

How would you solve the problem if you did not use ML? What would be the outcome?
Where does the training data come from?
How can you ensure that all your critical datasources align to the desired outcome that the machine learning algorithms powered solution offer you? 
Can you leverage historical information vs. real time (streaming) data?
How can you leverage the data you get from these tools to train these machine learning models to give you the best outcome for your business?

---Capabilities and Time to Value.
You want to have some realistic expectation about when you start to show an ROI or are there opportunities for quick wins, this is particualarly important if your organization is early in the adoption of AI and ML. We are seeing a lot of healthy skepticism around what ML can actually achieve and if you are sort of champing some of these technologies it is important for you and for the overall adoption of the solution to actually show that this stuff really works, and eliminate some of the skepticism that no doubt is lurking in the back of the minds of a lot of decision makers and executive sponsors.

Can you achieve quick wins or rapid ROI to gain credibility? 
What is your overall time to value for deploying these ML capabilities? and can you implement them incrementally? 
How much of that training data do you need to actually start to see really value out of the ML solution?

---Compliance and Interpretability
In some case, black box is easier, but when dealing with something as critical as mission critical IT Operations data, some of the most caucious and most concervative in IT that I've ever encountered, it's crtitical that they have confidence in the output and the decisions that are coming from ML. In many cases, even if the black box is easier, it's better to take an open box approach so the team will actually trust, have level of transparency, and they can control it the same way that they control other aspect of the IT estate.

Are you dealing with "open box" apporach for machine learning or more of "black box" approach? 

How important for you is to have that control about the machine learning algorithm is doing? And How much do you have the ability to audit the system to remain in compliance in terms of actions you perform, you can learn and improve your processes, as well as be held accountable to your business in terms of the outcome.


---ROT and TCO:
You would think some of these questions are obvious, but we've run into some projects that have gone side ways because they were not considered early enough in the process.

Are you looking at IT operations pain points, and teams that are experiencing these pain points that are really well aligned to the benefits that the ML solution can deliver? 

The ML model is going to find patterns in large amount of data, are you really thinking through those patterns and how to be able to process those at scale are aligned to the teams that are working through those IT processes today? 

In some cases those teams may feel introducing ML is threatning their jobs, what is that perception is going to have on rollout and the effectiveness of the solution, or they are going to have different perception, this will help them do their jobs more effectively, a force multiplier, and enabler and allows them to do less busy work. So making sure the real benefits of ML are well aligned to the painpoints of your teams within the domain that the data can provide for you is really important aspect.

Does achieving the ROI from ML requires your team to make major changes to the processes and tools they are making today? The biggest challenges we face in this space is getting adoption at scale. A lots of teams make huge investments in tools, but they don't see enough adoption for these tools. ML is no different that that, fewer changes on processes and tools that your team has to make to successfully adopt, the greater the chances for overall success. So we highly recommend that you think about the process change, and align expectation for the results based on how drastic the changes are that you are asking your team to make. 







How could you leverage this data to develop a better or more customized offering? 
Could you actually build an entirely new product entirely based on this data? What if you consider the use case/ industry from first principles, can you develop an entirely new product based on a mixture of existing and new data sources?


Will it be able to work on best of breat ecosystem where you cannot live without different types of your monitoring tools, be it synthetic tool, be it network intelligence tool, be it different application performance tools?

How long do you think it will take to incorporate an automatic workflow? 


There are some very natural ways to approach a large ML rollout in phases, so looking at specific dataset, specific applications, specific teams. There are usually some ones that are natural outliers that would dispropostionally benefit from AIOps tools and these could be your best choice for quick win. The more agile you could be in your approach to deploying ML the more likely it's going to be successfully adopted and achieve the outcome you are shooting for.



Knowing if you are fundamentally using supervised or unsupervised algorithm under the hood. This can have really significant impact on the user experience. In case of supervised model, it's very important to know where the training data is coming from, and how much of that training data do you need to actually start to see really value out of the ML solution? It is very common for that to get unspoken and lead to mis-aligned expectation.




