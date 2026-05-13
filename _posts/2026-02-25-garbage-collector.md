---
layout: post
title: JVM Garbage Collection - What Actually Matters in Production?
date: 2026-02-25 10:59:00-0400
description: JVM Garbage Collection in Production Systems
tags:
categories:
giscus_comments: false
related_posts: false
toc:
  beginning: true
---
<span style="margin-left: 10px;"></span>
Garbage collection is not just JVM internals. For a Java backend developer, GC is connected to request latency, 
allocation patterns, object lifetime, framework behavior, container memory limits, observability, and incident response.

## From HTTP Request to Heap Allocation

<span style="margin-left: 10px;"></span>
When a request enters a Java backend service, it does not only consume CPU and network resources. It also creates objects.

<span style="margin-left: 10px;"></span>
A lot of them!!!

<span style="margin-left: 10px;"></span>
For a single request, that may not matter. But in a production service handling hundreds or thousands of requests per 
second, those allocations become one of the main drivers of garbage collection behavior.

### A Request Is an Allocation Pipeline

<span style="margin-left: 10px;"></span>
Considering a typical Spring Boot REST API request:

```java
@PostMapping("/beers")
public BeerResponse createBeer(@RequestBody CreateBeerRequest request) {
    Beer beer = beerService.createBeer(request);
    return beerMapper.toResponse(beer);
}
```

<span style="margin-left: 10px;"></span>
This may look simple, but several allocations happen before and after this method runs.

<span style="margin-left: 10px;"></span>
A typical request may involve:
- request and response DTOs
- service-layer objects
- domain entities
- collections and strings
- logging messages
- database result objects
- etc...

<span style="margin-left: 10px;"></span>
Most of these objects are short-lived. They exist only during the lifetime of the request.

<span style="margin-left: 10px;"></span>
That is good. The JVM is optimized for this pattern.

<span style="margin-left: 10px;"></span>
But at scale, even short-lived objects can become expensive.

### Backend Services Allocate Constantly

<span style="margin-left: 10px;"></span>
A Java backend service is usually allocation-heavy by design.

<span style="margin-left: 10px;"></span>
Frameworks like Spring, Hibernate, Jackson, Bean Validation, Micrometer, logging libraries, and HTTP servers create 
many temporary objects to provide abstraction and developer productivity.

<span style="margin-left: 10px;"></span>
That is not automatically a bad thing, the problem starts when allocation rate becomes too high.

<span style="margin-left: 10px;"></span>
Allocation rate means how much memory the application allocates per second.

<span style="margin-left: 10px;"></span>
For example:

```
500 MB/s allocation rate
```

<span style="margin-left: 10px;"></span>
This means that the application is creating half a gigabyte of new objects every second.

<span style="margin-left: 10px;"></span>
Even if most objects die quickly, the garbage collector still needs to process that memory.  This is why allocation 
rate is often more important than total heap usage. A service can have plenty of free heap and still suffer from 
frequent garbage collections because it allocates too aggressively.

### Most Request Objects Should Die Young

<span style="margin-left: 10px;"></span>
Modern JVM garbage collectors rely heavily on the hypothesis that most objects die young. This fits backend applications 
very well. For example, these objects usually die quickly:

```java
CreateBeerRequest request
BeerResponse response
List<BeerBreweryResponse> breweries
String formattedMessage
Map<String, Object> logContext
```

<span style="margin-left: 10px;"></span>
They are created during request processing and become unreachable shortly after the response is sent.  These objects are 
usually allocated in the young generation, specifically in Eden space. When Eden fills up, the JVM performs a young GC 
to remove dead objects. This is normally cheap because most young objects are already garbage. That is the happy path :)

<span style="margin-left: 10px;"></span>
Problems start when objects that should be short-lived accidentally survive.

<span style="margin-left: 10px;"></span>
For example:

```java
private static final List<CreateBeerRequest> requests = new ArrayList<>();
```
<span style="margin-left: 10px;"></span>
If these objects remain reachable, the GC cannot collect them. After surviving enough young collections, they may be 
promoted to the old generation, which changes the situation completely.

<span style="margin-left: 10px;"></span>
Young generation garbage is expected. Old generation growth is more serious.
In backend systems, old generation growth often means one of these things:
- a legitimate cache is growing
- a queue is not being drained fast enough
- a memory leak exists
- request-scoped data escaped into long-lived state
- the application is holding too many database entities
- the service is processing payloads that are too large

<span style="margin-left: 10px;"></span>
This code looks harmless at first glance:

```java
public List<BeerResponse> getBeers(User user) {
    return beerRepository.findByUserId(user.getId())
            .stream()
            .map(beer -> new BeerResponse(
                    beer.getId(),
                    beer.getStatus().name(),
                    beer.getCreatedAt().toString()
            ))
            .toList();
}
```

<span style="margin-left: 10px;"></span>
But under load, it may allocate many objects:
- database entity objects
- stream pipeline objects
- lambda-related structures
- response DTOs
- strings from name()
- strings from toString()
- internal list storage

<span style="margin-left: 10px;"></span>
This does not mean streams are bad. It means hot paths must be understood in terms of allocation behavior. In 
low-traffic code, readability may matter more. In high-QPS endpoints, allocation patterns can affect GC frequency and latency.

### JSON Serialization: a Major Allocation Source

<span style="margin-left: 10px;"></span>
For backend APIs, JSON processing is one of the most common sources of allocation.

<span style="margin-left: 10px;"></span>
When using Jackson, the application may allocate:
- request DTOs
- nested DTOs
- temporary parsing buffers
- strings
- collections
- reflection metadata structures
- response serialization buffers

<span style="margin-left: 10px;"></span>
Example:

```java
@PostMapping("/users/search")
public List<UserResponse> search(@RequestBody SearchRequest request) {
    return userService.search(request);
}
```

