title: 利用AOP实现SqlSugar自动事务
author: 若邪
tags:
 - AOP
 - SqlSugar
 - 事务
 - Castle.DynamicProxy
categories:
 - 后端
date: 2017/10/25 0025
---
先看一下效果,带接口层的三层架构:

BL层：

``` cs
  public class StudentBL : IStudentService
      {
          private ILogger mLogger;
          private readonly IStudentDA mStudentDa;
          private readonly IValueService mValueService;

          public StudentService(IStudentDA studentDa,IValueService valueService)
          {
              mLogger = LogManager.GetCurrentClassLogger();
              mStudentDa = studentDa;
              mValueService = valueService;

          }

          [TransactionCallHandler]
          public IList<Student> GetStudentList(Hashtable paramsHash)
          {
              var list = mStudentDa.GetStudents(paramsHash);
              var value = mValueService.FindAll();
              return list;
          }
      }
```
假设GetStudentList方法里的mStudentDa.GetStudents和mValueService.FindAll不是查询操作，而是更新操作，当一个失败另一个需要回滚，就需要在同一个事务里，当一个出现异常就要回滚事务。

特性TransactionCallHandler就表明当前方法需要开启事务，并且当出现异常的时候回滚事务,方法执行完后提交事务。

DA层：
``` cs
 public class StudentDA : IStudentDA
     {

         private SqlSugarClient db;
         public StudentDA()
         {
             db = SugarManager.GetInstance().SqlSugarClient;
         }
         public IList<Student> GetStudents(Hashtable paramsHash)
         {
             return db.Queryable<Student>().AS("T_Student").With(SqlWith.NoLock).ToList();
         }
     }
```


### 对[SqlSugar](https://github.com/sunkaixuan/sqlsugar/)做一下包装

``` cs
 public class SugarManager
     {
         private static ConcurrentDictionary<string,SqlClient> _cache =
             new ConcurrentDictionary<string, SqlClient>();
         private static ThreadLocal<string> _threadLocal;
         private static readonly string _connStr = @"Data Source=localhost;port=3306;Initial Catalog=thy;user id=root;password=xxxxxx;Charset=utf8";
         static SugarManager()
         {
             _threadLocal = new ThreadLocal<string>();
         }

         private static SqlSugarClient CreatInstance()
         {
             SqlSugarClient client = new SqlSugarClient(new ConnectionConfig()
             {
                 ConnectionString = _connStr, //必填
                 DbType = DbType.MySql, //必填
                 IsAutoCloseConnection = true, //默认false
                 InitKeyType = InitKeyType.SystemTable
             });
             var key=Guid.NewGuid().ToString().Replace("-", "");
             if (!_cache.ContainsKey(key))
             {
                 _cache.TryAdd(key,new SqlClient(client));
                 _threadLocal.Value = key;
                 return client;
             }
             throw new Exception("创建SqlSugarClient失败");
         }
         public static SqlClient GetInstance()
         {
             var id= _threadLocal.Value;
             if (string.IsNullOrEmpty(id)||!_cache.ContainsKey(id))
                 return new SqlClient(CreatInstance());
             return _cache[id];
         }


         public static void Release()
         {
             try
             {
                 var id = GetId();
                 if (!_cache.ContainsKey(id))
                     return;
                 Remove(id);
             }
             catch (Exception e)
             {
                 throw e;
             }
         }
         private static bool Remove(string id)
         {
             if (!_cache.ContainsKey(id)) return false;

             SqlClient client;

             int index = 0;
             bool result = false;
             while (!(result = _cache.TryRemove(id, out client)))
             {
                 index++;
                 Thread.Sleep(20);
                 if (index > 3) break;
             }
             return result;
         }
         private static string GetId()
         {
             var id = _threadLocal.Value;
             if (string.IsNullOrEmpty(id))
             {
                 throw new Exception("内部错误: SqlSugarClient已丢失.");
             }
             return id;
         }

         public static void BeginTran()
         {
             var instance=GetInstance();
             //开启事务
             if (!instance.IsBeginTran)
             {
                 instance.SqlSugarClient.Ado.BeginTran();
                 instance.IsBeginTran = true;
             }
         }

         public static void CommitTran()
         {
             var id = GetId();
             if (!_cache.ContainsKey(id))
                 throw new Exception("内部错误: SqlSugarClient已丢失.");
             if (_cache[id].TranCount == 0)
             {
                 _cache[id].SqlSugarClient.Ado.CommitTran();
                 _cache[id].IsBeginTran = false;
             }
         }

         public static void RollbackTran()
         {
             var id = GetId();
             if (!_cache.ContainsKey(id))
                 throw new Exception("内部错误: SqlSugarClient已丢失.");
             _cache[id].SqlSugarClient.Ado.RollbackTran();
             _cache[id].IsBeginTran = false;
             _cache[id].TranCount = 0;
         }

         public static void TranCountAddOne()
         {
             var id = GetId();
             if (!_cache.ContainsKey(id))
                 throw new Exception("内部错误: SqlSugarClient已丢失.");
             _cache[id].TranCount++;
         }
         public static void TranCountMunisOne()
         {
             var id = GetId();
             if (!_cache.ContainsKey(id))
                 throw new Exception("内部错误: SqlSugarClient已丢失.");
             _cache[id].TranCount--;
         }
     }
```
_cache保存SqlSugar实例，_threadLocal确保同一线程下取出的是同一个SqlSugar实例。


