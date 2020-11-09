# Dynamically train and score Form Recognizer model to extract key-value pairs from different type of forms at scale using REST API with Python

In my first blog about the automated form processing, I described how you can extract key-value pairs from your forms in real-time using the Azure Form Recognizer cognitive service. We successfully implemented that solution for many customers. 
In this part 2 of the blog about the Form Recognizer Cognitive service, we will discuss common requirements across many customers we came across and how we address them as the product itself evolves. 
Often, after a successful PoC or MVP, our customers realize that, not only they need this real time solution to process their forms but, they also have a huge backlog of forms they would like to ingest into their relational or NoSQL databases, in a batch fashion. In the backlog, they different types of forms and they don’t want to build a model for each type.
In this blog, we’ll describe how to automatically (re)train existing models or create new ones extract the key-value pairs of off different type of forms and at scale.

# Business Problem

Most organizations are now aware of how valuable the data they have in different format (pdf, images, videos…) in their closets are. They are looking for best practices and most cost-effective ways and tools to digitize those assets.  By extracting the data from those forms and combining it with existing operational systems and data warehouses, they can build powerful AI and ML models to get insights from it to deliver value to their customers and business users.
With the [Form Recognizer Cognitive Service link](https://docs.microsoft.com/en-us/azure/cognitive-services/form-recognizer/overview), we help organizations to harness their data, automate processes (invoice payments, tax processing …), save money and time and get better accuracy.

