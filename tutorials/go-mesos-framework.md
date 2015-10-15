---
layout: doc
title: Creating a Mesos Framework with Go

---
 
This tutorial creates a Mesos framework in the Go programming language and launches from within a Vagrant VM. 


#### The framework template

The minimum requirements for a framework are a scheduler, executor, and file server.

- The scheduler receives resource offers from Mesos and decides which tasks consume which resources.

- The executor knows how to run the tasks that the scheduler launches.  A more detailed explanation is [available here](http://mesos.apache.org/documentation/latest/mesos-architecture/).

- The file server provides Mesos a location from which it can retrieve the executor binary.  This component is usually not explicitly called out as a requirement of a framework and while it is not technically a component of the framework, it is required for a framework to run end-to-end.  Of course, multiple frameworks can share the same server.


### Step 1:  Setup the environment

1.  Deploy a Mesos sandbox environment by using [Playa Mesos](https://github.com/mesosphere/playa-mesos) and login to the Mesos Vagrant instance. Playa Mesos creates an Apache Mesos test environment.

1.  From within Vagrant box test environment:

    1.  Install Git and Mercurial:
	
	    ```
	    $ sudo apt-get install -y git
	    $ sudo apt-get install -y mercurial
	    ```
    
	1.  Install Go:
	    
	    ```
		$ mkdir tmp && cd tmp
		$ git clone https://go.googlesource.com/go
		$ cd go
		$ git checkout go1.4.1
		$ cd src
		$ ./all.bash
		$ export PATH=$PATH:/home/vagrant/tmp/go/bin
		```
	     
	1.  <a href="https://golang.org/doc/code.html#GOPATH" target="_blank">Setup the required Go workspace</a>:
	
	    ```
	    $ mkdir $HOME/go && cd $HOME/go
	    $ mkdir pkg && mkdir bin && mkdir src
        $ export GOPATH=$HOME/go
        $ export PATH=$PATH:$GOPATH/bin
        ```

### Step 2: Create a new framework

1.  Get the Mesos Go framework template code:

	```
	$ mkdir -p $GOPATH/src/github.com/mesosphere 
	$ cd $GOPATH/src/github.com/mesosphere  
	$ git clone https://github.com/mesosphere/mesos-framework-tutorial.git
	$ go get ./... 
	$ cd mesos-framework-tutorial/
	$ git checkout -b tutorial origin/tutorial
	```

	You should now have all of the tutorial code and be in the 'tutorial' branch.  This next steps add framework functionality as we go and commit to the `tutorial` branch.
		
### Step 3: Initialize the framework configuration 

In this step, the framework is initialized. 

1.  Checkout the initial branch:
    
    ```
    $ git checkout aae4f846a6dd7e5e0fba2d737dc82718ddde9e2b
    ```
    
1.  Compile the framework code:
	
	```
	$ cd $GOPATH/src/github.com/mesosphere/mesos-framework-tutorial/
	$ go build -o example_scheduler
	$ cd $GOPATH/src/github.com/mesosphere/mesos-framework-tutorial/executor
	$ go build -o example_executor
	```

1.  Run the framework code:

	```
	$ cd $GOPATH/src/github.com/mesosphere/mesos-framework-tutorial
	$ ./example_scheduler --master=127.0.0.1:5050 --executor="$GOPATH/src/github.com/mesosphere/mesos-framework-tutorial/executor/example_executor" --logtostderr=true
	```
	
	The scheduler receives one resource offer from Mesos and then appears to block. By not accepting the resource offer the scheduler has implicitly rejected the offer.  No tasks are launched.  A configurable timeout will eventually occur and the resource will again be offered to the scheduler.  The output should look like this:

	```
	...
	I0713 19:03:42.775536   25174 scheduler.go:446] Framework registered with ID=20150713-1...
	I0713 19:03:42.775962   25174 example_scheduler.go:48] Scheduler Registered with Master...
	I0713 19:03:42.776181   25174 utils.go:32] Received Offer <20150713-...> with cpus=2 mem=1000
	```

	- 'main.go' initializes the configuration of all three components and packages them into a configuration object. This object is passed to the MesosSchedulerDriver, which is then started.

	- 'scheduler/example_scheduler.go' implements the required scheduler interface and logs all calls from Mesos.

	- 'executor/example_executor.go' compiles to an executable binary which is capable of hosting tasks.  It implements the executor interface and for the most part just logs calls from Mesos.  The exception is the LaunchTask method which makes status updates regarding tasks, but does not actually do any work.
	
1.  Press CTRL + C to stop the framework.  

### Step 4.  Accept offers and launch tasks

In this step, the framework begins accepting offers from Mesos and launching tasks.  The framework can now iterate across the offers provided by Mesos and launch tasks until we run out of resources.

1.  Checkout the branch:
    
    ```
    $ git checkout bc5da5bb52ad91871fb842e454133fe45d08d319
    ```
    
1.  Compile the framework code:
	
	```
	$ cd $GOPATH/src/github.com/mesosphere/mesos-framework-tutorial/
	$ go build -o example_scheduler
	$ cd $GOPATH/src/github.com/mesosphere/mesos-framework-tutorial/executor
	$ go build -o example_executor
	```

1.  Run the framework code:

	```
	$ cd $GOPATH/src/github.com/mesosphere/mesos-framework-tutorial
	$ ./example_scheduler --master=127.0.0.1:5050 --executor="$GOPATH/src/github.com/mesosphere/mesos-framework-tutorial/executor/example_executor" --logtostderr=true
	```

	The executor launches the tasks and reports status to Mesos indicating that the tasks are finished.  This frees the resources and they are offered to the scheduler again.  This loop continues endlessly as long as the scheduler process doesn't crash. A long-running distributed service has now been completed.  'example_executor.go' indicates where the real work is done in its 'LaunchTask' method.

	The output indicates that tasks are running:

	```
	I0713 19:06:52.967857   25228 utils.go:32] Received Offer <20150713-...> with cpus=2 mem=1000
	I0713 19:06:52.967939   25228 example_scheduler.go:90] Prepared task: go-task-1 with offer 20150713-... for launch
	I0713 19:06:52.967973   25228 example_scheduler.go:90] Prepared task: go-task-2 with offer 20150713-... for launch
	I0713 19:06:52.968075   25228 example_scheduler.go:96] Launching  2 tasks for offer 20150713-...
	I0713 19:06:54.173174   25228 example_scheduler.go:103] Status update: task 1  is in state  TASK_RUNNING
	I0713 19:06:54.174417   25228 example_scheduler.go:103] Status update: task 2  is in state  TASK_RUNNING
	I0713 19:06:54.176197   25228 example_scheduler.go:103] Status update: task 1  is in state  TASK_FINISHED
	I0713 19:06:54.178064   25228 example_scheduler.go:103] Status update: task 2  is in state  TASK_FINISHED
	...
	```
	
1.  Press CTRL + C to stop the framework. 

### Step 5. Create batch image processing task

In this step, we implement a simple batch image processing framework that shows data and metadata flowing back and forth between the scheduler and the executor.  In this case we invert the images, as shown below.

1.  Checkout the branch:
    
    ```
    $ git checkout e5e18f6f8c1a28d2d2d3ba725a99841b26e2f425
    ```
    
1.  Compile the framework code:
	
	```
	$ cd $GOPATH/src/github.com/mesosphere/mesos-framework-tutorial/
	$ go build -o example_scheduler
	$ cd $GOPATH/src/github.com/mesosphere/mesos-framework-tutorial/executor
	$ go build -o example_executor
	```

1.  Run the framework code:

	```
	$ cd $GOPATH/src/github.com/mesosphere/mesos-framework-tutorial
	$ ./example_scheduler --master=127.0.0.1:5050 --executor="$GOPATH/src/github.com/mesosphere/mesos-framework-tutorial/executor/example_executor" --logtostderr=true
	```

	The framework takes [a list of image URLs](https://github.com/mesosphere/mesos-framework-tutorial/blob/tutorial/images), assigns each URL to one task, and collects the results of the image processing.  In order for each task to know which image it should process we need to [encapsulate the URL in the task here](https://github.com/mesosphere/mesos-framework-tutorial/blob/tutorial/scheduler/example_scheduler.go#L104).  The executor then [reads this information on the other side](https://github.com/mesosphere/mesos-framework-tutorial/blob/tutorial/executor/example_executor.go#L65).  The executor then [processes the image and uploads it](https://github.com/mesosphere/mesos-framework-tutorial/blob/tutorial/executor/example_executor.go#L72-87) to the same HTTP server that previously served the executor binary.

	The HTTP server was modified to allow for image uploads, by [registering an upload handler function](https://github.com/mesosphere/mesos-framework-tutorial/blob/tutorial/executor/example_executor.go#L72-87).  It saves images to the directory from which the Vagrant VM was launched.  This [directory mapping was added](https://github.com/mesosphere/mesos-framework-tutorial/blob/tutorial/Vagrantfile#L63) in the VagrantFile.


	<img src="https://raw.githubusercontent.com/mesosphere/mesos-framework-tutorial/tutorial/original.jpg?token=AAinR_TyrX7bO_bT7H4QJRMtj5Be-jYAks5VrZPSwA%3D%3D" alt="Original image" width="100%" align="middle">

	<img src="https://raw.githubusercontent.com/mesosphere/mesos-framework-tutorial/tutorial/inverted.jpg?token=AAinR7nLI_1CmA9ImzaiARIniQt0K1lyks5VrZQRwA%3D%3D" alt="Inverted image" width="100%" align="middle">
	
1.  Press CTRL + C to stop the framework. 





