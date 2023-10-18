# Dividing a Legacy ERP for Agile Cloud Delivery with Microservices and MongoDB

As the lead architect at [Hybrid Web Agency](https://hybridwebagency.com/), one of the most challenging projects I've worked on was migrating a legacy monolithic ERP system into a modern microservices architecture. The outdated monolith had become expensive to maintain and was holding my client's business back from innovation.

Luckily, I've seen this problem countless times before in our years of providing custom [Software Development Services in Anaheim](https://hybridwebagency.com/anaheim-ca/best-software-development-company/). I know the struggles that organizations face when burdened by rigid and difficult-to-change systems. That's why I'm excited to share the step-by-step approach we took to refactor the data model and decompose the monolith at both the code and database layers.

During the migration, my team and I discovered some lesser known best practices for modeling domain entities in a document database like MongoDB to support fully independent microservices. If you've ever wondered how to future-proof your database architecture for the cloud while preserving historical data, you won't want to miss these strategies.

By the end of this article, you'll walk away with a clear blueprint for how to migrate your own legacy systems to a modern architecture. I'm going to give away tons of practical insights so you can avoid common pitfalls and deliver new features to your customers faster. Let's get started!


## The Benefits of a Microservices Architecture

Implementing a microservices architecture provides numerous advantages over a monolithic approach. Services are independently deployable, allowing features to be developed and released quickly without disrupting the whole application.  

Individual services can also be developed using different programming languages and frameworks, enabling best of breed technologies to be selected for each domain. For example, a recommendation engine could leverage machine learning libraries in Python while a user interface is built in React.

This decoupling of concerns facilitates specialized teams to work autonomously on separate services. Ideas can be validated rapidly through prototypes before full investment in a monolithic rewrite. New hires can easily be productive contributing to a single service relevant to their skills.

Technologies emerge and fade frequently. Microservices mitigate risk from these shifts since replacements involve only small, isolated pieces of the system rather than a rewrite of the entire monolith. Descriptive interfaces make migrating components to new implementations seamless.

#### Scaling Independently

Management of compute resources is optimized when services scale autonomously in response to demand. The frontend API gateway routes traffic based on URL yet can securely deploy behind a load balancer to handle peaks. When orders spike on holidays, only order processing needs more servers - no affecting the whole system.

```
ordersAPI.scale(replicas: 5);
```

Horizontal scaling at the service layer yields significant cost savings from granular right-sizing of resources. Idle support microservices like user profiles don't require costly overprovisioning to handle traffic that doesn't impact them.
   
## Analyzing the Data Model
### Understanding Entity Relationships
The first step in migrating to microservices is analyzing how entities relate within the monolithic data model. We examined each collection in the MongoDB database to identify clustered domains and transaction boundaries.

Common entities like Users, Products and Orders formed the nucleus of bounded contexts. Relationships between these core objects served as candidates for service decomposition. For example, we noticed Orders held foreign keys to Users for the customer and Products to represent items purchased.

To better understand cross-dependencies, example documents were printed to visualize associated fields. We noticed legacy code merged data that now pertained to separate business capabilities. For instance, shipping addresses repeated user profiles rather than storing a lightweight reference.

Analyzing relationships uncovered questionable tight coupling between modules that caused rippling updates. Normalizing redundant data removed impediments to independent development of user profiles versus shipping namespaces.

Database tools helped explore the network of connections. In MongoDB Compass, we diagrammed relations through $lookup pipelines and ran aggregate queries tallying references between entities. This exposed prime breakpoints for dividing logic into coherent services.

Relations informed domain boundaries and ensured services exposed clean interfaces. Well-defined contracts enable autonomous teams to develop and deploy modules incrementally as micro Frontends without blocking each other.

  
### Identifying Transactional Boundaries
Beyond relationships, we also examined transactions in the existing codebase to understand business process flows. This helped identify where data modifications needed to be contained within single services to maintain data consistency and integrity.

For order processing, we noticed any updates to the order itself, associated payments, inventory levels, and shipment notifications needed to occur transactionally within a single service. This informed the definition of our Order Management service boundary.

Analyzing both relationships and transactions provided critical insights for refactoring the data model and logic into independently deployable microservices with clearly defined interfaces.


## Refactoring for Microservices

### Normalizing Data Schemas

To support independent services deploying to different data stores if needed, we normalized schemas to remove duplication and only include the minimal data required by each service.

For example, the original Orders schema contained the full User object. We refactored this to a lightweight reference:

```
// Before
Orders: {
  user: {
    name: 'John',
    address: '123 Main St...' 
  }
  //...
}

// After  
Orders: {
  userId: 1234
  //...  
}
```

Similarly, product details were removed from Orders into their own collections to allow these entities to evolve independently over time.

### Applying Domain-Driven Design

Using bounded contexts from Domain-Driven Design helped logically separate services such as Order Fulfillment versus User Profiles. Interfaces abstracted data access:

```
interface UserRepository {
  getUser(id): User;
  updateProfile(user: User): void;
}

class MongoUserRepository implements UserRepository {

  users = db.collection('users');

  async getUser(id) {
    return this.users.findOne({_id: id}); 
  }

  //...
}
```

### Evolving Data Access Patterns

Queries and commands also needed refactoring to support the new architecture. For example, services previously accessed data directly via calls like `db.collection.find()`. We introduced abstraction via data access libraries:

```
// order.service.ts

constructor(private ordersRepo: OrderRepository) {}

getOrders() {
  return this.ordersRepo.find();
}

// mongodb.repo.ts

@Injectable()
class MongoOrderRepository implements OrderRepository {

  constructor(private mongoClient: MongoClient) {}

  find() {
   return this.mongoClient.db.collection('orders').find(); 
  }

}
```

This ensured flexibility to migrate databases without changing consumer code.

## Deploying Microservices

### Scaling Independently

With microservices, autoscaling is done at the individual service level rather than application-wide. We implemented scaling logic utilizing Docker Swarm and Kubernetes.

Deployment manifests defined scaling policies based on CPU and memory usage:

```yaml
# docker-compose.yml

services:
  orders:
    image: orders
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
      restart_policy: any
      placement:
        constraints: [node.role == worker]  
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      rollback_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      scaler:
        min_replicas: 2    
        max_replicas: 6
```

Scaling tiers provided reserve capacity and elastic overflow handling. The orders service monitored itself and spawned/terminated containers as needed:

```js
// orders.js

const cpusUsed = process.cpuUsage();

if(cpusUsed > 80) {
  swarm.scaleService('orders', 1);
}

if(cpusUsed < 20) {
  swarm.scaleService('orders', -1);  
}
```

Load balancers like Nginx and Traefik routed to scaled replica sets. This achieved optimal resource utilization, improving throughput and latency while reducing costs.

### Implementing Resiliency

Other resiliency techniques included retry policies, timeouts, and circuit breakers. Rate limiting and throttling prevented cascading failures. The Platform service owned transience policy for dependent services.

Homemade and open-source solutions like Polly, Hystrix, and Resilience4j provided protections. Centralized logging via Elasticsearch helped trace errors across the distributed applications.

  
### Implementing Resiliency

Building reliability into microservices involved implementing various techniques to prevent single points of failure. We focused on automated responses to transient errors and overload protection.

Using the Resilience4J library provided circuit breakers to gracefully handle faults:

```java
// OrdersService.java

@Slf4j
@Service
public class OrdersService {

  @CircuitBreaker(name="ordersService", fallbackMethod="fallback")
  public List<Order> getOrders() {
    return orderRepository.getOrders();
  }

  public List<Order> fallback(Exception e) {
    log.error("Circuit open, returning empty orders");
    return Collections.emptyList();
  }
}
```

Rate limiting prevented flooding services under stress:

```java 
// RateLimiterConfig.java

@Configuration
public class RateLimiterConfig {

  @Bean
  public RateLimiter ordersLimiter() {
    return RateLimiter.create(maxOrdersPerSecond); 
  }

}
```

Timeouts aborted long running calls:

```java
// ProductClient.java

@HystrixCommand(fallbackMethod="getProductFallback",
               commandProperties={
                 @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds", value="1000")  
               })
public Product getProduct(Long id) {
  return productService.getProduct(id);
}
```

Implemented retry logic via policies defined at the client:

```java
// RetryConfig.java

@Configuration
public class RetryConfig {

  @Bean
  public RetryTemplate ordersRetryTemplate() {
    SimpleRetryPolicy policy = new SimpleRetryPolicy();
    policy.setMaxAttempts(3);
    return new RetryTemplate(policy);
  }

} 
```

These techniques ensured consistent responses and prevented cascading failures across services.


## Conclusion

Migrating our monolithic ERP system to microservices has been a tremendous learning experience. More than just a technical migration, it represented an organizational transformation that will allow our client to far better serve their customers' evolving needs.

By breaking apart tightly coupled layers and establishing clear boundaries between domains, our development team unlocked a new level of agility. Features can now be developed and deployed independently according to business priority rather than the architecture. This ability to rapidly experiment and refine will keep the application ahead of changing market demands.

At the same time, our operations team now has full visibility and control over each system component. Anomalous behavior can be caught early through improved monitoring of discrete services. Scaling and failovers are automated rather than manual toasts. This increased resilience will provide a reliable foundation for the client's continued growth.

While the benefits of microservices are clear, such a migration is not trivial. By taking the time to properly analyze relationships, define interfaces, and introduce abstraction - rather than a crude 'rip and replace' - we created a flexible architecture that can continue evolving along with customer needs.

Most of all, I am grateful for the opportunity this project provided to partner so closely with a client on their digital transformation. Demystifying and sharing our learnings here is my way of paying it forward, so others may benefit from both our successes and mistakes along the way. I hope it empowers more businesses to take the plunge into modernization - the rewards for customers will be well worth it.

## References 

- MongoDB Documentation - Official docs on data modeling, queries, deployment, etc. https://docs.mongodb.com/

- Microservices Pattern - Martin Fowler's seminal overview of microservices architecture principles. https://martinfowler.com/articles/microservices.html 

- Domain-Driven Design - Eric Evans book that introduced DDD concepts for structuring services around business domains. https://domainlanguage.com/ddd/

- Twelve-Factor App Methodology - Best practices for building software-as-a-service apps that can be readily deployed to the cloud. https://12factor.net/

- Container Journal - In-depth articles on containerization, orchestration, cloud platforms. https://containerjournal.com/

- Docker Documentation - Comprehensive guides for building, deploying and managing containerized apps. https://docs.docker.com/

- Kubernetes Documentation - Kubernetes, the leading container orchestrator, and its architecture. https://kubernetes.io/docs/home/ 
