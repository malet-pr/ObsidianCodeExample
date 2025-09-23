#interface-based #nativeQuery #dto-projection #hibernate #jpa #modelMapper #mapStruct

1. Create an interface projection
	- Each method in the interface corresponds to a column in the query by alias name.  Spring will match the alias to the method name. 
	- The getter names **must match** the SQL aliases (case-insensitive).    
	- For tricky conversions like `'Y'/'N'` booleans or enums, map them as `String` or `Integer` here, then convert later in service code.
```java title:"Projection Interface" fold:true
package com.example.dto;  

import java.math.BigDecimal; 
import java.time.LocalDate;  

public interface OrderSummaryView {      
	Long getOrderId();     
	String getCustomerName();     
	BigDecimal getTotalAmount();     
	LocalDate getOrderDate();      // Spring will try to convert automatically   
	String getUrgentFlag();        // We will post-process to Boolean if needed   
	String getStatus(); 
}	
```

2. Write the native query
	- The aliases used in the sql query must match exactly the interface getter names.
```sql title:"Native Query" fold:true
SELECT 
    o.id            AS order_id,
    c.name          AS customer_name,
    o.total_amount  AS total_amount,
    o.order_date    AS order_date,
    o.urgent_flag   AS urgent_flag,  -- 'Y' or 'N'
    o.status        AS status        -- 'NEW', 'SHIPPED', etc.
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE (:customerId IS NULL OR o.customer_id = :customerId)
ORDER BY o.order_date DESC
```

3. Repository method
	- Reference the native query directly
	- Return the projection
	- Spring maps the result set by matching each SQL alias (`order_id`, `customer_name`, etc.) to the corresponding `getOrderId()`, `getCustomerName()`, etc. in the interface.
```java title:"Repository" fold:true
package com.example.repo;

import com.example.dto.OrderSummaryView;
import com.example.domain.Order;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.Repository;
import org.springframework.data.repository.query.Param;

import java.util.List;

public interface OrderRepository extends Repository<Order, Long> {

    @Query(value =
        "SELECT " +
        "  o.id AS order_id, " +
        "  c.name AS customer_name, " +
        "  o.total_amount AS total_amount, " +
        "  o.order_date AS order_date, " +
        "  o.urgent_flag AS urgent_flag, " +
        "  o.status AS status " +
        "FROM orders o " +
        "JOIN customers c ON c.id = o.customer_id " +
        "WHERE (:customerId IS NULL OR o.customer_id = :customerId) " +
        "ORDER BY o.order_date DESC",
        nativeQuery = true)
    List<OrderSummaryView> findOrderSummaries(@Param("customerId") Long customerId);
}
```

4. Use in other clases
```java title:"Use in Service" fold:true
@Service
public class OrderService {

	@Autowired
    private final OrderRepository orderRepository;

    public List<OrderSummaryDto> getSummaries(Long customerId) {
        return orderRepository.findOrderSummaries(customerId)
            .stream()
            .map(view -> new OrderSummaryDto(
                view.getOrderId(),
                view.getCustomerName(),
                view.getTotalAmount(),
                view.getOrderDate(),
                "Y".equalsIgnoreCase(view.getUrgentFlag()), 
			    OrderSummaryDto.Status.valueOf(view.getStatus()
				    .trim().toUpperCase())
		    ))
            .toList();
    }
}
```

5. Handling conversions
	1. Boolean stored as `'Y'/'N'`:
		- Option A: Return `String` and convert in service layer: `Boolean urgent = "Y".equalsIgnoreCase(view.getUrgentFlag());`
		- Option B: Handle conversion at the database level: If the database supports CASE:`CASE WHEN o.urgent_flag = 'Y' THEN 1 ELSE 0 END AS urgent_flag`. Then define the getter as `Boolean getUrgentFlag();`.
	2. Enums: For enums like `"NEW"`, `"SHIPPED"`, etc., map as `String` in the interface and convert later:`OrderSummaryDto.Status status =      OrderSummaryDto.Status.valueOf(view.getStatus().trim().toUpperCase());`
	
