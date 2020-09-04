# JDBC编程

## 查询

```Java
String connectionUrl = "jdbc:sqlserver://127.0.0.1:5346;databaseName=learnjdbc;user=learn;password=learnpassword";
try (Connection conn = DriverManager.getConnection(connectionUrl)) {
    try (PreparedStatement ps = conn.prepareStatement("SELECT id, name, gender, grade FROM students WHERE gender=? AND grade=?")) {
        ps.setObject(1, 1);
        ps.setObject(2, 3);
        try (ResultSet rs = ps.executeQuery()) {
            while (rs.next()) {
                long id = rs.getLong("id");
                String name = rs.getString("name");
                int gender = rs.getInt("gender");
                int grade = rs.getInt("grade");
                System.out.printf("%d %s %d %d\n", id, name, gender, grade);
            }
        }
    }
}
```

## 插入

```Java
try (PreparedStatement ps = conn.prepareStatement("INSERT INTO students (name, gender, grade, score) VALUES (?,?,?,?)")) {
    ps.setObject(1, "Bob");
    ps.setObject(2, 1);
    ps.setObject(3, 3);
    ps.setObject(4, 66);
    ps.executeUpdate();
}
```

## 插入并获取主键

```Java
try (PreparedStatement ps = conn.prepareStatement("INSERT INTO students (name, gender, grade, score) VALUES (?,?,?,?)", Statement.RETURN_GENERATED_KEYS)) {
    ps.setObject(1, "Bob");
    ps.setObject(2, 0);
    ps.setObject(3, 2);
    ps.setObject(4, 55);
    ps.executeUpdate();
    try (ResultSet rs = ps.getGeneratedKeys()) {
        if (rs.next())
            System.out.println(rs.getLong(1));
    }
}
```

## 更新

```Java
try (PreparedStatement ps = conn.prepareStatement("UPDATE students SET name=? WHERE id=?")) {
    ps.setObject(1, "Bob");
    ps.setObject(2, 15);
    ps.executeUpdate();
}
```

## 删除

```Java
try (PreparedStatement ps = conn.prepareStatement("DELETE FROM students WHERE id>?")) {
    ps.setObject(1, 12);
    System.out.println(ps.executeUpdate());
}
```

## 事务

> When a connection is created, it is in auto-commit mode. This means that each individual SQL statement is treated as a transaction and is automatically committed right after it is executed. (To be more precise, the default is for a SQL statement to be committed when it is completed, not when it is executed. A statement is completed when all of its result sets and update counts have been retrieved. In almost all cases, however, a statement is completed, and therefore committed, right after it is executed.)

创建连接时，它处于自动提交模式。**这意味着每个独立的SQL语句都被视为一个事务，并在执行后立即自动提交**。（更确切地说，默认情况下，SQL语句在完成时提交，而不是在执行时提交。当检索了语句的所有结果集和更新技术后，语句就完成了。然而，在几乎所有情况下，语句都是在执行之后立即完成并提交的。）

> A transaction is a set of one or more statements that is executed as a unit, so either all of the statements are executed, or none of the statements is executed.

事务是作为一个单元执行的一条或多条语句的集合，因此要么执行所有语句，要么不执行任何语句。

```Java
Connection conn = openConnection();
try {
    // 关闭自动提交:
    conn.setAutoCommit(false);
    // 执行多条SQL语句:
    insert(); update(); delete();
    // 提交事务:
    conn.commit();
} catch (SQLException e) {
    // 回滚事务:
    conn.rollback();
} finally {
    conn.setAutoCommit(true);
    conn.close();
}
```

## 批量

```Java
try (PreparedStatement ps = conn.prepareStatement("INSERT INTO students (name, gender, grade, score) VALUES (?, ?, ?, ?)")) {
    String[] names = new String[] { "AI", "BI", "CI" };
    for (String name : names) {
        ps.setString(1, name);
        ps.setInt(2, 1);
        ps.setInt(3, 1);
        ps.setInt(4, 66);
        ps.addBatch();
    }
    ps.executeBatch();
}
```

## 连接池

> 场景：向数据库中插入 50 行记录，单次插入 5s，批量插入 3s，连接池-单次插入 2s。