<span style="margin-left: 10px;"></span>
If the response contains thousands of users, the service may allocate a large object graph:

```
List<UserResponse>
  -> UserResponse
  -> AddressResponse
  -> List<RoleResponse>
  -> String fields
  -> serialization buffers
```

<span style="margin-left: 10px;"></span>
Large responses increase memory pressure, even if the endpoint seems functionally correct.
This is why pagination is not only a database concern, but it is also a memory and GC concern.

### Hibernate Can Increase Object Lifetime

<span style="margin-left: 10px;"></span>
ORM frameworks can also influence GC behavior. With Hibernate/JPA, a query does not just return raw database rows. It may create:
- entity objects
- proxy objects
- collections
- persistence context entries
- lazy-loading structures

<span style="margin-left: 10px;"></span>
For example:

```java
List<Beer> beers = beerRepository.findAll();
```

<span style="margin-left: 10px;"></span>
This can be dangerous if the result set is large. The persistence context may keep entities reachable longer than 
expected. That means objects that could have died quickly may stay alive until the transaction or session ends.
In batch jobs or large backend operations, this can cause old generation pressure.

<span style="margin-left: 10px;"></span>
A safer pattern may be:

```java
Page<Beer> page = beerRepository.findAll(pageable);
```

or streaming/clearing the persistence context carefully in batch processing.

<span style="margin-left: 10px;"></span>
This seems obvious, but I thought it was worth mentioning that database result size affects heap pressure.

### Logging Can Allocate More Than Expected

<span style="margin-left: 10px;"></span>
Logging is another common source of hidden allocation.  This is especially true when logs are built eagerly:

```java
log.debug("Beer details: " + expensiveObject.toString());
```

<span style="margin-left: 10px;"></span>
Even if debug logging is disabled, the string concatenation may still happen before the logger decides whether to write the message.

<span style="margin-left: 10px;"></span>
I prefer parameterized logging:

```java
log.debug("Beer details: {}", expensiveObject);
```

<span style="margin-left: 10px;"></span>
But even parameterized logging is not free if the object’s toString() is eventually called.  This becomes important 
in high-throughput systems where every request logs multiple lines. Logging full payloads is especially risky. This can 
create large strings, increase allocation rate, slow down request processing and of course expose sensitive data.

<span style="margin-left: 10px;"></span>
Good backend logging should be intentional, bounded, and production-safe. So only log what is necessary.

### Heap Usage Alone Is Not Enough

<span style="margin-left: 10px;"></span>
I've seen many developers look only at heap usage:
```
Heap used: 2.5 GB / 4 GB
```

<span style="margin-left: 10px;"></span>
That number is useful, but incomplete. For backend performance, we also need to understand:
- Allocation rate
- Young GC frequency
- Old generation growth
- Promotion rate
- GC pause duration
- Live object set size

<span style="margin-left: 10px;"></span>
A service with stable heap usage may still have bad GC behavior if it allocates too much temporary memory.

### What This Means for Java Backend Developers

<span style="margin-left: 10px;"></span>
A Java backend developer does not need to manually manage memory like in C or C++. But that does not mean memory is irrelevant. 
In Java, our responsibility is different:
- keep object lifetimes short when possible
- avoid unbounded data structures
- be careful with large responses
- understand framework allocation behavior
- monitor allocation rate and GC pauses
- investigate old-generation growth
- use evidence before tuning JVM flags

<span style="margin-left: 10px;"></span>
Backend engineers must know that performance is not only about algorithms or database indexes. 
It is also about runtime behavior!

## The JVM Heap Model for Backend Services

<span style="margin-left: 10px;"></span>
A Java backend developer does not need to manage memory manually, but they do need to understand where objects live, 
how long they live, and why that affects latency.
The JVM heap is not just “where objects go.” In production, it is the space where request traffic, framework behavior, 
caching, JSON processing, database access, and application design all meet.

<span style="margin-left: 10px;"></span>
The most important idea is this:
```
The heap is shaped by object lifetime.
```

<span style="margin-left: 10px;"></span>
Not all objects are equal. A request DTO that lives for 20 milliseconds is very different from a cache entry that lives for hours.

### Heap vs Stack

<span style="margin-left: 10px;"></span>
In Java, local variables and method calls are managed on thread stacks, while objects usually live on the heap. Example:
```java
public BeerResponse getBeer(Long id) {
    Beer beer = beerRepository.findById(id).orElseThrow();
    return beerMapper.toResponse(beer);
}
```

In simplified terms:

Stack:
- id
- beer reference
- return value reference

Heap:
- Long object, depending on boxing/caching
- Beer entity
- BeerResponse DTO
- Strings, collections, nested objects

<span style="margin-left: 10px;"></span>
The stack stores references and method execution state. The heap stores the actual objects.

<span style="margin-left: 10px;"></span>
When the method returns, stack variables disappear. But heap objects are only collectible if nothing still references them, and 
that distinction is critical. This object can be collected after the request:

```java
BeerResponse response = new BeerResponse(...);
```
<span style="margin-left: 10px;"></span>
Unless something keeps a reference to it:

```java
recentResponses.add(response);
```

<span style="margin-left: 10px;"></span>
The garbage collector does not care whether the request is finished. It only cares whether the object is still reachable.

### Reachability

<span style="margin-left: 10px;"></span>
The JVM collects objects that are no longer reachable from known starting points called GC roots. Common GC roots may include:
- active thread stacks
- static fields
- class metadata references
- running thread references

<span style="margin-left: 10px;"></span>
Example:

```java
private static final Map<Long, UserProfile> cache = new HashMap<>();
```