不知道SqlSugar判断当前实例是否已经开启事务，所以又将SqlSugar包了一层。
``` cs
  public class SqlClient
     {
         public SqlSugarClient SqlSugarClient;
         public bool IsBeginTran = false;
         public int TranCount = 0;

         public SqlClient(SqlSugarClient sqlSugarClient)
         {
             this.SqlSugarClient = sqlSugarClient;
         }
     }
```
IsBeginTran标识当前SqlSugar实例是否已经开启事务，TranCount是一个避免事务嵌套的计数器。

一开始的例子

``` cs
            [TransactionCallHandler]
            public IList<Student> GetStudentList(Hashtable paramsHash)
            {
                var list = mStudentDa.GetStudents(paramsHash);
                var value = mValueService.FindAll();
                return list;
            }
```
TransactionCallHandler表明该方法要开启事务，但是如果mValueService.FindAll也标识了TransactionCallHandler,又要开启一次事务？所以用TranCount做一个计数。

### 使用[Castle.DynamicProxy](https://github.com/castleproject/Core)

要实现标识了TransactionCallHandler的方法实现自动事务，就要代理BL类。这里使用Castle.DynamicProxy

Castle.DynamicProxy一般操作
``` cs
  public class MyClass : IMyClass
   {
       public void MyMethod()
       {
           Console.WriteLine("My Mehod");
       }
  }
  public class TestIntercept : IInterceptor
      {
          public void Intercept(IInvocation invocation)
          {
              Console.WriteLine("before");
              invocation.Proceed();
              Console.WriteLine("after");
          }
      }

   var proxyGenerate = new ProxyGenerator();
   TestIntercept t=new TestIntercept();
   var pg = proxyGenerate.CreateClassProxy<MyClass>(t);
   pg.MyMethod();
   //输出是
   //before
   //My Mehod
   //after
```
before就是要开启事务的地方，after就是提交事务的地方
最后实现
``` cs
  public class TransactionInterceptor : IInterceptor
      {
          private readonly ILogger logger;
          public TransactionInterceptor()
          {
              logger = LogManager.GetCurrentClassLogger();
          }
          public void Intercept(IInvocation invocation)
          {
              MethodInfo methodInfo = invocation.MethodInvocationTarget;
              if (methodInfo == null)
              {
                  methodInfo = invocation.Method;
              }

              TransactionCallHandlerAttribute transaction =
                  methodInfo.GetCustomAttributes<TransactionCallHandlerAttribute>(true).FirstOrDefault();
              if (transaction != null)
              {
                  SugarManager.BeginTran();
                  try
                  {
                      SugarManager.TranCountAddOne();
                      invocation.Proceed();
                      SugarManager.TranCountMunisOne();
                      SugarManager.CommitTran();
                  }
                  catch (Exception e)
                  {
                      SugarManager.RollbackTran();
                      logger.Error(e);
                      throw e;
                  }

              }
              else
              {
                  invocation.Proceed();
              }
          }
      }
```

``` cs
     [AttributeUsage(AttributeTargets.Method, Inherited = true)]
     public class TransactionCallHandlerAttribute : Attribute
     {
         public TransactionCallHandlerAttribute()
         {

         }
     }

```
### [Autofac](https://github.com/autofac/Autofac)与Castle.DynamicProxy结合使用

创建代理的时候一个BL类就要一次操作
``` cs
 proxyGenerate.CreateClassProxy<MyClass>(t);
```
而且项目里BL类的实例化是交给IOC容器控制的，我用的是Autofac。当然Autofac和Castle.DynamicProxy是可以结合使用的
``` cs
using System.Reflection;
using Autofac;
using Autofac.Extras.DynamicProxy;
using Module = Autofac.Module;
public class BusinessModule : Module
    {
        protected override void Load(ContainerBuilder builder)
        {
            var business = Assembly.Load("FTY.Business");
            builder.RegisterAssemblyTypes(business)
                .AsImplementedInterfaces().InterceptedBy(typeof(TransactionInterceptor)).EnableInterfaceInterceptors();
            builder.RegisterType<TransactionInterceptor>();
        }
    }

```









