# Command Query Responsibility Segregation (CQRS) Pattern
**Overview**
CQRS (Command Query Responsibility Segregation) is a software architecture pattern that separates read and write operations into distinct models. Instead of using a single model for both commands (write operations) and queries (read operations), CQRS divides them into two separate models:

- **Command Model:** Handles operations that modify data (Create, Update, Delete).
- **Query Model:** Handles operations that retrieve data (Read).

This separation allows better scalability, improved performance, and enhanced security.

----------

**Key Concepts of CQRS**

1. **Command Model (Write Model)**
    - Accepts commands that modify data.
    - Enforces business rules and validation.
    - Often interacts with the database in a transactional manner.
    - Uses domain models and aggregates to ensure consistency.
2. **Query Model (Read Model)**
    - Retrieves and returns data.
    - Optimized for fast read operations (can use caching, denormalized data, or NoSQL).
    - Does not modify data.
    - Can use different databases or projections optimized for read performance.
3. **Event Sourcing (Optional)**
    - Instead of directly updating the database, changes are stored as a sequence of events.
    - Events represent state changes and can be replayed to reconstruct the current state.
    - Works well with CQRS to maintain a reliable audit trail.
----------

**How CQRS Works?**

1. **A user sends a command (e.g., "Create Order")** → Processed by the Command Model.
2. **The Command Model validates and updates the database** → Often triggers an event.
3. **The event updates the Read Model (Query Model)** → Could be a separate database or an optimized structure.
4. **A user requests data (e.g., "Get Order Details")** → Processed by the Query Model for fast retrieval.
----------

**Benefits of CQRS**
✅ **Scalability:** Reads and writes can be scaled independently.
✅ **Performance Optimization:** Queries can use denormalized structures, caching, or different storage mechanisms.
✅ **Improved Security:** Write operations can enforce stricter rules, while read operations can be exposed differently.
✅ **Better Maintainability:** Separation of concerns makes the system easier to manage and evolve.
✅ **Event Sourcing Support:** Helps in audit logging and reconstructing historical states.

----------

**Example Use Case: Order Management System**

- **Command Model:** Handles "Create Order," "Update Order," "Cancel Order."
- **Query Model:** Retrieves order details, order history, customer order list.
- **Data Storage:**
    - Write database: Stores normalized transactional data.
    - Read database: Stores precomputed views for fast retrieval.
----------

**When to Use CQRS?**
✅ High read-to-write ratio applications (e.g., e-commerce, social media).
✅ Systems requiring event sourcing and audit logs.
✅ Large-scale applications with multiple read/write optimizations.
✅ Microservices architecture where read and write services need to scale separately.
🚫 Avoid CQRS if the system is simple and doesn’t need read/write separation, as it adds complexity.

![](https://paper-attachments.dropboxusercontent.com/s_2F3D3E9526748A0D70829C44D40E6D0C1E690F6ED411792DBD3EF07A3CFF11E2_1743743877671_image.png)


## CQRS Implementation in Spring Boot

### Overview

CQRS (Command Query Responsibility Segregation) separates read and write operations into distinct models:
- Command Model (Write Model): Handles operations that modify data.
- Query Model (Read Model): Handles operations that retrieve data.

In this implementation, we'll create a Spring Boot Order Management System using CQRS.

### Project Structure

```
order-service-cqrs
│── src/main/java/com/example/cqrs
│   ├── command/           # Write model (commands)
│   │   ├── controller/    # REST API for commands
│   │   ├── service/       # Business logic for commands
│   │   ├── entity/        # Command-side entity
│   │   ├── repository/    # Command-side repository
│   │   ├── event/         # Events for propagating changes
│   ├── query/             # Read model (queries)
│   │   ├── controller/    # REST API for queries
│   │   ├── service/       # Business logic for queries
│   │   ├── entity/        # Query-side entity
│   │   ├── repository/    # Query-side repository
│   ├── CqrsOrderServiceApplication.java
│   ├── application.yml    # Spring configuration
└── pom.xml                # Dependencies
```
### Command Side (Write Model)

**Order Entity**
```java
package com.example.cqrs.command.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "orders")
@Getter @Setter
@AllArgsConstructor @NoArgsConstructor
public class Order {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String product;
    private Integer quantity;
    private Double price;
}
```

**Order Created Event**
```java
package com.example.cqrs.command.event;

import lombok.AllArgsConstructor;
import lombok.Getter;

@Getter
@AllArgsConstructor
public class OrderCreatedEvent {
    private Long orderId;
    private String product;
    private Integer quantity;
    private Double price;
}
```

**Order Repository**
```java
package com.example.cqrs.command.repository;

import com.example.cqrs.command.entity.Order;
import org.springframework.data.jpa.repository.JpaRepository;

public interface OrderRepository extends JpaRepository<Order, Long> {
}
```

**Order Service**
```java
package com.example.cqrs.command.service;

import com.example.cqrs.command.entity.Order;
import com.example.cqrs.command.event.OrderCreatedEvent;
import com.example.cqrs.command.repository.OrderRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    public Order createOrder(Order order) {
        Order savedOrder = orderRepository.save(order);
        eventPublisher.publishEvent(new OrderCreatedEvent(
                savedOrder.getId(), 
                savedOrder.getProduct(), 
                savedOrder.getQuantity(), 
                savedOrder.getPrice()
        ));
        return savedOrder;
    }
}
```

**Order Controller**
```java
package com.example.cqrs.command.controller;

import com.example.cqrs.command.entity.Order;
import com.example.cqrs.command.service.OrderService;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderCommandController {

    private final OrderService orderService;

    @PostMapping
    public Order createOrder(@RequestBody Order order) {
        return orderService.createOrder(order);
    }
}
```
### Query Side (Read Model)

**Query Side Entity**
```java
package com.example.cqrs.query.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "orders_read")
@Getter @Setter
@AllArgsConstructor @NoArgsConstructor
public class OrderReadModel {

    @Id
    private Long id;
    private String product;
    private Integer quantity;
    private Double price;
}
```

**Query Side Repository**
```java
package com.example.cqrs.query.repository;

import com.example.cqrs.query.entity.OrderReadModel;
import org.springframework.data.jpa.repository.JpaRepository;

public interface OrderReadRepository extends JpaRepository<OrderReadModel, Long> {
}
```

**Query Side Service**
```java
package com.example.cqrs.query.service;

import com.example.cqrs.command.event.OrderCreatedEvent;
import com.example.cqrs.query.entity.OrderReadModel;
import com.example.cqrs.query.repository.OrderReadRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class OrderQueryService {

    private final OrderReadRepository orderReadRepository;

    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        OrderReadModel orderReadModel = new OrderReadModel(
                event.getOrderId(),
                event.getProduct(),
                event.getQuantity(),
                event.getPrice()
        );
        orderReadRepository.save(orderReadModel);
    }
}
```

**Query Controller**

```java
package com.example.cqrs.query.controller;

import com.example.cqrs.query.entity.OrderReadModel;
import com.example.cqrs.query.repository.OrderReadRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderQueryController {

    private final OrderReadRepository orderReadRepository;

    @GetMapping
    public List<OrderReadModel> getAllOrders() {
        return orderReadRepository.findAll();
    }

    @GetMapping("/{id}")
    public OrderReadModel getOrderById(@PathVariable Long id) {
        return orderReadRepository.findById(id).orElseThrow();
    }
}
```
