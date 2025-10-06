---
date: 2025-10-05
authors:
  - David Sarmiento Patrón
title: Introduction to Spring Modulith
tags:
  - spring-modulith
---
# Introduction

In this article, I would like to introduce to Spring Modulith. And I will resolve some questions such as What is it? Why I have to use?. 
The Idea of the monolith start with:

**Martin Fowler (2015)** - "Monolith First":
> "**you shouldn't start a new project with microservices, even if you're sure your application will be big enough to make it worthwhile.** ." [ˆ1]

**Sam Newman** - "Building Microservices":
> "Start with a monolith, extract microservices"

Also I show a simple demo how to start.

## 1. What is Spring Modulith
Spring Modulith is a library that allows create modular monoliths. Where each module is a domain of the application.
Internally the monolithic system is structured to be easy to maintain, test and if required allows division into microservices when necessary.

Spring Modulith offers the next features 
### Features
- Encourage to create modules with bounded context
- Supports event-driven communication (sync and async)
- Generation of documentation as UML diagrams like components
- Transactional Events: Spring Modulith introduces a way of publish events only when the transaction has been completed successfully. Each event is registered in the DB allowing subsequent retries if it fails.
- Architectural Validation
## 2. Why I should use Spring Modulith
Primarily because it creates a modular monolith with a well-defined structure that can be easily decomposed into microservices when needed.


## 3. Annotations more used
Spring Modulith provides usa package of annotations such as:

**@ApplicationModule**
- Marks a package/class as an application module.
- Defines explicit module boundaries.

**@ApplicationModuleListener**
 - Declares an internal event listener between modules.
- Similar to Spring's @EventListener, but focused on intra-module communication.

**@NamedInterface**
- Defines an interface within a module that can be accessed externally.
- Useful for controlling what is exposed and what remains internal.

**@ApplicationModuleTest**
- For testing focused on a specific module, isolating its dependencies.


## 4. Demo
In this example we work with two modules Order and Inventory:
- When an order is created, the Inventory module should update the stock
- The communication between both modules is based on Events
- Each module has Internal classes and some exposed classes


