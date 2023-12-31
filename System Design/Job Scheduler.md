## **Introduction**

Job scheduling is a well known system design interview question. Below are some areas where one might need to design a job scheduler.

- Design a system for payment processing. (i.e. monthly/weekly/daily payout etc.)
- Design a code deployment system. (i.e. code pipeline)

The purpose of this post is to design a simple yet scalable job scheduling system.

## **Problem Statement**

Design a job scheduler that runs jobs at a scheduled interval

## **Requirement**

#### Functional requirement

- User can schedule or view the job.
- User can provide job frequency.
	- Jobs can be run once or recurring. Jobs should be executed within X threshold time after the defined scheduled time. (let X = 15 minutes)
- Individual job execution time is no more than X minutes. (let x = 5 minutes)
- User can list all the submitted jobs with current status.
- Jobs can also have priority. Job the with higher priority should be executed first than lower priority
- Job output should be stored inside file system

#### Non-Functional requirement

- Highly available — system should always be available for users to add/view the job
- Highly scalable — system should scale for millions of jobs
- Reliability — system must execute a job at-least once, and the same Job can not run by different processes at the same time.
- Durable — system should not lose jobs information in case of any failure
- Latency — system should acknowledge the user as soon as the job is accepted. User doesn’t have to wait till job completion.

## **Traffic & Storage Estimation (Back of envelope calculation)**

- Total submitted jobs daily = 100 M (or 1000 QPS)

As we can see, an individual job can run at max 5 minutes. So, the system is highly CPU bounded.

**CPU Bound**

A modern CPU can have 16 cores, and each core can use 2 threads. Each job can run at max 5 minutes.

> # jobs can be executed by one machine = (16 cores*2 threads)/ (5 minutes*60) = **0.10 jobs per second** (or ~8000 jobs per day)

> # of machines needed to run 1000 QPS = 1000/0.10 = **10000** (wow 😮 !)

**Memory bound**

Let’s assume each job can use 5 MB of memory. You can fine tune this number based on discussion with interviewer and use case

> A modern machine with 16 GB ram can hold up-to = (16 GB/5 MB * 5 minutes * 60) =**10 jobs per second**

> # of machines needed to run 1000 QPS = 1000 / 10= **100**

This gives us a hint that single machine design for processing jobs is neither possible nor scalable. We need distributed system to design the solution

## **System interface**

Three APIs can be exposed to the user

_1. submitJob(api_key, user_id, job_schedule_time, job_type, priority, result_location)_

Here, _job_type = ONCE or RECURRING,_ and _result_location_ could be s3 location

API can return http response code 202 after accepting the job

_2. viewJob(api_key, user_id, job_id)_

Response includes the status as NOT_STARTED, STARTED or COMPLETED

3. listJobs(api_key, user_id, pagination_token)

User can query all jobs submitted, and a paginated response is returned

## **High Level design**

