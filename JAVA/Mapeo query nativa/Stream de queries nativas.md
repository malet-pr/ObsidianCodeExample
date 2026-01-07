
```java title:"Stream with interface-based projection" fold:true
public interface UserProjection {
    Long getId();
    String getUsername();
    String getEmail();
}

public interface UserRepository extends JpaRepository<User, Long> {
    
    @QueryHints(value = @QueryHint(name = HINT_FETCH_SIZE, value = "500"))
    @Query(value = "SELECT id, username, email FROM users WHERE active = true", nativeQuery = true)
    Stream<UserProjection> streamAllActiveUsers();
}
```

```java title:"Custom Repository" fold:true
@Repository
public class UserRepositoryCustom {

    @PersistenceContext
    private EntityManager entityManager;

    @Transactional(readOnly = true)
    public Stream<UserGridDto> streamAllForGrid() {
        // Use Hibernate's Session for low-level streaming control
        Session session = entityManager.unwrap(Session.class);
        
        return session.createNativeQuery(
            "SELECT id, username, status FROM users WHERE deleted = false")
            .setFetchSize(1000) // Critical for memory
            .setReadOnly(true)
            .stream() // In Java 11/Hibernate 5.x+, this returns a Stream<Object[]>
            .map(row -> {
                Object[] cells = (Object[]) row;
                return new UserGridDto(
                    ((Number) cells[0]).longValue(),
                    (String) cells[1],
                    (String) cells[2]
                );
            });
    }
}
```

```java title:"Class-based projection" fold:true
public class LargeReportDto {
    private Long id;
    private String clientName;
    private Double totalAmount;
    // ... all other fields from your 9 joins

    public LargeReportDto(Object[] row) {
        this.id = ((Number) row[0]).longValue();
        this.clientName = (String) row[1];
        this.totalAmount = ((Number) row[2]).doubleValue();
        // Map all columns by index
    }
}

@Repository
public class ReportRepositoryImpl {

    @PersistenceContext
    private EntityManager entityManager;

    @Transactional(readOnly = true)
    public Stream<LargeReportDto> streamComplexReport() {
        Session session = entityManager.unwrap(Session.class);
        
        // Native query with 9 joins
        String sql = "SELECT u.id, c.name, o.amount ... " +
                     "FROM users u " +
                     "JOIN clients c ON ... " + 
                     "JOIN orders o ON ..."; // your 8-9 joins here

        return session.createNativeQuery(sql)
                .setFetchSize(500) // Batches of 500 rows
                .setReadOnly(true)
                .stream() 
                .map(row -> new LargeReportDto((Object[]) row));
    }
}
```

### One Critical Warning: Connection Pool

When you stream, the database connection stays **open** until the `Stream.close()` is called (which happens at the end of your `try-with-resources`).

If you have a lot of users running this report at the same time, you might exhaust your **HikariCP connection pool**.

- **Tip:** Ensure your connection pool size (`spring.datasource.hikari.maximum-pool-size`) is large enough, or set a specific timeout for these long-running stream exports.

```java 
import org.hibernate.Session;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;
import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;
import java.util.stream.Stream;

@Repository
public class OracleReportRepository {

    @PersistenceContext
    private EntityManager entityManager;

    @Transactional(readOnly = true)
    public Stream<Object[]> streamNativeQuery(String status) {
        Session session = entityManager.unwrap(Session.class);
        
        // Native Query with your 9 joins
        return session.createNativeQuery("SELECT ... FROM ... WHERE status = :status")
                .setParameter("status", status)
                .setFetchSize(500) // Crucial for Oracle to not buffer everything
                .setReadOnly(true)
                .stream();
    }
}
```

```java
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.servlet.mvc.method.annotation.StreamingResponseBody;

@GetMapping(value = "/reports/stream", produces = MediaType.APPLICATION_JSON_VALUE)
public ResponseEntity<StreamingResponseBody> streamReport() {
    return ResponseEntity.ok()
        .contentType(MediaType.APPLICATION_JSON)
        .body(outputStream -> {
            // 1. Initialize the Jackson Generator
            JsonGenerator jsonGenerator = objectMapper.getFactory().createGenerator(outputStream);
            
            try (Stream<Object[]> dataStream = repository.streamNativeQuery("ACTIVE")) {
                jsonGenerator.writeStartArray(); // Start JSON [
                
                dataStream.forEach(row -> {
                    try {
                        // Manually construct the DTO/JSON object per row
                        jsonGenerator.writeStartObject();
                        jsonGenerator.writeNumberField("id", ((Number) row[0]).longValue());
                        jsonGenerator.writeStringField("clientName", (String) row[1]);
                        // ... add your other 23 columns here
                        jsonGenerator.writeEndObject();
                        
                        // Optional: This frees up memory in the generator's internal buffer
                        jsonGenerator.flush(); 
                    } catch (Exception e) {
                        throw new RuntimeException("Error writing JSON row", e);
                    }
                });
                
                jsonGenerator.writeEndArray(); // End JSON ]
            } finally {
                jsonGenerator.close();
                outputStream.close();
            }
        });
}
```

```java
// Inside your MVC App Service
public List<MyDto> getReport() {
    return restTemplate.execute(serviceUrl, HttpMethod.GET, null, response -> {
        // This converts the incoming JSON stream directly into a List
        return objectMapper.readValue(response.getBody(), new TypeReference<List<MyDto>>(){});
    });
}
```

```java
@Service
public class ReportService {
    private final OracleReportRepository repository;

    public ReportService(OracleReportRepository repository) {
        this.repository = repository;
    }

    @Transactional(readOnly = true)
    public Stream<Object[]> getReportStream() {
        return repository.streamNativeQuery();
    }
}
```

```java
@GetMapping("/stream")
public ResponseEntity<StreamingResponseBody> stream(HttpServletResponse response) {
    return ResponseEntity.ok()
        .body(out -> {
            // The service call happens INSIDE the lambda to keep 
            // the transaction alive during serialization
            try (Stream<Object[]> data = reportService.getReportStream()) {
                serializeToStream(data, out);
            }
        });
}
```

```java
@Transactional(readOnly = true)
public void streamWithLogging() {
    Runtime runtime = Runtime.getRuntime();
    long before = (runtime.totalMemory() - runtime.freeMemory()) / 1024 / 1024;
    log.info("Heap before stream: {} MB", before);

    try (Stream<Object[]> stream = repository.streamNativeQuery()) {
        stream.forEach(row -> {
            // Processing logic
        });
    }

    long after = (runtime.totalMemory() - runtime.freeMemory()) / 1024 / 1024;
    log.info("Heap after stream: {} MB", after);
}
```