<span style="margin-left: 10px;"></span>
Anything inside this static map is reachable through a GC root. That means the GC cannot collect it, even if the data 
is no longer useful. This is why memory leaks in Java are not usually “lost memory” in the C/C++ sense. They are 
usually unwanted reachable objects. A Java memory leak means that the application still holds references to objects it no longer needs.

<span style="margin-left: 10px;"></span>
That is one of the most important distinctions for us backend developers.

### Young Generation

<span style="margin-left: 10px;"></span>
Most new objects are initially allocated in the young generation. The young generation commonly contains:
- Eden space
- Survivor space 1
- Survivor space 2

<span style="margin-left: 10px;"></span>
The usual flow is:
1. Eden
2. survives young GC
3. Survivor space
4. survives multiple collections
5. promoted to Old Generation

<span style="margin-left: 10px;"></span>
This is the ideal case. Short-lived request objects should be born, used, and collected quickly.

<span style="margin-left: 10px;"></span>
A young GC collects the young generation. Usually, it is efficient because most young objects are dead.

Before young GC:
- Eden: many request DTOs, JSON buffers, temporary lists

After young GC:
- dead objects removed
- surviving objects copied to Survivor or promoted

<span style="margin-left: 10px;"></span>
Young GCs are normally expected in Java backend applications. The problem is not that young GCs happen, it is when they 
happen too often or take too long. Frequent young GCs usually indicate high allocation rate. That often comes from:
- large responses
- excessive DTO mapping
- object-heavy transformations
- unnecessary intermediate collections
- verbose logging
- inefficient serialization
- high request volume
- large database result sets

### Survivor Spaces

<span style="margin-left: 10px;"></span>
Objects that survive a young GC may be moved into a Survivor space. Survivor spaces exist to avoid promoting objects 
to the old generation too quickly. It is something like this:
1. Request starts
2. Object allocated in Eden
3. Young GC happens while request is still running
4. Object is still reachable
5. Object moves to Survivor
6. Request finishes
7. Next young GC collects it

<span style="margin-left: 10px;"></span>
And this is normal!!!

<span style="margin-left: 10px;"></span>
A request object may survive one young GC simply because the request was still being processed at the time.  That does 
not mean it is a leak. But if many objects keep surviving multiple young GCs, the JVM may eventually promote them to the old generation.
That is where things become more expensive.

### Old Generation

<span style="margin-left: 10px;"></span>
The old generation stores objects that survived long enough to be considered long-lived. 
In backend services, old generation commonly contains:
- Spring beans
- application configuration
- caches
- connection pools
- thread pools
- loaded class structures and framework objects
- long-lived domain data
- retained Hibernate entities
- queued messages
- large collections
- session data

<span style="margin-left: 10px;"></span>
Old generation is not bad, the problem is unexpected old-generation growth. For example:

```java
private final List<BeerEvent> goodBeers = new ArrayList<>();
```

<span style="margin-left: 10px;"></span>
If this list grows without bounds, old generation usage may continuously increase.
The GC can run repeatedly and still fail to reclaim memory because those objects are still reachable.
That is not a garbage collector failure its object retention!

### Live Set

<span style="margin-left: 10px;"></span>
A very important production concept is the live set. The live set is the amount of memory still occupied after a full 
or major collection. For example:
- Heap before GC: 6 GB
- Heap after GC: 2 GB
- Live set: about 2 GB

<span style="margin-left: 10px;"></span>
The live set represents objects the application is actually retaining. If the live set keeps growing over time, you may have:
- a memory leak
- an unbounded cache
- growing queues
- large long-lived maps
- legitimate growth in business data

<span style="margin-left: 10px;"></span>
This is much more useful than looking only at total heap usage. A healthy service often has a pattern like this:
1. heap grows
2. GC runs
3. heap drops
4. heap grows
5. GC runs
6. heap drops

<span style="margin-left: 10px;"></span>
An unhealthy service may look like this:
- heap grows
- GC runs
- heap drops less than before
- heap grows again
- GC runs
- heap drops even less
- old generation keeps increasing

### Heap Size Lies

<span style="margin-left: 10px;"></span>
A common mistake is assuming that a bigger heap always means fewer problems, sometimes increasing heap may help.
For example, if a service has a bursty allocation pattern, a slightly larger heap may reduce GC frequency.
But bigger heap can also make things worse.

<span style="margin-left: 10px;"></span>
Larger heaps may:
- hide memory leaks for longer
- increase time spent scanning live objects
- delay failure instead of fixing it
- increase container memory cost
- make old-generation collections more expensive

<span style="margin-left: 10px;"></span>
For backend systems, heap sizing should be based on:
- live set size
- allocation rate
- latency requirements
- traffic profile
- container memory limits
- collector choice
- headroom for spikes

### Modern Collectors

<span style="margin-left: 10px;"></span>
When learning GC, diagrams often show this:
```
Heap
├── Young Generation
│    ├── Eden
│    ├── Survivor 0
│    └── Survivor 1
└── Old Generation
```

<span style="margin-left: 10px;"></span>
This model is useful, but modern collectors may implement it differently. For example, G1 divides the heap into regions. 
A region can be used as Eden, Survivor, or Old depending on current GC needs. Conceptually, the generational model still matters.
But physically, the heap may not be one large contiguous young area and one large contiguous old area.
The young/old model is still useful conceptually, but collectors like G1 implement the heap using regions rather than 
fixed contiguous generations.

### Backend Example

<span style="margin-left: 10px;"></span>
Considering this endpoint:

```java
@GetMapping("/beers/{id}")
public BeerResponse getBeers(@PathVariable Long id) {
    Beer beer = beerService.getBeer(id);
    return beerMapper.toResponse(beer);
}
```

<span style="margin-left: 10px;"></span>
A healthy memory profile might look like this:

Long-lived:
  - BeerService
  - BeerRepository
  - DataSource
  - BeerMapper
  - Spring MVC infrastructure

