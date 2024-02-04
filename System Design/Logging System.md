# Introduction

- Logging is essential in **understanding the flow of an event** in a distributed system.
- It helps in finding out the root cause of the failure.
- A file that records details of events that occurred during the usage of a software application is called a log.
- Logs are crucial for debugging and monitoring your application’s activity.
- Log analysis helps us troubleshoot applications and network issues and recognize security problems.

# Logging in a distributed system

In today’s world, an increasing number of designs are **moving to microservice architecture instead of monolithic architecture**. In microservice architecture, logs of each microservice are accumulated in the respective machine. If we want to know about a certain event that was processed by several microservices, it is difficult to go into every node, figure out the flow, and view error messages. 


# Functional Requirements

- All system services must be able to be written into the logging system
- Able to query the logs


# Non-functional Requirements

- **Low latency**:
	Logging is an I/O-intensive operation that is often much slower than CPU operations. We need to design the system in such a way that logging does not impact application performance.
- **Scalability:**
	We want our logging system to be scalable. It should be able to handle the increasing amounts of logs over time and a growing number of concurrent users.
- **High Availability:**
	The logging system should be highly available to log the data. If the system is not highly available, there will be consistency issues in logs and also some of the data will be lost forever.



# Restrain the log size

The number of logs increases over time. At a time, perhaps hundreds of concurrent messages need to be logged. But the question is, are they all important enough to be logged? To solve this, logs have to be structured. We need to decide what to log into the system on the application or logging level.

## Use sampling

We’ll determine which messages we should log into the system in this approach. Consider a situation where we have lots of messages from the same set of events.Instead of logging all the information, we can use a **sampler service** that only logs a smaller set of messages from a larger chunk. This way, we can decide on the most important messages to be logged.

## Use categorization

We can also categorize the types of messages and apply a filter that identifies the important messages and only logs them to the system.

The following severity levels are commonly used in logging:

- `DEBUG`
- `INFO`
- `WARNING`
- `ERROR`
- `FATAL/CRITICAL`

# High Level Design

