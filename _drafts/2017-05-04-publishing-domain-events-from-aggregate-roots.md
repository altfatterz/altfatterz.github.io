---
layout: post
title: Publishing domain events from aggregate roots
tags: [springdata, spring]
---

Starting with Spring Data Ingalls release publishing domain events by aggregate roots becomes easier. Instead of leveraging Spring's `ApplicationEventPublisher` you can use `@DomainEvents` annotation on a method of your aggregate root.
Let's look at an example.

```java
class BankTransfer {
    
    @DomainEvents 
    Collection<Object> domainEvents() {
            // … return events you want to get published here
    }
    
    @AfterDomainEventsPublication 
    void callbackMethod() {
           // … potentially clean up domain events list
    }
}
```

The method annotated with `@DomainEvents` will be called if one of the `save()` methods of a Spring Data repository is called. The method can return a single event instance or a collection of events and cannot have any arguments. After all events have been published a method annotated with `@AfterDomainEventsPublication` is called.

Spring Data Commons provides a convenient base class (`AbstractAggregateRoot`) to help to register domain events and is using the publication mechanism implied by `@DomainEvents` and `@AfterDomainEventsPublication`   
  
```java
public class AbstractAggregateRoot {

	/**
	 * All domain events currently captured by the aggregate.
	 */
	@Getter(onMethod = @__(@DomainEvents)) //
	private transient final List<Object> domainEvents = new ArrayList<Object>();

	/**
	 * Registers the given event object for publication on a call to a Spring Data repository's save method.
	 * 
	 * @param event must not be {@literal null}.
	 * @return
	 */
	protected <T> T registerEvent(T event) {
		Assert.notNull(event, "Domain event must not be null!");
		this.domainEvents.add(event);
		return event;
	}

	/**
	 * Clears all domain events currently held. Usually invoked by the infrastructure in place in Spring Data
	 * repositories.
	 */
	@AfterDomainEventPublication
	public void clearDomainEvents() {
		this.domainEvents.clear();
	}
}
```
  
Let's modify the example to extend from `AbstractAggregateRoot` base class.
  
```java
public class BankTransfer extends AbstractAggregateRoot {

   ...

    public BankTransfer complete() {
        id = UUID.randomUUID().toString();
        registerEvent(new BankTransferCompletedEvent(id));
        return this;
    }
    
    ...
}
```

In the example The `BankTransfer` aggregate root publishes a `BankTransferCompletedEvent` when its `complete` method is called.
The client calls this method in a transactional context saving the BankTransfer also via Spring Data Repository abstraction, which triggers the publication of `BankTransferCompletedEvent` event.

```java
@Service
public class BankTransferService {

    ...
    
    @Transactional
    public String completeTransfer(BankTransfer bankTransfer) {
        return repository.save(bankTransfer.complete()).getId();
    }

    ...
}
```

On the event listener side by using `TransactionalEventListener` the event handler can be bound to a phase of the transaction which published the event. The typical use case is to handle the event when the transaction completed successfully, which is the default setting for `@TransactionalEventListener`. It can be further customized via the `phase` attribute. 

```java
@Service
public class BankTransferProcessor {

    ...
    
    @Async
    @TransactionalEventListener
    public void handleBankTransferCompletedEvent(BankTransferCompletedEvent event) {
        BankTransfer bankTransfer = repository.findById(event.getBankTransferId());
        bankTransfer.markStarted();
        bankTransfer = repository.save(bankTransfer);

        log.info("Starting to process bank transfer {}.", bankTransfer);

        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }

        if (new Random().nextBoolean()) {
            bankTransfer.markCompleted();
        } else {
            bankTransfer.markFailed();
        }

        repository.save(bankTransfer);

        log.info("Finished processing bank transfer {}.", bankTransfer);
    }

}
``` 

In the example the `@Async` is used to return immediately the service call.   
  
  
```bash
$ echo 
      '{
            "from":"DE89 3704 0044 0532 0130 00",
            "to":"HU42 1177 3016 1111 1018 0000 0000",
            "amount":100.25
      }' | \
      http post :8080/bank-transfers
     
HTTP/1.1 201
Content-Type: application/json;charset=UTF-8
Date: Thu, 04 May 2017 13:54:12 GMT      
{
    "bankTransferId": "783b9b13-8424-4004-b59f-eef400d8a52c"
}      
```

```bash
$ http :8080/bank-transfers/783b9b13-8424-4004-b59f-eef400d8a52c

HTTP/1.1 200
Content-Type: application/json;charset=UTF-8
Date: Thu, 04 May 2017 13:55:33 GMT
Transfer-Encoding: chunked

{
    "bankTransferId": "783b9b13-8424-4004-b59f-eef400d8a52c",
    "from": "DE89 3704 0044 0532 0130 00",
    "to": "HU42 1177 3016 1111 1018 0000 0000",  
    "amount": 100.25,
    "status": "COMPLETED"
}
```



