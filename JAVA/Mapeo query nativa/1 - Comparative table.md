#java #nativeeQuery 

|**Feature**|**Interface Projection**|**Class DTO Mapping** (`@SqlResultSetMapping`)|**JdbcTemplate + RowMapper**|
|---|---|---|---|
|**Boilerplate code**|✅ Minimal – just interface and repository method|❌ High – requires DTO + annotations + extra entity holder|⚠️ Moderate – need custom repository and mapper|
|**Mapping complexity**|❌ Limited – only direct mapping of DB types to Java|✅ Full control inside DTO constructor|✅ Full control inside `RowMapper`|
|**Tricky conversions (`Y/N` → Boolean, enums, etc.)**|❌ Hard – must post-process manually in service layer|✅ Easy – handled directly in constructor|✅ Easy – handled directly in `RowMapper`|
|**Control over SQL**|⚠️ Limited – query must fit Spring Data rules, cannot dynamically build complex queries easily|⚠️ Limited – still tied to JPA|✅ Full – write any SQL you want|
|**Performance (overhead)**|⚡ Good – minimal JPA processing|⚡ Good – slightly more overhead than interface projection|⚡ Fastest – no JPA overhead|
|**Type safety**|⚠️ Medium – mapping relies on alias matching to getters|✅ Strong – explicit mapping defined in annotations|✅ Strong – explicit control over ResultSet mapping|
|**Ease of refactoring**|⚠️ Risky – SQL aliases must match interface method names|✅ Safe – explicit column mapping via `@ColumnResult`|✅ Safe – handled programmatically in one place|
|**Pagination support (`Pageable`)**|✅ Built-in with Spring Data|✅ Built-in with Spring Data|❌ Manual implementation required|
|**Spring Data integration**|✅ Full – native query result fits seamlessly|✅ Full – uses named native queries|⚠️ Partial – must wrap manually or return `List`|
|**Best for simple read-only queries**|✅ Perfect|⚠️ Overkill for very simple queries|❌ Too heavy for trivial queries|
|**Best for complex result sets**|❌ Poor – lacks flexibility|✅ Good – can handle moderately complex results|✅ Excellent – no limits|
|**Runtime flexibility (dynamic SQL)**|❌ Limited – must predefine query|❌ Limited – named query is static|✅ Full – build SQL dynamically|
|**Learning curve**|⭐ Easy|⭐⭐ Medium|⭐⭐⭐ Steeper – need JDBC, SQL, and custom repo knowledge|
|**Extra artifacts needed**|None beyond repository|Extra dummy `@Entity` for annotations|Separate custom repository and RowMapper|
|**Typical use case**|Quick data retrieval, dashboards, lists|Complex projections where you still want JPA to manage mapping|Complex reporting, performance-critical code, legacy SQL|

---

### **Summary**

|**When to choose**|**Recommended approach**|
|---|---|
|**Simple read-only query, little to no conversion needed**|**Interface-based projection** – cleanest and fastest to implement|
|**Need some conversion logic or non-trivial mapping but want to stay in JPA world**|**Class-based DTO mapping** – more explicit and safer|
|**Very complex SQL, performance-sensitive, or heavy transformations needed**|**JdbcTemplate + RowMapper** – absolute control and best performance|

---
### **Example Scenarios**

|**Scenario**|**Best fit**|**Why**|
|---|---|---|
|Displaying a paginated list of orders in UI|Interface projection|Lightweight and supports pagination automatically|
|Retrieving data with a `Y/N` column, complex enums, and computed totals|Class DTO mapping|Clear conversion inside the constructor|
|Querying a legacy Oracle database with huge joins and CASE expressions|JdbcTemplate|Avoids JPA parsing overhead and gives full SQL freedom|
|Building dynamic search filters from user input|JdbcTemplate|SQL can be built dynamically with `StringBuilder` or QueryBuilder|

---

### **Final recommendation**

- Start with interface projections for simple and stable queries.
- Move to class DTO mapping when conversion logic grows or mapping becomes fragile.
- Switch to JdbcTemplate for maximum performance and flexibility, especially when working with legacy or very complex SQL.