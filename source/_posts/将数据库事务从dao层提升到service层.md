title: 将数据库事务从dao层提升到service层
author: 若邪
tags:
 - C#
 - 数据库事务
categories:
 - 后端
date: 2017/7/30
---
学习后端语言的时候，都会涉及到数据库的相关操作，不同语言在操作数据库方面有不同的驱动程序，比如java的JDBC,C#的ADO.NET。当进行数据的新增，更新以及删除的时候，经常需要开启数据库事务。比如ADO.NET是这样使用：

```cs
SqlConnection con = new Sqlconnection("数据库连接语句");
con.Open();
var trans = con.BeginTransaction();
try
{
     SqlCommand com = new SqlCommand(trans);
     //处理插入或更新逻辑
     trans.Commit();
}catch(ex){
     trans.Rollback();//如果前面有异常则事务回滚
}
finally
{
     con.Close();
}
```
很多教程都将事务写在数据访问层(dao层)，但是更多时候我们需要的是业务逻辑层(service层)级别的事务控制。比如我们有一个学生表，一个班级表。学生表存有对应的班级字段，学生与班级表都有对应的dao和service操作类。每个dao只操作相关的数据，不能即操作学生的数据，又操作班级的数据。现在我们要删除一个班级，并且将该班级的学生一并删除。不管是先删除班级还是先删除学生(不存在外键约束)，反正就是要一起删除。因为每个dao只操作单一的对象，这时候dao中进行删除操作的时候开启事务是达不到我们目的。班级删除失败，学生的删除操作是不会回滚的，反之也一样。
删除班级的同时一并删除学生，某一个失败，另一个删除操作回滚。这属于一个业务层的原子操作。在班级的service操作类中可以引入班级和学生的dao进行操作，两个dao的操作放到同一事务中进行操作。

### 连接Id类
```cs
namespace RuoXieTranscation
{
   public class ConnId
   {
       private string _cconId = Guid.NewGuid().ToString().Replace("-", "");
       private DateTime _createTime=DateTime.Now;

       public ConnId()
       {

       }

       public string CconId
       {
           get { return _cconId; }
       }

       public DateTime CreateTime
       {
            get { return _createTime; }
       }
    }
}
```
生成一个guid，后面标识每个连接实例的唯一性。
### 连接类
```cs
namespace RuoXieTranscation
{
    public class DbConnection
    {
        private string _sConnStr = "";
        private ConnId _connId = null;
        private SqlConnection _sqlConnection = null;
        private SqlCommand _sqlCommand = null;

        public ConnId ConnId
        {
            get { return _connId; }
        }

        public SqlCommand SqlCommand
        {
            get { return _sqlCommand; }
        }

        public DbConnection(string connStr)
        {
            _sConnStr = connStr;
        }

        public ConnId ConnOpen()
        {
            try
            {
                this._sqlConnection = new SqlConnection(_sConnStr);
                this._sqlCommand = new SqlCommand();
                _sqlCommand.Connection = this._sqlConnection;
                this._connId = new ConnId();
                _sqlConnection.Open();
            }
            catch (Exception e)
            {
                if (this._sqlConnection.State != System.Data.ConnectionState.Closed)
                {
                    this._sqlConnection.Close();
                    this._sqlConnection.Dispose();
                }
                this._sqlConnection = null;
            }
            return this._connId;
        }

        public void BeginTransaction()
        {
            try
            {
                _sqlCommand.Transaction =
                    _sqlConnection.BeginTransaction(System.Data.IsolationLevel.ReadCommitted,
                        this._connId.CconId);
            }
            catch (Exception e)
            {
                if (this._sqlConnection.State != System.Data.ConnectionState.Closed)
                {
                    this._sqlConnection.Close();
                    this._sqlConnection.Dispose();
                }
                this._sqlConnection = null;
            }

        }

        public void Commit()
        {
            try
            {
                this._sqlCommand.Transaction.Commit();
            }
            catch (Exception e)
            {
                this._sqlCommand.Transaction.Rollback();
            }
        }

        public void Rollback()
        {
            try
            {
                this._sqlCommand.Transaction.Rollback();
            }
            catch (Exception e)
            {
                this._sqlCommand.Transaction.Rollback();
            }
        }

        public void Close()
        {

            if (this._sqlCommand != null)
            {
                this._sqlCommand.Dispose();
            }
            if (this._sqlConnection.State != System.Data.ConnectionState.Closed)
            {
                this._sqlConnection.Close();
                this._sqlConnection.Dispose();
            }

        }
    }
}
```
打开连接后可以显式调用BeginTransaction来决定使用事务
### 连接管理类

