# 从 7 分钟到 10 秒，Mybatis 批处理太强了！



处理批处理的方式有很多种，这里不分析各种方式的优劣，只是概述 ExecutorType.BATCH 这种的用法，另学艺不精，如果有错的地方，还请大佬们指出更正。



## **问题原因**

在公司写项目的时候，有一个自动对账的需求，需要从文件中读取几万条数据插入到数据库中，后续可能跟着业务的增长，会上升到几十万，所以对于插入需要进行批处理操作，下面我们就来看看我是怎么一步一步踩坑的。

**简单了解一下批处理背后的秘密，BatchExecutor**

批处理是 JDBC 编程中的另一种优化手段。JDBC 在执行 SQL 语句时，会将 SQL 语句以及实参通过网络请求的方式发送到数据库，一次执行一条 SQL 语句，一方面会减小请求包的有效负载，另一个方面会增加耗费在网络通信上的时间。



通过批处理的方式，我们就可以在 JDBC 客户端缓存多条 SQL 语句，然后在 flush 或缓存满的时候，将多条 SQL 语句打包发送到数据库执行，这样就可以有效地降低上述两方面的损耗，从而提高系统性能。



不过，有一点需要特别注意：

> 每次向数据库发送的 SQL 语句的条数是有上限的，如果批量执行的时候超过这个上限值，数据库就会抛出异常，拒绝执行这一批 SQL 语句，所以我们需要控制批量发送 SQL 语句的条数和频率。



### **版本1-呱呱坠地**

废话不多说，早先时候项目的代码里就已经存在了批处理的代码，伪代码的样子大概是这样子的：

```java
@Resourceprivate 某Mapper类 mapper实例对象;
private int BATCH = 1000;

  private void doUpdateBatch(Date accountDate, List<某实体类> data) {    
      SqlSession batchSqlSession = null;    
      try {      
          if (data == null || data.size() == 0) {        
              return;      
          }      
          batchSqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH, false);
          for (int index = 0; index < data.size(); index++) {        
              mapper实例对象.更新/插入Method(accountDate, data.get(index).getOrderNo());        
              if (index != 0 && index % BATCH == 0) {          
                 batchSqlSession.commit();          
                 batchSqlSession.clearCache();        
          }      
      }      
          batchSqlSession.commit();    
      } catch (Exception e) {      
          batchSqlSession.rollback();      
          log.error(e.getMessage(), e);    
      } finally {      
          if (batchSqlSession != null) {        
              batchSqlSession.close();      
          }    
      }  
  }
```



我们先来看看上述这种写法的几种问题



**你真的懂commit、clearCache、flushStatements嘛？**



我们先看看官网给出的解释：

