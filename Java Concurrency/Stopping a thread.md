# Stopping a Thread

There's a task running in a separate thread and your job is to stop the task if it exceeds 10 minutes so basically we have a thread let's say the main thread that's going to start another thread which will actually run the task and then after a certain time out in this case it's 10 minutes you want to be able to stop that particular task.

## Thread using Runnable

so let's say we create a new thread in our main method which is our main thread so we are going to create a new thread we are going to submit a runnable to it and in that runnable will have all that steps which are there in a task in this case it doesn't matter what and then we are going to start that particular thread once we start the thread we want to wait for 10 minutes and then we want to stop that particular thread so to stop the thread all we have currently is the handle of thread which is t1 so let's see if there are any api's in the thread class which will help us stop the thread and the short answer is no we do not have a stop method in the thread class okay so we cannot really kill a thread or stop a thread so what other alternative do we have so let's say instead of a raw thread we can try something else

```java
public static void main(String[] args) {
	Thread t1 = new Thread(() -> {
		// task step 1
		// task step 2
		// ...
	});

	// 1. start the thread
	t1.start();

	// 2. TODO: timeout for 10 minutes
	// 3. stop the thread
}
```

## Thread Pool using `ExecutorService`

### Submit a Runnable task
let's try a thread pool instead so let's say we create a thread pool and we are going to submit our task in the form of a runnable to that particular thread pool so now it's thread pools responsibility to assign a thread to that task and to run that task we will have a time out of ten minutes we'll discuss that later how once that ten minutes has passed we are going to stop the task so for the thread pool we could do either a shutdown or we could do shutdown now in an attempt to stop the threads which are running our task


```java
public static void main(String[] args) {
	ExecutorService threadPool =. Executors.newFixedThreadPool(2);
	
	// 1. submit the runnable task
	threadPool.submit(() -> {
		// task
		// ...
	});
	
	// 2. TODO: timeout for 10 minutes
	
	// 3. stop the thread (by shutting down the pool)
	threadPool.shutdown();
	threadPool.shutdownNow();
}
```

so now the question is if we execute the method shut down or shut down now does it really stop the thread if we check the actual documentation of shutdown & shutdownNow

| Methods | Description |
|-------------|-------------|
| `shutdown()` | 1. No new tasks accepted |
| |2. Previously submitted tasks waiting in the queue are executed |
| `shutdownNow()` | 1. No new tasks accepted |
| | 2. Previously submitted tasks waiting in the queue are returned |
| | 3. Tasks being run by the thread(s) are _attempted_ to stop (No guarantees) |

### Submit a Callable task
we could try something like a callable instead of runnable so we'll use the same thread pool but now instead of submitting a runnable we're going to submit a callable if we submit a callable the thread pool returns us a handle in the form of a future and future class has this method called cancel so maybe after waiting for 10 minutes if we do future dot cancel it will stop the thread but again if you check the documentation of future it does not guarantee that it's going to stop the task it will only attempt to stop the thread

```java
public static void main(String[] args) {
	ExecutorService threadPool =. Executors.newFixedThreadPool(2);
	
	// 1. submit the callable task
	Future<?> future = threadPool.submit(() -> {
		// task
		// ...
	});
	
	// 2. TODO: timeout for 10 minutes
	
	// 3. stop the thread (by cancelling the future)
	future.cancel(true);
}
```

> Java threads cannot be killed. They are cooperative in nature. You need to ask politely.

There are two ways to ask the thread to stop:
- using `interrupt()` method
- using `volatile`
- using `AtomicBoolean`

## `interrupt()` method

### Thread class

so let's start with the interrupts so for using interrupts we'll create a new thread we'll wait for 10 minutes and we are going to call the interrupts method on that thread and the task itself needs to listen for the interrupts so within the task we need a condition and if condition saying that is the current thread interrupted if yes that means that someone wants me to stop doing what I am doing and then it's up to this task's responsibility whether to return or whether to continue in this case we know that the timeout has passed and we want to stop the task so we'll do a return we'll use a while loop so we'll keep checking whether the thread is interrupted and if it is not interrupted then we'll perform the next steps so let's say we have ten steps in our task after every step we are going to check if the thread is interrupted if not we'll continue with this step and if the task is interrupted we'll just come out of the while loop and the task is finished and by default the threads job is done and it will be killed