数据库连接池是一种复用Connection的组件，它可以避免反复创建新连接，提高JDBC代码的运行效率。

创建线程是一个昂贵的操作，如果有大量的小任务需要执行，并且频繁地创建和销毁线程，实际上会消耗大量的系统资源，往往创建和消耗线程所耗费的时间比执行任务的时间还长，所以，为了提高效率，可以用线程池。

类似的，在执行JDBC的增删改查的操作时，如果每一次操作都来一次打开连接，操作，关闭连接，那么创建和销毁JDBC连接的开销就太大了。为了避免频繁地创建和销毁JDBC连接，我们可以通过连接池（Connection Pool）复用已经创建好的连接。

JDBC连接池有一个标准的接口`javax.sql.DataSource`，注意这个类位于Java标准库中，但仅仅是接口。要使用JDBC连接池，我们必须选择一个JDBC连接池的实现。目前使用最广泛的是`HikariCP`。**记得顺带导入mysql-connector-java依赖**。

我们需要创建一个DataSource实例，这个实例就是连接池，**注意创建DataSource也是一个非常昂贵的操作，所以通常DataSource实例总是作为一个全局变量存储，并贯穿整个应用程序的生命周期**。

和前面的代码类似，只是获取Connection时，把`DriverManage.getConnection()`改为`ds.getConnection()`。

通过连接池获取连接时，并不需要指定JDBC的相关URL、用户名、口令等信息，因为这些信息已经存储在连接池内部了（创建HikariDataSource时传入的HikariConfig持有这些信息）。**一开始，连接池内部并没有连接，所以，第一次调用ds.getConnection()，会迫使连接池内部先创建一个Connection，再返回给客户端使用。当我们调用conn.close()方法时（在try(resource){...}结束处），不是真正“关闭”连接，而是释放到连接池中，以便下次获取连接时能直接返回**。

**所以，可以将连接池的连接创建理解为是一种“懒”创建，只有真正需要用到一个新的连接时，才在连接池内部创建一个连接并返回，使用完毕后会释放到连接池中**。

因此，连接池内部维护了若干个Connection实例，如果调用ds.getConnection()，就选择一个空闲连接，并标记它为“正在使用”然后返回，如果对Connection调用close()，那么就把连接再次标记为“空闲”从而等待下次调用。这样一来，我们就通过连接池维护了少量连接，但可以频繁地执行大量的SQL语句。

通常连接池提供了大量的参数可以配置，例如，维护的最小、最大活动连接数，指定一个连接在空闲一段时间后自动关闭等，需要根据应用程序的负载合理地配置这些参数。此外，大多数连接池都提供了详细的实时状态以便进行监控。

```Java
public static void main(String[] args) throws SQLException {
    HikariConfig config = new HikariConfig();
    config.setJdbcUrl(jdbcUrl);
    config.setUsername(jdbcUsername);
    config.setPassword(jdbcPassword);
    config.addDataSourceProperty("cachePrepStmts", "true");
    config.addDataSourceProperty("prepStmtCacheSize", "100");
    config.addDataSourceProperty("maximumPoolSize", "10");
    DataSource ds = new HikariDataSource(config);
    List<Student> students = queryStudents(ds);
    students.forEach(System.out::println);
}

static List<Student> queryStudents(DataSource ds) throws SQLException {
    List<Student> students = new ArrayList<>();
    try (Connection conn = ds.getConnection()) {
        try (PreparedStatement ps = conn.prepareStatement("SELECT * FROM students WHERE grade = ? AND score >= ?")) {
            ps.setInt(1, 3);
            ps.setInt(2, 90);
            try (ResultSet rs = ps.executeQuery()) {
                while (rs.next()) {
                    students.add(extractRow(rs));
                }
            }
        }
    }
    return students;
}

static Student extractRow(ResultSet rs) throws SQLException {
    Student std = new Student();
    std.setId(rs.getLong("id"));
    std.setName(rs.getString("name"));
    std.setGender(rs.getBoolean("gender"));
    std.setGrade(rs.getInt("grade"));
    std.setScore(rs.getInt("score"));
    return std;
}
```
