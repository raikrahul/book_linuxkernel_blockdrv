# request-mode

So far we've looked into bio-based IO.
Our mybrd driver receives a bio, followed by another, followed by another, from kernel and processes them in order.
As I had explained before, bio is the least unit of I/O.
Of course, there is bigger unit than bio.
It is called as the request.

In previous chapters, I explained that request-queue is a queue in which requests are stored.
But we didn't use the request at all.
We just exported a bio-handler function for kernel and had kernel call it with bio.
Finally we will use request-queue with request in this chapter.

Again, bio is a set of sequential sectors.
Sectors are sequential but _pages_, which _store sectors_, are _not sequential_.
So we extracted segments from bio and check _which page_ includes the segment.

The request is a set of bios.
When Kernel starts I/O processing, it creates a request which includes one bio and inserts the request into the request-queue.
Soon or later, driver will extract a request from the request-queue and process it.
While the request is in the request-queue, I/O scheduler does something for the request.

As I told you, there is only one bio in the request at the beginning.
What I/O scheduler does is, it adds more bios into the request.
For example, let's assume there is a write-request including 8 sectors (sector numbers are 0 ~ 7) in the request-queue.
And user application continues to write data at sector 8 ~ 15.
Kernel also _creates a request_ for sector 8 ~ 15 and adds it into the request-queue.
Now _I/O scheduler_ does its job.
It _merges two requests_ into one for sector ranging from 0 to  15.

The job of I/O scheduler is _watching the request-queue_ and merging sequential I/O requests.
It can increase disk throughput because head of HDD disk moves less.

I/O scheduler has many  scheduling  policies such as noop, cfq, deadline and etc.
And it has more features to increase block I/O performance.
Please refer other documents for the detail.

In this chapter, we will modify mybrd driver source to handle requests.

source: https://github.com/gurugio/mybrd/blob/ch04-request-mode/mybrd.c

## create the request-queue for request-mode

### create a new queue

In previous chapters, we created mybrd_device object in ```mybrd_alloc()``` and _generated a request-queue_ with ```blk_alloc_queue_node()```.
The driver can get bio from kernel via the request-queue.

So the first thing we need to do is to change the request-queue.
Please see ```mybrd_alloc()``` function in the new source file, there is new variable, global variable, of type, int, called as, queue_mode, that shows what is the base unit of I/O processing.
If I/O processing is based on the bio, queue_mode is set as ```MYBRD_Q_BIO```, so called as bio-mode.
If you want to change I/O processing to be based on the request, you need to set the queue_mode value as ```MYBRD_Q_RQ```, so called as request-mode.

The queue_mode value is defined as ```MYBRD_Q_RQ```, so the request-queue is created by ```blk_init_queue_node()``` in ```mybrd_alloc()``` of the source file.
There is still code to create the bio-based request-queue with ```blk_alloc_queue_node()```.

I'll explain the _difference_ of ```blk_alloc_queue_node()``` and ```blk_init_queue_node()``` _later_.

```blk_init_queue_node()``` registers a function of driver, ```mybrd_request_fn()``` as the request handler.
And driver should provide spin-lock for synchronization of the request-queue.
Finally ```blk_queue_prep_rq()``` is called to ```register blk_queue_prep_rq()```.
I'll explain ```blk_queue_prep_rq()``` later.

For now, please notice that we need to do three things:
1. create a request-queue with ```blk_init_queue_node()```
2. provide a spin-lock
3. call ```blk_queue_prep_rq()```.

### blk_init_queue_node()

From now, we start reading kernel source.
I think you've done downloading kernel source and generating tag.
Please find ```blk_init_queue_node()``` function.
You can see ```blk_init_queue_node()``` is _combination_ of ```blk_alloc_queue_node()``` and ```blk_init_allocated_queue()```.

We already know what ```blk_alloc_queue_node()``` does.
It generates a bio-based request-queue.
So what ```blk_alloc_queue_node()``` does are
1. create a bio-based request-queue
2. _initialize_ the request-queue to be request-handler