Short-lived:
  - request wrapper
  - path variable objects
  - Beer entity
  - BeerResponse DTO
  - JSON serialization buffer

<span style="margin-left: 10px;"></span>
After the response is sent, most request-specific objects should become unreachable. That is exactly what the JVM is good at.

<span style="margin-left: 10px;"></span>
Now consider this:

```java
@Component
public class ProductDebugStore {

    private final List<ProductResponse> responses = new ArrayList<>();

    public void save(ProductResponse response) {
        responses.add(response);
    }
}
```

<span style="margin-left: 10px;"></span>
And this endpoint:

```java
@GetMapping("/products/{id}")
public ProductResponse getProduct(@PathVariable Long id) {
    Product product = productService.getProduct(id);
    ProductResponse response = productMapper.toResponse(product);

    productDebugStore.save(response);

    return response;
}
```

<span style="margin-left: 10px;"></span>
Now every response is kept alive. The object is no longer request-scoped. It is reachable through a Spring singleton!
The GC cannot collect it. Eventually, these objects may move to the old generation and stay there.
This kind of issue often appears in production as:
- old generation keeps growing
- GC runs more often
- pause times increase
- eventually OutOfMemoryError or container restart

<span style="margin-left: 10px;"></span>
The root cause is not “bad GC.” Its accidental retention.

## GC and API Latency: p99 Over Average

<span style="margin-left: 10px;"></span>
In backend systems, garbage collection becomes important when it affects request latency.
A GC event is not just a memory-management detail. During certain phases of garbage collection, the JVM may pause 
application threads. When that happens, your service temporarily stops processing requests. That means GC can directly affect:
- HTTP response time
- p95 / p99 latency
- timeouts
- retries
- service-to-service communication

<span style="margin-left: 10px;"></span>
The key production idea is that GC problems usually appear as latency problems before they appear as memory problems.

### Average Latency Can Hide GC Issues

<span style="margin-left: 10px;"></span>
Many developers look at average latency first. For example the average latency is 25ms. That number may look healthy, 
but averages are often misleading in backend systems. A service can have a good average latency while still having 
serious tail-latency problems.

<span style="margin-left: 10px;"></span>
For example:
- Average latency: 25ms
- p95 latency: 80ms
- p99 latency: 750ms

<span style="margin-left: 10px;"></span>
This means most requests are fine, but the slowest 1% are very slow and for high-traffic services, that 1% matters.
If a service handles 2,000 requests per second, then 1% means 20 slow requests every second!

<span style="margin-left: 10px;"></span>
For backend APIs:
- p50  tells you the normal user experience
- p95  tells you about common slowness
- p99  tells you about production pain
- p999 tells you about rare but dangerous behavior

<span style="margin-left: 10px;"></span>
GC often appears in p99 or p999 because pauses may not happen constantly. They happen periodically, under pressure, or during traffic bursts.
That is why GC issues can stay hidden if you only monitor average latency.

### Stop-the-World Pauses

<span style="margin-left: 10px;"></span>
A stop-the-world pause means application threads are paused so the JVM can perform some GC work safely. During this pause:
- request threads stop running
- scheduled tasks stop running
- database calls may wait for application code to resume

<span style="margin-left: 10px;"></span>
From the outside, the service looks slow or frozen. For example:
1. Request starts
2. Service begins processing
3. GC pause happens for 300ms
4. Application resumes
5. Request completes

<span style="margin-left: 10px;"></span>
Even if the application logic only needed 20ms, the observed latency may become 20ms application work + 300ms GC pause = 320ms response time.
The user does not care that only 20ms was “real work.” The user sees a 320ms response.

<span style="margin-left: 10px;"></span>
A database query usually affects one request. A slow external API call usually affects one request.
A GC pause can affect many requests at the same time.
Suppose a given service that has 100 active request-processing threads. If a stop-the-world pause lasts 200ms, all those threads may be paused.
That means one GC event can delay many in-flight requests simultaneously. For example:
1. 100 requests in progress
2. 200ms stop-the-world pause
3. all 100 requests become at least 200ms slower

<span style="margin-left: 10px;"></span>
This is why GC has a strong effect on tail latency. The pause is not isolated to one unlucky request. It can hit the whole process.
And that why the problem gets worse under load.

<span style="margin-left: 10px;"></span>
While application threads are paused, new requests may continue arriving.
Those requests wait in queues and after the JVM resumes, the service may need to process both requests that were paused
and requests that arrived during the pause

<span style="margin-left: 10px;"></span>
This can create a temporary backlog. A 200ms GC pause may cause more than 200ms of user-visible latency if the service 
was already close to capacity. The pattern is something like this:
1. GC pause
2. request processing stops
3. queues grow
4. service resumes
5. backlog causes more latency
6. clients timeout or retry

<span style="margin-left: 10px;"></span>
And it gets worse in microservices, because latency does not stay local. Imagine Service A calls Service B.
If Service B has a GC pause, Service A may timeout. Then Service A retries:
1. Service B pauses
2. Service A times out
3. Service A retries
4. Service B receives even more requests

<span style="margin-left: 10px;"></span>
Now Service B has more load than before. More load means:
- more request objects
- more JSON parsing
- more DTO creation
- more logging
- more allocations
- more GC pressure

<span style="margin-left: 10px;"></span>
This creates a dangerous feedback loop:
1. GC pause
2. latency spike
3. timeout
4. retry
5. more traffic
6. higher allocation rate
7. more GC pressure
8. more pauses

<span style="margin-left: 10px;"></span>
This is how a local JVM memory issue can become a distributed-system incident.

### GC and Thread Pools

<span style="margin-left: 10px;"></span>
Java backend services usually depend on thread pools:
- Tomcat request threads
- database connection pools
- async executor pools
- scheduler pools
- Kafka listener threads