6. Adding pagination
```java title:"Pagination" fold:true
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

@Query(value =
    "SELECT ...", // same query as before
    countQuery = "SELECT COUNT(*) FROM orders o JOIN customers c ON c.id = o.customer_id",
    nativeQuery = true)
Page<OrderSummaryView> findPagedSummaries(@Param("customerId") Long customerId, Pageable pageable);
```

7. Complex mapping or large DTO
	1. ModelMapper: dynamic, runtime mapping, simpler for quick prototypes.
	2. MapStruct: compile-time generation, faster and safer for very large DTOs.
	3. Custom mappers are preferred for large DTOs to avoid too much reflection overhead.

Simple With Model Mapper

```java title:"Mapper Configuration"
import org.modelmapper.ModelMapper;

@Service
public class ActaMapper {

    private final ModelMapper modelMapper = new ModelMapper();

    public ActaDTO toDTO(ActaProjection projection) {
        return modelMapper.map(projection, ActaDTO.class);
    }
}
```

```java title:"Service"
List<ActaDTO> dtos = actaRepository.findActaById(actaId)
                                   .stream()
                                   .map(actaMapper::toDTO)
                                   .toList();
```

Custom With Model Mapper

```java title:"Mapper Configuration" fold:true
import org.modelmapper.ModelMapper;
import org.modelmapper.PropertyMap;
import org.springframework.stereotype.Service;
import java.time.LocalDate;
import java.time.ZoneId;

@Service
public class ActaMapper {

    private final ModelMapper modelMapper;

    public ActaMapper() {
        this.modelMapper = new ModelMapper();

        // Custom mapping for Booleans and Dates
        modelMapper.addMappings(new PropertyMap<ActaProjection, ActaDTO>() {
            @Override
            protected void configure() {
                map(source.getTarea(), destination.getTarea());

                // Boolean mapping
                using(ctx -> "Y".equals(ctx.getSource())).map(source.getExcluida(), destination.getExcluida());

                // Date mapping
                using(ctx -> ((java.sql.Timestamp) ctx.getSource()).toInstant()
                           .atZone(ZoneId.systemDefault())
                           .toLocalDate())
                           .map(source.getFechaCierre(), destination.getFechaCierre());
            }
        });
    }

    public ActaDTO toDTO(ActaProjection projection) {
        return modelMapper.map(projection, ActaDTO.class);
    }
}
```

```java title:"Service" fold:true
List<ActaDTO> dtos = actaRepository.findActaById(actaId)
                                   .stream()
                                   .map(actaMapper::toDTO)
                                   .toList();
```

With MapStruct

```java title:"Mapper Configuration" fold:true
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.Named;
import org.mapstruct.factory.Mappers;
import java.time.LocalDate;
import java.time.ZoneId;

@Mapper(componentModel = "spring")
public interface ActaMapper {

    ActaMapper INSTANCE = Mappers.getMapper(ActaMapper.class);

    @Mapping(source = "excluida", target = "excluida", qualifiedByName = "mapYnToBoolean")
    @Mapping(source = "fechaCierre", target = "fechaCierre", qualifiedByName = "mapTimestampToLocalDate")
    ActaDTO toDTO(ActaProjection projection);

    // Custom mapping for 'Y'/'N' → Boolean
    @Named("mapYnToBoolean")
    default Boolean mapYnToBoolean(String value) {
        if (value == null) return null;
        return "Y".equals(value);
    }

    // Custom mapping for java.sql.Timestamp → LocalDate
    @Named("mapTimestampToLocalDate")
    default LocalDate mapTimestampToLocalDate(java.sql.Timestamp ts) {
        if (ts == null) return null;
        return ts.toInstant().atZone(ZoneId.systemDefault()).toLocalDate();
    }
```

```java title:Service fold:true
@Service
public class ActaService {

	@Autowired
    private final ActaRepository actaRepository;
    @Autowired
    private final ActaMapper actaMapper; 

    public List<ActaDTO> getActas(Long actaId) {
        return actaRepository.findActaById(actaId)
                             .stream()
                             .map(actaMapper::toDTO)
                             .toList();
    }
}
```

