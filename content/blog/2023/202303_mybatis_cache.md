---
title: 'mybatis cache 导致 密码重复解密，报错'
date: 2023-03-31 10:54:00
updated: 2023-03-31 00:58:52
categories: 
- review
tags: 
- cache
- mybatis
description: mybatis cache 导致 密码重复解密，报错
---

> mybatis cache 导致 密码重复解密，报错
<!--more-->

## 背景
生产环境报错：
```java
Error while decrypting: str=123456

// 123456 是明文密码
```

根据 traceId 找到接口，
看了下，业务是：查询获取商户信息，同时会被 db 里面加密的用户名和密码解密出来。
但是，报错显示：传了`明文密码`去解密。

取数据库看了下，没有这个 明文密码，

看一下简化的代码：
```java

@Transactional(rollbackFor = Exception.class, transactionManager = JpaConfig.TRANSACTION_MANAGER)
public void sureConfirmation(...) {
  meetingRepo.saveMeetingBrandActionLog(logs)

}

public void saveMeetingBrandActionLog(logs) { logs.forEach(this::saveMeetingBrandActionLog); }

public void saveMeetingBrandActionLog(log) { 
  // queryInfo 里面会
  // - 1. 去 db 查询 info
  // - 2. 解密 密码
  // 这里执行多次的话，只有第一次会成功，第二次开始，就把明文密码传入 解密方法了。
  BizBrandOperateInfo operateInfo = bizBrandOperateInfoService.queryInfo( 
    BizSourceEnum.fpo, 
    log.getBizBrandId()
  );

```

复现了一下，可以很明显地看出是 myabatis 的缓存问题：
不加 `@Transactional` 注解的话，第二次是正常去 db 查询，然后正常解密的。

