---
layout: post
title: 数据库连接
date: 2021-08-20 20:30:32 +0800
categories: 学习笔记
---

### 短连接

查询之前创建连接，SQL执行结束之后断开连接。

ShortConnection.java

```
package org.example;

import com.google.common.collect.Maps;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import java.sql.*;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

public class ShortConnection {
    private static Log LOG = LogFactory.getLog(ShortConnection.class);

    private static String jdbcTemplate  = "jdbc:mysql://%s:%s/?user=%s&password=%s";
    private String jdbcUrl;

    static {
        try {
            Class.forName("com.mysql.jdbc.Driver");
        } catch (ClassNotFoundException e) {
            LOG.warn("class not found for jdbc driver", e);
        }
    }

    public void init(String url, int port, String user, String password) throws SQLException {
        this.jdbcUrl = String.format(jdbcTemplate, url, port, user, password);
    }

    public List<Map<String, String>> selectData(String sql) throws SQLException {
        List<Map<String, String>> data = new ArrayList<Map<String, String>>();
        try (Connection connection = DriverManager.getConnection(jdbcUrl);
             Statement statement = connection.createStatement()) {  // 在try语句中创建的Connection和Statement对象，不需要手动关闭，代码块结束之后会自动关闭。
            try (ResultSet resultSet = statement.executeQuery(sql)) { // 在try语句中创建的ResultSet对象，不需要手动关闭，代码块结束之后会自动关闭。
                ResultSetMetaData meta = resultSet.getMetaData();
                int columeCount = meta.getColumnCount();
                while (resultSet.next()) {
                    Map<String, String> line = Maps.newHashMap();
                    for (int i = 1; i <= columeCount; i++) {
                        line.put(meta.getColumnName(i), resultSet.getString(i));
                    }
                    data.add(line);
                }
            }
            return data;
        } catch (SQLException e) {
            throw e;
        }
    }
}
```

Main.java

```
package org.example;

import java.sql.SQLException;
import java.util.List;
import java.util.Map;

public class Main
{
    public static void main( String[] args )
    {
        shortConnectionTest();
    }

    public static void shortConnectionTest() {
        ShortConnection jdbcService = new ShortConnection();
        try {
            jdbcService.init("IP", 端口, "用户名", "密码");
            String sql = "select * from test_db.table_test limit 5;";
            List<Map<String, String>> data = jdbcService.selectData(sql);
            int i = 1;
            for (Map<String, String> line : data) {
                System.out.println("Line id : " + i++);
                for (String column : line.keySet()) {
                    System.out.println("column= "+ column + ", value= " + line.get(column));
                }
            }
            System.out.println("row number : " + data.size());
        } catch (SQLException e) {
            System.out.println("execute sql failed" + e);
        }
    }
}
```

在pom.xml文件中配置相关的依赖项。

```
    <!--  https://mvnrepository.com/artifact/com.google.guava/guava  -->
    <dependency>
      <groupId>com.google.guava</groupId>
      <artifactId>guava</artifactId>
      <version>23.0</version>
    </dependency>
    <!--  https://mvnrepository.com/artifact/org.apache.commons/commons-io  -->
    <dependency>
      <groupId>org.apache.commons</groupId>
      <artifactId>commons-io</artifactId>
      <version>1.3.2</version>
    </dependency>
    <dependency>
      <groupId>commons-logging</groupId>
      <artifactId>commons-logging</artifactId>
      <version>1.2</version>
    </dependency>
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>5.1.39</version>
    </dependency>
```

### 长连接

查询之前创建连接，SQL执行结束之后保持连接，直到所有SQL都执行结束之后再关闭连接。

LongConnection.java

```
package org.example;

import com.google.common.collect.Maps;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import java.sql.*;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

public class LongConnection {
    private static Log LOG = LogFactory.getLog(LongConnection.class);
    private static String jdbcTemplate  = "jdbc:mysql://%s:%s/?user=%s&password=%s";
    private String jdbcUrl;
    private Connection connection;

    static {
        try {
            Class.forName("com.mysql.jdbc.Driver");
        } catch (ClassNotFoundException e) {
            LOG.warn(e);
        }
    }

    public void init(String url, int port, String user, String password) throws SQLException {
        this.jdbcUrl = String.format(jdbcTemplate, url, port, user, password);
        try {
            connection = DriverManager.getConnection(jdbcUrl);
        } catch (SQLException e) {
            LOG.error("mysql connection failed", e);
            throw e;
        }
    }

    public void closeConnection() throws SQLException {
        connection.close();
    }

    public boolean connectionValidate()
    {
        boolean isValidated = true;
        try {
            Statement statement = connection.createStatement();
            statement.executeQuery("SELECT 1");
        } catch (SQLException e) {
            isValidated = false;
        }
        return isValidated;
    }

    public List<Map<String, String>> selectData(String sql) throws SQLException {
        List<Map<String, String>> data = new ArrayList<Map<String, String>>();
        try (Statement statement = connection.createStatement()) {
            try (ResultSet resultSet = statement.executeQuery(sql)) {
                ResultSetMetaData meta = resultSet.getMetaData();
                int columeCount = meta.getColumnCount();
                while (resultSet.next()) {
                    Map<String, String> line = Maps.newHashMap();
                    for (int i = 1; i <= columeCount; i++) {
                        line.put(meta.getColumnName(i), resultSet.getString(i));
                    }
                    data.add(line);
                }
            }
            return data;
        } catch (SQLException e) {
            throw e;
        }
    }
}
```

Main.java