Let's see the code of ```blk_init_allocated_queue()```.
It stores the pointer of request handler to ```request_fn``` field of the request-queue.
Kernel will call the request_fn field to handle the request.
And it also stores _the spin-lock_ into the request-queue.

We can find another familiar function, ```blk_queue_make_request()``` which we used to make bio-based I/O processing.
It takes the request-queue and ```blk_queue_bio()``` function as parameters.
As we made before, kernel initializes ```blk_queue_bio()``` function as bio handler.
Lastly  kernel calls ```elevator_init()``` function.

There is something funny.
Do you see a comment ``/* init elevator */``?
Function name is elevator_init() that can explain itself.
Why did kernel developer write that comment?
Kernel code is reviewed by so many genius developers and tested by so many Linux servers, embedded devices and Android devices.
Kernel developers always try to keep code compact and simple.
They write comment when it is really necessary.
So I just guess that elevator initializing is very important thing in the IO processing.

Actually that elevator is the I/O scheduler.
When we make bio-based driver, _we create a request-queue without the elevator_.
Now we can see that kernel creates a request-queue with the elevator.
That means the request-queue will include the I/O scheduler and generally have _better throughput than the bio-based request-queue_.

You maybe wonder why scheduler is names as elevator.
Yes, kernel developers often choose wierd name.
Please google why it is called as elevator just for fun.
There should be a good reason.

### blk_queue_bio()

Calling ```blk_queue_bio()``` is the _only difference_ ```between blk_init_queue_node()``` and ```blk_alloc_queue_node()```.
When driver creates the request-queue with ```blk_alloc_queue_node()```, kernel calls the driver function _for bio processing_.
But, when  driver creates the request-queue with ```blk_init_queue_node()```, kernel calls the kernel function, ```blk_queue_bio()```, _for bio processing_.

Let's check ```blk_queue_bio()```.
It's a little long and complex, so let's read it briefly.
It calls ```bio_endio()``` when error happens.
Then it calls ```bio_attempt_back_merge()``` and ```bio_attempt_front_merge()``` to _merge bios_ **in** _the request-queue_.

The ```get_request()``` function creates new request.
There is a _limit_ over _number of requests_ in a request-queue.
It is called as the _depth of the request-queue_.
If there are more requests than the depth, ```get_request()``` makes the process sleep.

There is a plug.
I'll skip it because it's beyond of this document.
Please refer other documents for block layer.

Finally ``__blk_run_queue()`` function _calls_ request_fn function which is the pointer to ```mybrd_request_fn()```.

I skipped many details and show only brief flow.
But I hope this can help you start in-depth investigation of kernel code and read other books like "Understanding the Linux kernel".

## request handling

Let's see the code for the request handling.

### mybrd_request_fn()

We registered ```mybrd_request_fn()``` as the request handler when we created the request-queue in ```mybrd_alloc()```.
If I/O is generated, kernel stores a request in the request-queue and calls ```mybrd_request_fn()``` with the request-queue.
As you can see, ```mybrd_request_fn()``` has an argument, which is  a pointer to the request-queue.
What mybrd_request_fn() job is, to be able to perform  of extracting the request from the request-queue and processing data in the request.

#### blk_fetch_request()

It _extracts a request from the request-queue_.
In ```blk_queue_bio()```, kernel locked the spin-lock before calling ``__blk_run_queue()``.
So ```blk_fetch_request()``` will be executed with _locked request-queue_.
Driver should unlock the request-queue after ```blk_fetch_request()```.
User application can generate more I/Os and kernel can add more requests into the request-queue during driver releases the spin-lock.
And before calling ```blk_fetch_request()```, driver should lock the request-queue.
If driver doesn't unlock the request-queue, user application would have to wait until all requests are handled.
So performance of mybrd would be bad.