![](https://miro.medium.com/v2/resize:fit:2000/1*tCNCtTWb0IYD7DaNKUCJEw.png)

Fig 1.0: High Levl Design of distributed logging system

Let’s list the major components of our system:

- **Log accumulator**: Each server has its log accumulator, and does three actions receives logs, storing them locally, and push the logs to pub/sub system.Let’s consider a situation where we have multiple different applications on a server, such as App 1, App 2, and so on. Each application has various microservices running as well. We use an ID with `application-id`, `service-id`, and its time stamp to uniquely identify various services of multiple applications.
- **Pub/Sub**:All servers in a data center push the logs to a pub-sub system. Since we use a horizontally-scalable pub-sub system, it is possible to manage huge amounts of logs. We may use multiple instances of the pub-sub per data center. It makes our system scalable, and we can avoid bottlenecks. Then, the pub-sub system pushes the data to the blob storage
- **Filterer:** It identifies the application and stores the logs in the blob storage reserved for that application since **we do not want to mix logs** of two different applications
- **Error aggregator**: It is critical to identify an error as quickly as possible. We use a service that picks up the error messages from the pub-sub system and informs the respective client. It saves us the trouble of searching the logs.
- **Alert aggregator**: Alerts are also crucial. So, it is important to be aware of them early. This service identifies the alerts and notifies the appropriate stakeholders if a fatal error is encountered, or sends a message to a monitoring tool.
- **Blob storage**: The logs need to be stored somewhere after accumulation. We’ll choose blob storage to save our logs.
- **Log indexer**: The growing number of log files affects the searching ability. The log indexer will use the distributed search to search efficiently.
- **Visualizer**: The visualizer is used to provide a unified view of all the logs.
- **Expiration checker:** Verifying the logs that have to be deleted. Verifying the logs to store in cold storage.




## High Level Design:

In a distributed system, clients across the globe generate events by requesting services from different serving nodes. The nodes generate logs while handling each of the requests. These logs are accumulated on the respective nodes.

In addition to the building blocks, let’s list the major components of our system:

![[Pasted image 20240129002754.png]]

Figure: High Level Design of Logging service (Image by Author, Idea from System Design Interview Prep Crash Course from educative)

**Log accumulator:** This is the component that collects logs from each node and sends them into storage. So, all the logs are in one place now. If we want to know about a specific event, we don’t need to visit each server as we can get them from our storage.

**Storage:** The logs need to be stored somewhere after collection. We may use blob storage to save our logs.

**Log indexer:** The growing number of log files will surely affect the searching quality of the logs. It will be problematic while log analysis. The indexer will use the distributed search to help analysis effortlessly.

**Centralized Visualizer:** The component is needed to provide a total view of all the logs.

More than one server exists in a distributed system. So, a single-log accumulator is not going to work for any system. We need to use tactics for scalability.

The initial design is mostly based on System Design Interview Prep Crash Course from educative.io.

## Scalability:

Let’s think about an e-commerce application which has various microservices running like authentication of users, cart management, bill payment, etc. So, we need an ID for the application, another for identifying the microservice and a timestamp to identify a log from which service and what event might have occurred.

Each microservice will push data to Log accumulator service. So, the log accumulator will receive the log and store it. But we need these logs to be in a centralized storage. Now, to remove the scalability issue, we can use the pub-sub system. Pub-sub system will manage various types of logs.

![[Pasted image 20240129002824.png]]

Figure: Pub-Sub to serialize and save data for further use (Image by [Author](https://medium.com/@ashchk))

For more detailed scalability solution you check [this course](https://www.educative.io/courses/grokking-modern-system-design-interview-for-engineers-managers/design-of-a-distributed-logging-service).

## Performance:

Another important factor we should always keep in mind is that logging is a continuous service; it should not affect the performance of the application. We should send logs asynchronously through a low-priority thread so that it does not interfere with other services.

Now here is a twist. For applications which support financial or bank services, the logs must be so secure that hackers cannot steal the data. So, we need to encrypt the logs such a way that no unauthorized user can decrypt the logs.

This encryption decryption may hamper logging service latency. We need to consider this trade-off for application requirements.

## Availability:

We should be careful about the data loss scenario in case of logging huge amounts of messages. For better performance, log services may keep data in RAM and persist them asynchronously, which may cause data loss. So, here is a tradeoff scenario between latency/performance and data availability. We may consider adding redundant log accumulators in case of a growing number of users.

## Security:

Log data can be used for training machine learning models to find patterns of user interactions. This huge amount of data can be used for security analytics like DDOS attack detection, fraud detection etc.

![[Pasted image 20240129002834.png]]

Figure: DDOS attack, Attacker controls other computers to send lots of requests to server (Image by author)

## What’s Next After Pub-Sub?

We need to consider that pub-sub is not a storage where we may keep data forever, there is a time interval after which the data will be removed from pub-sub. So, we need to use a blob storage to store the logs. For analysis purposes, we need to filter the logs using tags. We may have a filter that will store the logs tag-wise in a Blob storage.

An important part of logging is to identify if an error has occurred in the system, and if an error has occurred, we should be able to find it as early as possible. For this reason, developers use error messages while printing logs. We need to have a component which keeps all the error messages in a different component. It will save us time to search for error messages occurring in the system from huge amounts of logs.

Now, in case of serious failure scenarios, we may not have time to analyze the logs. So, we should have an Error Notifier which sends alerts to designated stakeholders if a fatal error has occurred.

Those who are more experienced in system design, you may check [this course](https://www.educative.io/courses/grokking-modern-system-design-interview-for-engineers-managers/design-of-a-distributed-logging-service) from educative for more detailed explanation.

## Log Analytics:

Let’s assume a scenario of a system like youtube. As a user watches a video, Log Accumulators will save user history logs in database. Now, Data analytics server has to analyze the logs and generate some potential candidates for the user using machine learning models. So, the log data of user history is being used for prediction of user preference. Now, when user again goes to the home page, the server goes to Recommendation server, which eventually goes to the database and get the list of video links for the user.

![[Pasted image 20240129002846.png]]

Figure: Log analytics of recommendation system (Image by Author)

# Conclusion:

Logging is important to analyze event flows in a distributed system. It helps engineers decrease the time to repair the system by helping find out the root cause of a problem. As logs are I/O intensive operations, they are slow. So, we should be careful to handle them so that logging does not overlap with the application’s main thread. Logging is also needed to monitor server health. By recording events and data, you can gain insights into performance and identify issues faster.