<span style="margin-left: 10px;"></span>
GC pauses interact badly with these pools.
During a pause, worker threads stop making progress. If requests keep arriving, thread pools and queues may fill up.
This is why GC should be analyzed together with thread pool metrics.
A p99 spike may not be caused only by “slow code.” It may be caused by a JVM pause that prevented otherwise healthy threads from running.

### Latency-Oriented vs Throughput-Oriented GC

<span style="margin-left: 10px;"></span>
Garbage collectors make trade-offs.
A throughput-oriented collector tries to maximize the amount of useful application work completed over time.
A latency-oriented collector tries to minimize pause duration, often by doing more work concurrently.

<span style="margin-left: 10px;"></span>
Backend services usually care about latency because they serve users or other services.
But that does not mean the lowest-pause collector is always the best choice.
There is a trade-off, lower pause times mean more concurrent GC work, more CPU overhead and potentially lower throughput.
For a backend service, the question is not which GC is fastest? The better question is which GC gives acceptable p99 
latency at acceptable CPU and memory cost for this workload? That is the production mindset in my point of view.

### Example: A GC Pause Hidden in p99

<span style="margin-left: 10px;"></span>
Imagine this service:
- Endpoint: GET /products/{id}
- Average latency: 18ms
- p95 latency: 45ms
- p99 latency: 480ms
- Error rate: low
- CPU: normal
- Database latency: normal

<span style="margin-left: 10px;"></span>
At first glance, nothing obvious is broken. Then you check GC logs and see something like this:

```
12:00:01.200 Young GC pause: 12ms
12:00:05.700 Young GC pause: 18ms
12:00:10.400 Young GC pause: 22ms
12:00:15.900 Mixed GC pause: 410ms
12:00:16.000 p99 latency spike observed
```

<span style="margin-left: 10px;"></span>
The application code did not suddenly become slower, the JVM paused request processing.
The important skill is not memorizing GC terminology. The important skill is connecting the runtime event to user-visible behavior.

### GC Pauses Can Break Health Checks

<span style="margin-left: 10px;"></span>
In containerized systems, GC pauses can also interfere with health checks.
If a service pauses long enough, Kubernetes or a load balancer may see failed readiness or liveness checks. 
Possible outcomes may be:
- service temporarily removed from load balancer
- pod restarted
- traffic shifted to other pods
- remaining pods receive more load
- more GC pressure on remaining pods

<span style="margin-left: 10px;"></span>
This can create a cascading failure.
A single service with unstable GC behavior can cause the platform to make the situation worse by restarting or rerouting traffic.
That does not mean health checks are bad. It means GC pauses need to be considered when setting timeouts and thresholds.

### Why Low Traffic Testing Misses GC Problems

<span style="margin-left: 10px;"></span>
GC behavior depends heavily on workload. Local testing usually has:
- low request volume
- small payloads
- short test duration
- low concurrency

<span style="margin-left: 10px;"></span>
Production has:
- high QPS
- large payloads
- long-running process
- real user traffic
- traffic bursts
- large caches
- background jobs
- database variability
- retries

<span style="margin-left: 10px;"></span>
A service can look perfect locally and still have GC-related p99 spikes in production.
This is why load testing matters. A realistic performance test should simulate:
- expected QPS
- concurrent users
- payload size distribution
- large responses
- database behavior
- cache warmup
- background jobs
- traffic bursts

<span style="margin-left: 10px;"></span>
GC tuning without realistic load is mostly guesswork.

### What to Monitor
<span style="margin-left: 10px;"></span>
For backend services, monitor GC together with application metrics. Useful JVM metrics:
- GC pause duration
- GC pause count
- young GC frequency
- old GC frequency
- heap used before/after GC
- old generation usage
- allocation rate
- promotion rate

<span style="margin-left: 10px;"></span>
Useful application metrics:
- request latency p95/p99
- request throughput
- error rate
- timeout rate
- retry rate
- thread pool usage
- connection pool usage
- CPU usage
- container memory usage

<span style="margin-left: 10px;"></span>
The goal is correlation. You may want to answer:
- Did p99 latency increase at the same time as GC pauses?
- Did old generation grow before the incident?
- Did allocation rate increase after a deployment?
- Did retries increase after GC pauses?
- Did thread pools saturate after the JVM resumed?

## Modern Collectors

<span style="margin-left: 10px;"></span>
I do not need to know every garbage collector in detail. What matters is knowing which collector fits which production problem.
The decision is usually about the trade-off: latency vs throughput vs CPU cost vs memory footprint

### G1 GC

<span style="margin-left: 10px;"></span>
For most modern Java backend services, G1 GC is the default choice.
Oracle’s Java 25 documentation states that G1 is selected by default on most hardware and operating system configurations. 
It also recommends starting with G1 defaults, optionally setting a pause-time goal and maximum heap size. G1 is a good fit for:
- Spring Boot APIs
- microservices
- medium-size heaps
- general backend workloads
- systems that need balanced latency and throughput

<span style="margin-left: 10px;"></span>
Why it works well:
- region-based heap
- incremental collection
- predictable pause-time goals
- good default behavior
- less manual tuning than older collectors

<span style="margin-left: 10px;"></span>
The production mindset is to start with G1 unless you have evidence that your latency requirements need something else.

### ZGC

<span style="margin-left: 10px;"></span>
ZGC is designed for low-latency applications where long GC pauses are unacceptable.
Modern ZGC is generational, it splits the heap into young and old generations so it can collect recently allocated 
objects separately from long-lived objects. ZGC is interesting for backend services with:
- large heaps
- strict p99/p999 latency targets
- real-time-ish APIs
- highly interactive systems
- services where GC pauses are visible to users

<span style="margin-left: 10px;"></span>
You might consider ZGC when:
- G1 pause times are too high
- old-gen collections affect p99 latency
- heap size is large
- the business cares more about predictable latency than maximum throughput