The ```blk_fetch_request()``` calls ```blk_peek_request()``` and _prep_rq_fn function pointer_.
prep_rq_fn function should return one of ```BLKPREP_OK, BLKPREP_DEFER and BLKPERP_KILL```.

The ```mybrd_alloc()``` passes ```mybrd_prep_rq_fn()``` to ```blk_queue_prep_rq()```, so prep_rq_fn function pointer indicates ```mybrd_prep_rq_fn()```.
prep_rq_fn function should be a function to make a request ready.
```mybrd_prep_rq_fn()``` only stores the pointer of mybrd object at the special field of the request.

#### blk_end_request_all()

It informs kernel the end of the request.
Kernel creates the request, so kernel should free it.
Driver passes the request pointer to be freed and error status to kernel.
The error could be -EIO.
If all is good, driver passes 0.

### _mybrd_request_fn()

This function processes one request.
The request from ```blk_getch_request()``` is be readied by ```mybrd_prep_rq_fn()```.
So the special field has pointer to mybrd object.
You can make your own preparation function and set the special field with your own data.

```blk_rq_pos()``` gets the first sector of the request.
You can check ```include/linux/blkdev.h``` file that includes many helper functions to get information of the request, such as ```blk_rq_pos()```.
You should not access the request data structure directly and use those helper functions, because the request data structure can be changed in any kernel version.
Even-if the request data structure is changed, helper function will exist with same interface.

The mybrd driver use ```rq_for_each_segment()``` to get each segment of the request.
As I told you before, the _request always includes sequential sectors_.
So actually we don't need to extract each segment with rq_for_each_segment.
But I used rq_for_each_segment() because I just wanted to use already implemented sector-based routines for radix-tree.
You can find a better way to process the request.

Handling segment is the same to bio-mode processing that we already implemented in previous chapter.

## request handling in SOFTIRQ

source: https://github.com/gurugio/mybrd/blob/ch04-request-mode-softirq/mybrd.c

Let's add one more mode, so-called irq-mode.
For request-mode, driver processes a request whenever user generates IO.
It can be possible because mybrd driver stores data on memory, not real disk.

If we should store data in real hard-disk, we could process the request when the disk generates interrupt to show previous I/O handling is finished.
Of course, we cannot process the request in the interrupt handler.
Execution in an Interrupt context has many limitations.

Therefore we need a way to delay the request handling.
That is the softirq mechanism.
Please refer following document for the detail.

https://lwn.net/Articles/520076/

I'll just introduce very simple example to help you understand how it works.
If you cannot understand above document for softirq, please read this example first.
Then you can understand softirq better.

### blk_complete_request()

There is a per-cpu list, ```blk_cpu_done```.
If driver calls ```blk_complete_request()``` with a request, kernel adds the request into _blk_cpu_done_ list.

If you want to know detail of blk_cpu_done, please check ```blk_softirq_init()``` function.
It creates blk_cpu_done list and registers it as ```BLOCK_SOFTIRQ```.

Soon or later, when kernel executes softirq routines, ```blk_done_softirq()``` is called.
The ```blk_done_softirq()``` extracts requests from blk_cpu_done list and _calls softirq_done_fn function pointer_ with the extracted request.
The function pointer, softirq_done_fn, is pointer to ```mybrd_softirq_done_fn()```.

In other words, driver function ```mybrd_softirq_done_fn()``` is called only when kernel processes softirq routines.
Before it, driver just delays the request handling.

### mybrd_softirq_done_fn()

The job of ```mybrd_softirq_done_fn()``` is the same to ```mybrd_request_fn()```.
It _processes one request_.
So ```mybrd_softirq_done_fn()``` also calls ```mybrd_request_fn()``` for I/O processing.

Only one different thing is extracting the request from the request-queue: ``list_del_init(&req->queuelist);``
The queuelist field of the request is the list node linked to the request-queue.

Calling ```blk_end_request_all()``` is also same to other mode.

# experiment

I will not write how to test the request-mode because it's the same to the bio-mode.
Please check which mode has better performance and what behavior is different.
