#jdbcTemplate #nativeQuery #hibernate #jdbc #rowMapper #customRepository 

ARREGLAR 
https://chatgpt.com/c/68b9f664-b1ec-8332-9bd6-6c74478eb155


# **1. Create the DTO**

Exactly like before, but now we don’t need to worry about constructor parameter types matching JPA.  
We can freely choose any types we want since we will control the mapping manually.

```java 
package com.example.dto;

import java.math.BigDecimal;
import java.time.LocalDate;

public class OrderSummaryDto {

    public enum Status { NEW, PROCESSING, SHIPPED, CANCELLED }

    private Long orderId;
    private String customerName;
    private BigDecimal totalAmount;
    private LocalDate orderDate;
    private Boolean urgent;
    private Status status;

    public OrderSummaryDto(Long orderId, String customerName, BigDecimal totalAmount,
                           LocalDate orderDate, Boolean urgent, Status status) {
        this.orderId = orderId;
        this.customerName = customerName;
        this.totalAmount = totalAmount;
        this.orderDate = orderDate;
        this.urgent = urgent;
        this.status = status;
    }

    // Getters
    public Long getOrderId() { return orderId; }
    public String getCustomerName() { return customerName; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public LocalDate getOrderDate() { return orderDate; }
    public Boolean getUrgent() { return urgent; }
    public Status getStatus() { return status; }
}

```


---

# **2. Write the native SQL query**

The SQL query remains the same as before. **Column aliases are optional** now because you control the mapping.

```sql
SELECT 
    o.id AS order_id,
    c.name AS customer_name,
    o.total_amount,
    o.order_date,
    o.urgent_flag,     -- 'Y' or 'N'
    o.status           -- 'NEW', 'SHIPPED', etc.
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE (:customerId IS NULL OR o.customer_id = :customerId)
ORDER BY o.order_date DESC
```

---

# **3. Create a custom repository interface**

Spring Data allows you to define **custom repository logic** that works alongside your standard JPA repository.

```java
package com.example.repo;

import com.example.dto.OrderSummaryDto;
import java.util.List;

public interface CustomOrderRepository {
    List<OrderSummaryDto> findOrderSummaries(Long customerId);
}
```
---

# **4. Implement the custom repository**

Use `JdbcTemplate` and a `RowMapper` to map rows to your DTO.

```java
package com.example.repo.impl;

import com.example.dto.OrderSummaryDto;
import com.example.repo.CustomOrderRepository;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.stereotype.Repository;

import java.sql.ResultSet;
import java.sql.SQLException;
import java.time.LocalDate;
import java.util.List;

@Repository
public class CustomOrderRepositoryImpl implements CustomOrderRepository {

	@Autowired
    private final JdbcTemplate jdbcTemplate;

    // RowMapper to map ResultSet → DTO
    private static final RowMapper<OrderSummaryDto> ORDER_SUMMARY_ROW_MAPPER = new RowMapper<>() {
        @Override
        public OrderSummaryDto mapRow(ResultSet rs, int rowNum) throws SQLException {
            return new OrderSummaryDto(
                rs.getLong("order_id"),
                rs.getString("customer_name"),
                rs.getBigDecimal("total_amount"),
                rs.getDate("order_date") != null ? rs.getDate("order_date").toLocalDate() : null,
                mapUrgentFlag(rs.getString("urgent_flag")),
                mapStatus(rs.getString("status"))
            );
        }

        private Boolean mapUrgentFlag(String flag) {
            if (flag == null) return null;
            return flag.trim().equalsIgnoreCase("Y") || flag.trim().equals("1");
        }

        private OrderSummaryDto.Status mapStatus(String statusStr) {
            if (statusStr == null || statusStr.isBlank()) return null;
            return OrderSummaryDto.Status.valueOf(statusStr.trim().toUpperCase());
        }
    };

    @Override
    public List<OrderSummaryDto> findOrderSummaries(Long customerId) {
        String sql = """
            SELECT 
                o.id AS order_id,
                c.name AS customer_name,
                o.total_amount,
                o.order_date,
                o.urgent_flag,
                o.status
            FROM orders o
            JOIN customers c ON c.id = o.customer_id
            WHERE (:customerId IS NULL OR o.customer_id = :customerId)
            ORDER BY o.order_date DESC
            """;

        // Use parameter binding
        return jdbcTemplate.query(sql, ps -> {
            if (customerId != null) {
                ps.setLong(1, customerId);
            } else {
                ps.setNull(1, java.sql.Types.BIGINT);
            }
        }, ORDER_SUMMARY_ROW_MAPPER);
    }
}

```
---

# **5. Integrate with your main repository**

Combine the custom repository with a standard JPA repository:

```java
package com.example.repo;

import com.example.domain.Order;
import org.springframework.data.jpa.repository.JpaRepository;

public interface OrderRepository extends JpaRepository<Order, Long>, CustomOrderRepository {
    // now has both JPA and custom JDBC methods
}
```

> **Why this works:**
> 
> - `OrderRepository` gives you all the usual Spring Data JPA methods (`save`, `findById`, etc.).
>     
> - `CustomOrderRepositoryImpl` provides high-performance native SQL mapping with `JdbcTemplate`.
>     

---

# **6. Usage in a service**

```java
package com.example.service;

import com.example.dto.OrderSummaryDto;
import com.example.repo.OrderRepository;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class OrderService {

    @Autowired
    private final OrderRepository orderRepository;

    public List<OrderSummaryDto> getOrderSummaries(Long customerId) {
        return orderRepository.findOrderSummaries(customerId);
    }
}
```

---

# **7. Example call**

```java
List<OrderSummaryDto> summaries = orderService.getOrderSummaries(123L);

for (OrderSummaryDto dto : summaries) {
    System.out.printf("Order %d | Customer: %s | Urgent: %s | Status: %s%n",
        dto.getOrderId(), dto.getCustomerName(), dto.getUrgent(), dto.getStatus());
}
```

---
