1. Tutoriales
	1. https://www.ehcache.org/documentation/3.3/getting-started.html 
	2. https://springframework.guru/using-ehcache-3-in-spring-boot
	3. https://www.geeksforgeeks.org/java/spring-boot-ehcaching/
	4. https://www.baeldung.com/spring-boot-ehcache
	
2. Estrategia de implementación
	1. **Setup and Validation:** Get Spring Boot to recognize and initialize Ehcache using a small _on-heap_ cache first. Verify the `@Cacheable` annotations work.
	2. **Size Analysis:** Re-confirm the exact memory size your  cache requires.
	3. **Off-Heap Configuration:** Implement the `ehcache.xml` or `ehcache.yaml` with the `offheap` tier.
	4. **JVM Tuning:** Add the essential `-XX:MaxDirectMemorySize` JVM argument to JBoss 7.4 configuration.
	5. **Performance Test:** Test for GC pause time reduction and overall execution speed (including the serialization/deserialization cost).

## 🛠️ Ehcache 3.x Off-Heap Implementation Guide for Drools

The core concept is to define a tiered cache that moves your large Drools `KieContainer` objects out of the Java heap and into separate **Direct Memory** managed by Ehcache.

### Step 1: Add Dependencies (Maven/Gradle)

Spring Boot caching abstraction, the JCache API (JSR-107), and the Ehcache 3.x implementation.

|**Dependency**|**Purpose**|
|---|---|
|`spring-boot-starter-cache`|Provides Spring's caching infrastructure and `@Cacheable` support.|
|`javax.cache:cache-api`|The standardized JSR-107 API that Ehcache implements.|
|`org.ehcache:ehcache`|The core Ehcache 3.x library.|

**Important Note on Serialization:** for storing complex Drools objects off-heap, they **must** be serialized. By default, Ehcache uses Java serialization (which is slow). Implement or configure a custom serializer (like **Kryo** or **FST**) for optimal performance. Drools object class (`KieContainer` or the object holding it) must implement `java.io.Serializable` or provide a custom `Serializer` to Ehcache.

### Step 2: Spring Boot Configuration

1. **Enable Caching:** Add `@EnableCaching` to one of your `@Configuration` classes or your main Spring Boot application class.
    
2. **Add this code to the CacheConfig class:** (mark Caffeine as `@Primary`).
    
```java title:"add to Cache Configuration Class" fold:true
@Bean
public JCacheCacheManager ehcacheOffHeapCacheManager() {
    try {
        CachingProvider cachingProvider = Caching.getCachingProvider();
        javax.cache.CacheManager manager = cachingProvider.getCacheManager(
            getClass().getResource("/ehcache.yaml").toURI(), // Point to YAML
            getClass().getClassLoader()
        );
        return new JCacheCacheManager(manager);
    } catch (URISyntaxException | IllegalStateException e) {
         // ... error handling ...
    }
}
```
    
### Step 3: Ehcache YAML Configuration 

Create an `ehcache.yaml` file in your `src/main/resources` folder. This file is the single source of truth for your cache memory management.

#### 📝 Key Configuration Elements:

|**Element**|**Description**|**Your Goal**|
|---|---|---|
|`resource-pools` (Global)|**Crucial:** Defines the total shared off-heap capacity available to _all_ Ehcache caches.|Set this slightly _less_ than your `-XX:MaxDirectMemorySize` (Step 4).|
|`<cache alias="..."/>`|Defines a specific cache for your Drools rules (e.g., `droolsRuleCache`).|The name must match the `cacheNames` parameter in your `@Cacheable` annotation.|
|`key-type` / `value-type`|**Important:** Must match the Java types being cached (e.g., `java.lang.String` and `org.kie.api.runtime.KieContainer`).|Explicitly define the types for safety.|
|`resources` (Per Cache)|Defines the memory tiers for this specific cache. **Must be pyramidal (Heap < Off-Heap < Disk).**|Configure `heap` (e.g., by entries) and `offheap` (e.g., by MB).|

#### Example  // cambiar esto

```yml title:"ehcache configuration" fold:true
---
# The primary configuration block for the CacheManager
# This configuration is loaded by the JCacheCacheManager
config:
  # --- 1. GLOBAL RESOURCE POOLS ---
  # Defines the total off-heap memory available to ALL caches managed by this configuration.
  resource-pools:
    offheap: 4 # Value
    unit: GB # Unit (MB, GB, TB)

  # --- 2. CACHE DEFINITIONS ---
  caches:
    # Definition for your Drools Rules Cache
    droolsRuleCache:
      # Key and Value Types
      key-type: java.lang.String
      value-type: com.yourcompany.rules.

	  # Enables JMX statistics for this cache 
	  statistics: true

      # Resource Tiers (Must be pyramidal: Heap < Off-Heap)
      resources:
        # Tier 1: Small On-Heap Buffer (fastest access)
        heap: 20
        unit: entries

        # Tier 2: Large Off-Heap Storage (for GC pressure relief)
        offheap: 3.5
        unit: GB
      
	  serializers:
		# Specifies the class to use for serializing the VALUE type 
		value: com.yourcompany.rules.KieContainerKryoSerializer  
        
      # Expiry Policy (Time-To-Idle: expires 1 hour after last access)
      expiry:
        tti: 1
        unit: hours
---
```