<span style="margin-left: 10px;"></span>
The trade-off is that ZGC reduces pause times by doing more work concurrently, but that work consumes CPU.
So ZGC is not automatically “better”. It may reduce latency while increasing CPU usage.

### Shenandoah

<span style="margin-left: 10px;"></span>
Shenandoah is also a low-pause collector. Its key idea is to reduce pause times by doing more garbage collection work 
concurrently with the running Java application, including concurrent compaction. It is useful for similar cases as ZGC.

<span style="margin-left: 10px;"></span>
Like ZGC, Shenandoah shifts more work out of stop-the-world pauses and into concurrent execution.
The trade-offs are also similar.

### How to Choose in Backend Systems

<span style="margin-left: 10px;"></span>
A practical decision model:
- Default service? Use G1
- Typical Spring Boot microservice? Use G1
- No clear GC problem? Stay with G1
- Strict p99/p999 latency target? Test ZGC or Shenandoah
- Large heap with painful pauses? Test ZGC or Shenandoah
- CPU-constrained environment? Be careful with concurrent collectors
- Batch job focused on total throughput? Do not blindly choose low-latency GC

<span style="margin-left: 10px;"></span>
The important point is that the collector choice should be driven by workload and latency requirements, not by hype.

## Java Backend Memory Leaks That Look Like GC Problems

<span style="margin-left: 10px;"></span>
Many production “GC problems” are not caused by the garbage collector. They are caused by the application keeping 
references to objects it no longer needs. The key idea is that GC cannot collect objects that are still reachable. 
A memory leak is often an object-retention problem, not a garbage-collector problem.
In Java, memory leaks usually do not happen because memory is “lost.” They happen because objects are still reachable 
from somewhere: a static field, a cache, a queue, a thread, a session, a listener, or a framework context.

### Unbounded Caches

<span style="margin-left: 10px;"></span>
Caching is one of the most common sources of Java backend leaks. For example:

```java
private final Map<String, UserProfile> cache = new HashMap<>();
```

<span style="margin-left: 10px;"></span>
If this map has no size limit, expiration policy, or eviction strategy, it can grow forever. This often happens with:
- user profiles
- authorization data
- product catalogs
- API responses
- tenant configuration
- lookup tables

<span style="margin-left: 10px;"></span>
A cache should usually have:
- maximum size
- time-based expiration
- metrics
- eviction policy

<span style="margin-left: 10px;"></span>
Using a library like Caffeine is usually better than building a cache with a raw HashMap.

### Static Collections
<span style="margin-left: 10px;"></span>
Static collections are dangerous because they live as long as the classloader lives. For example:

```java
import java.util.ArrayList;
import java.util.List;

public class DebugStore {
    private static final List<Object> EVENTS = new ArrayList<>();

    public static void add(Object event) {
        EVENTS.add(event);
    }
}
```

<span style="margin-left: 10px;"></span>
Every object added to EVENTS remains reachable. This is especially dangerous for:
- debug stores
- temporary registries
- in-memory audit logs
- test utilities accidentally used in production
- static maps of request data

<span style="margin-left: 10px;"></span>   
A static reference is effectively a global root. If it grows without bounds, GC cannot help.

### Forgotten Listeners and Callbacks

<span style="margin-left: 10px;"></span>
Listeners can leak memory when they are registered but never removed. For example:

```java
eventBus.register(listener);
```

<span style="margin-left: 10px;"></span>
If the event bus is long-lived and the listener references request, user, or component state, that state may remain alive. 
Common places are:
- event buses
- message listeners
- application lifecycle hooks
- WebSocket subscriptions
- observer patterns
- reactive streams
- scheduler callbacks

<span style="margin-left: 10px;"></span>
The fix is to unregister listeners when they are no longer needed:

```java
eventBus.unregister(listener);
```

<span style="margin-left: 10px;"></span>
This matters especially in long-running backend services where leaks accumulate slowly.

### ThreadLocal Misuse

<span style="margin-left: 10px;"></span>
ThreadLocal is useful, but dangerous in backend applications because request threads are reused. For example:

```java
private static final ThreadLocal<RequestContext> context = new ThreadLocal<>();

public void handle(RequestContext requestContext) {
    context.set(requestContext);
}
```
<span style="margin-left: 10px;"></span>
If you do not call remove(), the request context may stay attached to the thread after the request finishes.
In servlet containers, executor pools, and async processing, those threads may live for the lifetime of the application. 
The correct pattern is:

```java
try {
    context.set(requestContext);
    process();
} finally {
    context.remove();
}
```

<span style="margin-left: 10px;"></span>
Leaks through ThreadLocal are common with:
- security context
- tenant context
- correlation IDs
- request metadata
- large user/session objects
- custom tracing data

<span style="margin-left: 10px;"></span>
The rule is simple, if you set a ThreadLocal in request processing, remove it in a finally block or try-with-resources.

### Classloader Leaks

<span style="margin-left: 10px;"></span>
Classloader leaks usually appear in application servers, plugin systems, hot reload environments, or repeated redeployments. 
A classloader can be retained by:
- static fields
- running threads
- ThreadLocals
- JDBC drivers
- logging frameworks
- custom registries

<span style="margin-left: 10px;"></span>
The result is that old application classes and objects cannot be collected after redeployment.
In modern Spring Boot applications packaged as containers, this is less common than in traditional app servers, but it still matters in:
- Tomcat deployments
- application servers

### Large Session Objects

<span style="margin-left: 10px;"></span>
Sessions can retain much more memory than expected. Example:

```java
session.setAttribute("cart", largeCartObject);
session.setAttribute("userProfile", fullUserProfile);
session.setAttribute("lastSearchResults", hugeResultList);
```
<span style="margin-left: 10px;"></span>
This becomes dangerous when:
- sessions are long-lived
- many users are active
- objects stored in session are large
- session replication is enabled
- old session data is not removed

