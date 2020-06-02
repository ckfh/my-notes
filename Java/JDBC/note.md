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