### Step 4: JBoss EAP 7.4 JVM Tuning (Crucial Step)

The off-heap memory is consumed from the JVM's **Direct Memory** pool. This pool is _not_ limited by the Java heap (`-Xmx`), but it _is_ limited by its own configuration parameter. **If you don't set this, you will get OutOfMemoryErrors in native code.**

You must configure the `-XX:MaxDirectMemorySize` JVM argument in your JBoss EAP 7.4 instance.

1. **Locate Configuration File:** On JBoss/Wildfly, JVM options are typically configured in a script or configuration file inside the `<JBOSS_HOME>/bin` directory:
    
    - **Standalone Mode:** Look for `standalone.conf` (Linux/Mac) or `standalone.conf.bat` (Windows).
        
    - **Domain Mode:** Look for `domain.conf` or the specific host configuration file.
        
2. **Add the Parameter:** You need to locate the `JAVA_OPTS` environment variable definition (often a large block of comments or an `if/then` block). Add the `-XX:MaxDirectMemorySize` flag, ensuring its value is slightly **larger** than the total global `<offheap>` size you defined in your `ehcache.xml` (4GB in the example above).
    
    ```shell title:"add to standalone.conf" fold:true
    # Existing default JAVA_OPTS might be defined here
    # ...
    
    # Add your Direct Memory setting (Example: 4.5 GB)
    # The 'g' stands for gigabytes. Use 'm' for megabytes.
    JAVA_OPTS="$JAVA_OPTS -XX:MaxDirectMemorySize=4500m" 
    ```
    
3. **Review Heap Size:** Since 40% of your heap is moving off-heap, you should consider **reducing** your main heap size (`-Xmx`) to compensate, which will lower the load on the garbage collector.
    
### Step 5: Apply Caching in Code

Use the standard Spring `@Cacheable` annotation on the method that fetches or builds your Drools `KieContainer`.

```java title:"example caching KieContainer" fold:true
@Service
public class DroolsRulesService {
    
    // The value must match the 'alias' in ehcache.xml
    @Cacheable(value = "droolsRuleCache", key = "#rulesetName")
    public KieContainer loadKieContainer(String rulesetName) {
        // This is the expensive operation that loads and compiles the rules
        System.out.println("Building KieContainer for ruleset: " + rulesetName);
        
        // ... Drools compilation logic goes here ...
        
        return kieContainer; // The large object now stored off-heap
    }
}
```

### Step 6: Eviction Controller

```java title:"off-heap on demand cache eviction" fold:true
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.web.bind.annotation.*;
// ... other imports ...

@RestController
@RequestMapping("/api/cache")
public class CacheManagementController {

    // IMPORTANT: 'cacheManager' must match your @Bean name: "ehcacheOffHeapCacheManager"

    /**
     * Endpoint to evict ALL entries from the rules cache.
     * Use this after deploying new rules.
     * Example: POST /api/cache/rules/evict-all
     */
    @CacheEvict(
        value = "droolsRuleCache", 
        allEntries = true, 
        cacheManager = "ehcacheOffHeapCacheManager"
    )
    @PostMapping("/rules/evict-all")
    public String evictAllRulesCache() {
        return "Rules cache successfully cleared (droolsRuleCache). New rules will be loaded on next access.";
    }

    /**
     * Endpoint to evict a SPECIFIC ruleset by its key.
     * Example: POST /api/cache/rules/evict-key/rulesetA
     */
    @CacheEvict(
        value = "droolsRuleCache", 
        key = "#rulesetName", 
        cacheManager = "ehcacheOffHeapCacheManager"
    )
    @PostMapping("/rules/evict-key/{rulesetName}")
    public String evictSpecificRuleset(@PathVariable String rulesetName) {
        return "Ruleset evicted: " + rulesetName;
    }
}
```

### Step 7: Add Actuator to see both Caffeine and Ehcache 3 

Add Actuator dependency to pom.xml 
```properties title:"add to appliction.properties" fold:true
# Expose the 'caches' endpoint to view all cache statistics
management.endpoints.web.exposure.include=health,info,caches,metrics
```