![](https://yzp-note.4everland.store/img/2022/09/0bdd0394a8389e1faf8a98c0b66243ff.png)

## 原理
debug 一下：
mybatis 的代理类是：`org.apache.ibatis.binding.MapperProxy`:
![](https://yzp-note.4everland.store/img/2022/09/d3640ff213bcc5ccafd73c0beefc921d.png)
![](https://yzp-note.4everland.store/img/2022/09/d3640ff213bcc5ccafd73c0beefc921d.png)

### MapperProxy#invoke
可以看到，这里做了判断：
如果 proxy 是 原始类，就调用 `method.invoke(this, args)`,
否则，就调用 `cachedInvoker(method).invoke(proxy, method, args, sqlSession)`

```java
// org.apache.ibatis.binding.MapperProxy#invoke
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  try {
  // todo，这是啥
    if (Object.class.equals(method.getDeclaringClass())) {
      return method.invoke(this, args);
    } else {
      // here !!
      return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
    }
  } catch (Throwable t) {
    throw ExceptionUtil.unwrapThrowable(t);
  }
}
```

methodCache 的内容：
- key 是 原始 Method
- value 时 `MapperProxy$PlainMethodInvoker`
  - mapperMethod
    - command 包括 method signature 和 SqlCommandType=`SELECT`
    - method
      - returnsMany, returnsMap, returnsVoid, returnsCursor, returnsOptional 定义了好多返回值的类型啊，为啥不用枚举。
      - paramNameResolver 处理参数映射

![](https://yzp-note.4everland.store/img/2022/09/smart-mybatis-cache-3.png)

### SqlSessionInterceptor#invoke
```Java
// org.mybatis.spring.SqlSessionTemplate.SqlSessionInterceptor#invoke
private class SqlSessionInterceptor implements InvocationHandler {
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    SqlSession sqlSession = getSqlSession(SqlSessionTemplate.this.sqlSessionFactory,
        SqlSessionTemplate.this.executorType, SqlSessionTemplate.this.exceptionTranslator);
    try {
      Object result = method.invoke(sqlSession, args);
      if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
        // force commit even on non-dirty sessions because some databases require
        // a commit/rollback before calling close()
        sqlSession.commit(true);
      }
      return result;
```


### createCacheKey
cacheKey 的组成：
```java
{mappedStatement.id}

// 没有带参数，所有多个同名方法，这里是相同的，
// 那么后面 sql 也可能相同，参数数量也可能相同，
// 然后，再全部传 null。忽略类型的话，两个查询的 cacheKey 貌似就是一样的了。
// 1. todo 再看下这里面几个数字参数的定义。
// 2. 两个不同地方的完全相同的查询，返回相同的结果，也是正常的。
:{原始方法名} 

:{rowBounds.offset}:{rowBounds.limit}
:{boundSql.sql}  // 完整的 sql
:{所有 param value 的拼接}


```


```Java

@Override
public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  CacheKey cacheKey = new CacheKey();
  cacheKey.update(ms.getId());
  cacheKey.update(rowBounds.getOffset());
  cacheKey.update(rowBounds.getLimit());
  cacheKey.update(boundSql.getSql());
  
  
  for (ParameterMapping parameterMapping : parameterMappings) {
  if (parameterMapping.getMode() != ParameterMode.OUT) {
    Object value;
    String propertyName = parameterMapping.getProperty();
    if (boundSql.hasAdditionalParameter(propertyName)) {
      value = boundSql.getAdditionalParameter(propertyName);
    } else if (parameterObject == null) {
      value = null;
    } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
      value = parameterObject;
    } else {
      MetaObject metaObject = configuration.newMetaObject(parameterObject);
      value = metaObject.getValue(propertyName);
    }
    cacheKey.update(value);
  }
}
  
  
  cacheKey 内容：

733920730:-108631220:com.kezaihui.smart.biz.task.dao.BizBrandOperateInfoDao.findByBizBrandId:0:2147483647:select
         
        `id`,
        `source`,
        `biz_brand_id`,
        `biz_brand_name`,
        `biz_brand_alias`,
        `start_date`,
        `expire_date`,
        `expire_type`,
        `shop_count`,
        `create_time`,
        `update_time`,
        `delete_time`,

        `kai_dian_bao_account`,
        `kai_dian_bao_pwd`,
        `kai_dian_bao_info`,
        `category`,
        `budget`,
        `expect`,
        `now_urgent`
     
        from biz_brand_operate_info
        where `source` = ? and biz_brand_id = ?
         
        and delete_time = '1970-01-01 00:00:00':fpo:39365
```


### BaseExecutor localCache

这里可以看到 cache 的内容：
key 就是上面的 CacheKey
value 是**上次**查询的结果。
![](https://yzp-note.4everland.store/img/2022/09/smart-mybatis-cache-4.png)


### newExecutor
如果 cacheEnabled=true 的话，就会去走 cachingExecutor 的逻辑。

```Java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
  executorType = executorType == null ? defaultExecutorType : executorType;
  executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
  Executor executor;
  if (ExecutorType.BATCH == executorType) {
    executor = new BatchExecutor(this, transaction);
  } else if (ExecutorType.REUSE == executorType) {
    executor = new ReuseExecutor(this, transaction);
  } else {
    executor = new SimpleExecutor(this, transaction);
  }
  if (cacheEnabled) { 
    executor = new CachingExecutor(executor);
  }
  // 这里，pagehelper interceptor 注册上去了。
  executor = (Executor) interceptorChain.pluginAll(executor);
  return executor;
}

```

![](https://yzp-note.4everland.store/img/2022/09/smart-mybatis-20220903182817.png)


### 一级缓存 具体实现

```java
// 每次查询结束后，
// 默认 put 缓存。

// 如果 配置了 localCacheScope = statement 的话，
// 再去 clear cache？？？
if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
  // issue #482
  clearLocalCache();
}

```


完整一点的：
```Java
@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
  ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  if (queryStack == 0 && ms.isFlushCacheRequired()) {
    clearLocalCache();
  }
  List<E> list;
  try {
    queryStack++;
    list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
    if (list != null) {
      handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
    } else {
      list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
    }
  } finally {
    queryStack--;
  }
  if (queryStack == 0) {
    for (DeferredLoad deferredLoad : deferredLoads) {
      deferredLoad.load();
    }
    // issue #601
    deferredLoads.clear();
    if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
      // ##### 2. clear cache
      // issue #482
      clearLocalCache();
    }
  }
  return list;
}
```

#### `flushLocalCacheAfterEachStatement` -> `localCacheScope`

issue 482 这个注释，在这个 pr 中 引入：
https://github.com/mybatis/mybatis-3/commit/b32ac15d156e745ce3df0cafdd5e12272b1db481

改动是 配置参数：
```java
// before
flushLocalCacheAfterEachStatement=false|true
// after
localCacheScope=session|statement
```

```java
// before
configuration.setClearLocalCacheAfterEachStatement(booleanValueOf(props.getProperty("flushLocalCacheAfterEachStatement"), false));
// after
configuration.setLocalCacheScope(LocalCacheScope.valueOf(stringValueOf(props.getProperty("localCacheScope"), "SESSION")));
```

一个很好的实践：
很多时候，不要用 bool，优先选择枚举，扩展性更高。

一个坏的实践：
虽然参数名字上叫做 `Scope`，但是实现上，做的还是，`flush cache after statement`。
参数改了，但是实现没有同步更改，代码看起来，令人困惑，增加了认知成本。
简单来说，他们选择了 `总改动量更小`的方案。
mybatis 好感度 -1。

>[形成代码屎山的一大原因，就是`总代码量更小`的方案和`总改动量更小`的方案 PK 的时候，出于主观或客观的因素，往往是后者胜出。](https://twitter.com/disksing/status/1562612769903366144?s=20&t=2rdLzhbjJhvR6GDGsxnXgg)

实现应该在 put 的地方，判断是否需要存入 cache，
而不是现在这样，默认 cache，然后，如果 `localCacheScope=STATEMENT` 的话，再 `flush cache`.



### 读取 localCacheScope 的时机
在 注册 SqlSessionFactory 的时候，就读取了 localCacheScope 配置。
也就是说，应用启动后，这个配置就 apply 了，要让新的配置生效的话，只能重启。

没有找到 refresh config 的渠道。

也不能在执行代码时，决定这次 session 或者 statement 是否 cache。
可以叫做： `tunable cache level`

### mybatis 的国产周边项目
debug 的过程中，突然看到了中文注释，很奇怪，再一看，是国内的 增强项目：
`mybatis-plus` ，`pagehelper`

看了下面两段注释，难受。
mybatis 好感度 -1。
![](https://yzp-note.4everland.store/img/2022/09/smart-mybatis-cache-20220903184808.png)

![](https://yzp-note.4everland.store/img/2022/09/smart-mybatis-cache-202209031848147.png)

## 解决
关闭 session 级别的缓存。
```xml
// 默认值是 session
<setting name="localCacheScope" value="STATEMENT"/>
```
## summary
-   不建议用 cache，在`分布式环境下`，在有 `可变对象`的时候。
> shared mutable state is the root of all evil
-   ORM 缓存配置 默认关闭，需要的时候，手动配置在 ~~方法级别~~ 请求级别。
-   is ORM cache a good/bad design?

![](https://yzp-note.4everland.store/img/2022/11/202211100050746.png)

![](https://yzp-note.4everland.store/img/2022/11/202211100050209.png)



## 类比
share nothing 架构
https://zhuanlan.zhihu.com/p/386257529

share nothing in DDIA
https://github.com/Vonng/ddia/blob/master/part-ii.md

redis 为什么不用多线程

## ref
https://tech.meituan.com/2018/01/19/mybatis-cache.html