<span style="margin-left: 10px;"></span>
A common mistake is storing full domain objects or large result sets in session instead of storing small identifiers. Prefer:
- user ID instead of full user object
- cart ID instead of full cart graph
- pagination token instead of full search result list

<span style="margin-left: 10px;"></span>
Sessions should be small, intentional, and time-bounded.

### ORM Persistence Context Growth

<span style="margin-left: 10px;"></span>
Hibernate/JPA can retain entities longer than expected.
Inside a transaction, the persistence context tracks managed entities for dirty checking and identity management.
This is fine for small operations:

```java
Beer beer = entityManager.find(Beer.class, id);
```

<span style="margin-left: 10px;"></span>
But it becomes dangerous in large operations:

```java
List<Beer> beers = beerRepository.findAll();

for (Beer beer : beers) {
    process(beer);
}
```

<span style="margin-left: 10px;"></span>
The persistence context may retain many entities until the transaction ends.
In batch jobs, this can cause old-generation pressure or OutOfMemoryError. Safer approaches may include:
- pagination
- streaming carefully
- smaller transactions
- read-only queries
- clearing the persistence context
- avoiding huge findAll operations

<span style="margin-left: 10px;"></span>
Example in batch processing:

```java
entityManager.flush();
entityManager.clear();
```

### Queues Growing Faster Than Consumers

<span style="margin-left: 10px;"></span>
Queues are another common backend leak pattern. Example:

```java
private final BlockingQueue<Event> queue = new LinkedBlockingQueue<>();
```

<span style="margin-left: 10px;"></span>
If the queue is unbounded and producers are faster than consumers, memory usage grows continuously. This happens with:
- async event processing
- Kafka/RabbitMQ consumers
- email jobs
- audit logging
- background task queues
- retry queues

<span style="margin-left: 10px;"></span>
Symptoms:
- heap grows during traffic spikes
- old generation increases
- latency gets worse
- consumer lag increases

## Spring Boot, Hibernate, Jackson, and GC Pressure

<span style="margin-left: 10px;"></span>
In real Java backend systems, GC pressure rarely comes from one obvious new statement.
It usually comes from the ecosystem around the request:
- Spring MVC
- Jackson
- Hibernate/JPA
- Bean Validation
- logging
- DTO mappers
- AOP proxies
- reflection
- metrics/tracing

<span style="margin-left: 10px;"></span>
These tools are productive and valuable, but they also create objects. At production traffic levels, framework allocation 
becomes part of your performance profile.
The point is not to avoid frameworks, its to understand their runtime cost.

###DTO Mapping Overhead

<span style="margin-left: 10px;"></span>
Most backend services use DTOs to separate API models from domain models. For example:

```java
public BeerResponse toResponse(Beer beer) {
    return new BeerResponse(
            beer.getId(),
            beer.getStatus().name(),
            beer.getCustomer().getName(), 
            beer.getBreweries().stream()
            .map(this::toBreweryResponse)
            .toList()
    );
}
```

<span style="margin-left: 10px;"></span>
This is clean design, as is MapStruct, but it allocates:
- BeerResponse
- BreweryResponse objects
- lists
- strings
- stream pipeline objects
- temporary mapping structures

<span style="margin-left: 10px;"></span>
DTO mapping becomes expensive when:
- responses are large
- mapping happens in hot endpoints
- nested object graphs are deep
- multiple mapping layers exist
- intermediate collections are created

<span style="margin-left: 10px;"></span>
This does not mean DTOs are bad. It means large mappings should be intentional, measured, and paginated where possible.

### Jackson Object Creation

<span style="margin-left: 10px;"></span>
JSON serialization and deserialization are major allocation sources in REST APIs.
For inbound requests, Jackson may allocate:
- request DTOs
- nested objects
- collections
- strings
- parser buffers
- temporary metadata structures

<span style="margin-left: 10px;"></span>
For outbound responses, it may allocate:
- serialization buffers
- field name strings
- nested DTO traversal structures
- temporary byte/char arrays

<span style="margin-left: 10px;"></span>
Example:
```java
@PostMapping("/search")
public List<BeerResponse> search(@RequestBody SearchRequest request) {
    return beerService.search(request);
}
```

<span style="margin-left: 10px;"></span>
If this endpoint returns thousands of beers, the cost is not only database time. It is also heap pressure from the 
returned object graph and serialization process. 
Use pagination, streaming responses when appropriate, response-size limits, and avoid returning unnecessary fields.

### Hibernate Persistence Context

<span style="margin-left: 10px;"></span>
Hibernate does not just return objects from the database it also manages them.
Inside a transaction, Hibernate keeps entities in the persistence context for identity tracking and dirty checking.
For example:

```java
@Transactional
public void processBeers() {
List<Beer> beers = beerRepository.findAll();

    for (Beer beer : beers) {
        process(beer);
    }
}
```

<span style="margin-left: 10px;"></span>
This can retain a large number of entities until the transaction ends.
The persistence context may hold:
- entity instances
- entity snapshots
- proxy objects
- collections
- dirty-checking data
- relationship graphs

<span style="margin-left: 10px;"></span>
For small requests, this is fine. For large reads or batch operations, it can create old-generation pressure.
Safer patterns:
- pagination
- read-only transactions
- smaller transaction boundaries
- DTO projections
- clearing the persistence context in batches
- avoiding large findAll operations

<span style="margin-left: 10px;"></span>
Example:

```java
entityManager.flush();
entityManager.clear();
```

<span style="margin-left: 10px;"></span>
The key idea is that a large Hibernate session can keep objects alive longer than your code suggests.

### N+1 Queries Can Create Excessive Entities