```
package org.example;

import java.sql.SQLException;
import java.util.List;
import java.util.Map;

public class Main
{
    public static void main( String[] args )
    {
        try {
            longConnectionTest();
        } catch (Exception e) {
            System.out.println("Failed to close Connection!");
        }
    }

    public static void longConnectionTest() throws SQLException {
        LongConnection jdbcService = new LongConnection();
        try {
            jdbcService.init("IP", 端口, "用户名", "密码");
            for (int j = 0; j < 5; j++) {
                System.out.println("Connection Valid:" + jdbcService.connectionValidate());
                String sql = "select * from test_db.table_test limit 3;";
                List<Map<String, String>> data = jdbcService.selectData(sql);
                int i = 1;
                for (Map<String, String> line : data) {
                    System.out.println("Line id : " + i++);
                    for (String column : line.keySet()) {
                        System.out.println("column= " + column + ", value= " + line.get(column));
                    }
                }
                System.out.println("row number : " + data.size());
            }
        } catch (SQLException e) {
            System.out.println("execute sql failed" + e);
        } finally {
            jdbcService.closeConnection();
        }
    }
}
```

在pom.xml文件中配置相关的依赖项。

```
    <!--  https://mvnrepository.com/artifact/com.google.guava/guava  -->
    <dependency>
      <groupId>com.google.guava</groupId>
      <artifactId>guava</artifactId>
      <version>23.0</version>
    </dependency>
    <!--  https://mvnrepository.com/artifact/org.apache.commons/commons-io  -->
    <dependency>
      <groupId>org.apache.commons</groupId>
      <artifactId>commons-io</artifactId>
      <version>1.3.2</version>
    </dependency>
    <dependency>
      <groupId>commons-logging</groupId>
      <artifactId>commons-logging</artifactId>
      <version>1.2</version>
    </dependency>
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>5.1.39</version>
    </dependency>
```

### 连接池

查询之前从连接池申请连接，查询结束之后将连接归还给连接池。

ConnectionPool.java

```
package org.example;

import com.google.common.collect.Maps;
import org.apache.commons.dbcp.BasicDataSource;

import java.sql.*;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

public class ConnectionPool {
    private static BasicDataSource bds;

    public ConnectionPool() {
        //创建核心类
        bds = new BasicDataSource();
        //配置4个基本参数
        bds.setDriverClassName("com.mysql.jdbc.Driver");
        bds.setUrl("jdbc:mysql://IP:端口");
        bds.setUsername("用户名");
        bds.setPassword("密码");

        //管理连接配置
        bds.setMaxActive(5); //最大活动数
        bds.setMaxIdle(5);  //最大空闲数
        bds.setMinIdle(1);  //最小空闲数
        bds.setInitialSize(5);  //初始化个数
        bds.setValidationQuery("SELECT 1");
    }

    public Connection getConnection() {
        //获取连接
        try {
            return bds.getConnection();
        } catch (SQLException e) {
            throw new RuntimeException(e);
        }
    }

    public List<Map<String, String>> selectData(Connection connection, String sql) throws SQLException {
        List<Map<String, String>> data = new ArrayList<Map<String, String>>();
        try (Statement statement = connection.createStatement()) {
            try (ResultSet resultSet = statement.executeQuery(sql)) {
                ResultSetMetaData meta = resultSet.getMetaData();
                int columeCount = meta.getColumnCount();
                while (resultSet.next()) {
                    Map<String, String> line = Maps.newHashMap();
                    for (int i = 1; i <= columeCount; i++) {
                        line.put(meta.getColumnName(i), resultSet.getString(i));
                    }
                    data.add(line);
                }
            }
            return data;
        } catch (SQLException e) {
            throw e;
        }
    }
}
```

Main.java

```
package org.example;

import java.sql.Connection;
import java.sql.SQLException;
import java.util.List;
import java.util.Map;

public class Main
{
    public static void main( String[] args )
    {
        try {
            connectionPoolTest();
        } catch (Exception e) {
            System.out.println("Failed to return connection to pool!");
        }
    }

    public static void connectionPoolTest() throws SQLException {
        ConnectionPool connectionPool = new ConnectionPool();
        Connection connection = connectionPool.getConnection();
        try {
            for (int j = 0; j < 5; j++) {
                System.out.println("Connection Valid:" + !connection.isClosed());
                String sql = "select * from test_db.table_test limit 3;";
                List<Map<String, String>> data = connectionPool.selectData(connection, sql);
                int i = 1;
                for (Map<String, String> line : data) {
                    System.out.println("Line id : " + i++);
                    for (String column : line.keySet()) {
                        System.out.println("column= " + column + ", value= " + line.get(column));
                    }
                }
                System.out.println("row number : " + data.size());
                connection.close();
            }
        } catch (SQLException e) {
            System.out.println("execute sql failed" + e);
        } finally {
            connection.close(); // 将connection归还给连接池，并不是断开连接
        }
    }
}
```

在pom.xml文件中配置相关的依赖项。

```
    <!--  https://mvnrepository.com/artifact/com.google.guava/guava  -->
    <dependency>
      <groupId>com.google.guava</groupId>
      <artifactId>guava</artifactId>
      <version>23.0</version>
    </dependency>
    <!--  https://mvnrepository.com/artifact/org.apache.commons/commons-io  -->
    <dependency>
      <groupId>org.apache.commons</groupId>
      <artifactId>commons-io</artifactId>
      <version>1.3.2</version>
    </dependency>
    <dependency>
      <groupId>commons-logging</groupId>
      <artifactId>commons-logging</artifactId>
      <version>1.2</version>
    </dependency>
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>5.1.39</version>
    </dependency>
    <dependency>
      <groupId>commons-dbcp</groupId>
      <artifactId>commons-dbcp</artifactId>
      <version>1.2.2</version>
    </dependency>
```