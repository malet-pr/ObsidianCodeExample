#class-based #dto-projection #nativeQuery #hibernate #jpa #SqlResultSetMapping

1. Create DTO
	- `@ConstructorResult` expects your constructor parameter types to match what JPA will produce for each column.
	- Use conversion logic _inside the constructor_ for tricky fields like `Y/N` booleans or enums.
	- For booleans stored as `CHAR(1)` (`Y/N`) or `0/1`, map as `String`/`Integer` and convert yourself.
	- For dates, mapping to `Timestamp` in the constructor is the most compatible route, then convert to `LocalDate`/`LocalDateTime`.
```java title:"DTO" fold:true
package com.example.dto;

import java.math.BigDecimal;
import java.sql.Timestamp;
import java.time.LocalDate;

public class OrderSummaryDto {

    public enum Status { NEW, PROCESSING, SHIPPED, CANCELLED }

    private final Long orderId;
    private final String customerName;
    private final BigDecimal totalAmount;
    private final LocalDate orderDate;       // stored as DATE/TIMESTAMP in DB
    private final Boolean urgent;            // stored as 'Y'/'N' in DB
    private final Status status;             // stored as text in DB

    // Constructor must match the @ColumnResult types (see Step 2)
    public OrderSummaryDto(
            Long orderId,
            String customerName,
            BigDecimal totalAmount,
            Timestamp orderDateTs,   // map raw DB timestamp and convert
            String urgentFlag,       // map 'Y'/'N' and convert
            String statusStr         // map text and convert to enum
    ) {
        this.orderId = orderId;
        this.customerName = customerName;
        this.totalAmount = totalAmount;
        this.orderDate = orderDateTs != null ? orderDateTs.toLocalDateTime().toLocalDate() : null;

        this.urgent = urgentFlag == null
                ? null
                : urgentFlag.trim().equalsIgnoreCase("Y") || urgentFlag.trim().equals("1");

        this.status = (statusStr == null || statusStr.isBlank())
                ? null
                : Status.valueOf(statusStr.trim().toUpperCase());
    }

    // getters...
    public Long getOrderId() { return orderId; }
    public String getCustomerName() { return customerName; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public LocalDate getOrderDate() { return orderDate; }
    public Boolean getUrgent() { return urgent; }
    public Status getStatus() { return status; }
}
```

2. On any managed or a dummy @Entity, declare the query and the mapping
	- You put `@SqlResultSetMapping` and `@NamedNativeQuery` on an `@Entity` (often the main table you’re querying). The SQL must alias columns to the names used in `@ColumnResult`.
	- - **Aliases matter**: `as order_id`, `as urgent_flag`, etc. must exactly match the `@ColumnResult(name=...)`.
    - The `type = ...` in each `@ColumnResult` should match the constructor parameter types you chose in the DTO.
```java title:"Query and Mapping" fold:true
package com.example.domain;

import com.example.dto.OrderSummaryDto;
import jakarta.persistence.*;
import java.math.BigDecimal;
import java.sql.Timestamp;

@Entity
@Table(name = "orders")
@NamedNativeQuery(
    name = "Order.summaryByCustomer",
    query =
        "select " +
        "  o.id              as order_id, " +
        "  c.name            as customer_name, " +
        "  o.total_amount    as total_amount, " +
        "  o.order_date      as order_date, " +
        "  o.urgent_flag     as urgent_flag, " +   // 'Y'/'N' in DB
        "  o.status          as status " +         // e.g. 'NEW', 'SHIPPED'
        "from orders o " +
        "join customers c on c.id = o.customer_id " +
        "where (:customerId is null or o.customer_id = :customerId) " +
        "order by o.order_date desc",
    resultSetMapping = "OrderSummaryMapping"
)
@SqlResultSetMapping(
    name = "OrderSummaryMapping",
    classes = @ConstructorResult(
        targetClass = OrderSummaryDto.class,
        columns = {
            @ColumnResult(name = "order_id",      type = Long.class),
            @ColumnResult(name = "customer_name", type = String.class),
            @ColumnResult(name = "total_amount",  type = BigDecimal.class),
            @ColumnResult(name = "order_date",    type = Timestamp.class),
            @ColumnResult(name = "urgent_flag",   type = String.class),
            @ColumnResult(name = "status",        type = String.class)
        }
    )
)
public class Order {
    @Id
    private Long id;

    // other mapped fields if you need them; not required for the DTO mapping itself
}
```

3. Reference the native query in the repository
	- Let Spring Data call your named native query and **return the DTO directly**.
```java title:"Repository" fold:true
package com.example.repo;

import com.example.dto.OrderSummaryDto;
import com.example.domain.Order;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.Repository;
import org.springframework.data.repository.query.Param;

import java.util.List;

public interface OrderRepository extends Repository<Order, Long> {

    @Query(name = "Order.summaryByCustomer", nativeQuery = true)
    List<OrderSummaryDto> findOrderSummariesByCustomer(@Param("customerId") Long customerId);
}
```

4. Use it in other classes
```java title:"Uses" fold:true
List<OrderSummaryDto> summaries =
    orderRepository.findOrderSummariesByCustomer(123L);
```

5. Tips
 - **Booleans in Oracle (`CHAR(1)` = `Y/N` or `0/1`)**  
    Don’t declare `Boolean` in `@ColumnResult`. Map as `String` or `Integer` and convert in the DTO constructor (as shown). If Oracle pads `CHAR(1)` with spaces, use `TRIM` in SQL or `trim()` in Java.
- **Date/Time types**  
    Mapping native results directly to `LocalDate/LocalDateTime` can be provider-specific. The safest bet is `Timestamp` (or `Date`) in the constructor and convert to Java Time.
- **Enums**  
    Map the column as `String` and convert with `Enum.valueOf(…)` in the DTO constructor. Normalize the case and trim to avoid surprises.
- **Column name mismatches**  
    If your query aliases don’t match the `@ColumnResult(name=...)`, the mapping won’t fire. Double-check spelling/case.
- **Can’t modify an existing entity?**  
    Create a tiny “holder” entity just to keep the `@SqlResultSetMapping` / `@NamedNativeQuery` annotations. It only needs `@Entity` + `@Table` + an `@Id`, not actual usage.