![No alt text provided for this image](https://media.licdn.com/dms/image/D4E12AQFTTRXQ1tMnYQ/article-inline_image-shrink_1500_2232/0/1655006113657?e=1707955200&v=beta&t=VvQDG1si-0m6Tl9DUcUcd6Qo1gUowZQaN7T4akoAmuA)

## **User request Flow**

(1 & 2) User will submit/get the job by connecting with load balancer(or API Gateway)

(3) Request will persist in the Database, and acknowledgment is sent back to user

(4 & 5) Job Executor service will continuously poll the due jobs from the Database and keep entry in the queue

(6 & 7) job executor service will execute the actual job business logic and update the final result into the file system and update the status as COMPLETED

## **Database design**

Since we don’t have a strict requirement of transaction support or any other ACID properties to follow, and keeping peak QPS (2*1000 = 2000 QPS) in mind, we can use both SQL or NoSql database. Considering obvious advantages of NoSql in terms of scale, maintenance and cost, I would go with NoSql solution using DynamoDb

User Query pattern:

- Given userId, add job
- Given userId, retrieve all jobIds

Database schema:

Table: JOB

+------------------------------+--------+
|          Attribute           |  Type  |
+------------------------------+--------+
| user_id (partition key)      | uuid   |
| job_id (sort key)            | uuid   |
| actual_job_execution_time    | date   |
| job_status                   | string |
| job_type                     | string |
| job_interval                 | int    |
| result_location              | string |
| current_retries              | int    |
| max_retries                  | int    |
| scheduled_job_execution_time | date   |
| execution_status             | string |
+------------------------------+--------+

_job_status_: This is the job status that user will see. It could have : NOT_STARTED, STARTED, COMPLETED

_execution_status_: This is the actual execution status that our service will maintain. It could have: NOT_STARTED, CLAIMED, PROCESSING, SUCCESS, RETRIABLE_FAILURE, FATAL_FAILURE

Apart from user, our job scheduler service will poll the db to get the tasks that are due. There are different ways we can achieve this

1. Partition based on X minute size bucket window

We can create index named, _scheduledJob_ to retrieve last X minutes due jobs

Index: ScheduledJob

+----------------------------------------------+------+
|                  Attribute                   | Type |
+----------------------------------------------+------+
| scheduled_job_execution_time (partition key) | time |
| job_id (sort key)                            | uuid |
+----------------------------------------------+------+
Query (SQL equivalent):
SELECT * FROM ScheduledJob WHERE scheduled_job_execution_time == now() - X

2. Partition based on X minute size bucket window plus shard id

It is possible, on a particular time window, many jobs have been received (let’s say 100K). In that case, above query performance will be very slow. We can further partition the database based on random (let’s say between 1 to Y) shard_id .

Index: ScheduledJob

+----------------------------------------------+------+
|                  Attribute                   | Type |
+----------------------------------------------+------+
| scheduled_job_execution_time (partition key) | uuid |
| shard_id (partition key)                     | int  |
| job_id (sort key)                            | uuid |
+----------------------------------------------+------+
Query (SQL equivalent):
SELECT * FROM ScheduledJob WHERE scheduled_job_execution_time == now() - X and shard_id == Y

## **Deep Dive**

![No alt text provided for this image](https://media.licdn.com/dms/image/D4E12AQGPiRU97srVnw/article-inline_image-shrink_1000_1488/0/1655006132575?e=1707955200&v=beta&t=OcL-1f9stx8r_cPO62ELUoRiX7ocqUpM0aYv74zaCHM)

1. How does a job scheduler work ?

_Job Scheduling Flow_

- Every X minute, the master node creates an authoritative UNIX timestamp and assigns a _shard_id_ and _scheduled_job_execution_time_ to each worker.
- Worker node will execute below query, and push jobs inside the Kafka queue for execution.

worker 1:

SELECT * FROM ScheduledJob WHERE scheduled_job_execution_time == now() - X and shard_id = 1
worker 2:
SELECT * FROM ScheduledJob WHERE scheduled_job_execution_time == now() - X and shard_id = 2

Fault-tolerance

- Master monitors the health of workers and knows which worker is died and how to re-assign the query to a new worker.
- If master dies, we can allocate other worker node as a master
- Further, we can also introduce the local database to track the status if worker has successfully queried the db and put the entry inside queue

2. How does a job executor work ?

Job executor service has multiple consumers that pull data from the queue. Consumer machine also has master and worker processes. Both master and worker processes operate on a Pull based model. Master will poll jobs from the queue and worker process will continuously poll the job from master by executing below code

while True:
  w = get_next_work()
  do_work(w)

_Job Execution flow & Fault-tolerance_

- When a job is picked up from the queue, consumer’s master updates JOB db attribute execution_status=CLAIMED.
- When worker process picks up the work, it updates execution_status=PROCESSING and continuously send health check to local DB.
- Upon completion of a job, worker process will push the result inside s3, update JOB db _execution_status=COMPLETED or FATAL_FAILED_, and local db with the status
- Both worker processes and master will update the health check inside the local database.

Health checker service

- Health checker service runs periodically (say every x seconds), and scan the database where last received health check from the worker process is less than the defined threshold. In that case, it considers the Job has failed to process and push back the entry to the queue.

## **Conclusion**

System design is a broad topic, and it is hard to cover every aspect of the system in 1 hour long interview. In this design, we touched most of the areas, where interviewer can further drill.

Keep Learning!!