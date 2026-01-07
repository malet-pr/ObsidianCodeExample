
```java title:"implements ApplicationListener" fold:true
import org.springframework.cache.Cache;  
import org.springframework.cache.CacheManager;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.context.ApplicationListener;  
import org.springframework.context.event.ContextRefreshedEvent;  
import org.springframework.stereotype.Component;  
import java.util.UUID;  
  
@Component  
public class CachePopulatorListener implements ApplicationListener<ContextRefreshedEvent> {  
  
    // Spring will auto-wire the primary CacheManager here  
    private final CacheManager cacheManager;  
    private static final int TARGET_ENTRIES = 30000;  
  
    // Constructor Injection  
    @Autowired  
    public CachePopulatorListener(CacheManager cacheManager) {  
        this.cacheManager = cacheManager;  
    }  
      
    /**  
     * This method runs only once, AFTER the entire Spring application context     * is fully refreshed, initialized, and all beans are available.     */    @Override  
    public void onApplicationEvent(ContextRefreshedEvent event) {  
  
        // 1. FINAL Lookup: Should now work guaranteed  
        Cache pruebaCache = cacheManager.getCache("pruebaCache");  
  
        if (pruebaCache == null) {  
            System.err.println("FATAL ERROR: 'pruebaCache' still not found after context refresh. Check wiring.");  
            return;  
        }  
  
        System.out.println("--- Starting Off-Heap Cache Population (Context Refreshed) ---");  
        long startTime = System.currentTimeMillis();  
  
        // 2. Population Logic  
        for (int i = 0; i < TARGET_ENTRIES; i++) {  
            String key = "testKey-" + i;  
            String value = UUID.randomUUID().toString() + UUID.randomUUID().toString();  
            pruebaCache.put(key, value);  
        }  
  
        long duration = System.currentTimeMillis() - startTime;  
        System.out.printf(  
                "--- Cache Population Complete. %d entries inserted in %dms. ---%n",  
                TARGET_ENTRIES,  
                duration  
        );  
    }  
}
```