![图片](https://mmbiz.qpic.cn/mmbiz/x0kXIOa6owVEc5rMHofJEiaT1hjhctxJOoicA3onD2fEF8ldYpF6mAmjhJMO34aw7GQicwLcBGic5DnEibK0JJ0VScg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



然后我们结合上述写法，它会在判断批处理条数达到1000条的时候会去手动commit，然后又手动clearCache，我们先来看看commit到底都做了一些什么，以下为调用链

```java
  @Override  public void commit() {    
      commit(false);  
  }  

  @Override  public void commit(boolean force) {
      try {      
          executor.commit(isCommitOrRollbackRequired(force));      
          dirty = false;    
      } catch (Exception e) {      
          throw ExceptionFactory.wrapException("Error committing transaction.  Cause: " + e, e);    
      } finally {      
          ErrorContext.instance().reset();    
      }  
  }
  

  private boolean isCommitOrRollbackRequired(boolean force) {    
      // autoCommit默认为false，调用过插入、更新、删除之后的dirty值为true    
      return (!autoCommit && dirty) || force;  
  }

  @Override  public void commit(boolean required) throws SQLException {    
      if (closed) {      
          throw new ExecutorException("Cannot commit, transaction is already closed");    
      }    
      clearLocalCache();    
      flushStatements();    
      if (required) {      
          transaction.commit();    
      }  
  }
```



我们会发现，其实你直接调用commit的情况下，它就已经做了clearLocalCache这件事情，所以大可不必在commit后加上一句clearCache，而且clearCache是做了什么你又知道嘛？就搁这调用！！



![图片](https://mmbiz.qpic.cn/mmbiz/x0kXIOa6owVEc5rMHofJEiaT1hjhctxJOb01BlQqU7ZJYZhx0XpDWOnuR0bXic2PzpDsmLInB3qktmHRbx2oIOGA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



另外flushStatements的作用，官网里也有详细解释：



![图片](https://mmbiz.qpic.cn/mmbiz/x0kXIOa6owVEc5rMHofJEiaT1hjhctxJO2F6jImED1Gs5XVFnPF9EwwRzpS3ocVAbtotbNicPIv7WRhciaYsia6LtQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



此方法的作用就是将前面所有执行过的INSERT、UPDATE、DELETE语句真正刷新到数据库中。底层调用了JDBC的statement.executeBatch方法。



这个方法的返回值通俗来说如果执行的是同一个方法并且执行的是同一条SQL，注意这里的SQL还没有设置参数，也就是说SQL里的占位符'?'还没有被处理成真正的参数，那么每次执行的结果共用一个BatchResult，真正的结果可以通过BatchResult中的getUpdateCounts方法获取。



另外如果执行了SELECT操作，那么会将先前的UPDATE、INSERT、DELETE语句刷新到数据库中。这一点去看BatchExecutor中的doQuery方法即可。



### **反例**

看到这里，我们在来看点反例，你就会觉得这都是啥跟啥啊！！！误人子弟啊，直接在百度搜一段关键字：mybatis ExecutorType.BATCH 批处理，反例如下：



![图片](https://mmbiz.qpic.cn/mmbiz/x0kXIOa6owVEc5rMHofJEiaT1hjhctxJOL8FTWDPePwxoqKQDtJUtIlhrCP8RMSc3NqU8UrmKUINtvTFFOpdUfw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



**不具备通用性**

由于项目中用到批处理的地方肯定不止一个，那每用一次就需要CV一下，0.0 那会不会显得太菜了？能不能一劳永逸？这个时候就得用上Java8中的接口函数了~

### **版本2-初具雏形**

在解决完上述两个问题后，我们的代码版本来到了第2版，你以为这就对了？这就完事了？别急，我们继续往下看！

```java
import lombok.extern.slf4j.Slf4j;
import org.apache.ibatis.session.ExecutorType;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.springframework.stereotype.Component;
import javax.annotation.Resource;import java.util.List;
import java.util.function.ToIntFunction;

@Slf4j
@Component
public class MybatisBatchUtils {    
    /**     
     * 每次处理1000条     
     */    
    private static final int BATCH = 1000;
    @Resource    
    private SqlSessionFactory sqlSessionFactory;    
    /**     
     * 批量处理修改或者插入     
     *     
     * @param data     需要被处理的数据     
     * @param function 自定义处理逻辑    
     * @return int 影响的总行数    
    */    
    public  <T> int batchUpdateOrInsert(List<T> data, ToIntFunction<T> function) {        
        int count = 0;
        SqlSession batchSqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH);
        try {
            for (int index = 0; index < data.size(); index++) {
                count += function.applyAsInt(data.get(index));
                if (index != 0 && index % BATCH == 0) {
                    batchSqlSession.flushStatements();
                }
            }
            batchSqlSession.commit();
        } catch (Exception e) {
            batchSqlSession.rollback();
            log.error(e.getMessage(), e);
        } finally {
            batchSqlSession.close();
        } 
        return count;
    }
}
```



伪代码使用案例

```
@Resourceprivate 某Mapper类 mapper实例对象;
batchUtils.batchUpdateOrInsert(数据集合, item -> mapper实例对象.insert方法(item));
```



这个时候我兴高采烈的收工了，直到过了一两天，导师问我，考虑过这个业务的性能嘛，后续量大了可能每天有十多万笔数据，问我现在每天要多久，我才发现 0.0 两三万条数据插入居然要7分钟（不完全是这个问题导致这么慢，还有Oracle插入语句的原因，下面会描述），，哈哈，笑不活了，简直就是Bug制造机，我就开始思考为什么会这么慢，肯定是批处理没生效，我就思考为什么会没生效？



### **版本3-标准写法**

我们知道上面我们提到了BatchExecutor执行器，我们知道每个SqlSession都会拥有一个Executor对象，这个对象才是执行 SQL 语句的幕后黑手，我们也知道Spring跟Mybatis整合的时候使用的SqlSession是SqlSessionTemplate，默认用的是ExecutorType.SIMPLE，这个时候你通过自动注入获得的Mapper对象其实是没有开启批处理的

```java
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
      executor = (Executor) interceptorChain.pluginAll(executor);
      return executor; 
  }
```



那么我们实际上是需要通过sqlSessionFactory.openSession(ExecutorType.BATCH)得到的sqlSession对象（此时里面的Executor是BatchExecutor）去获得一个新的Mapper对象才能生效！！！



所以我们更改一下这个通用的方法，把MapperClass也一块传递进来

```java
public class MybatisBatchUtils {
    /**
     * 每次处理1000条
     */    
    private static final int BATCH_SIZE = 1000;
    @Resource    
    private SqlSessionFactory sqlSessionFactory;
    /**    
     * 批量处理修改或者插入 
     *  
     * @param data     需要被处理的数据  
     * @param mapperClass  Mybatis的Mapper类  
     * @param function 自定义处理逻辑  
     * @return int 影响的总行数 
     */   
    public <T,U,R> int batchUpdateOrInsert(List<T> data, Class<U> mapperClass, BiFunction<T,U,R> function) { 
        int i = 1;        
        SqlSession batchSqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH); 
        try {      
            U mapper = batchSqlSession.getMapper(mapperClass);           
            int size = data.size();       
            for (T element : data) {       
                function.apply(element,mapper);       
                if ((i % BATCH_SIZE == 0) || i == size) {         
                    batchSqlSession.flushStatements();       \
                }         
                i++;        
            }           
            // 非事务环境下强制commit，事务情况下该commit相当于无效          
            batchSqlSession.commit(!TransactionSynchronizationManager.isSynchronizationActive()); 
        } catch (Exception e) {      
            batchSqlSession.rollback();     
            throw new CustomException(e);     
        } finally {          
            batchSqlSession.close();     
        }       
        return i - 1;  
    }
}
```



这里会判断是否是事务环境，不是的话会强制提交，如果是事务环境的话，这个commit设置force值是无效的，这个在前面的官网截图中有提到。



使用案例：

```
batchUtils.batchUpdateOrInsert(数据集合, xxxxx.class, (item, mapper实例对象) -> mapper实例对象.insert方法(item));
```



**附：Oracle批量插入优化**

我们都知道Oracle主键序列生成策略跟MySQL不一样，我们需要弄一个序列生成器，这里就不详细展开描述了，然后Mybatis Generator生成的模板代码中，insert的id是这样获取的

```
<selectKey keyProperty="id" order="BEFORE" resultType="java.lang.Long">  select XXX.nextval from dual</selectKey>
```



如此，就相当于你插入1万条数据，其实就是insert和查询序列合计预计2万次交互，耗时竟然达到10s多。我们改为用原生的Batch插入，这样子的话，只要500多毫秒，也就是0.5秒的样子

```
<insert id="insert" parameterType="user">        insert into table_name(id, username, password)        values(SEQ_USER.NEXTVAL,#{username},#{password})</insert>
```