[link Spring Initializr](
https://start.spring.io/#!type=gradle-kotlin-project&language=java&java-version=21&bootVersion=3.5.6&baseDir=spring-modulith-app&groupId=dev.davidsarmiento&artifactId=example-modulith&name=Example%20Spring%20Modulith&description=Spring%20Modulith%20Tutorial&packageName=dev.davidsarmiento.modulith&dependencies=web,data-jpa,lombok,h2,modulith,validation)
![](Pasted%20image%2020250922220422.png#center)
We have the following project structure for the Order module. This structure use the default package/module configuration, where each sub-package under the main package is considered an application module.
```java
└── src
    ├── main
    │   ├── java
    │   │   └── dev
    │   │       └── davidsarmiento
    │   │           └── modulith
    │   │               ├── ExampleSpringModulithApplication.java
    │   │               └── order
    │   │                   ├── controller
    │   │                   │   └── OrderController.java
    │   │                   ├── dto
    │   │                   │   ├── CreateOrderRequest.java
    │   │                   │   ├── OrderItemDTO.java
    │   │                   │   └── package-info.java
    │   │                   ├── event
    │   │                   │   ├── OrderCreatedEvent.java
    │   │                   │   └── package-info.java
    │   │                   ├── model
    │   │                   │   ├── Order.java
    │   │                   │   ├── OrderItem.java
    │   │                   │   └── OrderStatus.java
    │   │                   ├── repository
    │   │                   │   ├── OrderItemRepository.java
    │   │                   │   └── OrderRepository.java
    │   │                   └── service
    │   │                       ├── OrderService.java
    │   │                       └── OrderServiceImpl.java
```

The most relevant part of the module is:
- The use of some annotations such as 
in package dto package-info.java :
```java
@org.springframework.modulith.NamedInterface("dto")  
package dev.davidsarmiento.modulith.order.dto;
```
in package event -> package-info.java:
```java
@org.springframework.modulith.NamedInterface("event")  
package dev.davidsarmiento.modulith.order.event;
```
In both cases we use NamedInterface to expose that package classes to other modules, in this case we will share these to Inventory module.

- Publishing of Events  used in createOrder() method.
```java
public class OrderServiceImpl implements OrderService {  
  
    private final OrderRepository orderRepository;  
    private final ApplicationEventPublisher eventPublisher;
    
@Override  
@Transactional  
public Order createOrder(CreateOrderRequest request) {  
    Order order = new Order();  
    order.setOrderDate(LocalDateTime.now());  
    order.setStatus(OrderStatus.COMPLETED);  
  
    List<OrderItem> orderItems = request.items().stream()  
            .map(itemDTO -> toOrderItem(itemDTO, order))  
            .collect(Collectors.toList());  
    order.setItems(orderItems);  
  
    Order savedOrder = orderRepository.save(order);  
  
    eventPublisher.publishEvent(new OrderCreatedEvent(this, request.items()));  
  
    return savedOrder;  
}
```
We use an instance of ApplicationEventPublisher to publish a Event for our following module called Inventory.

Inventory module has the responsibility of keep updated the stock of Product.

```java
└── src
    ├── main
    │   ├── java
    │   │   └── dev
    │   │       └── davidsarmiento
    │   │           └── modulith
    │   │               ├── ExampleSpringModulithApplication.java
    │   │               ├── inventory
    │   │               │   ├── controller
    │   │               │   │   └── InventoryController.java
    │   │               │   ├── dto
    │   │               │   │   └── ProductInventoryDTO.java
    │   │               │   ├── event
    │   │               │   │   └── OrderEventListener.java
    │   │               │   ├── model
    │   │               │   │   └── Product.java
    │   │               │   ├── package-info.java
    │   │               │   ├── repository
    │   │               │   │   └── ProductRepository.java
    │   │               │   └── service
    │   │               │       ├── InventoryService.java
    │   │               │       └── InventoryServiceImpl.java
```

The most relevant part of the Inventory module is:
- Use of  @EventListener annotation to process the OrderCreatedEvent published by Order module

```java
public class OrderEventListener {  
  
    private final InventoryService inventoryService;  
  
    @EventListener  
    public void handleOrderCreatedEvent(OrderCreatedEvent event) {  
        int retries = 3;  
        while (retries > 0) {  
            try {  
                inventoryService.updateInventory(event.getItems());  
                return; // Success  
            } catch (ObjectOptimisticLockingFailureException e) {  
                retries--;  
                if (retries == 0) {  
                    throw e;  
                }  
                System.out.println("Retrying inventory update due to optimistic locking failure...");  
            }  
        }  
    }  
}
```

- The ApplicactionModule annotation defined in inventory/package-info.java, used to defined the allowedDependencies from Order module.
```java
@org.springframework.modulith.ApplicationModule(allowedDependencies = {"order::dto", "order::event"})  
package dev.davidsarmiento.modulith.inventory;
```

- Verification of the valid module structures we can use this provided method verify():
```java
@Test  
void verifyModules() {  
    ApplicationModules.of(ExampleSpringModulithApplication.class).verify();  
}
```

- Documentation
- Spring Modulith has some tools to generate documentation in format uml and C4 , by the diagrama default folder is: build/spring-modulith-docs
> Tip: To be able to see this diagram you have to install PlantUML plugin for IntelliJ 
```java
public class DocumentationTests {  
  
    ApplicationModules modules = ApplicationModules.of(ExampleSpringModulithApplication.class);  
  
    @Test  
    void writeDocumentationSnippets() {  
  
        new Documenter(modules)  
                .writeModulesAsPlantUml()  
                .writeIndividualModulesAsPlantUml();  
    }  
}
```
 Component Diagram Generated by Spring Modulith
![](Pasted%20image%2020251005221717.png)
## 5. Conclusion
- Spring Modulith is more than a library,  it is a framework.
- In this demo,  we were introduced to basic use of modules with Spring Modulith. We also understand the use of some annotations : **`@ApplicationModule`** y **`@NamedInterface`** 
- Spring Modulith promotes the use of asynchronous event communication among modules.
- Spring Modulith provides a series of tools to test and document our modular application.

## Source Code 
[Github source](https://github.com/davis243/example-modulith)
## References
[ˆ1]  https://martinfowler.com/bliki/MonolithFirst.html
https://docs.spring.io/spring-modulith/reference/fundamentals.html