```java
public static void main(String[] args) {
	Thread t1 = new Thread(() -> {
		while (!Thread.currentThread().isInterrupted()) {
			if (!Thread.currentThread().isInterrupted()) {
				// task step 1
			}
			// ...
			// task step n
		}
	});

	// 1. start the thread
	t1.start();

	// 2. TODO: timeout for 10 minutes
	
	// 3. stop the thread
	t1.interrupt();
}
```

we spoke about how thread pool dot shutdown now does not guarantee the stopping of the threads and that is because all shutdown now does is internally it will go to all the threads and it will do a thread dot interrupt that means that even if we used the thread pool within the thread pool task also we can apply the same concept of checking for interrupt so when we do thread pool dot shut down now internally it will apply the interrupt and our task will catch that interrupt it will come out of the while loop and that particular task will be marked as complete

```java
public static void main(String[] args) {
	ExecutorService threadPool =. Executors.newFixedThreadPool(2);
	
	// 1. submit the runnable task
	threadPool.submit(() -> {
		while (!Thread.currentThread().isInterrupted()) {
			if (!Thread.currentThread().isInterrupted()) {
				// task step 1
			}
			// ...
			// task step n
		}
	});
	
	// 2. TODO: timeout for 10 minutes
	
	// 3. stop the thread (by shutting down the pool)
	threadPool.shutdownNow();
}
```

so  the exact same thing with future dot cancel so the same example that we saw earlier where instead of a runnable we had a callable which returned the future and after 10 minutes we were attempting future dot cancel and when we have the parameter value as true that means that whenever cancel is called internally whatever thread is running our future related task that particular task is interrupted so even here if we have the check for interrupts we'll be able to come out of the while loop and we can complete our tasks

```java
public static void main(String[] args) {
	ExecutorService threadPool =. Executors.newFixedThreadPool(2);
	
	// 1. submit the callable task
	Future<?> future = threadPool.submit(() -> {
		while (!Thread.currentThread().isInterrupted()) {
			if (!Thread.currentThread().isInterrupted()) {
				// task step 1
			}
			// ...
			// task step n
		}
	});
	
	// 2. TODO: timeout for 10 minutes
	
	// 3. stop the thread (by cancelling the future)
	future.cancel(true);
}
```


so interrupts are one of the ways to check if some other thread wants us to stop the other way is to simply use a volatile variable so let's say in this example and using Java 8 lambdas we are going to have a class called my task that implements runnable and it overrides the run method and all the steps which are involved in the task are mentioned here but these steps are you run only as long as the variable keepRunning is true and this variable is public and of the type volatile so as a task it is saying that I'll keep running until the value of this volatile variable is true just start a raw thread with that task and after 10 minutes if the main thread just changes the value of this volatile from true to false since it's a volatile when the thread t1 reaches the while loop again it will find the value of keep running as false and that is when it will come out of the while loop and the run method has been completed and the thread automatically dies and since volatiles and atomic boolean's are very similar we can replace this volatile with an atomic boolean and the entire code will remain the same so either we use atomic boolean or volatile it really doesn't matter so in all these earlier examples we assumed that our task is running some Java operations as a set of steps and after every step we had a while operation where it checked for the interrupts or the volatiles and it performed the next step 

```java
public void process() {
	// 1. Create a task and submit to a thread
	MyTask task = new MyTask();
	Thread t1 = new Thread(task);
	t1.start();
	
	// 2. TODO: timeout for 10 minutes

	// 3. ask task to stop using volatile
	task.keepRunning = false;
}

private class MyTask implements Runnable {
	public volatile boolean keepRunning = true;

	@Override
	public void run() {
		while (keepRunning) {
			// steps
		}
	}
}
```


