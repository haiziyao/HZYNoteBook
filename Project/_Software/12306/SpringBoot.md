
##### 配置驱动

```java
public class FastJsonSafeMode implements InitializingBean {  
  
    @Override  
    public void afterPropertiesSet() throws Exception {  
        System.setProperty("fastjson2.parser.safeMode", "true");  
    }  
}
```

```java
public class ApplicationBaseAutoConfiguration {  
  
    @Bean  
    @ConditionalOnMissingBean    
    @ConditionalOnProperty(value = "framework.fastjson.safa-mode", havingValue = "true")  
    public FastJsonSafeMode congoFastJsonSafeMode() {  
        return new FastJsonSafeMode();  
    }  
}
```