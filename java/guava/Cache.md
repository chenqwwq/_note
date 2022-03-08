# Guava Cache 





## 使用示例

```java
// Guava 推荐使用 CacheBuilder 来创建本地缓存对象
LoadingCache<String, String> build = CacheBuilder.newBuilder()
  // 最多的缓存个数
  .maximumSize(2)
  // 在写入后多久会被淘汰
  .expireAfterWrite(1, TimeUnit.MINUTES)
  // 多久没有被访问会被淘汰
  .expireAfterAccess(2,TimeUnit.DAYS)
  // 缓存被淘汰的监听器
  .removalListener(new RemovalListener<String, String>() {
    @Override
    public void onRemoval(RemovalNotification<String, String> notification) {
      // 监听器包括了原因 notification.getCause()，例如因为 maximumSize 被淘汰会显示为 SIZE
      System.out.printf("[%s:%s - cause:%s]\n ",notification.getKey(),notification.getValue(),notification.getCause());
    }
  })
  // 创建，并传入 CacheLoader 的实现方法，通过 LoadingCache.get(Key) 没获取到值的时候会调用该方法
  .build(new CacheLoader<String, String>() {
    @Override
    public String load(String key) throws Exception {
      return key + "+2";
    }
  });
```

> 简单使用。