```cs
namespace RuoXieTranscation
{
    public class ConnManager
    {
        private static ConcurrentDictionary<string, DbConnection> _cache =
            new ConcurrentDictionary<string, DbConnection>();
        private static ThreadLocal<string> _threadLocal;
        private static readonly string _connStr = @"Password=977865769;Persist Security Info=True;User ID=sa;Initial Catalog=RuoXie;Data Source=5ENALIZN94GYJZZ\SQLEXPRESS";

        static ConnManager()
        {
            _threadLocal=new ThreadLocal<string>();
        }

        public static bool CreateConn()
        {
            DbConnection dbconn = new DbConnection(_connStr);
            ConnId key = dbconn.ConnOpen();
            if (!_cache.ContainsKey(key.CconId))
            {
                _cache.TryAdd(key.CconId, dbconn);
                _threadLocal.Value = key.CconId;
                Console.WriteLine("创建数据库连接,Id: " + key.CconId);
                return true;
            }
            throw new Exception("打开数据库连接失败");
        }
        public static void BeginTransaction()
        {
            var id = GetId();
            if (!_cache.ContainsKey(id))
                throw new Exception("内部错误，链接已丢失");
            _cache[id].BeginTransaction();
        }

        public static void Commit()
        {
            try
            {
                var id = GetId();
                if(!_cache.ContainsKey(id))
                    throw new Exception("内部错误，链接已丢失");
                _cache[id].Commit();
            }
            catch (Exception e)
            {
                throw e;
            }
        }

        public static void Rollback()
        {
            try
            {
                var id = GetId();
                if (!_cache.ContainsKey(id))
                    throw new Exception("内部错误，链接已丢失");
                _cache[id].Rollback();
            }
            catch (Exception e)
            {
                throw e;
            }
        }

        public static void ReleaseConn()
        {
            try
            {
                var id = GetId();
                if (!_cache.ContainsKey(id))
                    throw new Exception("内部错误，链接已丢失");
                _cache[id].Close();
                Remove(id);
            }
            catch (Exception e)
            {
                throw e;
            }
        }

        public static SqlCommand GetSqlCommand()
        {

            var id = GetId();
            if (!_cache.ContainsKey(id))
                throw new Exception("内部错误: 连接已丢失.");
            return _cache[id].SqlCommand;
        }

        private static string GetId()
        {
            var id = _threadLocal.Value;
            if (string.IsNullOrEmpty(id))
            {
                throw new Exception("内部错误: 连接已丢失.");
            }
            return id;
        }

        private static bool Remove(string id)
        {
            if (!_cache.ContainsKey(id)) return false;

            DbConnection dbConnection;

            int index = 0;
            bool result = false;
            while (!(result = _cache.TryRemove(id, out dbConnection)))
            {
                index++;
                Thread.Sleep(20);
                if (index > 3) break;
            }
            return result;
        }
    }
}
```
通过静态属性_cache保存每个连接的Id,_threadLocal保存当前线程中的连接Id，不管一个service中涉及多少个dao操作，都是处于同一线程中，通过_threadLocal就可以取出同一个连接对象进行操作。
### 使用
```cs
public class SQLHelper
    {
        public static int ExecuteNonQuery(string sql, SqlParameter[] parameters = null)
        {
            var command = ConnManager.GetSqlCommand();
            command.CommandText = sql;
            command.CommandType = System.Data.CommandType.Text;
            if (parameters != null)
            {
                command.Parameters.Clear();
                command.Parameters.AddRange(parameters);
            }
            return command.ExecuteNonQuery();
        }

        public static object ExecuteScalar(string sql, SqlParameter[] parameters = null)
        {
            var command = ConnManager.GetSqlCommand();
            command.CommandText = sql;
            command.CommandType = System.Data.CommandType.Text;
            if (parameters != null)
            {
                command.Parameters.Clear();
                command.Parameters.AddRange(parameters);
            }
            return command.ExecuteScalar();
        }
    }
```

```cs
public class StudentDao
    {
        public bool Add(string name, string no)
        {
            string sql = string.Format("insert into T_Student(Name12,No) values(@name,@no)");
            var nameParameter = new SqlParameter("@name", SqlDbType.NVarChar);
            var noParameter = new SqlParameter("@no", SqlDbType.NVarChar);
            nameParameter.Value = name;
            noParameter.Value = no;
            SqlParameter[] paras = new SqlParameter[]{
                nameParameter,noParameter
            };
            return SQLHelper.ExecuteNonQuery(sql, paras) > 0;
        }
    }
```

```cs
public class StudentBll
   {
       private StudentDao mDao;

       public StudentBll()
       {
           mDao=new StudentDao();
       }

       public bool AddStudent(string name, string no)
       {
           return mDao.Add(name, no);
       }
    }
```

```cs
class Program
    {
        static void Main(string[] args)
        {
            test();
            test2();
            test3();
            Console.ReadLine();
        }

        static void test()
        {
            ConnManager.CreateConn();
            ConnManager.BeginTransaction();
            try
            {
                var classService = new ClassBll();
                classService.AddClass("7班");
                ConnManager.Commit();
                ConnManager.ReleaseConn();
            }
            catch (Exception e)
            {
                ConnManager.Rollback();
                ConnManager.ReleaseConn();
            }
        }
        static void test2()
        {
            ConnManager.CreateConn();
            ConnManager.BeginTransaction();
            try
            {
                var classService = new ClassBll();
                var studentService=new StudentBll();
                classService.AddClass("8班");
                studentService.AddStudent("李四","001");
                ConnManager.Commit();
                ConnManager.ReleaseConn();
            }
            catch (Exception e)
            {
                ConnManager.Rollback();
                ConnManager.ReleaseConn();
            }

        }
        static void test3()
        {
            ConnManager.CreateConn();
            //ConnManager.BeginTransaction();
            try
            {
                var classService = new ClassBll();
                var studentService = new StudentBll();
                classService.AddClass("8班");
                studentService.AddStudent("李四", "001");
                //ConnManager.Commit();
                ConnManager.ReleaseConn();
            }
            catch (Exception e)
            {
                //ConnManager.Rollback();
                ConnManager.ReleaseConn();
            }

        }
    }
```
虽然将事务提取到了service层，但是每次都要写这样的代码
```cs
            ConnManager.CreateConn();
            ConnManager.BeginTransaction();
            try
            {
                //业务逻辑调用
                ConnManager.Commit();
                ConnManager.ReleaseConn();
            }
            catch (Exception e)
            {
                ConnManager.Rollback();
                ConnManager.ReleaseConn();
            }
```
使用过spring或者spring.net的应该都知道将事务控制转到业务层事多简单，比如spring.net
```cs
        [Transaction]
        public void DeleteData(string name)
        {
            UserDao.Delete(name);
            AccountDao.Delete(name);
        }
```
只需要在service方法加上Transaction attribute。原理就是AOP编程。