<span style="margin-left: 10px;"></span>
N+1 queries are usually discussed as a database performance problem, but they are also a memory problem. Example:

```java
List<Beer> beers = beerRepository.findAll();

for (Beer beer : beers) {
    beer.getBreweries().size(); // may trigger lazy loading
}
```

<span style="margin-left: 10px;"></span>
This can create:
- many SQL queries
- many entity objects
- many Hibernate proxies
- many collections
- large object graphs
- extra persistence context entries

<span style="margin-left: 10px;"></span>
So the cost is not only too many database round trips it is also too many Java objects allocated and retained.
Fixes may include:
- fetch joins
- entity graphs
- DTO projections
- batch fetching
- pagination
- query-specific data loading

<span style="margin-left: 10px;"></span>
A backend engineer should understand both sides: database impact and heap impact.

### Large Result Sets

<span style="margin-left: 10px;"></span>
This is one of the simplest ways to create GC pressure. Example:

```java
@GetMapping("/beers")
public List<BeerResponse> getBeers() {
    return beerRepository.findAll()
        .stream()
        .map(beerMapper::toResponse)
        .toList();
}
```

<span style="margin-left: 10px;"></span>
This may allocate:
- all beer entities
- related entities
- DTOs
- lists
- strings
- JSON serialization buffers

<span style="margin-left: 10px;"></span>
Large result sets are dangerous because they create memory pressure across multiple layers:
- database driver
- Hibernate
- DTO mapper
- Jackson
- HTTP response buffer

<span style="margin-left: 10px;"></span>
A production-safe API should usually use:
- pagination
- limits
- filters
- cursor-based pagination
- streaming for specific use cases

### Logging Full Payloads

<span style="margin-left: 10px;"></span>
Logging is often underestimated. Example:

```java
log.info("Request: {}", request);
log.info("Response: {}", response);
```

<span style="margin-left: 10px;"></span>
This can allocate large strings, especially when DTOs have generated toString() methods that include nested objects.
Problems with full-payload logging:
- large string allocations
- sensitive data exposure
- slower request processing
- larger log volume
- more pressure on logging appenders
- possible async logging queue growth

<span style="margin-left: 10px;"></span>
This is worse when logging happens on every request.
Prefer:
- request ID
- user/tenant ID
- endpoint
- status code
- duration
- small business identifiers
- error code
- payload size

<span style="margin-left: 10px;"></span>
Instead of logging the whole object graph, log the useful identifiers.

### Validation Frameworks

<span style="margin-left: 10px;"></span>
Bean Validation is convenient and common in Spring Boot APIs. Example:

```java
@PostMapping("/beers")
public BeerResponse create(@Valid @RequestBody CreatebeerRequest request) {
    return beerService.create(request);
}
```

<span style="margin-left: 10px;"></span>
Validation may allocate:
- constraint violation objects
- property paths
- message templates
- interpolated messages
- metadata lookups
- temporary collections

<span style="margin-left: 10px;"></span>
Usually this is fine, it becomes more relevant when:
- payloads are large
- nested validation is deep
- many invalid requests arrive
- validation runs in hot paths
- custom validators allocate heavily
- error responses include many details

<span style="margin-left: 10px;"></span>
Validation should be clear and useful, but avoid creating huge validation responses for massive payloads.

### Reflection and Proxy-Heavy Frameworks

<span style="margin-left: 10px;"></span>
Spring Boot applications use reflection, dynamic proxies, annotations, and generated infrastructure heavily. Examples:
- Spring AOP proxies
- @Transactional proxies
- security proxies
- repository proxies
- reflection-based binding
- annotation scanning
- method interceptors
- metrics/tracing instrumentation

<span style="margin-left: 10px;"></span>
Most of this cost is acceptable and often paid during startup or cached internally. But in request paths, proxies and 
interceptors can still add allocation and call overhead. This matters especially in:
- very high-QPS endpoints
- deep service call chains
- heavy AOP usage
- reflection-heavy mappers
- dynamic serialization/deserialization
- excessive instrumentation

<span style="margin-left: 10px;"></span>
Again, the point is not to avoid Spring. The point is to know that abstractions are not free.

### How This Connects to GC

<span style="margin-left: 10px;"></span>
A typical Spring Boot request may look like this:

- Spring MVC objects
- Jackson request DTOs
- validation objects
- service-layer allocations
- Hibernate entities/proxies
- DTO mapping
- Jackson response serialization
- logging/tracing/metrics
- GC later cleans temporary objects

<span style="margin-left: 10px;"></span>
This is why GC behavior is connected to everyday backend code.
If traffic increases, all of these allocations increase too.
If payloads get larger, object graphs get larger.
If Hibernate loads too much data, old-generation pressure increases.
If logs include full payloads, string allocation increases.
If queues or sessions retain these objects, they stop being temporary.

## Conclusion
<span style="margin-left: 10px;"></span>
Garbage collection issues in Java backend systems are rarely just “GC problems.” More often, they are symptoms of how 
the application allocates, retains, and processes objects under real production load.

<span style="margin-left: 10px;"></span>
For backend engineers, the important skill is not memorizing every GC algorithm. It is understanding the connection 
between application code and runtime behavior: request allocation patterns, object lifetime, old-generation growth, p99 latency, 
container memory limits, and observability data.
A strong Java developer knows that GC tuning should come after evidence. Before changing JVM flags, look at allocation 
rate, GC logs, heap usage after collection, live set growth, thread pools, database pools, and request latency.

<span style="margin-left: 10px;"></span>
The core lesson is simple, GC performance starts in application design, not in JVM flags.
If most request-scoped objects die young, caches and queues are bounded, ThreadLocals are cleaned up, large payloads 
are controlled, and GC metrics are correlated with production behavior, the JVM can do its job efficiently :)