```java
public void process() {
	// 1. Create a task and submit to a thread
	MyTask task = new MyTask();
	Thread t1 = new Thread(task);
	t1.start();
	
	// 2. TODO: timeout for 10 minutes

	// 3. stop the thread
	task.stop();
}

private class MyTask implements Runnable {
	private AtomicBoolean keepRunning = new AtomicBoolean(true);

	@Override
	public void run() {
		while (keepRunning.get()) {
			// steps
		}
	}

	public void stop() {
		keepRunning.set(false);
	}
}
```


> Cannot react to interrups not volatile if control not in Java, ie, during I/O operations

but what if instead of Java operations it is doing some IO operations so let's say in this case our task instead of doing some Java operation is making a stored procedure call which is a database operation and let's say this store procedure is very slow and it takes more than 10 minutes in that case our thread will be stuck at this point and it will never be able to reach the while again to check if there is an interrupt or a volatile changed right so it's important to confirm that it's not an i/o operation it has to be a Java operation so that you can keep going back to the while loop so in such cases the libraries itself like the JDBC library and HTTP client library they have these timeout methods where you can specify them the maximum timeout you can wait for this operation


```java
public void process() {
	// 1. Create a task and submit to a thread
	MyTask task = new MyTask();
	Thread t1 = new Thread(task);
	t1.start();
	
	// 2. TODO: timeout for 10 minutes

	// 3. stop the thread
	task.stop();
}

private class MyTask implements Runnable {
	private AtomicBoolean keepRunning = new AtomicBoolean(true);

	@Override
	public void run() {
		while (keepRunning.get()) {
			// step 1
			// step 2: stuck here for 15 minutes..
			jdbcClient.storedProcedureCall();

			// step 3: HTTP call..
			httpClient.request();
		}
	}

	public void stop() {
		keepRunning.set(false);
	}
}
```

## Conditional Timeout

so now that we have understood how to stop a thread let's go back to the second part of the question which is you want to be able to do that only after a certain time out in this case the time out of 10 minutes and this part is relatively straightforward so the first option we have is we start the thread and for the timeout we just say thread dot sleep for 10 minutes so this will make the main thread sleep for 10 minutes after it wakes up we can use any of the previously discussed methods to stop our task we can also use a scheduler service for it so here we have a scheduled executor service where we are scheduling stopping off our task after 10 minutes so we are saying that run this particular operation after 10 minutes and within that operation we are saying thus not stopped and third instead of runnable if you have a caller bill where do you get a future object even the future object has this method called future dot get and you can give it a duration and in this case we can guess a get me the value of this future and wait maximum for 10 minutes if in that 10 minutes you do not get the value of the future then there will be a timeout exception and within the timeout exception we can use the same thing as earlier either we can use future dot cancel or we can use the task dot stop using the volatiles so that's it that's how we solve this particular question so that's it for this video this was a very interesting question to me if you have any similar interesting questions that you want us to solve together please let me know in the comments thanks a lot for watching and see you in the next one bye

```java
public void process() {
	// 1. Create a task and submit to a thread
	MyTask task = new MyTask();
	Thread t1 = new Thread(task);
	t1.start();
	
	// 2. timeout for 10 minutes
	try {
		Thread.sleep(10 * 60 * 1000);
	} catch (InterruptedException e) {
		// process exception
	}

	// 3. stop the thread
	task.stop();
}
```

```java
public void process() {
	// 1. Create a task and submit to a thread
	MyTask task = new MyTask();
	Thread t1 = new Thread(task);
	t1.start();
	
	// 2. scheduler will stop the thread for 10 minutes
	ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
	scheduler.schedule(MyTask::stop, 10, TimeUnit.MINUTES);
}
```

```java
public void process() {
	ExecutorService threadPool =. Executors.newFixedThreadPool(2);
	
	// 1. Create a task and submit to the thread pool
	MyTask task = new MyTask();
	final Future<?> future = threadPool.submit(task);
	
	// 2. wait for 10 minutes to get response
	try {
		future.get(10, TimeUnit.MINUTES);
	} catch (InterruptedException | ExecutionException e) {
		// process exception
	} catch (TimeoutException e) {
		future.cancel(true); // if using interrupts
		task.stop(); // if using volatile
	}
}
```

