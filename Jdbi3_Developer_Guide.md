# Jdbi 3 开发指南

<a name="1___1__Jdbi_简介"></a>
## 1. Jdbi 简介

Jdbi提供了对Java中关系数据的方便、惯用的访问。Jdbi 3是第三个主要版本，它引入了对Java 8的增强支持，对设计和实现的无数改进，以及对模块化插件的增强支持。

> **💡提示:** 不想升级了吗?[v2文档](jdbi2/index.html)仍然可用。

Jdbi构建在JDBC之上。如果您的数据库有JDBC驱动程序，则可以使用Jdbi。Jdbi改进了JDBC的粗糙接口，提供了更自然的Java数据库接口，易于绑定到域数据类型。与ORM不同的是，我们的目标不是提供一个完整的对象关系映射框架—与隐藏的复杂性不同，我们提供的构建块允许您根据您的应用程序构建关系和对象之间的映射。

Jdbi的API有两种形式:

<a name="2____1_1__流式_API"></a>
### 1.1. 流式 API

Core API提供了流畅的命令式接口。使用Builder样式对象将SQL连接到富Java数据类型。

```java
Jdbi jdbi = Jdbi.create("jdbc:h2:mem:test"); // (H2 in-memory database)

List<User> users = jdbi.withHandle(handle -> {
    handle.execute("CREATE TABLE user (id INTEGER PRIMARY KEY, name VARCHAR)");

    // Inline positional parameters
    handle.execute("INSERT INTO user(id, name) VALUES (?, ?)", 0, "Alice");

    // Positional parameters
    handle.createUpdate("INSERT INTO user(id, name) VALUES (?, ?)")
            .bind(0, 1) // 0-based parameter indexes
            .bind(1, "Bob")
            .execute();

    // Named parameters
    handle.createUpdate("INSERT INTO user(id, name) VALUES (:id, :name)")
            .bind("id", 2)
            .bind("name", "Clarice")
            .execute();

    // Named parameters from bean properties
    handle.createUpdate("INSERT INTO user(id, name) VALUES (:id, :name)")
            .bindBean(new User(3, "David"))
            .execute();

    // Easy mapping to any type
    return handle.createQuery("SELECT * FROM user ORDER BY name")
            .mapToBean(User.class)
            .list();
});

assertThat(users).containsExactly(
        new User(0, "Alice"),
        new User(1, "Bob"),
        new User(2, "Clarice"),
        new User(3, "David"));
```

<a name="3____1_2__声明式_API"></a>
### 1.2. 声明式 API

SQL Object扩展位于Core之上，并提供了一个声明式接口。通过声明一个带注释的Java“接口”，告诉Jdbi要执行什么SQL以及您喜欢的结果的形状，并且提供实现。

```java
// Define your own declarative interface
public interface UserDao {
    @SqlUpdate("CREATE TABLE user (id INTEGER PRIMARY KEY, name VARCHAR)")
    void createTable();

    @SqlUpdate("INSERT INTO user(id, name) VALUES (?, ?)")
    void insertPositional(int id, String name);

    @SqlUpdate("INSERT INTO user(id, name) VALUES (:id, :name)")
    void insertNamed(@Bind("id") int id, @Bind("name") String name);

    @SqlUpdate("INSERT INTO user(id, name) VALUES (:id, :name)")
    void insertBean(@BindBean User user);

    @SqlQuery("SELECT * FROM user ORDER BY name")
    @RegisterBeanMapper(User.class)
    List<User> listUsers();
}
Jdbi jdbi = Jdbi.create("jdbc:h2:mem:test");
jdbi.installPlugin(new SqlObjectPlugin());

// Jdbi implements your interface based on annotations
List<User> userNames = jdbi.withExtension(UserDao.class, dao -> {
    dao.createTable();

    dao.insertPositional(0, "Alice");
    dao.insertPositional(1, "Bob");
    dao.insertNamed(2, "Clarice");
    dao.insertBean(new User(3, "David"));

    return dao.listUsers();
});

assertThat(userNames).containsExactly(
        new User(0, "Alice"),
        new User(1, "Bob"),
        new User(2, "Clarice"),
        new User(3, "David"));
```

Jdbi有一个灵活的插件架构，可以很容易地支持你喜欢的库(Guava, jodatatime, Spring, Vavr)或数据库供应商(H2, Oracle, Postgres)。

Jdbi不是ORM。没有会话缓存、更改跟踪、“视图中打开的会话”，也没有诱使库理解您的模式。

相反，Jdbi提供了SQL和简单表格数据结构之间的直接映射。

您可以使用自己的SQL，而Jdbi只运行您告诉它的命令——按照上帝希望的方式。

> **💡提示:**已经在使用Jdbi v2了吗?参见[从v2升级到v3](#_upgrading_from_v2_to_v3).

<a name="4___2__开始"></a>
## 2. 开始

Jdbi很容易包含在您的Java项目 - 一个[Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0.html)许可证,一些外部依赖,和通过[Maven Central](http://search.maven.org/#search|ga|1|g%3A"org.jdbi" AND v%3A"3.16.0")发布的JARs,你可以在POM中包含相关的工件:

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.jdbi</groupId>
      <artifactId>jdbi3-bom</artifactId>
      <type>pom</type>
      <version>3.16.0</version>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

然后，在你的`<dependencies>`小节中，为你想使用的每个Jdbi模块声明一个依赖项:

```xml
<dependencies>
  <dependency>
    <groupId>org.jdbi</groupId>
    <artifactId>jdbi3-core</artifactId>
  </dependency>
</dependencies>
```

Jdbi提供了其他几个模块，这些模块用其他特性增强了核心API。

<a name="5____2_1__模块"></a>
### 2.1. 模块

- jdbi3-sqlobject

  SQL对象扩展。大多数Jdbi用户使用这个。

- jdbi3-guava

  支持Guava的集合和可选类型。

- jdbi3-jodatime2

  支持JodaTime v2的DateTime类型。

- jdbi3-jpa

  对JPA注解的最小支持。

- jdbi3-kotlin

  自动映射kotlin数据类。

- jdbi3-kotlin-sqlobject

  增强SQL Object扩展以支持Kotlin默认方法和方法的默认参数。

- jdbi3-oracle12

  支持Oracle返回DML语句。

- jdbi3-postgres

  支持Postgres驱动程序支持的大多数数据类型。

- jdbi3-spring4

  提供一个工厂bean来设置Jdbi单例。

- jdbi3-stringtemplate4

  使用StringTemplate4模板引擎，而不是JDBI内置的引擎。

- jdbi3-vavr

  支持Vavr元组，集合和值参数

<a name="6____2_2__事先申明"></a>
### 2.2. 事先申明

> **💡提示:** 您可能想要添加我们的注释`org.jdbi.v3.meta.Beta` 将被列入IDE的“不稳定API使用”黑名单。

> **☢警告:** 我们的`org.jdbi.*.internal`包不被认为是公共API;它们的内容可能会在没有警告的情况下发生根本变化。

<a name="7___3__核心_API"></a>
## 3. 核心 API

<a name="8____3_1__Jdbi"></a>
### 3.1. Jdbi

这个 [Jdbi](apidocs/org/jdbi/v3/core/Jdbi.html) 类是库的主要入口点。

每个`Jdbi`实例包装一个JDBC [DataSource](https://docs.oracle.com/javase/8/docs/api/javax/sql/DataSource.html)。它也是数据库会话的配置存储库。

有几种方法可以创建`Jdbi`实例。你可以使用JDBC URL:

```java
// H2 in-memory database
Jdbi jdbi = Jdbi.create("jdbc:h2:mem:test");
```

如果你有一个`DataSource`对象，你可以直接使用它:

```java
DataSource ds = ...
Jdbi jdbi = Jdbi.create(ds);
```

`Jdbi`实例是线程安全的，不拥有任何数据库资源。

通常，应用程序创建一个单例的、共享的`Jdbi`实例，并在那里设置任何公共配置。更多细节请参见[Configuration](#_configuration)。

`Jdbi `本身不提供连接池或其他[高可用性](#_high_availability)特性，但它可以与其他提供这些特性的软件结合使用。

在一个更有限的范围内(比如HTTP请求或事件回调)，您将从您的`Jdbi`实例请求一个`Handle`对象。

<a name="9____3_2__Handle_句柄_"></a>
### 3.2. Handle(句柄)

句柄表示一个活动的[数据库连接](https://docs.oracle.com/javase/8/docs/api/java/sql/Connection.html).

[Handle](apidocs/org/jdbi/v3/core/Handle.html)用于准备和运行针对数据库的SQL语句，并管理数据库事务。它提供对流式语句API的访问，这些API可以绑定参数、执行语句，然后将任何结果映射到Java对象。

`Handle`在创建时从`Jdbi`继承配置。更多细节请参见[Configuration](#_configuration)。

> **👁小心:** 因为`Handle`持有一个打开的连接，所以必须小心确保当您使用完它时，每个Handle都是关闭的。如果关闭句柄失败，将会导致数据库被打开的连接淹没，或者耗尽连接池。

有几种方法可以在运行时获得`Handle`实例。

如果您的操作将返回一些结果，请使用 [jdbi.withHandle()](apidocs/org/jdbi/v3/core/Jdbi.html#withHandle-org.jdbi.v3.core.HandleCallback-):

```java
List<String> names = jdbi.withHandle(handle ->
    handle.createQuery("select name from contacts")
          .mapTo(String.class)
          .list());
assertThat(names).contains("Alice", "Bob");
```

如果您的操作不需要返回结果，请使用 `Jdbi.useHandle(HandleConsumer)`:

```java
jdbi.useHandle(handle -> {
    handle.execute("create table contacts (id int primary key, name varchar(100))");
    handle.execute("insert into contacts (id, name) values (?, ?)", 1, "Alice");
    handle.execute("insert into contacts (id, name) values (?, ?)", 2, "Bob");
});
```

`withHandle`和`useHandle`都打开一个临时句柄，调用你的回调，并在回调返回时立即释放这个句柄。

> **💡提示:** 您可能会注意到在Jdbi的一些地方出现了“consumer”vs“callback”命名模式。回调函数返回一个值，并与`with-`方法相结合。消费者不返回值，并且与`use-`方法结合。

或者，如果你想自己管理句柄的生命周期，使用`jdbc.open()`:

```java
try (Handle handle = jdbi.open()) {
    handle.execute("insert into contacts (id, name) values (?, ?)", 3, "Chuck");
}
```

> **👁小心:** 当使用`jdbc.open()`时，应该始终使用try-with-resources或try-finally块来确保数据库连接被释放。不释放Handle将泄漏连接。我们建议在可能的情况下使用`withHandle`或`useHandle`而不是`open`。

<a name="10____3_3__参数"></a>
### 3.3. 参数

Arguments是Jdbi对JDBC语句参数的表示(the `?` in `select * from Foo where bar = ?`).

在JDBC `PreparedStatement`上设置参数`?`，您可以使用`ps.setString(1, "Baz")`。 使用Jdbi，当你绑定字符串`"Baz"`时，它会搜索所有注册的*ArgumentFactory*实例，直到找到一个愿意将String转换为*Argument*的实例。参数负责为占位符设置字符串，就像`setString`所做的那样。

参数可以执行比简单JDBC支持的更高级的绑定:BigDecimal可以绑定为SQL decimal，java.time.Year可以绑定为SQL int，或者一个复杂对象可以序列化为字节数组并绑定为SQL blob。

> **🏷注意:** Jdbi参数的使用仅限于JDBC `prepared statement`语句参数。 值得注意的是，arguments通常不能用于改变查询的结构(例如表或列名，`SELECT`或`INSERT`等)，也不能将参数插入到字符串字面量中。 更多信息请参见[Templating](#_templating) 和 [TemplateEngine](#_templateengine)。

<a name="11_____3_3_1__位置参数"></a>
#### 3.3.1. 位置参数

当SQL语句使用`?`令牌，Jdbi可以将值绑定到对应索引的参数上(以0开始):

```java
handle.createUpdate("insert into contacts (id, name) values (?, ?)")
      .bind(0, 3)
      .bind(1, "Chuck")
      .execute();

String name = handle.createQuery("select name from contacts where id = ?")
                    .bind(0, 3)
                    .mapTo(String.class)
                    .one();
```

<a name="12_____3_3_2__Named_Arguments"></a>
#### 3.3.2. Named Arguments(命名参数)

当SQL语句使用`:name`这样的冒号前缀标记时，Jdbi可以通过名称绑定参数:

```
handle.createUpdate("insert into contacts (id, name) values (:id, :name)")
      .bind("id", 3)
      .bind("name", "Chuck")
      .execute();

String name = handle.createQuery("select name from contacts where id = :id")
                    .bind("id", 3)
                    .mapTo(String.class)
                    .one();
```

> **🏷注意:** 这个`:foo`语法是可以改变的默认行为;请参阅`ColonPrefixSqlParser`类。Jdbi也提供了对`#foo`语法的开箱即用的支持，您也可以创建自己的语法。

> **💡提示:** 不允许混合命名参数和位置参数，因为这会变得混乱。

<a name="13_____3_3_3__Supported_Argument_Types"></a>
#### 3.3.3. Supported Argument Types(支持的参数类型)

开箱即用，Jdbi 支持以下类型作为 SQL 语句参数：

- 基本类型: `boolean`, `byte`, `short`, `int`, `long`, `char`, `float`, and `double`
- java.lang: `Boolean`, `Byte`, `Short`, `Integer`, `Long`, `Character`, `Float`, `Double`, `String`, 和 `Enum` (默认情况下作为枚举值的名称存储)
- java.math: `BigDecimal`
- java.net: `Inet4Address`, `Inet6Address`, `URL`, and `URI`
- java.sql: `Blob`, `Clob`, `Date`, `Time`, and `Timestamp`
- java.time: `Instant`, `LocalDate`, `LocalDateTime`, `LocalTime`, `OffsetDateTime`, `ZonedDateTime`, and `ZoneId`
- java.util: `Date`, `Optional` (围绕任何其他受支持的类型), 和 `UUID`
- `java.util.Collection` 和 Java 数组（存储为 SQL 数组）。 根据数组元素的类型，可能需要一些额外的设置。

您还可以配置Jdbi以支持其他参数类型。稍后再详细介绍。

<a name="14_____3_3_4__Binding_Arguments"></a>
#### 3.3.4. Binding Arguments(绑定参数)

SQL语句的参数可以通过几种不同的方式绑定。

你可以绑定单个参数:

```java
handle.createUpdate("insert into contacts (id, name) values (:id, :name)")
      .bind("id", 1)
      .bind("name", "Alice")
      .execute();
```

您可以从`Map` 的条目一次绑定多个参数：

```java
Map<String, Object> contact = new HashMap<>();
contact.put("id", 2)
contact.put("name", "Bob");

handle.createUpdate("insert into contacts (id, name) values (:id, :name)")
      .bindMap(contact)
      .execute();
```

您可以从`List<T>` 或可变参数一次绑定多个值：

```java
List<String> keys = new ArrayList<String>()
keys.add("user_name");
keys.add("street");

handle.createQuery("SELECT value FROM items WHERE kind in (<listOfKinds>)")
      .bindList("listOfKinds", keys)
      .mapTo(String.class)
      .list();

// 或者，使用“vararg”定义
handle.createQuery("SELECT value FROM items WHERE kind in (<varargListOfKinds>)")
      .bindList("varargListOfKinds", "user_name", "docs", "street", "library")
      .mapTo(String.class)
      .list();
```

> **🏷注意:** 使用`bindList`需要编写带有属性的SQL，而不是绑定，尽管你的值是绑定的。该属性是一个占位符，它将被安全地呈现到以逗号分隔的绑定占位符列表中。

你可以从Java Bean的属性绑定多个参数:

```java
Contact contact = new Contact();
contact.setId(3);
contact.setName("Cindy");

handle.createUpdate("insert into contacts (id, name) values (:id, :name)")
      .bindBean(contact)
      .execute();
```

你也可以绑定一个对象的公共字段:

```java
Object contact = new Object() {
    public int id = 0;
    public String name = "Cindy";
};

handle.createUpdate("insert into contacts (id, name) values (:id, :name)")
      .bindFields(contact)
      .execute();
```

或者您可以绑定对象的公共无参数方法：

```java
Object contact = new Object() {
    public int theId() {
        return 0;
    }

    public String theName() {
        return "Cindy";
    }
};

handle.createUpdate("insert into contacts (id, name) values (:theId, :theName)")
      .bindMethods(contact)
      .execute();
```

或者，您可以使用前缀限定每个绑定的 bean/对象。 在两个或多个绑定 bean 具有相似属性名称的情况下，这有助于消除歧义：

```java
Folder folder = new Folder(1, "Important Documents");
Document document =
    new Document(100, "memo.txt", "Business business business. Numbers.");

handle.createUpdate("insert into documents (id, folder_id, name, contents) " +
                    "values (:d.id, :f.id, :d.name, :d.contents)")
      .bindBean("f", folder)
      .bindMethods("f", folder)
      .bindFields("d", document)
      .execute();
```

> **🏷注意:** `bindBean()`、`bindFields()` 和 `bindMethods()` 可用于绑定嵌套属性，例如 `:user.address.street`。

> **☢警告:** `bindMap()` 不绑定嵌套属性——映射键应该与绑定的参数名称完全匹配。

> **💡提示:** 作者建议检查[Immutables](#109____7_4__Immutables)对高级方法的支持，方便地绑定和映射值类型。

<a name="15_____3_3_5__Custom_Arguments"></a>
#### 3.3.5. Custom Arguments(自定义参数)

Occasionally your data model will use data types not natively supported by Jdbi (see [Supported Argument Types](#_supported_argument_types)).

Fortunately, Jdbi can be configured to bind custom data types as arguments, by implementing a few simple interfaces.

> **🏷注意:** JDBC的核心特性通常得到所有数据库供应商的良好支持。然而，更高级的用法，如数组支持或几何类型，往往很快就会变成特定于供应商的。

<a name="16______Argument"></a>
##### Argument(参数)

[Argument](apidocs/org/jdbi/v3/core/argument/Argument.html)接口将单个值封装到绑定中。

```java
static class UUIDArgument implements Argument {
    private UUID uuid;

    UUIDArgument(UUID uuid) {
        this.uuid = uuid;
    }

    @Override
    public void apply(int position, PreparedStatement statement, StatementContext ctx)
    throws SQLException {
        statement.setString(position, uuid.toString()); <1>
    }
}

@Test
public void uuidArgument() {
    UUID u = UUID.randomUUID();
    assertThat(handle.createQuery("SELECT CAST(:uuid AS VARCHAR)")
        .bind("uuid", new UUIDArgument(u))
        .mapTo(String.class)
        .one()).isEqualTo(u.toString());
}
```

| <1> | 由于 Argument 通常直接调用 JDBC，因此在应用时会给出从 1 开始的索引（正如 JDBC 所期望的）。 |
| --- | ----------------------------------------------------------------------------------- |

这里我们使用 **Argument** 直接绑定一个 UUID。 在这种特殊情况下，最明显的方法是将 UUID 作为字符串发送到数据库。 如果您的 JDBC 驱动程序直接支持自定义类型或高效的二进制传输，您可以在此处轻松利用它们。

<a name="17______ArgumentFactory"></a>
##### ArgumentFactory(参数工厂)

[ArgumentFactory](apidocs/org/jdbi/v3/core/argument/ArgumentFactory.html) 接口为它知道的任何数据类型提供 [Argument](#16______Argument) 实例。 通过实现和注册一个参数工厂，可以绑定自定义数据类型，而不必将它们显式地包装在 `Argument` 对象中。

Jdbi provides an `AbstractArgumentFactory` class which simplifies implementing the `ArgumentFactory` contract:

```
static class UUIDArgumentFactory extends AbstractArgumentFactory<UUID> {
    UUIDArgumentFactory() {
        super(Types.VARCHAR); <1>
    }

    @Override
    protected Argument build(UUID value, ConfigRegistry config) {
        return (position, statement, ctx) -> statement.setString(position, value.toString()); <2>
    }
}

@Test
public void uuidArgumentFactory() {
    UUID u = UUID.randomUUID();
    handle.registerArgument(new UUIDArgumentFactory());
    assertThat(handle.createQuery("SELECT CAST(:uuid AS VARCHAR)")
        .bind("uuid", u)
        .mapTo(String.class)
        .one()).isEqualTo(u.toString());
}
```

| <1> | 绑定 UUID 时使用的 JDBC [SQL 类型常量](https://docs.oracle.com/javase/8/docs/api/java/sql/Types.html)。 Jdbi 需要这个来绑定 `null` 的 UUID 值。 参见 [PreparedStatement.setNull(int,int)](https://docs.oracle.com/javase/8/docs/api/java/sql/PreparedStatement.html#setNull-int-int-) |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| <2> | 由于 `Argument` 是一个函数式接口，它可以实现为一个简单的 lambda 表达式。                                                                                                                                                                                                               |

<a name="18______Prepared_Arguments"></a>

##### Prepared Arguments(准备参数)

传统的参数工厂根据绑定的类型和实际值来决定绑定。 这是非常灵活的，但是当绑定一个大的 `PreparedBatch` 时，它会导致严重的性能损失，因为必须为每批添加的参数咨询整个参数工厂链。 为了解决这个问题，实现 `ArgumentFactory.Preparable`，它承诺处理给定 `Type` 的所有值。 大多数内置参数工厂现在都实现了 Preparable 接口。

<a name="19______Arguments_Registry"></a>
##### Arguments Registry(参数注册表)

当您注册一个 `ArgumentFactory` 时，注册信息存储在 Jdbi 持有的 [Arguments](apidocs/org/jdbi/v3/core/argument/Arguments.html) 实例中。 `Arguments` 是一个配置类，它存储所有注册的参数工厂（包括内置参数的工厂）。

实际上，当您将参数绑定到语句时，Jdbi会查询`Arguments`配置对象，并搜索`ArgumentFactory`，该对象知道如何将绑定对象转换为`Argument`。

稍后，当语句执行时，绑定期间定位的每个`Argument`都会应用到JDBC [PreparedStatement](https://docs.oracle.com/javase/8/docs/api/java/sql/PreparedStatement.html) .

> **🏷注意:**  有时，两个或多个参数工厂将支持相同数据类型的参数。 当这种情况发生时，最后注册的工厂获胜。 可准备参数工厂总是优先于基本参数工厂。 这意味着您可以覆盖任何数据类型的绑定方式，包括开箱即用支持的数据类型。

<a name="20____3_4__Queries"></a>
### 3.4. Queries(查询)

[Query](apidocs/org/jdbi/v3/core/statement/Query.html) 是一个 [result-bearing](apidocs/org/jdbi/v3/core/result/ResultBearing.html) SQL 语句，它返回一个 来自数据库的结果集。

```java
List<Map<String, Object>> users =
    handle.createQuery("SELECT id, name FROM user ORDER BY id ASC")
        .mapToMap()
        .list();

assertThat(users).containsExactly(
        map("id", 1, "name", "Alice"),
        map("id", 2, "name", "Bob"));
```

要从查询中获取单行，可以使用几种可能的方法，具体取决于结果集中可能的行数。

当你期望结果只包含一行时，调用`one()`。只有当返回的行映射为`null`时才返回`null`。如果结果有零行或多行，则抛出异常。

```java
String name = handle.select("select name from users where id = ?", 3)
    .mapTo(String.class)
    .one();
```

当您希望结果包含零或一行时调用 findOne() 。 如果没有行，或者有一行映射到 `null`，则返回 `Optional.empty()`。 如果结果有多行，则抛出异常。

```java
Optional<String> name = handle.select(...)
    .mapTo(String.class)
    .findOne();
```

当您希望结果至少包含一行时，请调用“first()”。 如果第一行映射到 `null`，则返回 `null`。 如果结果有零行，则抛出异常。

```java
String name = handle.select("select name from users where id = ?", 3)
    .mapTo(String.class)
    .first();
```

当结果可能包含任意数量的行时调用`findFirst()`。 如果没有行，或者第一行映射到 `null`，则返回 `Optional.empty()`。

```java
Optional<String> name = handle.select(...)
    .mapTo(String.class)
    .findFirst();
```

可以在List中返回多个结果行：

```java
List<String> name = handle.createQuery(
        "select title from films where genre = :genre order by title")
    .bind("genre", "Action")
    .mapTo(String.class)
    .list();
```

对于其他集合，使用带有 [collector](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collector.html) 的 `collect()`：

```java
Set<String> name = handle.createQuery(
        "select title from films where genre = :genre order by title")
    .bind("genre", "Action")
    .mapTo(String.class)
    .collect(Collectors.toSet());
```

您还可以流式传输结果：

```java
handle.createQuery(
        "select title from films where genre = :genre order by title")
    .mapTo(String.class)
    .useStream(stream -> {
      // do stuff with stream
    });
```

到目前为止，所有示例都显示了一个 `String` 结果类型。 当然，您可以映射到许多其他数据类型：

```java
LocalDate releaseDate = handle.createQuery(
        "select release_date from films where name = :name")
    .bind("name", "Star Wars: A New Hope")
    .mapTo(LocalDate.class)
    .one();
```

<a name="21____3_5__Mappers"></a>
### 3.5. Mappers(映射器)

Jdbi 使用映射器将结果数据转换为 Java 对象。 有两种类型的映射器：

- [Row Mappers](#22_____3_5_1__Row_Mappers), 映射整行结果集数据。
- [Column Mappers](#25_____3_5_2__Column_Mappers), 映射结果集行的单个列。

<a name="22_____3_5_1__Row_Mappers"></a>
#### 3.5.1. Row Mappers(行映射器)

[RowMapper](apidocs/org/jdbi/v3/core/mapper/RowMapper.html)是一个函数式接口，映射一个JDBC的当前行[ResultSet](https://docs.oracle.com/javase/8/docs/api/java/sql/ResultSet.html)  到映射类型。 为结果集中的每一行调用一次行映射器。

由于 `RowMapper` 是一个函数式接口，它们可以使用 lambda 表达式内联到查询中：

```java
List<User> users = handle.createQuery("SELECT id, name FROM user ORDER BY id ASC")
        .map((rs, ctx) -> new User(rs.getInt("id"), rs.getString("name")))
        .list();
```

> **💡提示:** 在上面的例子中使用了三种不同的类型。 由 Handle.createQuery() 返回的 Query 实现了 ResultBearing 接口。 `ResultBearing.map()` 方法接受一个 `RowMapper<T>` 并返回一个 `ResultIterable<T>`。 最后，`ResultBearing.list()` 将结果集中的每一行收集到一个 `List<T>` 中。

行映射器可以定义为类，允许重用：

```java
class UserMapper implements RowMapper<User> {
    @Override
    public User map(ResultSet rs, StatementContext ctx) throws SQLException {
        return new User(rs.getInt("id"), rs.getString("name"));
    }
}
List<User> users = handle.createQuery("SELECT id, name FROM user ORDER BY id ASC")
    .map(new UserMapper())
    .list();
```

这个 `RowMapper` 相当于上面的 lambda 映射器，但更明确。

<a name="23______RowMappers_registry"></a>
##### RowMappers registry(行映射器注册表)

可以为特定类型注册行映射器。这简化了使用，只需要指定要映射到的类型。Jdbi自动从注册表查找映射器，并使用它。

```java
jdbi.registerRowMapper(User.class,
    (rs, ctx) -> new User(rs.getInt("id"), rs.getString("name"));

try (Handle handle = jdbi.open()) {
  List<User> users = handle.createQuery("SELECT id, name FROM user ORDER BY id ASC")
        .mapTo(User.class)
        .list();
}
```
> **💡提示:** `翻译者WJW`提示,也可以handle调用registerRowMapper方法

可以在不指定映射类型的情况下注册使用显式映射类型（例如上一节中的 UserMapper 类）实现 `RowMapper` 的映射器：

```java
handle.registerRowMapper(new UserMapper());
```

使用此方法时，Jdbi 检查映射器的泛型类签名以自动发现映射类型。

可以为任何给定类型注册多个映射器。 发生这种情况时，给定类型的最后注册的映射器优先。 这允许优化，比如为某种类型注册一个“默认”映射器，同时允许在适当的时候用不同的映射器覆盖默认映射器。

<a name="24______RowMapperFactory"></a>
##### RowMapperFactory(行映射器工厂)

[RowMapperFactory](apidocs/org/jdbi/v3/core/mapper/RowMapperFactory.html) 可以为任意类型生成行映射器。

在以下情况下，实现工厂可能比常规行映射器更可取：

- 映射器实现是通用的，可以应用于多个映射类型。 例如，Jdbi 提供了一个通用的 [BeanMapper](#33______BeanMapper)，它将列映射到任何 bean 类的 bean 属性。
- 映射类型具有通用签名，和/或映射器可以由其他注册的映射器组成。 例如，Jdbi 提供了一个 [Map.Entry mapper](#35______Map_Entry_mapping)，前提是为类型 `K` 和 `V` 注册了一个映射器。
- 您想将多个映射器捆绑到一个类中。

让我们以`Pair<L, R>`类为例：

```java
public final class Pair<L, R> {
  public final L left;
  public final R right;

  public Pair(L left, R right) {
    this.left = left;
    this.right = right;
  }
}
```

现在，让我们实现一个行映射器工厂。 工厂应该为任何`Pair<L, R>`类型生成一个`RowMapper<Pair<L, R>>`，其中`L`类型从第一列映射，`R`从第二列映射——假设 `L` 和 `R`都注册了列映射器。

让我们一步一步来：

```java
public class PairMapperFactory implements RowMapperFactory {
  public Optional<RowMapper<?>> build(Type type, ConfigRegistry config) {
    ...
  }
}
```

`build` 方法接受一个映射类型和一个配置注册表。 如果它知道如何映射该类型，它可能会返回 `Optional.of(someMapper)`，否则返回 `Optional.empty()`。

首先，我们检查映射的类型是否为`Pair`:

```java
if (!Pair.class.equals(GenericTypes.getErasedType(type))) {
  return Optional.empty();
}
```

> **💡提示:** `GenericTypes`实用程序类在[使用泛型类型](#137____9_3__Working_with_Generic_Types)中进行了讨论。

接下来，我们从映射类型中提取 `L` 和 `R` 泛型参数：

```java
Type leftType = GenericTypes.resolveType(Pair.class.getTypeParameters()[0], type);
Type rightType = GenericTypes.resolveType(Pair.class.getTypeParameters()[1], type);
```

在第一行中，`Pair.class.getTypeParameters()[0]` 给出了类型变量 `L`。 同样在第二行，`Pair.class.getTypeParameters()[1]` 给出了类型变量 `R`。

我们使用 `resolveType()` 在映射类型的上下文中解析 `L` 和 `R` 类型变量的类型。

现在我们有了`L` 和`R` 的类型，我们可以通过配置注册表从`ColumnMappers` 配置类中查找这些类型的列映射器：

```java
ColumnMappers columnMappers = config.get(ColumnMappers.class);

ColumnMapper<?> leftMapper = columnMappers.findFor(leftType)
   .orElseThrow(() -> new NoSuchMapperException(
       "No column mapper registered for Pair left parameter " + leftType));
ColumnMapper<?> rightMapper = columnMappers.findFor(rightType)
   .orElseThrow(() -> new NoSuchMapperException(
       "No column mapper registered for Pair right parameter " + rightType));
```

配置注册表是配置类的定位器。 因此，当我们调用 `config.get(ColumnMappers.class)` 时，我们会返回一个带有当前列映射器配置的 `ColumnMappers` 实例。

接下来我们调用 `ColumnMappers.findFor()` 来获取left 和 right类型的列映射器。

> **💡提示:** 你可能已经注意到，虽然这个方法可以返回 `Optional`，但如果我们找不到左侧或右侧的映射器，我们就会抛出异常。 我们发现这是一个最佳实践：如果工厂对映射类型一无所知（在本例中为“Pair”），则返回`Optional.empty()`。 如果它知道映射类型但缺少一些配置以使其工作（例如，映射器未注册为`L`或`R`参数），则抛出带有信息性消息的异常更有帮助，因此用户可以诊断*为什么* 映射器未按预期工作。

最后，我们构造一个pair mapper，并返回它：

```java
RowMapper<?> pairMapper = (rs, ctx) ->
    new Pair(leftMapper.map(rs, 1, ctx), // 在 JDBC 中，列号从 1 开始
             rightMapper.map(rs, 2, ctx)); // ..for MOTHERF***ING REASONS

return Optional.of(pairMapper);
```

下面是工厂类PairMapperFactory的完整源代码：

```java
public class PairMapperFactory implements RowMapperFactory {
  public Optional<RowMapper<?>> build(Type type, ConfigRegistry config) {
    if (!Pair.class.equals(GenericTypes.getErasedType(type))) {
      return Optional.empty();
    }

    Type leftType = GenericTypes.resolveType(Pair.class.getTypeParameters()[0], type);
    Type rightType = GenericTypes.resolveType(Pair.class.getTypeParameters()[1], type);

    ColumnMappers columnMappers = config.get(ColumnMappers.class);

    ColumnMapper<?> leftMapper = columnMappers.findFor(leftType)
       .orElseThrow(() -> new NoSuchMapperException(
           "No column mapper registered for Pair left parameter " + leftType));
    ColumnMapper<?> rightMapper = columnMappers.findFor(rightType)
       .orElseThrow(() -> new NoSuchMapperException(
           "No column mapper registered for Pair right parameter " + rightType));

    RowMapper<?> pairMapper = (rs, ctx) ->
        new Pair(leftMapper.map(rs, 1, ctx),
                 rightMapper.map(rs, 2, ctx));

    return Optional.of(pairMapper);
  }
}
```

行映射器工厂的注册类似于常规行映射器:

```java
jdbi.registerRowMapper(new PairMapperFactory());

try (Handle handle = jdbi.open()) {
  List<Pair<String, String>> configPairs = handle
          .createQuery("SELECT key, value FROM config")
          .mapTo(new GenericType<Pair<String, String>>() {})
          .list();
}
```

> **💡提示:** `GenericType` 实用程序类在 [使用泛型类型](#137____9_3__Working_with_Generic_Types)中讨论.

<a name="25_____3_5_2__Column_Mappers"></a>
#### 3.5.2. Column Mappers(列映射器)

[ColumnMapper](apidocs/org/jdbi/v3/core/mapper/ColumnMapper.html) 是一个函数接口，从一个JDBC [ResultSet](https://docs.oracle.com/javase/8/docs/api/java/sql/ResultSet.html) 到映射类型。

由于 `ColumnMapper` 是一个函数式接口，所以可以使用 lambda 表达式将它们内联提供给查询：

```java
List<Money> amounts = handle
    .select("select amount from transactions where account_id = ?", accountId)
    .map((rs, col, ctx) -> Money.parse(rs.getString(col))) <1>
    .list();
```

> **<1>** 每当使用列映射器映射行时，仅映射每行的第一列。

列映射器可以定义为允许重用的类：

```java
public class MoneyMapper implements ColumnMapper<Money> {
  public Money map(ResultSet r, int columnNumber, StatementContext ctx) throws SQLException {
    return Money.parse(r.getString(columnNumber));
  }
}
List<Money> amounts = handle
    .select("select amount from transactions where account_id = ?", accountId)
    .map(new MoneyMapper())
    .list();
```

这个 `ColumnMapper` 相当于上面的 lambda 映射器，但更明确。

<a name="26______ColumnMappers_registry"></a>
##### ColumnMappers registry(列映射器注册表)

可以为特定类型注册列映射器。 这简化了使用，只需要您指定要映射到的类型。 Jdbi 会自动从注册表中查找映射器并使用它。

```java
jdbi.registerColumnMapper(Money.class,
    (rs, col, ctx) -> Money.parse(rs.getString(col)));

List<Money> amounts = jdbi.withHandle(handle ->
    handle.select("select amount from transactions where account_id = ?", accountId)
          .mapTo(Money.class)
          .list());
```

使用显式映射类型实现`ColumnMapper`的映射器(如前一节中的`MoneyMapper`类)可以不指定映射类型而被注册:

```java
handle.registerColumnMapper(new MoneyMapper());
```

使用此方法时，Jdbi 检查映射器的泛型类签名以自动发现映射类型。

可以为任何给定类型注册多个映射器。 发生这种情况时，给定类型的最后注册的映射器优先。 这允许优化，比如为某种类型注册一个“默认”映射器，同时允许在适当的时候用不同的映射器覆盖默认映射器。

开箱即用，列映射器缺省注册了以下几种类型：

- 基本类型: `boolean`, `byte`, `short`, `int`, `long`, `char`, `float`, and `double`
- java.lang: `Boolean`, `Byte`, `Short`, `Integer`, `Long`, `Character`, `Float`, `Double`, `String`, 和 `Enum` (默认情况下存储为枚举值的名称)
- java.math: `BigDecimal`
- `byte[]` 数组 (例如 对于 BLOB 或 VARBINARY 列)
- java.net: `InetAddress`, `URL`, 和 `URI`
- java.sql: `Timestamp`
- java.time: `Instant`, `LocalDate`, `LocalDateTime`, `LocalTime`, `OffsetDateTime`, `ZonedDateTime`, and `ZoneId` (翻译者WJW注: 不包括`java.util.Date`)
- java.util: `UUID`
- `java.util.Collection` 和 Java 数组（用于数组列）。 根据数组元素的类型，可能需要一些额外的设置——请参阅 [SQL 数组](#37____3_7__SQL_Arrays).

> **🏷注意:** 枚举值的绑定和映射方法可以通过 [Enums](apidocs/org/jdbi/v3/core/enums/Enums.html) 配置以及 [EnumByName](apidocs/org/jdbi/v3 /core/enums/EnumByName.html) 和 [EnumByOrdinal](apidocs/org/jdbi/v3/core/enums/EnumByOrdinal.html) 注解。

> 翻译者WJW添加: 如何注册`java.util.Date`的列映射器
> ```java
>     jdbi3.registerColumnMapper(new ColumnMapper<java.util.Date>() {
>       public java.util.Date map(ResultSet rs, int columnNumber, StatementContext ctx) throws SQLException {
>         return rs.getTimestamp(columnNumber);
>       }
>     });
>     
>     jdbi3.useHandle(handle -> {
>       java.util.Date releaseDate = handle.select("select create_time from e_bi limit 3")
>           .mapTo(java.util.Date.class)
>           .first();
>       
>       System.out.println("create_time: " + releaseDate);
>     });
> ```

<a name="27______ColumnMapperFactory"></a>
##### ColumnMapperFactory(列映射器工厂)

[ColumnMapperFactory](apidocs/org/jdbi/v3/core/mapper/ColumnMapperFactory.html) 可以生成任意类型的列映射器。

在以下情况下，实现工厂可能比常规列映射器更可取：

- 映射器类是泛型的，可以应用于多个映射类型。
- 被映射的类型是泛型的，并且/或者映射器可以由其他已注册的映射器组成。
- 您想将多个映射器捆绑到一个类中。

作为示例，让我们为 `Optional<T>` 创建一个映射器工厂。 工厂应该为任何`T`生成一个`ColumnMapper<Optional<T>>`，前提是为`T`注册了一个列映射器。

让我们一步一步来：

```java
public class OptionalColumnMapperFactory implements ColumnMapperFactory {
  public Optional<ColumnMapper<?>> build(Type type, ConfigRegistry config) {
    ...
  }
}
```

`build` 方法接受一个映射类型和一个配置注册表。 如果它知道如何映射该类型，它可能会返回 `Optional.of(someMapper)`，否则返回 `Optional.empty()`。

首先，我们检查映射类型是否为 `Optional`：

```java
if (!Optional.class.equals(GenericTypes.getErasedType(type))) {
  return Optional.empty();
}
```

> **💡提示:** `GenericTypes` 实用程序类在 [使用泛型类型](#137____9_3__Working_with_Generic_Types) 中讨论.

接下来，从映射类型中提取 `T` 泛型参数：

```java
Type t = GenericTypes.resolveType(Optional.class.getTypeParameters()[0], type);
```

表达式 `Optional.class.getTypeParameters()[0]` 给出了类型变量 `T`。

我们使用 `resolveType()` 在映射类型的上下文中解析 `T` 的类型。

现在我们有了 `T` 的类型，我们可以通过配置注册表从 `ColumnMappers` 配置类中查找该类型的列映射器：

```java
ColumnMapper<?> tMapper = config.get(ColumnMappers.class)
    .findFor(embeddedType)
    .orElseThrow(() -> new NoSuchMapperException(
        "No column mapper registered for parameter " + embeddedType + " of type " + type));
```

配置注册表是配置类的定位器。 因此，当我们调用 `config.get(ColumnMappers.class)` 时，我们会返回一个带有当前列映射器配置的 `ColumnMappers` 实例。

接下来我们调用 `ColumnMappers.findFor()` 来获取嵌入类型的列映射器。

> **💡提示:** 你可能已经注意到，虽然这个方法可以返回 `Optional`，但如果我们找不到嵌入类型的映射器，我们就会抛出异常。 我们发现这是一个最佳实践：如果工厂对映射类型一无所知（在本例中为`Optional`），则返回`Optional.empty()`。 如果它知道映射的类型但缺少一些使其工作的配置（例如没有为类型`T`数注册映射器），则抛出带有信息性消息的异常更有帮助，因此用户可以诊断*为什么*映射器 没有按预期工作。

最后，我们为optionals构造列映射器，并返回它：

```java
ColumnMapper<?> optionalMapper = (rs, col, ctx) ->
    Optional.ofNullable(tMapper.map(rs, col, ctx));

return Optional.of(optionalMapper);
```

下面是工厂类OptionalColumnMapperFactory的完整源代码：

```java
public class OptionalColumnMapperFactory implements ColumnMapperFactory {
  public Optional<ColumnMapper<?>> build(Type type, ConfigRegistry config) {
    if (!Optional.class.equals(GenericTypes.getErasedType(type))) {
      return Optional.empty();
    }

    Type t = GenericTypes.resolveType(Optional.class.getTypeParameters()[0], type);

    ColumnMapper<?> tMapper = config.get(ColumnMappers.class)
        .findFor(t)
        .orElseThrow(() -> new NoSuchMapperException(
            "No column mapper registered for parameter " + t + " of type " + type));

    ColumnMapper<?> optionalMapper = (rs, col, ctx) ->
        Optional.ofNullable(tMapper.map(rs, col, ctx));

    return Optional.of(optionalMapper);
  }
}
```

列映射器工厂可以类似于常规列映射器进行注册：

```java
jdbi.registerColumnMapper(new OptionalColumnMapperFactory());

try (Handle handle = jdbi.open()) {
  List<Optional<String>> middleNames = handle
          .createQuery("select middle_name from contacts")
          .mapTo(new GenericType<Optional<String>>() {})
          .list();
}
```

> **💡提示:** `GenericType` 实用程序类在 [使用泛型类型](#137____9_3__Working_with_Generic_Types) 中讨论。

<a name="28_____3_5_3__Primitive_Mapping"></a>
#### 3.5.3. Primitive Mapping(基本类型映射)

所有 Java 基本类型都有到它们相应的 JDBC 类型的默认映射。 一般情况下，Jdbi 在遇到包装器类型时会自动进行适当的装箱和拆箱。

默认情况下，映射到原始类型的 SQL `null` 将采用 Java 默认值。 这可以通过配置`jdbi.getConfig(ColumnMappers.class).setCoalesceNullPrimitivesToDefaults(false)`来禁用。

<a name="29_____3_5_4__Immutables_Mapping"></a>
#### 3.5.4. Immutables Mapping(不可变映射)

`Immutables` 值对象可能会被映射，详情参见 [Immutables](#109____7_4__Immutables) 部分。

<a name="30_____3_5_5__Freebuilder_Mapping"></a>
#### 3.5.5. Freebuilder Mapping(自由建造器映射)

`Freebuilder` 值对象可能会被映射，详情参见 [Freebuilder](#110____7_5__Freebuilder) 部分。

<a name="31_____3_5_6__Reflection_Mappers"></a>
#### 3.5.6. Reflection Mappers(反射映射器)

Jdbi 提供了一些开箱即用的基于反射的映射器。

Reflective mappers treat column names as bean property names (BeanMapper), constructor parameter names (ConstructorMapper), or field names (FieldMapper).

Reflective mappers are snake_case aware and will automatically match up these columns to camelCase field/argument/property names.

> **💡提示:** To instruct Jdbi to ignore an otherwise mappable method, annotate it as `@Unmappable`.

<a name="32______ConstructorMapper"></a>
##### ConstructorMapper(构造器映射器)

**Jdbi**提供了一个简单的构造函数映射器，它使用反射按名称将列分配给构造函数参数。

```java
@ConstructorProperties({"id", "name"})
public User(int id, String name) {
  this.id = id;
  this.name = name;
}
```

`@ConstructorProperties` 注解告诉 Jdbi 每个构造函数参数的属性名称，因此它可以找出每个构造函数参数对应的列。

> **💡提示:** Lombok的 `@AllArgsConstructor` 注解会为您生成 `@ConstructorProperties`注解。

启用`-parameters` Java 编译器标志消除了对`@ConstructorProperties` 注解的需要——参见[使用参数名称编译](#133____9_2__使用参数名称编译)。 因此：

```java
public User(int id, String name) {
    this.id = id;
    this.name = name;
}
```

使用 `factory()` 方法为你的映射类注册一个构造函数映射器：

```java
handle.registerRowMapper(ConstructorMapper.factory(User.class));
Set<User> userSet = handle.createQuery("SELECT * FROM user ORDER BY id ASC")
    .mapTo(User.class)
    .collect(Collectors.toSet());

assertThat(userSet).hasSize(4);
```

构造函数参数名称“id”、“name”与数据库列名称匹配，因此根本不需要自定义映射器代码。

构造函数映射器可以为每个映射的构造函数参数配置一个列名前缀。 这可以帮助消除映射连接的歧义，例如 当两个映射类具有相同的属性名称（如 `id` 或 `name`）时：

```java
handle.registerRowMapper(ConstructorMapper.factory(Contact.class, "c"));
handle.registerRowMapper(ConstructorMapper.factory(Phone.class, "p"));
handle.registerRowMapper(JoinRowMapper.forTypes(Contact.class, Phone.class);
List<JoinRow> contactPhones = handle.select("select " +
        "c.id cid, c.name cname, " +
        "p.id pid, p.name pname, p.number pnumber " +
        "from contacts c left join phones p on c.id = p.contact_id")
    .mapTo(JoinRow.class)
    .list();
```

通常，映射类将具有单个构造函数。 如果它有多个构造函数，Jdbi 将根据以下规则选择一个：

- 首先，使用带有`@JdbiConstructor` 注解的构造函数，如果有的话。
- 接下来，使用用`@ConstructorProperties` 注解的构造函数，如果有的话。
- 否则，抛出 Jdbi 不知道使用哪个构造函数的异常。

对于与属性名称不匹配列名称的，请使用 `@ColumnName` 批注来提供准确的列名称。

```java
public User(@ColumnName("user_id") int id, String name) {
  this.id = id;
  this.name = name;
}
```

> **🏷注意:** `@ColumnName` 注解仅在将 SQL 数据映射到 Java 对象时适用。 当绑定对象属性时（例如使用`bindBean()`），绑定属性名（`:id`）而不是列名（`:user_id`）。

嵌套的构造函数注入类型可以使用 `@Nested` 注解进行映射：

```java
public class User {
  public User(int id,
              String name,
              @Nested Address address) {
    ...
  }
}

public class Address {
  public Address(String street,
                 String city,
                 String state,
                 String zip) {
    ...
  }
}

```java
handle.registerRowMapper(ConstructorMapper.factory(User.class));

List<User> users = handle
    .select("select id, name, street, city, state, zip from users")
    .mapTo(User.class)
    .list();
```

`@Nested` 注解有一个可选的 `value()` 属性，可用于将列名前缀应用于每个嵌套的构造函数参数：

```java
public User(int id,
            String name,
            @Nested("addr") Address address) {
  ...
}
```

```java
handle.registerRowMapper(ConstructorMapper.factory(User.class));

List<User> users = handle
    .select("select id, name, addr_street, addr_city, addr_state, addr_zip from users")
    .mapTo(User.class)
    .list();
```

默认情况下，ConstructorMapper 期望结果集包含映射每个构造函数参数的列，如果任何参数无法映射，则会抛出异常。

结果集中可能会省略带有`@Nullable`注释的参数，其中`ConstructorMapper`会将`null`传递给该参数的构造函数。

```java
public class User {
  public User(int id,
              String name,
              @Nullable String passwordHash,
              @Nullable @Nested Address address) {
    ...
  }
}
```

在这个例子中，`id` 和 `name` 列必须出现在结果集中，但 `passwordHash` 和 `address` 是可选的。 如果它们存在，它们将被映射。 否则，

> **💡提示:** 可以使用任何包中的任何 `@Nullable` 注解。 `javax.annotation.Nullable` 是一个不错的选择。

<a name="33______BeanMapper"></a>
##### BeanMapper(Bean映射器)

We also provide basic support for mapping beans:

```
public class UserBean {
    private int id;
    private String name;

    public int getId() {
        return id;
    }
    public void setId(int id) {
        this.id = id;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
}
```

Register a bean mapper for your mapped class, using the `factory()` method:

```
handle.registerRowMapper(BeanMapper.factory(UserBean.class));

List<UserBean> users = handle
        .createQuery("select id, name from user")
        .mapTo(UserBean.class)
        .list();
```

Alternatively, call `mapToBean()` instead of registering a bean mapper:

```
List<UserBean> users = handle
        .createQuery("select id, name from user")
        .mapToBean(UserBean.class)
        .list();
```

Bean mappers can be configured with a column name prefix for each mapped property. This can help to disambiguate mapping joins, e.g. when two mapped classes have identical property names (like `id` or `name`):

```
handle.registerRowMapper(BeanMapper.factory(ContactBean.class, "c"));
handle.registerRowMapper(BeanMapper.factory(PhoneBean.class, "p"));
handle.registerRowMapper(JoinRowMapper.forTypes(ContactBean.class, PhoneBean.class));
List<JoinRow> contactPhones = handle.select("select "
        + "c.id cid, c.name cname, "
        + "p.id pid, p.name pname, p.number pnumber "
        + "from contacts c left join phones p on c.id = p.contact_id")
        .mapTo(JoinRow.class)
        .list();
```

For legacy column names that don’t match up to property names, use the `@ColumnName` annotation to provide an exact column name.

```
public class User {
  private int id;

  @ColumnName("user_id")
  public int getId() { return id; }

  public void setId(int id) { this.id = id; }
}
```

The `@ColumnName` annotation can be placed on either the getter or setter method.

|      | The `@ColumnName` annotation only applies while mapping SQL data into Java objects. When binding object properties (e.g. with `bindBean()`), bind the property name (`:id`) rather than the column name (`:user_id`). |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

Nested Java Bean types can be mapped using the `@Nested` annotation:

```
public class User {
  private int id;
  private String name;
  private Address address;

  ... (getters and setters)

  @Nested 
  public Address getAddress() { ... }

  public void setAddress(Address address) { ... }
}

public class Address {
  private String street;
  private String city;
  private String state;
  private String zip;

  ... (getters and setters)
}
```

|      | The `@Nested` annotation can be placed on either the getter or setter method. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

```
handle.registerRowMapper(BeanMapper.factory(User.class));

List<User> users = handle
    .select("select id, name, street, city, state, zip from users")
    .mapTo(User.class)
    .list();
```

The `@Nested` annotation has an optional `value()` attribute, which can be used to apply a column name prefix to each nested bean property:

```
@Nested("addr")
public Address getAddress() { ... }
handle.registerRowMapper(BeanMapper.factory(User.class));

List<User> users = handle
    .select("select id, name, addr_street, addr_city, addr_state, addr_zip from users")
    .mapTo(User.class)
    .list();
```

|      | `@Nested` properties are left unmodified (i.e. null) if the result set has no columns matching any properties of the nested object. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

<a name="34______FieldMapper"></a>

##### FieldMapper

[FieldMapper](apidocs/org/jdbi/v3/core/mapper/reflect/FieldMapper.html) uses reflection to map database columns directly to object fields (including private fields).

```
public class User {
  public int id;

  public String name;
}
```

Register a field mapper for your mapped class, using the `factory()` method:

```
handle.registerRowMapper(FieldMapper.factory(User.class));

List<UserBean> users = handle
        .createQuery("select id, name from user")
        .mapTo(User.class)
        .list();
```

Field mappers can be configured with a column name prefix for each mapped field. This can help to disambiguate mapping joins, e.g. when two mapped classes have identical property names (like `id` or `name`):

```
handle.registerRowMapper(FieldMapper.factory(Contact.class, "c"));
handle.registerRowMapper(FieldMapper.factory(Phone.class, "p"));
handle.registerRowMapper(JoinRowMapper.forTypes(Contact.class, Phone.class);
List<JoinRow> contactPhones = handle.select("select " +
        "c.id cid, c.name cname, " +
        "p.id pid, p.name pname, p.number pnumber " +
        "from contacts c left join phones p on c.id = p.contact_id")
    .mapTo(JoinRow.class)
    .list();
```

For legacy column names that don’t match up to field names, use the `@ColumnName` annotation to provide an exact column name:

```
public class User {
  @ColumnName("user_id")
  public int id;

  public String name;
}
```

|      | The `@ColumnName` annotation only applies while mapping SQL data into Java objects. When binding object properties (e.g. with `bindBean()`), bind the property name (`:id`) rather than the column name (`:user_id`). |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

Nested field-mapped types can be mapped using the `@Nested` annotation:

```
public class User {
  public int id;
  public String name;

  @Nested
  public Address address;
}

public class Address {
  public String street;
  public String city;
  public String state;
  public String zip;
}
handle.registerRowMapper(FieldMapper.factory(User.class));

List<User> users = handle
    .select("select id, name, street, city, state, zip from users")
    .mapTo(User.class)
    .list();
```

The `@Nested` annotation has an optional `value()` attribute, which can be used to apply a column name prefix to each nested field:

```
public class User {
  public int id;
  public String name;

  @Nested("addr")
  public Address address;
}
handle.registerRowMapper(FieldMapper.factory(User.class));

List<User> users = handle
    .select("select id, name, addr_street, addr_city, addr_state, addr_zip from users")
    .mapTo(User.class)
    .list();
```

|      | `@Nested` fields are left unmodified (i.e. null) if the result set has no columns matching any fields of the nested object. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

<a name="35______Map_Entry_mapping"></a>
##### Map.Entry mapping

Out of the box, Jdbi registers a `RowMapper<Map.Entry<K,V>>`. Since each row in the result set is a `Map.Entry<K,V>`, the entire result set can be easily collected into a `Map<K,V>` (or Guava’s `Multimap<K,V>`).

|      | A mapper must be registered for both the key and value type. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

Join rows can be gathered into a map result by specifying the generic map signature:

```
String sql = "select u.id u_id, u.name u_name, p.id p_id, p.phone p_phone "

    + "from user u left join phone p on u.id = p.user_id";
Map<User, Phone> map = h.createQuery(sql)
        .registerRowMapper(ConstructorMapper.factory(User.class, "u"))
        .registerRowMapper(ConstructorMapper.factory(Phone.class, "p"))
        .collectInto(new GenericType<Map<User, Phone>>() {});
```

In the preceding example, the `User` mapper uses a "u" column name prefix, and the `Phone` mapper uses "p". Since each mapper only reads columns with the expected prefix, the respective `id` columns are unambiguous.

A unique index (e.g. by ID column) can be obtained by setting the key column name:

```
Map<Integer, User> map = h.createQuery("select * from user")
        .setMapKeyColumn("id")
        .registerRowMapper(ConstructorMapper.factory(User.class))
        .collectInto(new GenericType<Map<Integer, User>>() {});
```

Set both the key and value column names to gather a two-column query into a map result:

```
Map<String, String> map = h.createQuery("select key, value from config")
        .setMapKeyColumn("key")
        .setMapValueColumn("value")
        .collectInto(new GenericType<Map<String, String>>() {});
```

All the above examples assume a one-to-one key/value relationship. What if there is a one-to-many relationship?

Google Guava provides a `Multimap` type, which supports mapping multiple values per key.

First, follow the instructions in the [Google Guava](#_google_guava) section to install `GuavaPlugin` into Jdbi.

Then, simply ask for a `Multimap` instead of a `Map`:

```
String sql = "select u.id u_id, u.name u_name, p.id p_id, p.phone p_phone "
    + "from user u left join phone p on u.id = p.user_id";
Multimap<User, Phone> map = h.createQuery(sql)
        .registerRowMapper(ConstructorMapper.factory(User.class, "u"))
        .registerRowMapper(ConstructorMapper.factory(Phone.class, "p"))
        .collectInto(new GenericType<Multimap<User, Phone>>() {});
```

The `collectInto()` method is worth explaining. When you call it, several things happen behind the scenes:

- Consult the `JdbiCollectors` registry to obtain a [CollectorFactory](apidocs/org/jdbi/v3/core/collector/CollectorFactory.html) which supports the given container type.
- Next, ask that `CollectorFactory` to extract the element type from the container type signature. In the above example, the element type of `Multimap<User,Phone>` is `Map.Entry<User,Phone>`.
- Obtain a mapper for that element type from the mapping registry.
- Obtain a [Collector](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collector.html) for the container type from the `CollectorFactory`.
- Finally, return `map(elementMapper).collect(collector)`.

|      | If the lookup for the collector factory, element type, or element mapper fails, an exception is thrown. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

Jdbi can be enhanced to support arbitrary container types. See [[JdbiCollectors\]](#JdbiCollectors) for more information.

<a name="36____3_6__Templating"></a>
### 3.6. Templating

Binding query parameters, as described above, is excellent for sending a static set of parameters to the database engine. Binding ensures that the parameterized query string (`… where foo = ?`) is transmitted to the database without allowing hostile parameter values to inject SQL.

Bound parameters are not always enough. Sometimes a query needs complicated or structural changes before being executed, and parameters just don’t cut it. Templating (using a `TemplateEngine`) allows you to alter a query’s content with general String manipulations.

Typical uses for templating are optional or repeating segments (conditions and loops), complex variables such as comma-separated lists for IN clauses, and variable substitution for non-bindable SQL elements (like table names). Unlike *argument binding*, the *rendering* of *attributes* performed by TemplateEngines is **not** SQL-aware. Since they perform generic String manipulations, TemplateEngines can easily produce horribly mangled or subtly defective queries if you don’t use them carefully.

|      | [Query templating is a common attack vector!](https://www.xkcd.com/327/) Always prefer binding parameters to static SQL over dynamic SQL when possible. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

```
handle.createQuery("select * from <TABLE> where name = :n")

    // -> "select * from Person where name = :n"
    .define("TABLE", "Person")

    // -> "select * from Person where name = 'MyName'"
    .bind("n", "MyName");
```

|      | Use a TemplateEngine to perform crude String manipulations on a query. Query parameters should be handled by Arguments. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

|      | TemplateEngines and SqlParsers operate sequentially: the initial String will be rendered by the TemplateEngine using attributes, then parsed by the SqlParser with Argument bindings. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

If the TemplateEngine outputs text matching the parameter format of the SqlParser, the parser will attempt to bind an Argument to it. This can be useful to e.g. have named parameters of which the name itself is also a variable, but can also cause confusing bugs:

```
String paramName = "arg";

handle.createQuery("select * from Foo where bar = :<attr>")
    .define("attr", paramName)
    ...
    .bind(paramName, "baz"); // <- does not need to know the parameter's name ("arg")!
handle.createQuery("select * from emotion where emoticon = <sticker>")
    .define("sticker", ":-)") // -> "... where emoticon = :-)"
    .mapToMap()
    // exception: no binding/argument named "-)" present
    .list();
```

Bindings and definitions are usually separate. You can link them in a limited manner using the `stmt.defineNamedBindings()` or `@DefineNamedBindings` customizers. For each bound parameter (including bean properties), this will define a boolean which is `true` if the binding is present and not `null`. You can use this to craft conditional updates and query clauses.

For example,

```
class MyBean {
    long id();
    String getA();
    String getB();
    Instant getModified();
}

handle.createUpdate("update mybeans set <if(a)>a = :a,<endif> <if(b)>b = :b,<endif> modified=now() where id=:id")
    .bindBean(mybean)
    .defineNamedBindings()
    .execute();
```

Also see the section about [TemplateEngine](#_templateengine).

<a name="37____3_7__SQL_Arrays"></a>
### 3.7. SQL Arrays

Jdbi can bind/map Java arrays to/from SQL arrays:

```
handle.createUpdate("insert into groups (id, user_ids) values (:id, :userIds)")
      .bind("id", 1)
      .bind("userIds", new int[] { 10, 5, 70 })
      .execute();

int[] userIds = handle.createQuery("select user_ids from groups where id = :id")
      .bind("id", 1)
      .mapTo(int[].class)
      .one();
```

You can also use Collections in place of arrays, but you’ll need to provide the element type if you’re using the fluent API, since it’s erased:

```
handle.createUpdate("insert into groups (id, user_ids) values (:id, :userIds)")
      .bind("id", 1)
      .bindArray("userIds", int.class, Arrays.asList(10, 5, 70))
      .execute();

List<Integer> userIds = handle.createQuery("select user_ids from groups where id = :id")
      .bind("id", 1)
      .mapTo(new GenericType<List<Integer>>() {})
      .one();
```

Use `@SingleValue` for mapping an array result with the SqlObject API:

```
public interface GroupsDao {
  @SqlQuery("select user_ids from groups where id = ?")
  @SingleValue
  List<Integer> getUserIds(int groupId);
}
```

<a name="38_____3_7_1__Registering_array_types"></a>
#### 3.7.1. Registering array types

Any Java array element type you want binding support for needs to be registered with Jdbi’s `SqlArrayTypes` registry. An array type that is directly supported by your JDBC driver can be registered using:

```
jdbi.registerArrayType(int.class, "integer");
```

Here, `"integer"` is the SQL type name that the JDBC driver supports natively.

|      | Plugins like `PostgresPlugin` and `H2DatabasePlugin` automatically register the most common array element types for their respective databases. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

|      | Postgres supports enum array types, so you can register an array type for `enum Colors { red, blue }` using `jdbi.registerArrayType(Colors.class, "colors")` where `"colors"` is a user-defined enum type name in your database. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

<a name="39_____3_7_2__Binding_custom_array_types"></a>
#### 3.7.2. Binding custom array types

You can also provide your own implementation of `SqlArrayType` that converts a custom Java element type to a type supported by the JDBC driver:

```
class UserArrayType implements SqlArrayType<User> {

    @Override
    public String getTypeName() {
        return "integer";
    }

    @Override
    public Object convertArrayElement(User user) {
        return user.getId();
    }
}
```

You can now bind instances of `User[]` to arguments of data type `integer[]`:

```
User user1 = new User(1, "bob")
User user2 = new User(42, "alice")

handle.registerArrayType(new UserArrayType());
handle.createUpdate("insert into groups (id, user_ids) values (:id, :users)")
      .bind("id", 1)
      .bind("users", new User[] { user1, user2 })
      .execute();
```

|      | Like the [Arguments Registry](#_arguments_registry), if there are multiple `SqlArrayType` s registered for the same data type, the last registered wins. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

<a name="40_____3_7_3__Mapping_array_types"></a>
#### 3.7.3. Mapping array types

`SqlArrayType` only allows you to bind Java array/collection arguments to their SQL counterparts. To map SQL array columns back to Java types, you can register a regular `ColumnMapper`:

```
public class UserIdColumnMapper implements ColumnMapper<UserId> {
    @Override
    public UserId map(ResultSet rs, int col, StatementContext ctx) throws SQLException {
        return new UserId(rs.getInt(col));
    }
}
handle.registerColumnMapper(new UserIdColumnMapper());
List<UserId> userIds = handle.createQuery("select user_ids from groups where id = :id")
      .bind("id", 1)
      .mapTo(new GenericType<List<UserId>>() {})
      .one();
```

|      | Array columns can be mapped to any container type registered with the `JdbiCollectors` registry. E.g. a `VARCHAR[]` may be mapped to an `ImmutableList<String>` if the guava plugin is installed. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

<a name="41____3_8__Results"></a>
### 3.8. Results

After executing a database query, you need to interpret the results. JDBC provides the **ResultSet** class which can do simple mapping to Java primitives and built in classes, but the API is often cumbersome to use. **Jdbi** provides configurable mapping, including the ability to register custom mappers for rows and columns.

A **RowMapper** converts a row of a **ResultSet** into a result object.

A **ColumnMapper** converts a single column’s value into a Java object. It can be used as a **RowMapper** if there is only one column present, or it can be used to build more complex **RowMapper** types.

The mapper is selected based on the declared result type of your query.

**jdbi** iterates over the rows in the ResultSet and presents the mapped results to you in a container such as a **List**, **Stream**, **Optional**, or **Iterator**.

```
public static class User {
    final int id;
    final String name;

    public User(int id, String name) {
        this.id = id;
        this.name = name;
    }
}

@Before
public void setUp() {
    handle.execute("CREATE TABLE user (id INTEGER PRIMARY KEY AUTO_INCREMENT, name VARCHAR)");
    for (String name : Arrays.asList("Alice", "Bob", "Charlie", "Data")) {
        handle.execute("INSERT INTO user(name) VALUES (?)", name);
    }
}

@Test
public void findBob() {
    User u = findUserById(2).orElseThrow(() -> new AssertionError("No user found"));
    assertThat(u.id).isEqualTo(2);
    assertThat(u.name).isEqualTo("Bob");
}

public Optional<User> findUserById(long id) {
    RowMapper<User> userMapper =
            (rs, ctx) -> new User(rs.getInt("id"), rs.getString("name"));
    return handle.createQuery("SELECT * FROM user WHERE id=:id")
        .bind("id", id)
        .map(userMapper)
        .findFirst();
}
```

<a name="42_____3_8_1__ResultBearing"></a>
#### 3.8.1. ResultBearing

The [ResultBearing](apidocs/org/jdbi/v3/core/result/ResultBearing.html) interface represents a result set of a database operation, which has not been mapped to any particular result type.

TODO:

- Query implements ResultBearing
- Update.executeAndReturnGeneratedKeys() returns ResultBearing
- PreparedBatch.executeAndReturnGeneratedKeys() returns ResultBearing
- A ResultBearing object can be mapped, which returns a ResultIterable of the mapped type.
  - mapTo(Type | Class | GenericType) if a mapper is registered for type
  - map(RowMapper | ColumnMapper)
  - mapToBean() for bean types
  - mapToMap() which returns Map<String,Object> mapping lower-cased column names to values
- reduceRows
  - RowView
- reduceResultSet
- collectInto e.g. with a GenericType token. Implies a mapTo() and a collect() in one operation. e.g. collectInto(new GenericType<List<User>>(){}) is the same as mapTo(User.class).collect(toList())
- Provide list of container types supported out of the box

<a name="43_____3_8_2__ResultIterable"></a>
#### 3.8.2. ResultIterable

[ResultIterable](apidocs/org/jdbi/v3/core/result/ResultIterable.html) represents a result set which has been mapped to a specific type, e.g. `ResultIterable<User>`.

TODO:

- ResultIterable.forEach
- ResultIterable.iterator()
  - Must be explicitly closed, to release database resources.
  - Use try-with-resources to ensure database resources get cleaned up. *

<a name="44______Find_a_Single_Result"></a>
##### Find a Single Result

`ResultIterable.one()` returns the only row in the result set. If zero or multiple rows are encountered, it will throw `IllegalStateException`.

`ResultIterable.findOne()` returns an `Optional<T>` of the only row in the result set, or `Optional.empty()` if no rows are returned.

`ResultIterable.first()` returns the first row in the result set. If zero rows are encountered, `IllegalStateException` is thrown.

`ResultIterable.findFirst()` returns an `Optional<T>` of the first row, if any.

<a name="45______Stream"></a>
##### Stream

**Stream** integration allows you to use a RowMapper to adapt a ResultSet into the new Java 8 Streams framework. As long as your database supports streaming results (for example, PostgreSQL will do it as long as you are in a transaction and set a fetch size), the stream will lazily fetch rows from the database as necessary.

**#stream** returns a **Stream<T>**. You should then process the stream and produce a result. This stream must be closed to release any database resources held, so we recommend **useStream**, **withStream** or alternately a **try-with-resources** block to ensure that no resources are leaked.

```
handle.createQuery("SELECT id, name FROM user ORDER BY id ASC")
      .map(new UserMapper())
      .useStream(stream -> {
          Optional<String> first = stream
              .filter(u -> u.id > 2)
              .map(u -> u.name)
              .findFirst();
          assertThat(first).contains("Charlie");
      });
```

**#withStream** and **#useStream** handle closing the stream for you. You provide a **StreamCallback** that produces a result or a **StreamConsumer** that produces no result, respectively.

<a name="46______List"></a>
##### List

**#list** emits a **List<T>**. This necessarily buffers all results in memory.

```
List<User> users =
    handle.createQuery("SELECT id, name FROM user")
        .map(new UserMapper())
        .list();
```

<a name="47______Collectors"></a>
##### Collectors

**#collect** takes a **Collector<T, ? , R>** that builds a resulting collection **R<T>**. The **java.util.stream.Collectors** class has a number of interesting **Collector** implementations to start with.

You can also write your own custom collectors. For example, to accumulate found rows into a **Map**:

```
h.execute("insert into something (id, name) values (1, 'Alice'), (2, 'Bob'), (3, 'Chuckles')");
Map<Integer, Something> users = h.createQuery("select id, name from something")
    .mapTo(Something.class)
    .collect(Collector.of(HashMap::new, (accum, item) -> {
        accum.put(item.getId(), item);   // Each entry is added into an accumulator map
    }, (l, r) -> {
        l.putAll(r);                     // While jdbi does not process rows in parallel,
        return l;                        // the Collector contract encourages writing combiners.
    }, Characteristics.IDENTITY_FINISH));
```

<a name="48______Reduction"></a>
##### Reduction

**#reduce** provides a simplified **Stream#reduce**. Given an identity starting value and a **BiFunction<U, T, U>** it will repeatedly combine **U** until only a single remains, and then return that.

<a name="49______ResultSetScanner"></a>
##### ResultSetScanner

The **ResultSetScanner** interface accepts a lazily-provided **ResultSet** and produces the result Jdbi returns from statement execution.

Most of the above operations are implemented in terms of **ResultSetScanner**. The Scanner has ownership of the ResultSet and may advance or seek it.

The return value ends up being the final result of statement execution.

Most users should prefer using the higher level result collectors described above, but someone’s gotta do the dirty work.

<a name="50_____3_8_3__Joins"></a>
#### 3.8.3. Joins

Joining multiple tables together is a very common database task. It is also where the mismatch between the relational model and Java’s object model starts to rear its ugly head.

Here we present a couple of strategies for retrieving results from more complicated rows.

Consider a contact list app as an example. The contact list contains any number of contacts. Contacts have a name, and any number of phone numbers. Phone numbers have a type (e.g. home, work) and a phone number:

```
class Contact {
  Long id;
  String name;
  List<Phone> phones = new ArrayList<>();

  void addPhone(Phone phone) {
    phones.add(phone);
  }
}

class Phone {
  Long id;
  String type;
  String phone;
}
```

We’ve left out getters, setters, and access modifiers for brevity.

Since we’ll be reusing the same queries, we’ll define them as constants now:

```
static final String SELECT_ALL = "select contacts.id c_id, name c_name, "
    + "phones.id p_id, type p_type, phones.phone p_phone "
    + "from contacts left join phones on contacts.id = phones.contact_id "
    + "order by c_name, p_type ";

static final String SELECT_ONE = SELECT_ALL + "where phones.id = :id";
```

Note that we’ve given aliases (e.g. `c_id`, `p_id`) to distinguish columns of the same name (`id`) from different tables.

Jdbi provides a few different APIs for dealing with joined data.

<a name="51______ResultBearing_reduceRows__"></a>

##### ResultBearing.reduceRows()

The [ResultBearing.reduceRows(U, BiFunction)](apidocs/org/jdbi/v3/core/result/ResultBearing.html#reduceRows-U-java.util.function.BiFunction-) method accepts an accumulator seed value and a lambda function. For each row in the result set, Jdbi calls the lambda with the current accumulator value and a [RowView](apidocs/org/jdbi/v3/core/result/RowView.html) over the current row of the result set. The value returned for each row becomes the input accumulator passed in for the next row. After the last row has been processed, `reducedRows()` returns the last value returned from the lambda.

```
List<Contact> contacts = handle.createQuery(SELECT_ALL)
    .registerRowMapper(BeanMapper.factory(Contact.class, "c"))
    .registerRowMapper(BeanMapper.factory(Phone.class, "p")) 
    .reduceRows(new LinkedHashMap<Long, Contact>(), 
                (map, rowView) -> {
      Contact contact = map.computeIfAbsent( 
          rowView.getColumn("c_id", Long.class),
          id -> rowView.getRow(Contact.class));

      if (rowView.getColumn("p_id", Long.class) != null) { 
        contact.addPhone(rowView.getRow(Phone.class));
      }

      return map; 
    })
    .values() 
    .stream()
    .collect(toList()); 
```

|      | Register row mappers for `Contact` and `Phone`. Note the `"c"` and `"p"` arguments used—these are column name prefixes. By registering mappers with prefixes, the `Contact` mapper will only map the `c_id` and `c_name` columns, whereas the `Phone` mapper will only map `p_id`, `p_type`, and `p_phone`. |
| ---- | ------------------------------------------------------------ |
|      | Use an empty [LinkedHashMap](https://docs.oracle.com/javase/8/docs/api/java/util/LinkedHashMap.html) as the accumulator seed, mapped by contact ID. `LinkedHashMap` is a good accumulator when selecting multiple master records, since it has fast storage and lookup while preserving insertion order (which helps honor `ORDER BY` clauses). If ordering is unimportant, a `HashMap` would also suffice. |
|      | Load the `Contact` from the accumulator if we already have it; otherwise, initialize it through the `RowView`. |
|      | If `p_id` column is not null, load the phone number from the current row and add it to the current contact. |
|      | Return the input map (now sporting an additional contact and/or phone) as the accumulator for the next row. |
|      | At this point, all rows have been read into memory, and we don’t need the contact ID keys. So we call `Map.values()` to get a `Collection<Contact>`. |
|      | Collect the contacts into a `List<Contact>`.                 |

Alternatively, the [ResultBearing.reduceRows(RowReducer)](apidocs/org/jdbi/v3/core/result/ResultBearing.html#reduceRows-org.jdbi.v3.core.result.RowReducer-) variant accepts a [RowReducer](apidocs/org/jdbi/v3/core/result/RowReducer.html) and returns a stream of reduced elements.

For simple master-detail joins, the [ResultBearing.reduceRows(BiConsumer method makes it easy to reduce these joins into a stream of master elements.

Adapting the example above:

```
List<Contact> contacts = handle.createQuery(SELECT_ALL)
    .registerRowMapper(BeanMapper.factory(Contact.class, "c"))
    .registerRowMapper(BeanMapper.factory(Phone.class, "p"))
    .reduceRows((Map<Long, Contact> map, RowView rowView) -> { 
      Contact contact = map.computeIfAbsent(
          rowView.getColumn("c_id", Long.class),
          id -> rowView.getRow(Contact.class));

      if (rowView.getColumn("p_id", Long.class) != null) {
        contact.addPhone(rowView.getRow(Phone.class));
      }
      
    })
    .collect(toList()); 
```

|      | The lambda receives a map where result objects will be stored, and a `RowView`. The map is a `LinkedHashMap`, so the result stream will yield the result objects in the same order they were inserted. |
| ---- | ------------------------------------------------------------ |
|      | No `return` statement needed. The same `map` is reused on every row. |
|      | This `reduceRows()` invocation produces a `Stream<Contact>` (i.e. from `map.values().stream()`. In this example, we collect the elements into a list, but we could call any `Stream` method here. |

You may be wondering about the `getRow()` and `getColumn()` calls to `rowView`. When you call `rowView.getRow(SomeType.class)`, `RowView` looks up the registered row mapper for `SomeType`, and uses it to map the current row to a `SomeType` object.

Likewise, when you call `rowView.getColumn("my_value", MyValueType.class)`, `RowView` looks up the registered column mapper for `MyValueType`, and uses it to map the `my_value` column of the current row to a `MyValueType` object.

Now let’s do the same thing, but for a single contact:

```
Optional<Contact> contact = handle.createQuery(SELECT_ONE)
    .bind("id", contactId)
    .registerRowMapper(BeanMapper.factory(Contact.class, "c"))
    .registerRowMapper(BeanMapper.factory(Phone.class, "p"))
    .reduceRows(LinkedHashMapRowReducer.<Long, Contact> of((map, rowView) -> {
      Contact contact = map.orElseGet(() -> rowView.getRow(Contact.class));

      if (rowView.getColumn("p_id", Long.class) != null) {
        contact.addPhone(rowView.getRow(Phone.class));
      }
    })
    .findFirst();
```

<a name="52______ResultBearing_reduceResultSet__"></a>
##### ResultBearing.reduceResultSet()

[ResultBearing.reduceResultSet()](apidocs/org/jdbi/v3/core/result/ResultBearing.html#reduceResultSet-U-org.jdbi.v3.core.result.ResultSetAccumulator-) is a low-level API similar to `reduceRows()`, except it provides direct access to the JDBC `ResultSet` instead of a `RowView` for each row.

This method can provide superior performance compared to `reduceRows()`, at the expense of verbosity:

```
List<Contact> contacts = handle.createQuery(SELECT_ALL)
    .reduceResultSet(new LinkedHashMap<Long, Contact>(),
                     (acc, resultSet, ctx) -> {
      long contactId = resultSet.getLong("c_id");
      Contact contact;
      if (acc.containsKey(contactId)) {
        contact = acc.get(contactId);
      } else {
        contact = new Contact();
        acc.put(contactId,contact);
        contact.setId(contactId);
        contact.setName(resultSet.getString("c_name");
      }

      long phoneId = resultSet.getLong("p_id");
      if (!resultSet.wasNull()) {
        Phone phone = new Phone();
        phone.setId(phoneId);
        phone.setType(resultSet.getString("p_type");
        phone.setPhone(resultSet.getString("p_phone");
        contact.addPhone(phone);
      }

      return acc;
    })
    .values()
    .stream()
    .collect(toList());
```

<a name="53______JoinRowMapper"></a>
##### JoinRowMapper

The JoinRowMapper takes a set of types to extract from each row. It uses the mapping registry to determine how to map each given type, and presents you with a JoinRow that holds all of the resulting values.

Let’s consider two simple types, User and Article, with a join table named Author. Guava provides a Multimap class which is very handy for representing joined tables like this. Assuming we have mappers already registered:

```
h.registerRowMapper(ConstructorMapper.factory(User.class));
h.registerRowMapper(ConstructorMapper.factory(Article.class));
```

we can then easily populate a Multimap with the mapping from the database:

```
Multimap<User, Article> joined = HashMultimap.create();
h.createQuery("SELECT * FROM user NATURAL JOIN author NATURAL JOIN article")
    .map(JoinRowMapper.forTypes(User.class, Article.class))
    .forEach(jr -> joined.put(jr.get(User.class), jr.get(Article.class)));
```

|      | While this approach is easy to read and write, it can be inefficient for certain patterns of data. Consider performance requirements when deciding whether to use high level mapping or more direct low level access with handwritten mappers. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

You can also use it with SqlObject:

```
public interface UserArticleDao {
    @RegisterJoinRowMapper({User.class, Article.class})
    @SqlQuery("SELECT * FROM user NATURAL JOIN author NATURAL JOIN article")
    Stream<JoinRow> getAuthorship();
}
Multimap<User, Article> joined = HashMultimap.create();

handle.attach(UserArticleDao.class)
        .getAuthorship()
        .forEach(jr -> joined.put(jr.get(User.class), jr.get(Article.class)));

assertThat(joined).isEqualTo(JoinRowMapperTest.getExpected());
```

<a name="54____3_9__Updates"></a>
### 3.9. Updates

Updates are operations that return an integer number of rows modified, such as a database **INSERT**, **UPDATE**, or **DELETE**.

You can execute a simple update with `Handle`'s `int execute(String sql, Object… args)` method which binds simple positional parameters.

```
count = handle.execute("INSERT INTO user(id, name) VALUES(?, ?)", 4, "Alice");
assertThat(count).isEqualTo(1);
```

To further customize, use `createUpdate`:

```
int count = handle.createUpdate("INSERT INTO user(id, name) VALUES(:id, :name)")
    .bind("id", 3)
    .bind("name", "Charlie")
    .execute();
assertThat(count).isEqualTo(1);
```

Updates may return [Generated Keys](#_generated_keys) instead of a result count.

<a name="55____3_10__Batches"></a>
### 3.10. Batches

A **Batch** sends many commands to the server in bulk.

After opening the batch, repeated add statements, and invoke **add**.

```
Batch batch = handle.createBatch();

batch.add("INSERT INTO fruit VALUES(0, 'apple')");
batch.add("INSERT INTO fruit VALUES(1, 'banana')");

int[] rowsModified = batch.execute();
```

The statements are sent to the database in bulk, but each statement is executed separately. There are no parameters. Each statement returns a modification count, as with an Update, and those counts are then returned in an `int[]` array. In common cases all elements will be `1`.

<a name="56____3_11__Prepared_Batches"></a>

### 3.11. Prepared Batches

A **PreparedBatch** sends one statement to the server with many argument sets. The statement is executed repeatedly, once for each batch of arguments that is **add**-ed to it.

The result is again a `int[]` of modified row count.

```
PreparedBatch batch = handle.prepareBatch("INSERT INTO user(id, name) VALUES(:id, :name)");
for (int i = 100; i < 5000; i++) {
    batch.bind("id", i).bind("name", "User:" + i).add();
}
int[] counts = batch.execute();
```

SqlObject also supports batch inserts:

```
public void testSqlObjectBatch() {
    BasketOfFruit basket = handle.attach(BasketOfFruit.class);

    int[] rowsModified = basket.fillBasket(Arrays.asList(
            new Fruit(0, "apple"),
            new Fruit(1, "banana")));

    assertThat(rowsModified).containsExactly(1, 1);
    assertThat(basket.countFruit()).isEqualTo(2);
}

public interface BasketOfFruit {
    @SqlBatch("INSERT INTO fruit VALUES(:id, :name)")
    int[] fillBasket(@BindBean Collection<Fruit> fruits);

    @SqlQuery("SELECT count(1) FROM fruit")
    int countFruit();
}
```

|      | Batching dramatically increases efficiency over repeated single statement execution, but many databases don’t handle extremely large batches well either. Test with your database configuration, but often extremely large data sets should be divided and committed in pieces - or risk bringing your database to its knees. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

<a name="57_____3_11_1__Exception_Rewriting"></a>
#### 3.11.1. Exception Rewriting

The JDBC SQLException class is very old and predates more modern exception facilities like Throwable’s suppressed exceptions. When a batch fails, there may be multiple failures to report, which could not be represented by the base Exception types of the day.

So SQLException has a bespoke [getNextException](https://docs.oracle.com/javase/8/docs/api/java/sql/SQLException.html#getNextException--) chain to represent the causes of a batch failure. Unfortunately, by default most logging libraries do not print these exceptions out, pushing their handling into your code. It is very common to forget to handle this situation and end up with logs that say nothing other than

```
java.sql.BatchUpdateException: Batch entry 1 insert into something (id, name) values (0, '') was aborted. Call getNextException to see the cause.
```

**jdbi** will attempt to rewrite such nextExceptions into "suppressed exceptions" (new in Java 8) so that your logs are more helpful:

```
java.sql.BatchUpdateException: Batch entry 1 insert into something (id, name) values (0, 'Keith') was aborted. Call getNextException to see the cause.
Suppressed: org.postgresql.util.PSQLException: ERROR: duplicate key value violates unique constraint "something_pkey"
  Detail: Key (id)=(0) already exists.
```

<a name="58____3_12__Generated_Keys"></a>
### 3.12. Generated Keys

An Update or PreparedBatch may automatically generate keys. These keys are treated separately from normal results. Depending on your database and configuration, the entire inserted row may be available.

|      | Unfortunately there is a lot of variation between databases supporting this feature so please test this feature’s interaction with your database thoroughly. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

In PostgreSQL, the entire row is available, so you can immediately map your inserted names back to full User objects! This avoids the overhead of separately querying after the insert completes.

Consider the following table:

```
public static class User {
    final int id;
    final String name;

    public User(int id, String name) {
        this.id = id;
        this.name = name;
    }
}

@Before
public void setUp() {
    db.useHandle(h -> h.execute("CREATE TABLE users (id SERIAL PRIMARY KEY, name VARCHAR)"));
    db.registerRowMapper(ConstructorMapper.factory(User.class));
}
```

You can get generated keys in the fluent style:

```
public void fluentInsertKeys() {
    db.useHandle(handle -> {
        User data = handle.createUpdate("INSERT INTO users (name) VALUES(?)")
                .bind(0, "Data")
                .executeAndReturnGeneratedKeys()
                .mapTo(User.class)
                .one();

        assertEquals(1, data.id); // This value is generated by the database
        assertEquals("Data", data.name);
    });
}
```

<a name="59____3_13__Stored_Procedure_Calls"></a>
### 3.13. Stored Procedure Calls

A **Call** invokes a database stored procedure.

Let’s assume an existing stored procedure as an example:

```
CREATE FUNCTION add(a IN INT, b IN INT, sum OUT INT) AS $$
BEGIN
  sum := a + b;
END;
$$ LANGUAGE plpgsql
```

Here’s how to call a stored procedure:

```
OutParameters result = handle
        .createCall("{:sum = call add(:a, :b)}") 
        .bind("a", 13) 
        .bind("b", 9) 
        .registerOutParameter("sum", Types.INTEGER)   
        .invoke(); 
```

|      | Call `Handle.createCall()` with the SQL statement. Note that JDBC has a peculiar SQL format when calling stored procedures, which we must follow. |
| ---- | ------------------------------------------------------------ |
|      | Bind input parameters to the procedure call.                 |
|      | Register out parameters, the values that will be returned from the stored procedure call. This tells JDBC what data type to expect for each out parameter. |
|      | Out parameters may be registered by name (as shown in the example) or by zero-based index, if the SQL is using positional parameters. Multiple output parameters may be registered, depending on the output of the stored procedure itself. |
|      | Finally, call `invoke()` to execute the procedure.           |

Invoking the stored procedure returns an [OutParameters](apidocs/org/jdbi/v3/core/statement/OutParameters.html) object, which contains the value(s) returned from the stored procedure call.

Now we can extract the result(s) from `OutParameters`:

```
int sum = result.getInt("sum");
```

It is possible to return open cursors as a result-like object by declaring it as `Types.REF_CURSOR` and then inspecting it via `OutParameters.getRowSet()`. Usually this must be done in a transaction, and the results must be consumed before closing the statement by processing it using the `Call.invoke(Consumer)` or `Call.invoke(Function)` callback style.

|      | Due to design constraints within JDBC, the parameter data types available through `OutParameters` is limited to those types supported directly by JDBC. This cannot be expanded through e.g. mapper registration. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

<a name="60____3_14__Scripts"></a>
### 3.14. Scripts

A **Script** parses a String into semicolon terminated statements. The statements can be executed in a single **Batch** or individually.

```
int[] results = handle.createScript(
        "INSERT INTO user VALUES(3, 'Charlie');"
        + "UPDATE user SET name='Bobby Tables' WHERE id=2;")
    .execute();

assertThat(results).containsExactly(1, 1);
```

<a name="61____3_15__Transactions"></a>
### 3.15. Transactions

**jdbi** provides full support for JDBC transactions.

**Handle** objects provide two ways to open a transaction — **inTransaction** allows you to return a result, and **useTransaction** has no return value.

Both optionally allow you to specify the transaction isolation level.

```
public Optional<User> findUserById(long id) {
    return handle.inTransaction(h ->
            h.createQuery("SELECT * FROM users WHERE id=:id")
                    .bind("id", id)
                    .mapTo(User.class)
                    .findFirst());
}
```

Here, we (probably unnecessarily) guard a simple *SELECT* statement with a transaction.

Additionally, Handle has a number of methods for direct transaction management: begin(), savepoint(), rollback(), commit(), etc. Normally, you will not need to use these. If you do not explicitly commit a manually opened transaction, it will be rolled back.

<a name="62_____3_15_1__Serializable_Transactions"></a>
#### 3.15.1. Serializable Transactions

For more advanced queries, sometimes serializable transactions are required. **jdbi** includes a transaction runner that is able to retry transactions that abort due to serialization failures. It is important that your transaction does not have side effects as it may be executed multiple times.

```
// Automatically rerun transactions
db.setTransactionHandler(new SerializableTransactionRunner());

// Set up some values
BiConsumer<Handle, Integer> insert = (h, i) -> h.execute("INSERT INTO ints(value) VALUES(?)", i);
handle.execute("CREATE TABLE ints (value INTEGER)");
insert.accept(handle, 10);
insert.accept(handle, 20);

// Run the following twice in parallel, and synchronize
ExecutorService executor = Executors.newCachedThreadPool();
CountDownLatch latch = new CountDownLatch(2);

Callable<Integer> sumAndInsert = () ->
    db.inTransaction(TransactionIsolationLevel.SERIALIZABLE, h -> {
        // Both read initial state of table
        int sum = h.select("SELECT sum(value) FROM ints").mapTo(int.class).one();

        // First time through, make sure neither transaction writes until both have read
        latch.countDown();
        latch.await();

        // Now do the write.
        insert.accept(h, sum);
        return sum;
    });

// Both of these would calculate 10 + 20 = 30, but that violates serialization!
Future<Integer> result1 = executor.submit(sumAndInsert);
Future<Integer> result2 = executor.submit(sumAndInsert);

// One of the transactions gets 30, the other will abort and automatically rerun.
// On the second attempt it will compute 10 + 20 + 30 = 60, seeing the update from its sibling.
// This assertion fails under any isolation level below SERIALIZABLE!
assertThat(result1.get() + result2.get()).isEqualTo(30 + 60);

executor.shutdown();
```

The above test is designed to run two transactions in lock step. Each attempts to read the sum of all rows in the table, and then insert a new row with that sum. We seed the table with the values 10 and 20.

Without serializable isolation, each transaction reads 10 and 20, and then returns 30. The end result is 30 + 30 = 60, which does not correspond to any serial execution of the transactions!

With serializable isolation, one of the two transactions is forced to abort and retry. On the second go around, it calculates 10 + 20 + 30 = 60. Adding to 30 from the other, we get 30 + 60 = 90 and the assertion succeeds.

<a name="63____3_16__ClasspathSqlLocator"></a>
### 3.16. ClasspathSqlLocator

You may find it helpful to store your SQL templates in individual files on the classpath, rather than in string inside Java code.

The `ClasspathSqlLocator` converts Java type and method names into classpath locations, and then reads, parses, and caches the loaded statements.

```
// reads classpath resource com/foo/BarDao/query.sql
ClasspathSqlLocator.findSqlOnClasspath(com.foo.BarDao.class, "query");

// same resource as above
ClasspathSqlLocator.findSqlOnClasspath("com.foo.BarDao.query");
```

<a name="64___4__Configuration"></a>
## 4. Configuration

`Jdbi` aims to be useful out of the box with minimal configuration. Sometimes you need to change default behavior, or add in extensions to handle additional database types. Each piece of core or extension that wishes to participate in configuration defines a configuration class, for example the `SqlStatements` class stores SqlStatement related configuration. Then, on any `Configurable` context (like a `Jdbi` or `Handle`) you can change configuration in a type safe way:

```
jdbi.getConfig(SqlStatements.class).setUnusedBindingAllowed(true);
jdbi.getConfig(Arguments.class).register(new MyTypeArgumentFactory());
jdbi.getConfig(Handles.class).setForceEndTransactions(true);

// Or, if you have a bunch of work to do:
jdbi.configure(RowMappers.class, rm -> {
    rm.register(new TypeARowMapperFactory();
    rm.register(new TypeBRowMapperFactory();
});
```

Generally, you should finalize all configuration changes before interacting with the database.

When a new context is created, it inherits a copy of the parent context configuration at the time of creation. So a `Handle` initializes its configuration from the creating `Jdbi`, but changes never propagate back up.

See [JdbiConfig](#_jdbiconfig) for more advanced implementation details.

<a name="65____4_1__Qualified_Types"></a>
### 4.1. Qualified Types

Sometimes the same Java object can correspond to multiple data types in a database. For example, a `String` could be `varchar` plaintext, `nvarchar` text, `json` data, etc, all with different handling requirements.

[QualifiedType](apidocs/org/jdbi/v3/core/qualifier/QualifiedType.html) allows you to add such context to a Java type:

```
QualifiedType.of(String.class).with(Json.class);
```

This `QualifiedType` still represents the `String` *type*, but *qualified* with the `@Json` annotation. It can be used in a way similar to [GenericType](#_generictype), to make components handling values (mainly `ArgumentFactories` and `ColumnMapperFactories`) perform their work differently, and to have the values handled by different implementations altogether:

```
@Json
public class JsonArgumentFactory extends AbstractArgumentFactory<String> {
    @Override
    protected Argument build(String value, ConfigRegistry config) {
        // do something specifically for json data
    }
}
```

Once registered, this `@Json` qualified factory will receive **only** `@Json String` values. Other factories *not* qualified as such will **not** receive this value:

```
QualifiedType<String> json = QualifiedType.of(String.class).with(Json.class);
query.bindByType("jsonValue", "{\"foo\":1}", json);
```

|      | Jdbi chooses factories to handle values by **exactly matching** their *qualifiers*. It’s up to the factory implementations to discriminate on the *type* of the value afterwards. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

|      | Qualifiers are implemented as `Annotations`. This allows factories to independently inspect values for qualifiers at the source, such as on their `Class`, to alter their own behavior or to *requalify* a value and have it re-evaluated by Jdbi’s lookup chain. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

|      | Qualifiers being annotations does **not** mean they inherently activate their function when placed in source classes. Each feature decides its own rules regarding their use. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

|      | Arguments can only be qualified for binding via `bindByType` calls, not regular `bind` or `update.execute(Object…)`. Also, arrays cannot be qualified. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

These features currently make use of qualified types:

- `@NVarchar` and `@MacAddr` (the latter in `jdbi3-postgres`) bind and map Strings as `nvarchar` and `macaddr` respectively, instead of the usual `varchar`.
- `jdbi3-postgres` offers [HStore](#_hstore).
- [JSON](#jdbi3-json)
- `BeanMapper`, `@BindBean`, `@RegisterBeanMapper`, `mapTobean()`, and `bindBean()` respect qualifiers on getters, setters, and setter parameters.
- `ConstructorMapper` and `@RegisterConstructorMapper` respect qualifiers on constructor parameters.
- `@BindMethods` and `bindMethods()` respect qualifiers on methods.
- `@BindFields`, `@RegisterFieldMapper`, `FieldMapper` and `bindFields()` respect qualifiers on fields.
- `SqlObject` respects qualifiers on methods (applies them to the return type) and parameters.
  - on parameters of type `Consumer<T>`, qualifiers are applied to the `T`.
- `@MapTo`
- `@BindJpa` and `JpaMapper` respect qualifiers on getters and setters.
- `@BindKotlin`, `bindKotlin()`, and `KotlinMapper` respect qualifiers on constructor parameters, getters, setters, and setter parameters.

<a name="66___5__SQL_Objects_SQL对象_"></a>
## 5. SQL Objects(SQL对象)

SQL对象是流畅式核心API的声明式替代。

要开始使用SQL Object插件，添加一个Maven依赖项:

```xml
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-sqlobject</artifactId>
</dependency>
```

然后将插件安装到你的 `Jdbi` 实例中：

```java
Jdbi jdbi = ...
jdbi.installPlugin(new SqlObjectPlugin());
```

使用 SQL Object，您可以声明一个公共接口，为每个数据库操作添加方法，并指定要执行的 SQL 语句。

你可以用以下两种方式指定每个方法的功能:

- 使用 `SQL方法注解` 对方法进行注解。 Jdbi 提供了四种开箱即用的注解（更新、查询、存储过程调用和批处理）。
- 将该方法声明为 Java 8 `default` 方法，并在方法主体中提供您自己的实现。

在运行时，您可以请求接口的实例，Jdbi 会根据您声明的注解和方法合成一个实现。

<a name="67____5_1__Annotated_Methods_注解方法_"></a>
### 5.1. Annotated Methods(注解方法)

使用Jdbi的SQL方法注释 ([@SqlBatch](apidocs/org/jdbi/v3/sqlobject/statement/SqlBatch.html), [@SqlCall](apidocs/org/jdbi/v3/sqlobject/statement/SqlCall.html), [@SqlQuery](apidocs/org/jdbi/v3/sqlobject/statement/SqlQuery.html), or [@SqlUpdate](apidocs/org/jdbi/v3/sqlobject/statement/SqlUpdate.html))注释的方法将基于方法上的注释及其参数自动生成实现。方法的参数用作语句的参数，SQL语句结果映射到方法返回类型。

<a name="68_____5_1_1__@SqlUpdate"></a>
#### 5.1.1. @SqlUpdate

将 `@SqlUpdate` 注解用于修改数据的操作（即插入、更新、删除）。

```java
public interface UserDao {
  @SqlUpdate("insert into users (id, name) values (?, ?)")
  void insert(long id, String name);
}
```

方法参数绑定到 SQL 语句中各自位置的`?`标记。 所以 `id` 绑定到第一个 `?`，而 `name` 绑定到第二个。

> **💡提示:** `@SqlUpdate` 也可用于 DDL（数据定义语言）操作，如创建或更改表。 我们建议使用架构迁移工具，例如 [Flyway](https://flywaydb.org/) 或 [Liquibase](http://www.liquibase.org/) 来维护您的数据库架构。

默认情况下，`@SqlUpdate` 方法可能会返回一些类型：

- `void`: 不返回任何内容（显然）
- `int` 或者 `long`: 返回更新计数。根据数据库供应商和JDBC驱动程序，这可能是更改的行数，也可能是查询匹配的行数(不管是否更改了任何数据)。
- `boolean`: 如果更新计数大于零则返回true。

<a name="69______@GetGeneratedKeys"></a>
##### @GetGeneratedKeys

有些SQL语句会在数据库中代表您生成数据，例如带有自动生成的主键的表，或从序列中选择的主键。我们需要一种方法从数据库中检索这些生成的值。

> 数据库对生成主键的支持各不相同。有些只支持每条语句生成一个键列，有些(如Postgres)可以返回整个行。在依赖此行为之前，您应该检查数据库供应商的文档。

`@GetGeneratedKeys` 注解告诉 Jdbi，返回值应该是从 SQL 语句生成的键，而不是更新计数。

```java
public interface UserDao {
  @SqlUpdate("insert into users (id, name) values (nextval('user_seq'), ?)")
  @GetGeneratedKeys("id")
  long insert(String name);
}
```

可以通过这种方式生成和返回多个列：

```java
public interface UserDao {
  @SqlUpdate("insert into users (id, name, created_on) values (nextval('user_seq'), ?, now())")
  @GetGeneratedKeys({"id", "created_on"})
  @RegisterBeanMapper(IdCreateTime.class)
  IdCreateTime insert(String name);
}
```

> **💡提示:** One True Database在返回生成的键时支持附加功能。详见[PostgreSQL](#__getgeneratedkeys_4)。

<a name="70_____5_1_2__绑定参数"></a>
#### 5.1.2. 绑定参数

在我们继续使用其他 `@Sql__` 注释之前，让我们讨论如何将方法参数作为参数绑定到 SQL 语句。

默认情况下，传递给该方法的参数被绑定为 SQL 语句中的位置参数。

```java
public interface UserDao {
  @SqlUpdate("insert into users (id, name) values (?, ?)")
  void insert(long id, String name);
}
```

您可以使用带有 `@Bind` 注解的命名参数：

```java
@SqlUpdate("insert into users (id, name) values (:id, :name)")
void insert(@Bind("id") long id, @Bind("name") String name);
```

[使用参数名称编译](#133____9_2__使用参数名称编译) 消除了对`@Bind` 注解的需要。 然后 Jdbi 会将每个未注解的参数绑定到参数的名称。

```java
@SqlUpdate("insert into users (id, name) values (:id, :name)")
void insert(long id, String name);
```

绑定值列表是通过`@BindList` 注释完成的。 这将以“a,b,c,d,...”形式展开列表。 请注意，此注释要求您使用 `<绑定>` 符号，这与 `@Bind`（使用 `:绑定`）不同：

```java
@SqlQuery("select name from users where id in (<userIds>)")
List<String> getFromIds(@BindList("userIds") List<Long> userIds)
```

您可以从`Map` 的条目进行绑定：

```java
@SqlUpdate("insert into users (id, name) values (:id, :name)")
void insert(@BindMap Map<String, ?> map);
```

在SQL Object中(但不在Core中)，你可以用前缀限定一个绑定映射:

```java
@SqlUpdate("insert into users (id, name) values (:user.id, :user.name)")
void insert(@BindMap("user") Map<String, ?> map);
```

你可以从Java Bean的属性进行绑定:

```java
@SqlUpdate("insert into users (id, name) values (:id, :name)")
void insert(@BindBean User user);
```

你可以从对象的公共字段进行绑定:

```java
@SqlUpdate("insert into users (id, name) values (:id, :name)")
void insert(@BindFields User user);
```

也可以从对象的公共无参数方法进行绑定:

```java
@SqlUpdate("insert into users (id, name) values (:functionThatReturnsTheId, :functionThatReturnsTheName)")
void insert(@BindMethods User user);
```

像 `@BindMap` 一样，`@BindBean`、`@BindMethods` 和 `@BindFields` 注解可以有一个可选的前缀：

```java
@SqlUpdate("insert into users (id, name) values (:user.id, :user.name)")
void insert(@BindBean("user") User user);
//void insert(@BindFields("user") User user);
//void insert(@BindMethods("user") User user);
```

> **🏷注意:** 在核心 API 中，`@BindBean`、`@BindFields` 和 `@BindMethods` 可用于绑定嵌套属性，例如 `:user.address.street`。

> **☢警告:** `@BindMap` 不绑定嵌套属性——映射键应该与绑定的参数名称完全匹配。

<a name="71_____5_1_3__@SqlQuery"></a>
#### 5.1.3. @SqlQuery

使用 `@SqlQuery` 注解进行选择操作。

查询方法可能返回单行或多行结果，具体取决于方法返回类型是否类似于集合。

```java
public interface UserDao {
  @SqlQuery("select name from users")
  List<String> listNames(); <1>

  @SqlQuery("select name from users where id = ?")
  String getName(long id);   <2> <3>

  @SqlQuery("select name from users where id = ?")
  Optional<String> findName(long id); <4>
}
```

| <1>  | 当多行方法返回空结果集时，将返回空集合。                     |
| ---- | ------------------------------------------------------------ |
| <2>  | 如果单行方法从查询中返回多行，则该方法只返回结果集中的第一行。 |
| <3>  | 如果单行方法返回空结果集，则返回 `null`。                    |
| <4>  | 方法可能返回“可选”值。 如果查询没有返回任何行（或者如果行中的值为 null），则返回 `Optional.empty()` 而不是 null。 如果查询返回多于一行，SQL 对象将引发异常。 |

> **💡提示:** 通过向 [JdbiCollectors](#JdbiCollectors) 配置注册表注册 [CollectorFactory](apidocs/org/jdbi/v3/core/collector/CollectorFactory.html)，可以“教”Jdbi 识别新的集合类型。

请参阅 [BuiltInCollectorFactory](apidocs/org/jdbi/v3/core/collector/BuiltInCollectorFactory.html) 以获取开箱即用支持的集合类型的完整列表。 某些 Jdbi 插件（例如`GuavaPlugin`）注册额外的集合类型。

查询方法也可能返回 [ResultIterable](#_resultiterable)、[ResultIterator](apidocs/org/jdbi/v3/core/result/ResultIterator.html) 或 `Stream`。

```java
public interface UserDao {
  @SqlQuery("select name from users")
  ResultIterable<String> getNamesAsIterable();

  @SqlQuery("select name from users")
  ResultIterator<String> getNamesAsIterator();

  @SqlQuery("select name from users")
  Stream<String> getNamesAsStream();
}
```

从这些方法返回的对象包含数据库资源，当您使用它们时必须显式关闭这些资源。 我们强烈建议在调用这些方法时使用 try-with-resource 块，以防止资源泄漏：

```java
try (ResultIterable<String> names = dao.getNamesAsIterable()) {
  ...
}

try (ResultIterator<String> names = dao.getNamesAsIterator()) {
  ...
}

try (Stream<String> names = dao.getNamesAsStream()) {
  ...
}
```

> **☢警告:** `ResultIterable`、`ResultIterator` 和 `Stream` 方法不适用于按需(on-demand) SQL对象。 除非以嵌套方式调用方法（请参阅 [On-Demand](#97_____5_5_3__On_Demand)），返回的对象将已经关闭。

<a name="72______@RegisterRowMapper"></a>

##### @RegisterRowMapper

使用 `@RegisterRowMapper` 注册一个具体的行映射器类：

```java
public interface UserDao {
  @SqlQuery("select * from users")
  @RegisterRowMapper(UserMapper.class)
  List<User> list();
}
```

与此注解一起使用的行映射器必须满足一些要求：

```java
public class UserMapper implements RowMapper<User> {   <1> <2>
  public UserMapper() { <3>
    ...
  }

  public T map(ResultSet rs, StatementContext ctx) throws SQLException {
    ...
  }
}
```

| <1>  | 必须是一个公共类。                                           |
| ---- | ------------------------------------------------------------ |
| <2>  | 必须使用显式类型参数（例如，`RowMapper<User>`）而不是类型变量（例如`RowMapper<T>`）来实现`RowMapper`。 |
| <3>  | 必须有一个公共的、无参数的构造函数（或一个默认构造函数）。   |

> **💡提示:** `@RegisterRowMapper` 注解可以在同一类型或方法上重复多次以注册多个映射器。

<a name="73______@RegisterRowMapperFactory"></a>
##### @RegisterRowMapperFactory

使用 `@RegisterRowMapperFactory` 注册一个 `RowMapperFactory`。

```java
public interface UserDao {
  @SqlQuery("select * from users")
  @RegisterRowMapperFactory(UserMapperFactory.class)
  List<User> list();
}
```

与此注解一起使用的行映射器工厂必须满足一些要求：

```java
public class UserMapperFactory implements RowMapperFactory { <1>
  public UserMapperFactory() { <2>
    ...
  }

  public Optional<RowMapper<?>> build(Type type, ConfigRegistry config) {
    ...
  }
}
```

| <1>  | 必须是一个公共类。                                         |
| ---- | ---------------------------------------------------------- |
| <2>  | 必须有一个公共的、无参数的构造函数（或一个默认构造函数）。 |

> **💡提示:** `@RegisterRowMapperFactory` 注解可以在同一类型或方法上重复多次以注册多个工厂。

<a name="74______@RegisterColumnMapper"></a>
##### @RegisterColumnMapper

使用 `@RegisterColumnMapper` 来注册一个列映射器：

```java
public interface AccountDao {
  @SqlQuery("select balance from accounts where id = ?")
  @RegisterColumnMapper(MoneyMapper.class)
  Money getBalance(long id);
}
```

与此注解一起使用的列映射器必须满足一些要求：

```java
public class MoneyMapper implements ColumnMapper<Money> {   <1> <2>
  public MoneyMapper() { <3>
    ...
  }

  public T map(ResultSet r, int columnNumber, StatementContext ctx) throws SQLException {
    ...
  }
}
```

| <1>  | 必须是一个公共类。                                           |
| ---- | ------------------------------------------------------------ |
| <2>  | 必须使用显式类型参数（例如 `ColumnMapper<User>`）而不是类型变量（例如 `ColumnMapper<T>`）来实现 `ColumnMapper`。 |
| <3>  | 必须有一个公共的、无参数的构造函数（或一个默认构造函数）。   |

> **💡提示:** `@RegisterColumnMapper` 注解可以在同一类型或方法上重复多次以注册多个映射器。

<a name="75______@RegisterColumnMapperFactory"></a>
##### @RegisterColumnMapperFactory

使用 `@RegisterColumnMapperFactory` 注册列映射器工厂：

```java
public interface AccountDao {
  @SqlQuery("select * from users")
  @RegisterColumnMapperFactory(MoneyMapperFactory.class)
  List<User> list();
}
```

与此注解一起使用的列映射器工厂必须满足一些要求：

```java
public class UserMapperFactory implements ColumnMapperFactory { <1>
  public UserMapperFactory() { <2>
    ...
  }

  public Optional<ColumnMapper<?>> build(Type type, ConfigRegistry config) {
    ...
  }
}
```

| <1>  | 必须是一个公共类。                                         |
| ---- | ---------------------------------------------------------- |
| <2>  | 必须有一个公共的、无参数的构造函数（或一个默认构造函数）。 |

> **💡提示:** `@RegisterColumnMapperFactory` 注解可以在同一类型或方法上重复多次以注册多个工厂。

<a name="76______@RegisterBeanMapper"></a>
##### @RegisterBeanMapper

使用 `@RegisterBeanMapper` 为 bean 类注册一个 [BeanMapper](#33______BeanMapper)：

```java
public interface UserDao {
  @SqlQuery("select * from users")
  @RegisterBeanMapper(User.class)
  List<User> list();
}
```

使用 `prefix` 属性会导致 bean 映射器只映射那些以前缀开头的列：

```java
public interface UserDao {
  @SqlQuery("select u.id u_id, u.name u_name, r.id r_id, r.name r_name " +
      "from users u left join roles r on u.role_id = r.id")
  @RegisterBeanMapper(value = User.class, prefix = "u")
  @RegisterBeanMapper(value = Role.class, prefix = "r")
  Map<User,Role> getRolesPerUser();
}
```

在这个例子中，`User` 映射器将把 `u_id` 和 `u_name` 列映射到 `User.id` 和 `User.name` 属性中。 同样，将 `r_id` 和 `r_name` 分别映射到 `Role.id` 和 `Role.name`。

> **💡提示:** `@RegisterBeanMapper` 注解可以在同一类型或方法上重复（如上所示）以注册多个 bean 映射器。

<a name="77______@RegisterConstructorMapper"></a>
##### @RegisterConstructorMapper

使用 `@RegisterConstructorMapper` 为通过构造函数使用所有属性实例化的类注册 [ConstructorMapper](32______ConstructorMapper)。

```java
public interface UserDao {
  @SqlQuery("select * from users")
  @RegisterConstructorMapper(User.class)
  List<User> list();
}
```

使用 `prefix` 属性会导致构造函数映射器只映射那些以前缀开头的列：

```java
public interface UserDao {
  @SqlQuery("select u.id u_id, u.name u_name, r.id r_id, r.name r_name " +
      "from users u left join roles r on u.role_id = r.id")
  @RegisterConstructorMapper(value = User.class, prefix = "u")
  @RegisterConstructorMapper(value = Role.class, prefix = "r")
  Map<User,Role> getRolesPerUser();
}
```

在这个例子中，`User` 映射器会将 `u_id` 和 `u_name` 列映射到 `User` 构造函数的 `id` 和 `name` 参数中。 同样，将 `r_id` 和 `r_name` 分别映射到 `Role` 构造函数的 `id` 和 `name` 参数。

> **💡提示:** `@RegisterConstructorMapper` 注解可以在同一类型或方法上重复多次以注册多个构造函数映射器。

<a name="78______@RegisterFieldMapper"></a>

##### @RegisterFieldMapper

使用 `@RegisterFieldMapper` 为给定的类注册一个 [FieldMapper](#34______FieldMapper)。

```java
public interface UserDao {
  @SqlQuery("select * from users")
  @RegisterFieldMapper(User.class)
  List<User> list();
}
```

使用 `prefix` 属性会导致字段映射器仅映射以前缀开头的列：

```java
public interface UserDao {
  @SqlQuery("select u.id u_id, u.name u_name, r.id r_id, r.name r_name " +
      "from users u left join roles r on u.role_id = r.id")
  @RegisterFieldMapper(value = User.class, prefix = "u")
  @RegisterFieldMapper(value = Role.class, prefix = "r")
  Map<User,Role> getRolesPerUser();
}
```

在这个例子中，`User` 映射器将把 `u_id` 和 `u_name` 列映射到 `User.id` 和 `User.name` 字段中。 同样，将 `r_id` 和 `r_name` 分别映射到 `Role.id` 和 `Role.name` 字段。

> **💡提示:** `@RegisterConstructorMapper` 注解可以在同一类型或方法上重复多次以注册多个构造函数映射器。

<a name="79______@SingleValue"></a>
##### @SingleValue

有时，在使用诸如数组之类的高级 SQL 功能时，诸如 `int[]` 或 `List<Integer>` 之类的容器类型可能会含糊不清地表示“单个 SQL int[]”或“一个 ResultSet of int”。

由于数组在规范化模式中不常用，因此 SQL 对象默认假定您将 **ResultSet(表示数据库结果集的当前行)** 收集到容器对象中。 您可以将返回类型注释为 `@SingleValue` 以覆盖它。

例如，假设我们想从一行中选择一个`varchar []`列:

```java
// split comma separated list
public static class ListStringMapper implements ColumnMapper<List<String>> {
    @Override
    public List<String> map(ResultSet r, int columnNumber, StatementContext ctx) throws SQLException {
      String ss = r.getString(columnNumber);
      return Arrays.asList(ss.split(","));
    }
}


public interface UserDao {
  @SqlQuery("select roles from users where id = ?")
  @RegisterColumnMapper(ListStringMapper.class)
  @SingleValue
  List<String> getUserRoles(long userId)
}
```

通常，Jdbi 会将 `List<String>` 解释为表示映射类型为 `String`，并将所有结果行收集到一个列表中。 `@SingleValue` 注释导致 Jdbi 将 `List<String>` 视为映射类型。

> **🏷注意:** `@SingleValue Optional<String>` 很诱人，但通常不需要。 `Optional` 被实现为一个包含零个或一个元素的容器。 添加`@SingleValue` 意味着数据库本身有一个类似`optional<varchar>` 类型的列。

<a name="80______Map<K_V>_Results"></a>

##### Map<K,V> Results

SQL 对象方法可能返回`Map<K,V>` 类型（参见核心API 中的[Map.Entry mapping](#35______Map_Entry_mapping)）。 在这种情况下，每一行都映射到一个 `Map.Entry<K,V>`，每行的条目都被收集到一个 单一的`Map` 实例中。

> **🏷注意:** 必须为键和值类型注册映射器。

只需为键和值类型注册映射器，即可将主/详细连接行收集到map中。

```java
@SqlQuery("select u.id u_id, u.name u_name, p.id p_id, p.phone p_phone "
    + "from user u left join phone p on u.id = p.user_id")
@RegisterConstructorMapper(value = User.class, prefix = "u")
@RegisterConstructorMapper(value = Phone.class, prefix = "p")
Map<User, Phone> getMap();
```

在前面的示例中，`User` 映射器使用“u”列名称前缀，`Phone` 映射器使用“p”。 由于每个映射器仅读取具有预期前缀的列，因此各自的 `id` 列是明确的。

可以通过设置键列名来获得唯一索引（例如通过 ID 列）：

```java
@SqlQuery("select * from user")
@KeyColumn("id")
@RegisterConstructorMapper(User.class)
Map<Integer, User> getAll();
```

设置键和值列名，将两列查询收集到映射结果中:

```java
@SqlQuery("select key, value from config")
@KeyColumn("key")
@ValueColumn("value")
Map<String, String> getAll();
```

以上所有示例都假设是一对一的键/值关系。

如果存在一对多关系怎么办？ Google Guava 提供了一个 `Multimap` 类型，它支持每个键映射多个值。

首先，按照 [Google Guava](#_google_guava) 部分中的说明安装 `GuavaPlugin`。

然后，只需指定一个 `Multimap` 返回类型而不是 `Map`：

```java
@SqlQuery("select u.id u_id, u.name u_name, p.id p_id, p.phone p_phone "
    + "from user u left join phone p on u.id = p.user_id")
@RegisterConstructorMapper(value = User.class, prefix = "u")
@RegisterConstructorMapper(value = Phone.class, prefix = "p")
Multimap<User, Phone> getMultimap();
```

到目前为止，所有示例都是“Map”类型，其中结果集中的每一行都是一个“Map.Entry”。 但是，如果我们要返回的 `Map` 实际上是单行甚至单列怎么办？

Jdbi 的 [MapMapper](apidocs/org/jdbi/v3/core/mapper/MapMapper.html) 将每一行映射到一个 `Map<String, Object>`，其中列名映射到列值。

> **🏷注意:** Jdbi 的默认设置是将列名转换为 Map 键的小写。 可以通过`MapMappers` 配置类更改此行为。

默认情况下，SQL 对象将`Map` 返回类型视为`Map.Entry` 值的集合。 使用 `@SingleValue` 注释覆盖它，以便将返回类型视为单个值而不是集合：

```java
@SqlQuery("select * from users where id = ?")
@RegisterRowMapper(MapMapper.class)
@SingleValue
Map<String, Object> getById(long userId);
```

从 Jdbi 3.6.0 开始，有 [GenericMapMapperFactory](apidocs/org/jdbi/v3/core/mapper/GenericMapperFactory.html)，它提供了相同的功能，但允许==除“Object”以外(对于`Map<String, Object>`还是要用MapMapper)==的值类型，只要合适的 ColumnMapper 已注册并且所有列都属于这种类型：

```java
@SqlQuery("select 1.0 as LOW, 2.0 as MEDIUM, 3.0 as HIGH")
@RegisterRowMapperFactory(GenericMapMapperFactory.class)
@SingleValue
Map<String, BigDecimal> getNumericLevels();
```

> **💡提示:** 你使用 PostgreSQL 的 `hstore` 列吗？ [PostgreSQL](#_postgresql) 插件提供了一个 `hstore` 到 `Map<String, String>` 列映射器。 有关更多信息，请参阅 [hstore](#_hstore)。

<a name="81______@UseRowReducer"></a>

##### @UseRowReducer

`@SqlQuery` 方法使用连接查询可以将主从连接减少到一个或多个主级对象。 请参阅 [ResultBearing.reduceRows()](#51______ResultBearing_reduceRows__) 以了解行减行器的介绍。

考虑一个包含文件夹和文档的文件系统比喻。 在连接中，我们将使用 `f_` 作为文件夹列的前缀，并使用 `d_` 作为文档列的前缀。

```java
@RegisterBeanMapper(value = Folder.class, prefix = "f") <1>
@RegisterBeanMapper(value = Document.class, prefix = "d")
public interface DocumentDao {
    @SqlQuery("select " +
            "f.id f_id, f.name f_name, " +
            "d.id d_id, d.name d_name, d.contents d_contents " +
            "from folders f left join documents d " +
            "on f.id = d.folder_id " +
            "where f.id = :folderId" +
            "order by d.name")
    @UseRowReducer(FolderDocReducer.class) <2>
    Optional<Folder> getFolder(int folderId); <3>

    @SqlQuery("select " +
            "f.id f_id, f.name f_name, " +
            "d.id d_id, d.name d_name, d.contents d_contents " +
            "from folders f left join documents d " +
            "on f.id = d.folder_id " +
            "order by f.name, d.name")
    @UseRowReducer(FolderDocReducer.class) <2>
    List<Folder> listFolders(); <3>

    class FolderDocReducer implements LinkedHashMapRowReducer<Integer, Folder> { <4>
        @Override
        public void accumulate(Map<Integer, Folder> map, RowView rowView) {
            Folder f = map.computeIfAbsent(rowView.getColumn("f_id", Integer.class), <5>
                                           id -> rowView.getRow(Folder.class));

            if (rowView.getColumn("d_id", Integer.class) != null) { <6>
                f.getDocuments().add(rowView.getRow(Document.class));
            }
        }
    }
}
```

| <1>  | 在此示例中，我们使用前缀注册文件夹和文档映射器，以便每个映射器仅查看具有该前缀的列。 这些映射器由 `getRow(Folder.class)` 和 `getRow(Document.class)` 调用中的行缩减器间接使用。 |
| ---- | ------------------------------------------------------------ |
| <2>  | 用`@UseRowReducer`注解该方法，并指定`RowReducer`实现类。     |
| <3>  | 同样的' RowReducer '实现可以用于单主记录和多主记录查询。     |
| <4>  | [LinkedHashMapRowReducer](apidocs/org/jdbi/v3/core/result/LinkedHashMapRowReducer.html) 是一个抽象的`RowReducer` 实现，它使用`LinkedHashMap` 作为结果容器，并返回`values()` 集合作为结果 . |
| <5>  | 通过 ID 从map中获取此行的`Folder`，如果不在map中，则创建它。 |
| <6>  | 在映射文档并将其添加到文件夹之前，确认该行有一个文档（这是左联接）。 |

<a name="82_____5_1_4__@SqlBatch"></a>
#### 5.1.4. @SqlBatch

使用 `@SqlBatch` 注解进行批量更新操作。 `@SqlBatch` 类似于 Core 中的 [PreparedBatch](#56____3_11__Prepared_Batches)。

```java
public interface ContactDao {
  @SqlBatch("insert into contacts (id, name, email) values (?, ?, ?)")
  void bulkInsert(List<Integer> ids,
                  Iterator<String> names,
                  String... emails);
}
```

批处理参数可以是集合、可迭代对象、迭代器、数组（包括可变参数）。 为简洁起见，我们将这些称为“可迭代对象”。

当调用批处理方法时，SQL 对象遍历该方法的可迭代参数，并使用每个参数中的相应元素执行 SQL 语句。

因此这样的语句:

```java
contactDao.bulkInsert(
    ImmutableList.of(1, 2, 3),
    ImmutableList.of("foo", "bar", "baz").iterator(),
    "a@example.com", "b@example.com", "c@fake.com");
```

将执行：

```java
insert into contacts (id, name, email) values (1, 'foo', 'a@example.com');
insert into contacts (id, name, email) values (2, 'bar', 'b@example.com');
insert into contacts (id, name, email) values (3, 'baz', 'c@fake.com');
```

常量值也可以用作 SQL 批处理的参数。 在这种情况下，对于批处理中的每个 SQL 语句，相同的值都绑定到该参数。

```java
public interface UserDao {
  @SqlBatch("insert into users (tenant_id, id, name) " +
      "values (:tenantId, :user.id, :user.name)")
  void bulkInsert(@Bind("tenantId") long tenantId, <1>
                  @BindBean("user") User... users);
}
```

| <1>  | 使用相同的`tenant_id`插入每个用户记录。 |
| ---- | --------------------------------------- |


> **☢警告:** `@SqlBatch` 方法必须至少有一个可迭代参数。

默认情况下，`@SqlBatch` 方法可能会返回一些类型：

- `void`: 不返回任何内容（显然）
- `int[]` 或者 `long[]`: 返回批处理中每次执行的更新计数。根据数据库供应商和JDBC驱动程序，这可能是语句更改的行数，也可能是查询匹配的行数(不管是否更改了任何数据)。
- `boolean[]`: 如果更新计数大于零，则返回true，批处理中每次执行一个值。

<a name="83______@GetGeneratedKeys"></a>
##### @GetGeneratedKeys

与`@SqlUpdate` 类似，`@GetGeneratedKeys` 注解告诉 SQL 对象返回值应该是每个 SQL 语句生成的键，而不是更新计数。 有关更深入的讨论，请参阅 [@GetGeneratedKeys](#__getgeneratedkeys)。

```java
public interface UserDao {
  @SqlBatch("insert into users (id, name) values (nextval('user_seq'), ?)")
  @GetGeneratedKeys("id")
  long[] bulkInsert(List<String> names); <1>
}
```

| <1>  | 为每个插入的名称返回生成的 ID。 |
| ---- | ------------------------------- |

可以通过这种方式生成和返回多个列：

```java
public interface UserDao {
  @SqlBatch("insert into users (id, name, created_on) values (nextval('user_seq'), ?, now())")
  @GetGeneratedKeys({"id", "created_on"})
  @RegisterBeanMapper(IdCreateTime.class)
  List<IdCreateTime> bulkInsert(String... names);
}
```

<a name="84______@SingleValue"></a>
##### @SingleValue

在某些情况下，您可能希望将可迭代参数视为常量 - 在方法参数上使用 `@SingleValue` 注解。 这会导致 SQL 对象将整个可迭代对象绑定为批处理中每个 SQL 语句的参数值（通常作为 SQL 数组参数）。

```java
public interface UserDao {
  @SqlBatch("insert into users (id, name, roles) values (?, ?, ?)")
  void bulkInsert(List<Long> ids,
                  List<String> names,
                  @SingleValue List<String> roles);
}
```

在上面的例子中，每个新行都会在 `roles` 列中获得相同的 `varchar[]` 值。

<a name="85_____5_1_5__@SqlCall"></a>
#### 5.1.5. @SqlCall

使用`@SqlCall` 注解来调用存储过程。

```java
public interface AccountDao {
  @SqlCall("{call suspend_account(:id)}")
  void suspendAccount(@Bind("id") long accountId);
}
```

`@SqlCall` 方法可以返回 `void`，如果存储过程有任何输出参数，也可以返回 `OutParameters`。 每个输出参数都必须使用 `@OutParameter` 注解注册。

```java
public interface OrderDao {
  @SqlCall("{call prepare_order_from_cart(:cartId, :orderId, :orderTotal)}")
  @OutParameter(name = "orderId",    sqlType = java.sql.Types.BIGINT)
  @OutParameter(name = "orderTotal", sqlType = java.sql.Types.DECIMAL)
  OutParameters prepareOrderFromCart(@Bind("cartId") long cartId);
}
```

可以从方法返回的 [OutParameters](apidocs/org/jdbi/v3/core/statement/OutParameters.html) 中提取单个输出参数：

```java
OutParameters outParams = orderDao.prepareOrderFromCart(cartId);
long orderId = outParams.getLong("orderId");
double orderTotal = outParams.getDouble("orderTotal");
```

通过传递 `Consumer<OutParameters>` 或 `Function<OutParameters, T>`，您可以在语句关闭之前处理结果。 这对于处理游标类型的结果很有用。

<a name="86_____5_1_6__@SqlScript"></a>
#### 5.1.6. @SqlScript

使用`@SqlScript` 批量执行一个或多个语句。 您可以定义要使用的模板引擎的属性。

```java
@SqlScript("CREATE TABLE <name> (pk int primary key)")
void createTable(@Define String name);

@SqlScript("INSERT INTO cool_table VALUES (5), (6), (7)")
@SqlScript("DELETE FROM cool_table WHERE pk > 5")
int[] doSomeUpdates(); // returns [ 3, 2 ]

@UseClasspathSqlLocator // load external SQL!
@SqlScript // use the method name
@SqlScript("secondScript") // or specify it yourself
int[] externalScript();
```

<a name="87_____5_1_7__@GetGeneratedKeys"></a>
#### 5.1.7. @GetGeneratedKeys

`@GetGeneratedKeys` z注解可用于 `@SqlUpdate` 或 `@SqlBatch` 方法以返回从 SQL 语句生成的键：

```java
public void sqlObjectBatchKeys() {
    db.useExtension(UserDao.class, dao -> {
        List<User> users = dao.createUsers("Alice", "Bob", "Charlie");
        assertEquals(3, users.size());

        assertEquals(1, users.get(0).id);
        assertEquals("Alice", users.get(0).name);

        assertEquals(2, users.get(1).id);
        assertEquals("Bob", users.get(1).name);

        assertEquals(3, users.get(2).id);
        assertEquals("Charlie", users.get(2).name);
    });
}

public interface UserDao {
    @SqlBatch("INSERT INTO users (name) VALUES(?)")
    @GetGeneratedKeys
    List<User> createUsers(String... names);
}
```

<a name="88_____5_1_8__SqlLocator"></a>
#### 5.1.8. SqlLocator

当 SQL 语句变得越来越复杂时，在 `@Sql__` 方法注释中将语句作为 Java 字符串提供可能会很麻烦。

Jdbi提供注解，允许您配置外部位置以加载SQL语句。

- @UseAnnotationSqlLocator (默认的行为;使用@Sql__(…)注释值)
- @UseClasspathSqlLocator - 根据SQL Object接口类型的包和名称从类路径上的文件加载SQL。

```java
package com.foo;
@UseClasspathSqlLocator
interface BarDao {
    // loads classpath resource com/foo/BarDao/query.sql
    @SqlQuery
    void query();
}
```

**@UseClasspathSqlLocator** 使用 [ClasspathSqlLocator](#63____3_16__ClasspathSqlLocator)实现，如上所述。

如果你喜欢 StringTemplate，[StringTemplate 4](#126____7_14__StringTemplate_4) 模块还提供了一个 SqlLocator，它可以从类路径上的 StringTemplate 4 文件中加载 SQL 模板。

<a name="89_____5_1_9__@CreateSqlObject"></a>
#### 5.1.9. @CreateSqlObject

使用@CreateSqlObject注释在另一个SqlObject中重用一个SqlObject。例如，您可以构建一个事务方法，该方法执行在其他SqlObject中定义的SQL更新，作为事务的一部分。Jdbi不会为对子SqlObject的调用打开新的句柄。

```java
public interface Bar {

    @SqlUpdate("insert into bar (name) values (:name)")
    @GetGeneratedKeys
    int insert(@Bind("name") String name);
}

public interface Foo {

    @CreateSqlObject
    Bar createBar();

    @SqlUpdate("insert into foo (bar_id, name) values (:bar_id, :name)")
    void insert(@Bind("bar_id") int barId, @Bind("name") String name);

    @Transaction
    default void insertBarAndFoo(String barName, String fooName) {
        int barId = createBar().insert(barName);
        insert(barId, fooName);
    }
}
```

<a name="90_____5_1_10__@Timestamped"></a>
#### 5.1.10. @Timestamped

你可以用`@Timestamped`注解任何语句，在`now`绑定下绑定一个`OffsetDateTime`，其值为当前时间：

```java
public interface Bar {
    @SqlUpdate("insert into times(val) values(:now)")
    @Timestamped
    int insert();
}
```

您可以自定义绑定名称：

```
@Timestamped("timestamp")
```

[TimestampedConfig](apidocs/org/jdbi/v3/sqlobject/customizer/TimestampedConfig.html) 允许您控制用于此的时区。

<a name="91____5_2__Consumer_Methods"></a>
### 5.2. Consumer Methods

作为一种特殊情况，除了其他绑定参数之外，您还可以额外在最后提供一个 `Consumer<T>` 参数。 提供的使用者对结果集中的每一行执行一次。 参数 T 的静态类型决定了行类型。

```java
@SqlQuery("select id, name from users")
void forEachUser(Consumer<User> consumer);
```

<a name="92____5_3__Default_Methods"></a>
### 5.3. Default Methods

偶尔会出现不适合SQL方法注解的用例。在这些情况下，您可以使用Java 8的 `default` 方法“drop down(下拉)”到 Core API。

Jdbi 提供了一个带有 `getHandle` 方法的 `SqlObject` 混合接口。 让你的 SQL Object 接口扩展 `SqlObject` mixin，然后在默认方法中提供你自己的实现：

```java
public interface SplineDao extends SqlObject {
  default void reticulateSplines(Spline spline) {
    Handle handle = getHandle();
    // do tricky stuff using the Core API.
  }
}
```

默认方法也可以用来将多个SQL操作组合到一个方法调用中:

```java
public interface ContactPhoneDao {
  @SqlUpdate("insert into contacts (id, name) values (nextval('contact_id'), :name)")
  long insertContact(@BindBean Contact contact);

  @SqlBatch("insert into phones (contact_id, type, phone) values (:contactId, :type, :phone)")
  void insertPhone(long contactId, @BindBean Iterable<Phone> phones);

  default long insertFullContact(Contact contact) {
    long id = insertContact(contact);
    insertPhone(id, contact.getPhones());
    return id;
  }
}
```

<a name="93____5_4__Transaction_Management"></a>
### 5.4. Transaction Management

您可以使用 SqlObject 注解声明事务：

```java
@Test
public void sqlObjectTransaction() {
    assertThat(handle.attach(UserDao.class).findUserById(3).map(u -> u.name)).contains("Charlie");
}

public interface UserDao {
    @SqlQuery("SELECT * FROM users WHERE id=:id")
    @Transaction
    Optional<User> findUserById(int id);
}
```

带有 `@Transaction` 注解的 SQL 方法可以选择指定事务隔离级别：

```java
@SqlUpdate("INSERT INTO USERS (name) VALUES (:name)")
@Transaction(TransactionIsolationLevel.READ_COMMITTED)
void insertUser(String name);
```

如果`@Transaction` 方法调用另一个`@Transaction` 方法，则它们必须指定相同的隔离级别，或者内部方法不得指定任何内容，在这种情况下使用外部方法的隔离级别。

```java
@Transaction(TransactionIsolationLevel.READ_UNCOMMITTED)
default void outerMethodCallsInnerWithSameLevel() {
    // this works: isolation levels agree
    innerMethodSameLevel();
}

@Transaction(TransactionIsolationLevel.READ_UNCOMMITTED)
default void innerMethodSameLevel() {}

@Transaction(TransactionIsolationLevel.READ_COMMITTED)
default void outerMethodWithLevelCallsInnerMethodWithNoLevel() {
    // this also works: inner method doesn't specify a level, so the outer method controls.
    innerMethodWithNoLevel();
}

@Transaction
default void innerMethodWithNoLevel() {}

@Transaction(TransactionIsolationLevel.REPEATABLE_READ)
default void outerMethodWithOneLevelCallsInnerMethodWithAnotherLevel() throws TransactionException {
    // error! inner method specifies a different isolation level.
    innerMethodWithADifferentLevel();
}

@Transaction(TransactionIsolationLevel.SERIALIZABLE)
default void innerMethodWithADifferentLevel() {}
```

<a name="94____5_5__Using_SQL_Objects_使用_SQL_对象_"></a>
### 5.5. Using SQL Objects(使用 SQL 对象)

定义接口后，有几种方法可以获取它的实例：

<a name="95_____5_5_1__Attached_to_Handle_附加到Handle_"></a>
#### 5.5.1. Attached to Handle(附加到Handle)

您可以获得附加到打开Handle的 SQL 对象。

```java
try (Handle handle = jdbi.open()) {
  ContactPhoneDao dao = handle.attach(ContactPhoneDao.class);
  dao.insertFullContact(contact);
}
```

附加的 `SQL对象`与句柄具有相同的生命周期——当句柄关闭时，`SQL对象`将变得不可用。

<a name="96_____5_5_2__Temporary_SQL_Objects_临时SQL对象_"></a>
#### 5.5.2. Temporary SQL Objects(临时SQL对象)

还可以通过传递回调(通常是lambda)，从Jdbi对象获得临时SQL对象。 使用[Jdbi.withExtension](apidocs/org/jdbi/v3/core/Jdbi.html#withExtension-java.lang.Class-org.jdbi.v3.core.extension.ExtensionCallback-)操作返回结果
, 或者[useExtension](apidocs/org/jdbi/v3/core/Jdbi.html#useExtension-java.lang.Class-org.jdbi.v3.core.extension.ExtensionConsumer-)用于没有结果的操作。

```java
jdbi.useExtension(ContactPhoneDao.class, dao -> dao.insertFullContact(alice));
long bobId = jdbi.withExtension(ContactPhoneDao.class, dao -> dao.insertFullContact(bob));
```

临时 `SQL对象` 仅在传递给方法的回调中有效。 当回调返回时，`SQL对象`（和关联的临时句柄）将关闭。

<a name="97_____5_5_3__On_Demand"></a>

#### 5.5.3. On-Demand(按需)

“On-demand(按需)”实例有一个开放式的生命周期，因为它们为每个方法调用获取和释放一个连接。它们是线程安全的，可以跨应用程序重用。当您一次只需要进行“单个调用”时，这很方便。

```java
ContactPhoneDao dao = jdbi.onDemand(ContactPhoneDao.class);
long aliceId = dao.insertFullContact(alice);
long bobId = dao.insertFullContact(bob);
```

按需状态存储在`ThreadLocal`中，以模拟词法作用域。

每次分配和释放连接时都会有性能损失。在上面的例子中，两个 `insertFullContact` 操作从你的数据库连接池中获取单独的 `Connection` 对象。为避免这种情况，请在使用 DAO 期间保持句柄打开：

```java
dao.useTransaction(txn -> {
    User bob = txn.readContact(bobId);
    Order order = txn.getOpenOrder(bobId);
    txn.createInvoice(computeInvoice(bob, metadata));
});
```

> ==wjw_note:==  DAO要扩展`org.jdbi.v3.sqlobject.transaction.Transactional`接口才能使用`useTransaction`
>
> 例如:  `public interface StoreDetailInfoDao extends Transactional<StoreDetailInfoDao> {`

接口 `default` 方法，以及混入，例如 [SqlObject](apidocs/org/jdbi/v3/sqlobject/SqlObject.html) 和 [Transactional](apidocs/org/jdbi/v3/sqlobject/transaction/Transactional.html)，允许您在按需句柄保持打开状态的情况下运行代码。 同一线程上的重入调用将收到相同的“句柄”。 当最外面的按需调用完成时，句柄将关闭。

> **☢警告:** 在最外层的按需调用之外返回类似游标的类型，例如 `Stream<T>` 或 `Iterable<T>` 不起作用。 由于`Handle`关闭，数据库游标被释放，读取将失败。

<a name="98____5_6__Additional_Annotations"></a>
### 5.6. Additional Annotations

Jdbi provides dozens of annotations out of the box:

- [org.jdbi.v3.sqlobject.config](apidocs/org/jdbi/v3/sqlobject/config/package-summary.html) 为可以在`Jdbi` 或`Handle` 级别配置的事物提供注释。 这包括映射器和参数的注册，以及用于配置 SQL 语句呈现和解析。
- [org.jdbi.v3.sqlobject.customizer](apidocs/org/jdbi/v3/sqlobject/customizer/package-summary.html) 为绑定参数、定义属性和控制语句结果集的获取行为提供了注解。
- [org.jdbi.v3.jpa](apidocs/org/jdbi/v3/jpa/package-summary.html) 提供了`@BindJpa`注解，用于根据JPA`@Column`注解将属性绑定到列。
- [org.jdbi.v3.sqlobject.locator](apidocs/org/jdbi/v3/sqlobject/locator/package-summary.html) 提供注解，配置Jdbi从其他源加载SQL语句，例如类路径上的文件。
- [org.jdbi.v3.sqlobject.statement](apidocs/org/jdbi/v3/sqlobject/statement/package-summary.html) 提供了`@MapTo`注解，用于在调用方法时动态指定映射类型。
- [org.jdbi.v3.stringtemplate4](apidocs/org/jdbi/v3/stringtemplate4/package-summary.html) 提供配置 Jdbi 以从类路径上的 StringTemplate 4 `.stg` 文件加载 SQL 和/或使用 ST4 模板引擎解析 SQL 模板的注解。
- [org.jdbi.v3.sqlobject.transaction](apidocs/org/jdbi/v3/sqlobject/transaction/package-summary.html) 为 SQL 对象中的事务管理提供注解。 详见 [Transaction Management](#_transaction_management)。

Jdbi被设计为支持用户定义的注解。请参阅[自定义注解](#145____9_8__User_Defined_Annotations)以获得创建自己的注解的指南。

<a name="99____5_7__Annotations_and_Inheritance"></a>
### 5.7. Annotations and Inheritance(注解 和 继承)

SQL 对象从它们扩展的接口继承方法和注解：

```java
package com.app.dao;

@UseClasspathSqlLocator  <1> <2>
public interface CrudDao<T, ID> {
  @SqlUpdate <3>
  void insert(@BindBean T entity);

  @SqlQuery <3>
  Optional<T> findById(ID id);

  @SqlQuery
  List<T> list();

  @SqlUpdate
  void update(@BindBean T entity);

  @SqlUpdate
  void deleteById(ID id);
}
```

| <1> | 参见 [SqlLocator](#88_____5_1_8__SqlLocator). |
| --- | --------------------------------------------- |
| <2> | 类注解由子类型继承。                            |
| <3> | 方法和参数注解由子类型继承，除非子类型覆盖了方法。 |

```java
package com.app.contact;

@RegisterBeanMapper(Contact.class)
public interface ContactDao extends CrudDao<Contact, Long> {}
```

```java
package com.app.account;

@RegisterConstructorMapper(Account.class)
public interface AccountDao extends CrudDao<Account, UUID> {}
```

在本例中，我们使用了 `@UseClasspathSqlLocator` 注解，因此每个方法都将使用从类路径加载的SQL。因此，`ContactDao` 方法将使用以下 SQL：

- `/com/app/contact/ContactDao/insert.sql`
- `/com/app/contact/ContactDao/findById.sql`
- `/com/app/contact/ContactDao/list.sql`
- `/com/app/contact/ContactDao/update.sql`
- `/com/app/contact/ContactDao/deleteById.sql`

而 `AccountDao` 将使用来自以下内容的 SQL：

- `/com/app/account/AccountDao/insert.sql`
- `/com/app/account/AccountDao/findById.sql`
- `/com/app/account/AccountDao/list.sql`
- `/com/app/account/AccountDao/update.sql`
- `/com/app/account/AccountDao/deleteById.sql`

Suppose `Account` used `name()`-style accessors instead of `getName()`. In that case, we’d want `AccountDao` to use `@BindMethods` instead of `@BindBean`.

Let’s override those methods with the right annotations:

```
package com.app.account;

@RegisterConstructorMapper(Account.class)
public interface AccountDao extends CrudDao<Account, UUID> {
  @Override
  @SqlUpdate 
  void insert(@BindMethods Account entity);

  @Override
  @SqlUpdate 
  void update(@BindMethods Account entity);
}
```

|      | Method annotations are not inherited on override, so we must duplicate those we want to keep. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

<a name="100___6__Testing"></a>
## 6. Testing(测试)

`jdbi3-testing` 工件提供了一个 [JdbiRule](apidocs/org/jdbi/v3/testing/JdbiRule.html) 类，它为编写与托管数据库实例集成的 JUnit 测试提供帮助。 这使得编写单元测试变得快速而简单！ 你必须记住包含数据库依赖本身，例如获得一个纯 H2 Java 数据库：

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <version>1.4.197</version>
    <scope>test</scope>
</dependency>
```

如果你想针对 Postgres 进行测试，你应该包含：

```xml
<dependency>
    <groupId>com.opentable.components</groupId>
    <artifactId>otj-pg-embedded</artifactId>
    <version>0.11.3</version>
    <scope>test</scope>
</dependency>
```

<a name="101___7__Third_Party_Integration"></a>
## 7. Third-Party Integration(第三方集成)

<a name="102____7_1__Google_Guava"></a>
### 7.1. Google Guava

This plugin adds support for the following types:

- `Optional<T>` - registers an argument and mapper. Supports `Optional` for any wrapped type `T` for which a mapper / argument factory is registered.
- Most Guava collection and map types - see [GuavaCollectors](apidocs/org/jdbi/v3/guava/GuavaCollectors.html) for a complete list of supported types.

To use this plugin, add a Maven dependency:

```
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-guava</artifactId>
</dependency>
```

Then install the plugin into your `Jdbi` instance:

```
jdbi.installPlugin(new GuavaPlugin());
```

With the plugin installed, any supported Guava collection type can be returned from a SQL object method:

```
public interface UserDao {
    @SqlQuery("select * from users order by name")
    ImmutableList<User> list();

    @SqlQuery("select * from users where id = :id")
    com.google.common.base.Optional<User> getById(long id);
}
```

<a name="103____7_2__H2_Database"></a>
### 7.2. H2 Database

This plugin configures Jdbi to correctly handle `integer[]` and `uuid[]` data types in an H2 database.

This plugin is included with the core jar (but may be extracted to separate artifact in the future). Use it by installing the plugin into your `Jdbi` instance:

```
jdbi.installPlugin(new H2DatabasePlugin());
```

<a name="104____7_3__JSON"></a>
### 7.3. JSON

The `jdbi3-json` module adds a `@Json` type qualifier that allows to store arbitrary Java objects as JSON data in your database.

The actual JSON (de)serialization code is not included. For that, you must install a backing plugin (see below).

|      | Backing plugins will install the `JsonPlugin` for you. You do **not** need to install it yourself or include the `jdbi3-json` dependency directly. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

The feature has been tested with Postgres `json` columns and `varchar` columns in H2 and Sqlite.

<a name="105_____7_3_1__Jackson_2"></a>
#### 7.3.1. Jackson 2

This plugin provides JSON backing through Jackson 2.

```
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-jackson2</artifactId>
</dependency>
jdbi.installPlugin(new Jackson2Plugin());
// optionally configure your ObjectMapper (recommended)
jdbi.getConfig(Jackson2Config.class).setMapper(myObjectMapper);

// now with simple support for Json Views if you want to filter properties:
jdbi.getConfig(Jackson2Config.class).setView(ApiProperty.class);
```

<a name="106_____7_3_2__Gson_2"></a>
#### 7.3.2. Gson 2

This plugin provides JSON backing through Gson 2.

```
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-gson2</artifactId>
</dependency>
jdbi.installPlugin(new Gson2Plugin());
// optional
jdbi.getConfig(Gson2Config.class).setGson(myGson);
```

<a name="107_____7_3_3__Operation"></a>
#### 7.3.3. Operation

Any bound object qualified as [@Json](apidocs/org/jdbi/v3/json/Json.html) — except `String` — will be converted by the [registered](apidocs/org/jdbi/v3/json/JsonConfig.html) [JsonMapper](apidocs/org/jdbi/v3/json/JsonMapper.html) and *requalified* as `@Json String`. A correspondingly qualified `ArgumentFactory` will then be called to store the JSON data, allowing special JSON handling for your database to be implemented. If none are found, a factory for plain `String` will be used instead, to handle the JSON as plaintext.

Mapping works just the same way, but in reverse: an output type qualified as `@Json T` will be fetched from a `@Json String` or `String` ColumnMapper, and then passed through the `JsonMapper`.

|      | Our PostgresPlugin provides qualified factories that will bind/map the `@Json String` to/from `json` or `jsonb`-typed columns. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

<a name="108_____7_3_4__Usage"></a>
#### 7.3.4. Usage

```
handle.execute("create table myjsons (id serial not null, value json not null)");
```

SqlObject:

```
// any json-serializable type
class MyJson {}

// use @Json qualifier:
interface MyJsonDao {
    @SqlUpdate("insert into myjsons (json) values(:value)")
    // on parameters
    int insert(@Json MyJson value);

    @SqlQuery("select value from myjsons")
    // on result types
    @Json
    List<MyJson> select();
}

// also works on bean or property-mapped objects:
class MyBean {
    private final MyJson property;
    @Json
    public MyJson getProperty() { return ...; }
}
```

With the Fluent API, you provide a `QualifiedType<T>` any place you’d normally provide a `Class<T>` or `GenericType<T>`:

```
QualifiedType<MyJson> qualifiedType = QualifiedType.of(MyJson.class).with(Json.class);

h.createUpdate("insert into myjsons(json) values(:json)")
    .bindByType("json", new MyJson(), qualifiedType)
    .execute();

MyJson result = h.createQuery("select json from myjsons")
    .mapTo(qualifiedType)
    .one();
```

<a name="109____7_4__Immutables"></a>
### 7.4. Immutables

[Immutables](https://immutables.github.io/) is an annotation processor that generates value types based on simple interface descriptions. The value types naturally map very well to `Jdbi` properties binding and row mapping.

|      | Immutables support is still experimental and does not yet support custom naming schemes. We do support the configurable `get`, `is`, and `set` prefixes. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

Just tell us about your types by installing the plugin and configuring your `Immutables` type:

```
jdbi.getConfig(JdbiImmutables.class).registerImmutable(MyValueType.class)
```

The configuration will both register appropriate `RowMapper`s as well as configure the new `bindPojo` (or `@BindPojo`) binders:

```
@Value.Immutable
public interface Train {
    String name();
    int carriages();
    boolean observationCar();
}

@Test
public void simpleTest() {
    jdbi.getConfig(JdbiImmutables.class).registerImmutable(Train.class);
    try (Handle handle = jdbi.open()) {
        handle.execute("create table train (name varchar, carriages int, observation_car boolean)");

        assertThat(
            handle.createUpdate("insert into train(name, carriages, observation_car) values (:name, :carriages, :observationCar)")
                .bindPojo(ImmutableTrain.builder().name("Zephyr").carriages(8).observationCar(true).build())
                .execute())
            .isEqualTo(1);

        assertThat(
            handle.createQuery("select * from train")
                .mapTo(Train.class)
                .one())
            .extracting("name", "carriages", "observationCar")
            .containsExactly("Zephyr", 8, true);
    }
}
```

<a name="110____7_5__Freebuilder"></a>
### 7.5. Freebuilder

[Freebuilder](https://https://freebuilder.inferred.org/) is an annotation processor that generates value types based on simple interface or abstract class descriptions. Jdbi supports Freebuilder in much the same way that it supports Immutables.

|      | Freebuilder support is still experimental and may not support all Freebuilder implemented feaatures. We do support both JavaBean style getters and setters as well as unprefixed getters and setters. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

Just tell us about your Freebuilder types by installing the plugin and configuring your `Freebuilder` type:

```
jdbi.getConfig(JdbiFreebuilder.class).registerFreebuilder(MyFreeBuilderType.class)
```

The configuration will both register appropriate `RowMapper`s as well as configure the new `bindPojo` (or `@BindPojo`) binders:

```
@FreeBuilder
public interface Train {
    String name();
    int carriages();
    boolean observationCar();

    class Builder extends FreeBuildersTest_Train_Builder {}
}

@Test
public void simpleTest() {
    jdbi.getConfig(JdbiFreeBuilders.class).registerFreeBuilder(Train.class);
    try (Handle handle = jdbi.open()) {
        handle.execute("create table train (name varchar, carriages int, observation_car boolean)");

        Train train = new Train.Builder()
            .name("Zephyr")
            .carriages(8)
            .observationCar(true)
            .build();

        assertThat(
            handle.createUpdate("insert into train(name, carriages, observation_car) values (:name, :carriages, :observationCar)")
                .bindPojo(train)
                .execute())
            .isEqualTo(1);

        assertThat(
            handle.createQuery("select * from train")
                .mapTo(Train.class)
                .one())
            .extracting("name", "carriages", "observationCar")
            .containsExactly("Zephyr", 8, true);
    }
}
```

<a name="111____7_6__JodaTime"></a>
### 7.6. JodaTime

This plugin adds support for using joda-time’s `DateTime` type.

To use this plugin, add a Maven dependency:

```
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-jodatime2</artifactId>
</dependency>
```

Then install the plugin into your `Jdbi` instance:

```
jdbi.installPlugin(new JodaTimePlugin());
```

<a name="112____7_7__JPA"></a>
### 7.7. JPA

Using the JPA plugin is a great way to trick your boss into letting you try Jdbi. "No problem boss, it already supports JPA annotations, easy peasy!"

This plugin adds mapping support for a small subset of JPA entity annotations:

- Entity
- MappedSuperclass
- Column

To use this plugin, add a Maven dependency:

```
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-jpa</artifactId>
</dependency>
```

Then install the plugin into your `Jdbi` instance:

```
jdbi.installPlugin(new JpaPlugin());
```

Honestly though.. just tear off the bandage and switch to Jdbi proper.

<a name="113____7_8__Kotlin"></a>
### 7.8. Kotlin

[Kotlin](https://kotlinlang.org/) support is provided by **jdbi3-kotlin** and **jdbi3-kotlin-sqlobject** modules.

Kotlin API documentation:

- [jdbi3-kotlin](apidocs-kotlin/jdbi3-kotlin/index.html)
- [jdbi3-kotlin-sqlobject](apidocs-kotlin/jdbi3-kotlin-sqlobject/index.html)

<a name="114_____7_8_1__ResultSet_mapping"></a>
#### 7.8.1. ResultSet mapping

The **jdbi3-kotlin** plugin adds mapping to Kotlin data classes. It supports data classes where all fields are present in the constructor as well as classes with writable properties. Any fields not present in the constructor will be set after the constructor call. The mapper supports nullable types. It also uses default parameter values in the constructor if the parameter type is not nullable and the value absent in the result set.

To use this plugin, add a Maven dependency:

```
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-kotlin</artifactId>
</dependency>
```

Ensure the Kotlin compiler’s [JVM target version](https://kotlinlang.org/docs/reference/using-maven.html#attributes-specific-for-jvm) is set to at least 1.8:

```
<kotlin.compiler.jvmTarget>1.8</kotlin.compiler.jvmTarget>
```

Then install the plugin into your `Jdbi` instance:

```
jdbi.installPlugin(KotlinPlugin());
```

The Kotlin mapper also supports `@ColumnName` annotation that allows to specify name for a property or parameter explicitly, as well as the `@Nested` annotation that allows mapping nested Kotlin objects.

|      | Instead of using `@BindBean`, `bindBean()`, and `@RegisterBeanMapper` use `@BindKotlin`, `bindKotlin()`, and `KotlinMapper` for qualifiers on constrictor parameters, getter, setters, and setter parameters of Kotlin class. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

|      | The `@ColumnName` annotation only applies while mapping SQL data into Java objects. When binding object properties (e.g. with `bindBean()`), bind the property name (`:id`) rather than the column name (`:user_id`). |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

If you load all Jdbi plugins via `Jdbi.installPlugins()` this plugin will be discovered and registered automatically. Otherwise, you can attach it using `Jdbi.installPlugin(KotlinPlugin())`.

An example from the test class:

```
data class IdAndName(val id: Int, val name: String)
data class Thing(@Nested val idAndName: IdAndName,
                 val nullable: String?,
                 val nullableDefaultedNull: String? = null,
                 val nullableDefaultedNotNull: String? = "not null",
                 val defaulted: String = "default value")
@Test fun testFindById() {
    val qry = db.sharedHandle.createQuery("select id, name from something where id = :id")
    val things: List<Thing> = qry.bind("id", brian.idAndName.id).mapTo<Thing>().list()
    assertEquals(1, things.size)
    assertEquals(brian, things[0])
}
```

There are two extensions to help:

- `<reified T : Any>ResultBearing.mapTo()`
- `<T : Any>ResultIterable<T>.useSequence(block: (Sequence<T>) → Unit)`

Allowing code like:

```
val qry = handle.createQuery("select id, name from something where id = :id")
val things = qry.bind("id", brian.id).mapTo<Thing>.list()
```

and for using a Sequence that is auto closed:

```
qryAll.mapTo<Thing>.useSequence {
    it.forEach(::println)
}
```

<a name="115_____7_8_2__SqlObject"></a>
#### 7.8.2. SqlObject

The **jdbi3-kotlin-sqlobject** plugin adds automatic parameter binding by name for Kotlin methods in SqlObjects as well as support for Kotlin default methods.

```
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-kotlin-sqlobject</artifactId>
</dependency>
```

Then install the plugin into your `Jdbi` instance:

```
jdbi.installPlugin(KotlinSqlObjectPlugin());
```

Parameter binding supports individual primitive types as well as Kotlin or JavaBean style objects as a parameter (referenced in binding as `:paramName.propertyName`). No annotations are needed.

If you load all Jdbi plugins via `Jdbi.installPlugins()` this plugin will be discovered and registered automatically. Otherwise, you can attach the plugin via: `Jdbi.installPlugin(KotlinSqlObjectPlugin())`.

An example from the test class:

```
interface ThingDao {
    @SqlUpdate("insert into something (id, name) values (:something.idAndName.id, :something.idAndName.name)")
    fun insert(something: Thing)

    @SqlQuery("select id, name from something")
    fun list(): List<Thing>
}
@Before fun setUp() {
    val dao = db.jdbi.onDemand<ThingDao>()

    val brian = Thing(IdAndName(1, "Brian"), null)
    val keith = Thing(IdAndName(2, "Keith"), null)

    dao.insert(brian)
    dao.insert(keith)
}
@Test fun testDao() {
    val dao = db.jdbi.onDemand<ThingDao>()

    val rs = dao.list()

    assertEquals(2, rs.size.toLong())
    assertEquals(brian, rs[0])
    assertEquals(keith, rs[1])

}
```

<a name="116____7_9__Lombok"></a>
### 7.9. Lombok

Lombok is a great tool for cutting the boilerplate out of POJO classes.

```
@Data
public void DataClass {
  private Long id;
  private String name;
  // autogenerates default constructor, getters, setters, equals, hashCode, and toString
}

@Value
public void ValueClass {
  private long id;
  private String name;
  // autogenerates all-args constructor, getters, equals, hashCode, and toString
}
```

Lombok and Jdbi mostly play nice out of the box:

- Use `BeanMapper` or `@RegisterBeanMapper` to map `@Data` classes.
- Use `ConstructorMapper` or `@RegisterConstructorMapper` to map `@Value` classes.
- Use `bindBean()` or `@BindBean` to bind `@Data` or `@Value` classes.

We say "mostly" because there’s a wrinkle once you start annotating fields with Jdbi annotations like `@Nested`, `@ColumnMapper`, or type qualifying annotations such as `@HStore`.

- BeanMapper looks for these annotations on getters, setters, or setter parameters.
- ConstructorMapper looks for them on constructor parameters.
- Lombok doesn’t move them there by default.

As of Lombok version 1.18.4, Lombok can be configured to copy any annotations you specify to generated getter, setter, setter parameters, and constructor parameters.

Create a file `lombok.config` in your project src tree (or edit the existing one), and add a line for each annotation type which should be copied, as in the following example:

```
lombok.copyableAnnotations += org.jdbi.v3.core.mapper.Nested
lombok.copyableAnnotations += org.jdbi.v3.core.mapper.reflect.ColumnName
lombok.copyableAnnotations += org.jdbi.v3.postgres.HStore
```

<a name="117____7_10__Oracle_12"></a>
### 7.10. Oracle 12

This module adds support for Oracle `RETURNING` DML expressions.

To use this feature, add a Maven dependency:

```
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-oracle12</artifactId>
</dependency>
```

Then, use the `OracleReturning` class with an `Update` or `PreparedBatch` to get the returned DML.

<a name="118____7_11__PostgreSQL"></a>
### 7.11. PostgreSQL

The **jdbi3-postgres** plugin provides enhanced integration with the [PostgreSQL JDBC Driver](https://jdbc.postgresql.org/).

To use this feature, add a Maven dependency:

```
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-postgres</artifactId>
</dependency>
```

Then install the plugin into your `Jdbi` instance.

```
Jdbi jdbi = Jdbi.create("jdbc:postgresql://host:port/database")
                .installPlugin(new PostgresPlugin());
```

The plugin configures mappings for the Java 8 **java.time** types like **Instant** or **Duration**, **InetAddress**, **UUID**, typed enums, and **hstore**.

It also configures SQL array type support for `int`, `long`, `float`, `double`, `String`, and `UUID`.

See the [javadoc](apidocs/org/jdbi/v3/postgres/package-summary.html) for an exhaustive list.

|      | Some Postgres operators, for example the `?` query operator, collide with `jdbi` or `JDBC` special characters. In such cases, you may need to escape operators to e.g. `??` or `\:`. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

<a name="119_____7_11_1__hstore"></a>
#### 7.11.1. hstore

The Postgres plugin provides an `hstore` to `Map<String, String>` column mapper and vice versa argument factory:

```
Map<String, String> accountAttributes = handle
    .select("select attributes from account where id = ?", userId)
    .mapTo(new GenericType<Map<String, String>>() {})
    .one();
```

With `@HStore` qualified type:

```
QualifiedType<> HSTORE_MAP = QualifiedType.of(new GenericType<Map<String, String>>() {})
    .with(HStore.class);

Map<String, String> caps = handle.createUpdate("update account set attributes = :hstore")
    .bindByType("hstore", mapOfStrings, HSTORE_MAP)
    .execute();
```

By default, SQL Object treats `Map` return types as a collection of `Map.Entry` values. Use the `@SingleValue` annotation to override this, so that the return type is treated as a single value instead of a collection:

```
public interface AccountDao {
  @SqlQuery("select attributes from account where id = ?")
  @SingleValue
  Map<String, String> getAccountAttributes(long accountId);
}
```

<a name="120_____7_11_2__@GetGeneratedKeys"></a>
#### 7.11.2. @GetGeneratedKeys

In Postgres, `@GetGeneratedKeys` can return the entire modified row if you request generated keys without naming any columns.

```
public interface UserDao {
  @SqlUpdate("insert into users (id, name, created_on) values (nextval('user_seq'), ?, now())")
  @GetGeneratedKeys
  @RegisterBeanMapper(User.class)
  User insert(String name);
}
```

If a database operation modifies multiple rows (e.g. an update that will modify several rows), your method can return all the modified rows in a collection:

```
public interface UserDao {
  @SqlUpdate("update users set active = false where id = any(?)")
  @GetGeneratedKeys
  @RegisterBeanMapper(User.class)
  List<User> deactivateUsers(long... userIds);
}
```

<a name="121_____7_11_3__Large_Objects"></a>
#### 7.11.3. Large Objects

Postgres supports storing large character or binary data in separate storage from table data. Jdbi allows you to stream this data in and out of the database as part of an enclosing transaction. Storing, reading, and a delete hook are provided. The test case serves as a simple example:

```
public void blobCrud(InputStream myStream) throws IOException {
    h.useTransaction(th -> {
        Lobject lob = th.attach(Lobject.class);
        lob.insert(1, myStream);
        readItBack = lob.readBlob(1);
        lob.deleteLob(1);
        assert lob.readBlob(1) == null;
    });
}

public interface Lobject {
    // CREATE TABLE lob (id int, lob oid
    @SqlUpdate("insert into lob (id, lob) values (:id, :blob)")
    void insert(int id, InputStream blob);

    @SqlQuery("select lob from lob where id = :id")
    InputStream readBlob(int id);

    @SqlUpdate("delete from lob where id = :id returning lo_unlink(lob)")
    void deleteLob(int id);
}
```

Please refer to [Pg-JDBC docs](https://jdbc.postgresql.org/documentation/head/binary-data.html) for upstream driver documentation.

<a name="122____7_12__Spring"></a>
### 7.12. Spring

This module provides `JdbiFactoryBean`, a factory bean which sets up a `Jdbi` singleton.

To use this module, add a Maven dependency:

```
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-spring4</artifactId>
</dependency>
```

Then configure the Jdbi factory bean in your Spring container, e.g.:

```
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xsi:schemaLocation="
       http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.0.xsd
       http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-2.0.xsd
       http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-2.0.xsd">

  
  <bean id="db" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
    <property name="url" value="jdbc:h2:mem:testing"/>
  </bean>

  
  <bean id="transactionManager"
    class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="db"/>
  </bean>
  <tx:annotation-driven transaction-manager="transactionManager"/>

  
  <bean id="jdbi"
    class="org.jdbi.v3.spring4.JdbiFactoryBean">
    <property name="dataSource" ref="db"/>
  </bean>

  
  <bean id="service"
    class="com.example.service.MyService">
    <constructor-arg ref="jdbi"/>
  </bean>
</beans>
```

|      | The SQL data source that Jdbi will connect to. In this example we use an H2 database, but it can be any JDBC-compatible database. |
| ---- | ------------------------------------------------------------ |
|      | Enable configuration of transactions via annotations.        |
|      | Configure `JdbiFactoryBean` using the data source configured earlier. |
|      | Inject `Jdbi` into a service class. Alternatively, use standard JSR-330 `@Inject` annotations on the target class instead of configuring it in your `beans.xml`. |

|      | This module has been tested to be compatible with Spring 5.1.8 (used by Spring Boot 2.1.4) as well. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

<a name="123_____7_12_1__Installing_plugins"></a>
#### 7.12.1. Installing plugins

Plugins may be automatically installed by scanning the classpath for [ServiceLoader](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html) manifests.

```
<bean id="jdbi" class="org.jdbi.v3.spring4.JdbiFactoryBean">
  ...
  <property name="autoInstallPlugins" value="true"/>
</bean>
```

Plugins may also be installed explicitly:

```
<bean id="jdbi" class="org.jdbi.v3.spring4.JdbiFactoryBean">
  ...
  <property name="plugins">
    <list>
      <bean class="org.jdbi.v3.sqlobject.SqlObjectPlugin"/>
      <bean class="org.jdbi.v3.guava.GuavaPlugin"/>
    </list>
  </property>
</bean>
```

Not all plugins are automatically installable. In these situations, you can auto-install some plugins and manually install the rest:

```
<bean id="jdbi" class="org.jdbi.v3.spring4.JdbiFactoryBean">
  ...
  <property name="autoInstallPlugins" value="true"/>
  <property name="plugins">
    <list>
      <bean class="org.jdbi.v3.core.h2.H2DatabasePlugin"/>
    </list>
  </property>
</bean>
```

<a name="124_____7_12_2__Global_Attributes"></a>
#### 7.12.2. Global Attributes

Global defined attributes may be configured on the factory bean:

```
<bean id="jdbi" class="org.jdbi.v3.spring4.JdbiFactoryBean">
  <property name="dataSource" ref="db"/>
  <property name="globalDefines">
    <map>
      <entry key="foo" value="bar"/>
    </map>
  </property>
</bean>
```

<a name="125____7_13__SQLite"></a>
### 7.13. SQLite

The **jdbi3-sqlite** plugin provides support for using the [SQLite JDBC Driver](https://bitbucket.org/xerial/sqlite-jdbc/) with Jdbi.

The plugin configures mapping for the Java **URL** type which is not supported by driver.

To use this plugin, first add a Maven dependency:

```
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-sqlite</artifactId>
</dependency>
```

Then install the plugin into your `Jdbi` instance.

```
Jdbi jdbi = Jdbi.create("jdbc:sqlite:database")
                .installPlugin(new SQLitePlugin());
```

<a name="126____7_14__StringTemplate_4"></a>
### 7.14. StringTemplate 4

This module allows you to plug in the StringTemplate 4 templating engine, in place of the standard Jdbi templating engine.

To use module plugin, add a Maven dependency:

```
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-stringtemplate4</artifactId>
</dependency>
```

To use StringTemplate format in SQL statements, set the template engine to `StringTemplateEngine`.

Defined attributes are provided to the StringTemplate engine to render the SQL:

```
String sortColumn = "name";
String sql = "select id, name " +
             "from account " +
             "order by <if(sort)> <sortBy>, <endif> id";

List<Account> accounts = handle.createQuery(sql)
      .setTemplateEngine(new StringTemplateEngine())
      .define("sort", true)
      .define("sortBy", sortColumn)
      .mapTo(Account.class)
      .list();
```

Alternatively, SQL templates can be loaded from StringTemplate group files on the classpath:

com/foo/AccountDao.sql.stg

```
group AccountDao;

selectAll(sort,sortBy) ::= <<
  select id, name
  from account
  order by <if(sort)> <sortBy>, <endif> id
>>
ST template = StringTemplateSqlLocator.findStringTemplate(
                  "com/foo/AccountDao.sql.stg", "selectAll");

String sql = template.add("sort", true)
                     .add("sortBy", sortColumn)
                     .render();
```

In SQL Objects, the `@UseStringTemplateEngine` annotation sets the statement locator, similar to first example above.

```
package com.foo;

public interface AccountDao {
  @SqlQuery("select id, name " +
            "from account " +
            "order by <if(sort)> <sortBy>, <endif> id")
  @UseStringTemplateEngine
  List<Account> selectAll(@Define boolean sort,
                          @Define String sortBy);
}
```

Alternatively, the `@UseStringTemplateSqlLocator` annotation sets the statement locator, and loads SQL from a StringTemplate group file on the classpath:

```
package com.foo;

public interface AccountDao {
  @SqlQuery
  @UseStringTemplateSqlLocator
  List<Account> selectAll(@Define boolean sort,
                          @Define String sortBy);
}
```

In this example, since the fully qualified class name is `com.foo.AccountDao`, SQL will be loaded from the file `com/foo/AccountDao.sql.stg` on the classpath.

By default, the template in the group with the same name as the method will be used. This can be overridden on the `@Sql___` annotation:

```
package com.foo;

public interface AccountDao {
  @SqlQuery("listSorted")
  @UseStringTemplateSqlLocator
  List<Account> selectAll(@Define boolean sort,
                          @Define String sortBy);
}
```

In this example, the SQL template will still be loaded from the file `com/foo/AccountDao.sql.stg` on the classpath, however the `listSorted` template will be used, regardless of the method name.

<a name="127____7_15__Vavr"></a>
### 7.15. Vavr

The Vavr Plugin offers deep integration of **Jdbi** with the Vavr functional library:

- Supports argument resolution of sever Vavr Value types such as `Option<T>`, `Either<L, T>`, `Lazy<T>`, `Try<T>` and `Validation<T>`. Given that for the wrapped type `T` a Mapper is registered.
- Return Vavr collection types from queries. Supported are `Seq<T>`, `Set<T>`, `Map<K, T>` and `Multimap<K, T>` as well as all subtypes thereof. It is possible to collect into a `Traversable<T>`, in this case a `List<T>` will be returned. For all interface types a sensible default implementation will be used (e.q. `List<T>` for `Seq<T>`). Furthermore `Multimap<K, T>`s are backed by a `Seq<T>` as default value container.
- Columns can be mapped into Vavr’s `Option<T>` type.
- Tuple projections for **Jdbi**! Yey! Vavr offers Tuples up to a maximum arity of 8. you can map your query results e.g. to `Tuple3<Integer, String, Long>`. If you select more columns than the arity of the projection the columns up to that index will be used.

To use the plugin, add a Maven dependency:

```
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-vavr</artifactId>
</dependency>
```

Currently Vavr >= 0.9.0 is supported and tested. The plugin pulls a supported version of Vavr and is ready to be used. As with other plugins: install via `Jdbi` instance or use auto install.

```
jdbi.installPlugin(new VavrPlugin());
```

Here are some usage examples of the features listed above:

```
String query = "select * from users where :name is null or name = :name";
Option<String> param = Option.of("eric");

// will fetch first user with given name or first user with any name (Option.none)
return handle.createQuery(query)
        .bind("name", param)
        .mapToBean(User.class)
        .findFirst();
```

where `param` may be one of `Option<T>`, `Either<L, T>`, `Lazy<T>`, `Try<T>` or `Validation<T>`. Note that in the case of these types, the nested value must be 'present' otherwise a `null` value is used (e.g. for `Either.Left` or `Validation.Invalid`).

```
handle.createQuery("select name from users")
        .collectInto(new GenericType<Seq<String>>() {});
```

This works for all the collection types supported. For the nested value row and column mappers already installed in **Jdbi** will be used. Therefore the following would work and can make sense if the column is nullable:

```
handle.createQuery("select middle_name from users") // nulls incoming!
        .collectInto(new GenericType<Seq<Option<String>>>() {});
```

The plugin will obey configured key and value columns for `Map<K, T>` and `Multimap<K, T>` return types. In the next example we will key users by their name, which is not necessarily unique.

```
Multimap<String, User> usersByName = handle.createQuery("select * from users")
        .setMapKeyColumn("name")
        .collectInto(new GenericType<Multimap<String, User>>() {});
```

Last but not least we can now project simple queries to Vavr tuples like that:

```
// given a 'tuples' table with t1 int, t2 varchar, t3 varchar, ...
List<Tuple3<Integer, String, String>> tupleProjection = handle
        .createQuery("select t1, t2, t3 from tuples")
        .mapTo(new GenericType<Tuple3<Integer, String, String>>() {})
        .list();
```

You can also project complex types into a tuple as long as a row mapper is registered.

```
// given that there are row mappers registered for both complex types
Tuple2<City, Address> tupleProjection = handle
        .createQuery("select cityname, zipcode, street, housenumber from " +
            "addresses where user_id = 1")
        .mapTo(new GenericType<Tuple2<City, Address>>() {})
        .one();
```

If you want to mix complex types and simple ones we also got you covered. Using the `TupleMappers` class you can configure your projections.(In fact, you have to - read below!)

```
handle.configure(TupleMappers.class, c ->
        c.setColumn(2, "street").setColumn(3, "housenumber"));

Tuple3<City, String, Integer> result = handle
        .createQuery("select cityname, zipcode, street, housenumber from " +
             "addresses where user_id = 1")
        .mapTo(new GenericType<Tuple3<City, String, Integer>>() {})
        .one();
```

Bear in mind:

- The configuration of the columns is 1-based, since they reflect the tuples' values (which you would query by e.g. `._1`).
- Tuples are always mapped fully column-wise or fully via row mappers. If you want to mix row-mapped types and single-column mappings the `TupleMappers` must be configured properly i.e. all non row-mapped tuple indices must be provided with a column configuration!

<a name="128___8__Cookbook"></a>
## 8. Cookbook

This section includes examples of various things you might like to do with `Jdbi`.

<a name="129____8_1__Simple_Dependency_Injection"></a>
### 8.1. Simple Dependency Injection

`Jdbi` tries to be independent of using a dependency injection framework, but it’s straightforward to integrate yours. Just do field injection on a simple custom config type:

```
class InjectedDependencies implements JdbiConfig<InjectedDependencies> {
    @Inject
    SomeDependency dep;

    public InjectedDependencies() {}

    @Override
    public InjectedDependencies createCopy() {
        return this; // effectively immutable
    }
}

Jdbi jdbi = Jdbi.create(myDataSource);
myIoC.inject(jdbi.getConfig(InjectedDependencies.class));

// Then, in any component that needs to access it:
getHandle().getConfig(InjectedDependencies.class).dep
```

<a name="130____8_2__LIKE_clauses_with_Parameters"></a>
### 8.2. LIKE clauses with Parameters

Since JDBC (and therefore `Jdbi`) does not allow binding parameters into the middle of string literals, you cannot interpolate bindings into `LIKE` clauses (`LIKE '%:param%'`).

Incorrect usage:

```
handle.createQuery("select name from things where name like '%:search%'")
    .bind("search", "foo")
    .mapTo(String.class)
    .list()
```

This query would try to select `where name like '%:search%'` *literally*, without binding any arguments. This is because JDBC drivers will not bind arguments *inside string literals*.

It never gets that far, though — this query will throw an exception, because we don’t allow unused argument bindings by default.

The solution is to use SQL string concatenation:

```
handle.createQuery("select name from things where name like '%' || :search || '%'")
    .bind("search", "foo")
    .mapTo(String.class)
    .list()
```

Now, `search` can be properly bound as a parameter to the statement, and it all works as desired.

|      | Check the string concatenation syntax of your database before doing this. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

<a name="131___9__Advanced_Topics"></a>
## 9. Advanced Topics

<a name="132____9_1__High_Availability"></a>
### 9.1. High Availability

Jdbi can be combined with connection pools and high-availability features in your database driver. We’ve used [HikariCP](https://brettwooldridge.github.io/HikariCP/) in combination with the [PgJDBC connection load balancing](https://jdbc.postgresql.org/documentation/head/connect.html) features with good success.

```
PGSimpleDataSource ds = new PGSimpleDataSource();
ds.setServerName("host1,host2,host3");
ds.setLoadBalanceHosts(true);
HikariConfig hc = new HikariConfig();
hc.setDataSource(ds);
hc.setMaximumPoolSize(6);
Jdbi jdbi = Jdbi.create(new HikariDataSource(hc)).installPlugin(new PostgresPlugin());
```

Each Jdbi may be backed by a pool of any number of hosts, but the connections should all be alike. Exactly which parameters must stay the same and which may vary depends on your database and driver.

If you want to have two separate pools, for example a read-only set that connects to read replicas and a smaller pool of writers that go only to a single host, you currently should have separate `Jdbi` instances each pointed at a separate `DataSource`.

<a name="133____9_2__使用参数名称编译"></a>

### 9.2. 使用参数名称编译

默认情况下，Java编译器不会将构造函数和方法的参数名写入类文件。在运行时，反射式请求参数名称会给出“arg0”、“arg1”等值。

开箱即用，Jdbi 使用注解来了解每个参数的名称，例如：

- `ConstructorMapper` 使用 `@ConstructorProperties` 注解.
- SQL 对象方法参数使用 `@Bind` 注解。

```java
@SqlUpdate("insert into users (id, name) values (:id, :name)")
void insert(@Bind("id") long id, @Bind("name") String name); <1>
```

|     | 如此冗长，非常样板。哇。 |
| --- | ---------------------- |

如果你使用 `-parameters` 编译器标志编译你的代码，那么就不需要这些注解 — Jdbi 自动使用方法参数名称：

```java
@SqlUpdate("insert into users (id, name) values (:id, :name)")
void insert(long id, String name);
```

<a name="134_____9_2_1__Maven_setup"></a>
#### 9.2.1. Maven 设置

在你的 POM 中配置 `maven-compiler-plugin`：

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <configuration>
    <compilerArgs>
      <arg>-parameters</arg>
    </compilerArgs>
  </configuration>
</plugin>
```

<a name="135_____9_2_2__IntelliJ_IDEA_setup"></a>
#### 9.2.2. IntelliJ IDEA 设置

- File → Settings
- Build, Execution, Deployment → Compiler → Java Compiler
- Additional command-line parameters: `-parameters`
- Click Apply, then OK.
- Build → Rebuild Project

<a name="136_____9_2_3__Eclipse_setup"></a>
#### 9.2.3. Eclipse 设置

- Window → Preferences
- Java → Compiler
- Under "Classfile Generation," check the option "Store information about method parameters (usable via reflection)."

<a name="137____9_3__Working_with_Generic_Types"></a>
### 9.3. Working with Generic Types(使用泛型类型)

Jdbi provides utility classes to make it easier to work with Java generic types.

<a name="138_____9_3_1__GenericType"></a>
#### 9.3.1. GenericType(泛型类型)

[GenericType](apidocs/org/jdbi/v3/core/generic/GenericType.html) 表示一个泛型类型签名，可以以类型安全的方式传递。

通过实例化匿名内部类来创建泛型类型引用:

```java
new GenericType<Optional<String>>() {}
```

此类型引用可以传递给任何接受 `GenericType<T>` 的 Jdbi 方法，例如：

```java
List<Optional<String>> middleNames = handle
    .select("select middle_name from contacts")
    .mapTo(new GenericType<Optional<String>>() {})
    .list();
```

`GenericType.getType()` 返回原始 [java.lang.reflect.Type](https://docs.oracle.com/javase/8/docs/api/java/lang/reflect/Type.html) 对象 用于表示 Java 中的泛型。

<a name="139_____9_3_2__GenericTypes"></a>
#### 9.3.2. GenericTypes

[GenericTypes](apidocs/org/jdbi/v3/core/generic/GenericTypes.html) 提供了处理 Java 泛型类型签名的方法。

`GenericTypes`中的所有方法都按照`java.lang.reflect.Type`操作。

`getErasedType(Type)`方法接受一个`Type`并返回该类型的原始`Class`，抹去所有泛型参数:

```java
Type listOfInteger = new GenericType<List<Integer>>() {}.getType();
GenericTypes.getErasedType(listOfInteger); // => List.class

GenericTypes.getErasedType(String.class); // => String.class
```

`resolveType(Type, Type)`方法接受泛型类型和用于解析它的上下文类型。

例如，给定来自 `Optional<T>` 的类型变量 `T`：

```java
Type t = Optional.class.getTypeParameters()[0];
```

并给定上下文类型`Optional<String>`：

```java
Type optionalOfString = new GenericType<Optional<String>>() {}.getType();
```

`resolveType()` 方法回答了这个问题：“在 Optional<String> 类型的上下文中，什么是类型 T？”

```java
GenericTypes.resolveType(t, optionalOfString);
// => String.class
```

这种解析某个泛型超类型的第一个类型参数的场景是如此常见，以至于我们为它创建了一个单独的方法:

```java
GenericTypes.findGenericParameter(optionalOfString, Optional.class);
// => Optional.of(String.class)

Type listOfInteger = new GenericType<List<Integer>>() {}.getType();
GenericTypes.findGenericParameter(listOfInteger, Collection.class);
// => Optional.of(Integer.class)
```

注意，如果类型参数不能被解析，或者类型之间没有任何关系，这个方法将返回`Optional.empty()`:

```java
GenericTypes.findGenericParameter(optionalOfString, List.class);
// => Optional.empty();
```

<a name="140____9_4__NamedArgumentFinder"></a>
### 9.4. NamedArgumentFinder(命名参数查找器)

[NamedArgumentFinder](apidocs/org/jdbi/v3/core/argument/NamedArgumentFinder.html) 接口，顾名思义，从某些来源按名称查找参数。 通常，单个`NamedArgumentFinder` 实例将为多个不同的名称提供参数。

在`bindBean()`，`bindFields()`， `bindMethods()` 和 `bindMap()`都不是很合适的情况下，你可以实现自己的`NamedArgumentFinder`并绑定它，而不是分别提取和绑定每个参数。

```java
Cache cache = ... // e.g. Guava Cache
NamedArgumentFinder cacheFinder = (name, ctx) ->
    Optional.ofNullable(cache.getIfPresent(name))
            .map(value -> ctx.findArgumentFor(Object.class, value));

stmt.bindNamedArgumentFinder(cacheFinder);
```

> **💡提示:** 在幕后，[SqlStatement.bindBean()](apidocs/org/jdbi/v3/core/statement/SqlStatement.html#bindBean-java.lang.Object-), [SqlStatement.bindMethods()](apidocs/org/jdbi/v3/core/statement/SqlStatement.html#bindMethods-java.lang.Object-), [SqlStatement.bindFields()](apidocs/org/jdbi/v3/core/statement/SqlStatement.html#bindFields-java.lang.Object-), and [SqlStatement.bindMap()](apidocs/org/jdbi/v3/core/statement/SqlStatement.html#bindMap-java.util.Map-) 方法只是创建和绑定的自定义实现 `NamedArgumentFinder` 分别用于  beans, methods, fields, 和 maps。

<a name="141____9_5__JdbiConfig"></a>
### 9.5. JdbiConfig

Configuration is managed by the [ConfigRegistry](apidocs/org/jdbi/v3/core/config/ConfigRegistry.html) class. Each Jdbi object that represents a distinct database context (for example, **Jdbi** itself, a **Handle** instance, or an attached SqlObject class) gets its own configuration registry. Most contexts implement the [Configurable](apidocs/org/jdbi/v3/core/config/Configurable.html) interface which allows modification of its configuration as well as retrieving the current context’s configuration for use by Jdbi core or extensions.

When a new configurable context is created, it inherits a copy of its parent configuration at the time of creation - further modifications to the original will not affect already created configuration contexts. Configuration context copies happen when producing a Handle from Jdbi, when opening a **SqlStatement** from the Handle, and when attaching or creating an on-demand extension such as **SqlObject**.

The configuration itself is stored in various implementations of the [JdbiConfig](apidocs/org/jdbi/v3/core/config/JdbiConfig.html) interface. Each implementation must adhere to the contract of the interface; in particular it must have a public no-argument constructor that provides useful defaults and a **createCopy** method that is invoked when a configuration registry is cloned.

Generally, configuration should be set on a context before that context is used, and not changed later. Some configuration classes may be thread safe but most are not.

Many of Jdbi’s core features, for example argument or mapper registries, are simply implementations of **JdbiConfig** that store the registered mappings for later use during query execution.

```
public class ExampleConfig implements JdbiConfig<ExampleConfig> {

    private String color;
    private int number;

    public ExampleConfig() {
        color = "purple";
        number = 42;
    }

    private ExampleConfig(ExampleConfig other) {
        this.color = other.color;
        this.number = other.number;
    }

    public ExampleConfig setColor(String color) {
        this.color = color;
        return this;
    }

    public String getColor() {
        return color;
    }

    public ExampleConfig setNumber(int number) {
        this.number = number;
        return this;
    }

    public int getNumber() {
        return number;
    }

    @Override
    public ExampleConfig createCopy() {
        return new ExampleConfig(this);
    }

}
```

<a name="142_____9_5_1__Creating_a_custom_JdbiConfig_type"></a>
#### 9.5.1. Creating a custom JdbiConfig type

- Create a public class that implements JdbiConfig.
- Add a public, no-argument constructor
- Add a private, copy constructor.
- Implement `createCopy()` to call the copy constructor.
- Add config properties, and provide sane defaults for each property.
- Ensure that all config properties get copied to the new instance in the copy constructor.
- Override `setConfig(ConfigRegistry)` if your config class wants to be able to use other config classes in the registry. E.g. RowMappers registry delegates to ColumnMappers registry, if it doesn’t have a mapper registered for a given type.
- Use that configuration object from other classes that are interested in it.
  - e.g. BeanMapper, FieldMapper, and ConstructorMapper all use the ReflectionMappers config class to keep common configuration.

<a name="143____9_6__JdbiPlugin"></a>
### 9.6. JdbiPlugin

JdbiPlugin can be used to bundle bulk configuration. Plugins may be installed explicitly via `Jdbi.installPlugin(JdbiPlugin)`, or may be installed automagically from the classpath using the ServiceLoader mechanism via `installPlugins()`.

Jars may provide a file in `META-INF/services/org.jdbi.v3.core.spi.JdbiPlugin` containing the fully qualified class name of your plugin.

In general, Jdbi’s separate artifacts each provide a single relevant plugin (e.g. `jdbi3-sqlite`), and such modules will be auto-loadable. Modules that provide no (e.g. `jdbi3-commons-text`) or multiple (e.g. `jdbi3-core`) plugins typically will not be.

|      | The developers encourage you to install plugins explicitly. Code with declared dependencies on the module it uses is more robust to refactoring and provides useful data for static analysis tools about what code is or isn’t used. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

<a name="144____9_7__StatementContext"></a>
### 9.7. StatementContext

The [StatementContext](apidocs/org/jdbi/v3/core/statement/StatementContext.html) class is a carrier for various state related to the creation and execution of statements that is not appropriate to hold on the **Query** or other particular statement class itself. Among other things, it holds open **JDBC** resources, processed SQL statements, and accumulated bindings. It is exposed to implementations of most user extension points, for example **RowMapper, \*ColumnMapper\*s, or \*CollectorFactory**.

The **StatementContext** itself is not intended to be extended and generally extensions should not need to mutate the context. Please read the JavaDoc for more information on advanced usage.

<a name="145____9_8__User_Defined_Annotations"></a>
### 9.8. User-Defined Annotations

SQL Object is designed to be extended with user-defined annotations. In fact, most of the annotations provided in Jdbi are wired up with the approach outlined below.

There are a few different categories of annotations in SQL Object, and it’s important to understand the differences between them:

- [Statement Customizing Annotations](#_statement_customizing_annotations) - configures the underlying [SqlStatement](apidocs/org/jdbi/v3/core/statement/SqlStatement.html) of a method prior to execution. These can only be used in tandem with annotations like `@SqlQuery`, `@SqlUpdate`, etc, and do not work on default methods.
- [Configuration Annotations](#_configuration_annotations) - modifies configuration in the [ConfigRegistry](apidocs/org/jdbi/v3/core/config/ConfigRegistry.html) within the scope of a SQL object or one of its methods.
- [Method Decorating Annotations](#_method_decorating_annotations) - decorates a method invocation with some additional behavior, e.g. the `@Transaction` annotation wraps the method call in a `handle.inTransaction()` call.

Once you know which type of annotation you want, proceed to the appropriate section below and follow the guide to set it up.

<a name="146_____9_8_1__Statement_Customizing_Annotations"></a>
#### 9.8.1. Statement Customizing Annotations

SQL statement customizing annotations are used to apply some change to the [SqlStatement](apidocs/org/jdbi/v3/core/statement/SqlStatement.html) associated with a SQL method.

Typically these annotations correlate to an API method in core. e.g. `@Bind` corresponds to `SqlStatement.bind()`, `@MaxRows` corresponds to `Query.setMaxRows()`, etc.

Customizing annotations are applied only after the [SqlStatement](apidocs/org/jdbi/v3/core/statement/SqlStatement.html) has been created.

You can create your own SQL statement customizing annotations and attach runtime behavior to them.

First, create an annotation that you want to attach a statement customization to:

```
@Retention(RetentionPolicy.RUNTIME) 
@Target({ElementType.TYPE, ElementType.METHOD, ElementType.PARAMETER}) 
public @interface MaxRows {
  int value();
}
```

|      | All statement customizing annotations should have a `RUNTIME` retention policy. |
| ---- | ------------------------------------------------------------ |
|      | Statement customizing annotations only work on types, methods, or parameters. Strictly speaking, the `@Target` annotation is not required, but it’s a good practice to include it, so that annotations can only be applied where they will actually do something. |

Placing a customizing annotation on a type means "apply this customization to every method."

When used on parameters, annotations may use the argument passed to the method while processing the annotation.

Next, we write an implementation of the [SqlStatementCustomizerFactory](apidocs/org/jdbi/v3/sqlobject/customizer/SqlStatementCustomizerFactory.html) class, to process the annotation and apply the customization to the statement.

The `SqlStatementCustomizerFactory` produces two different types of "statement customizer" command objects: [SqlStatementCustomizer](apidocs/org/jdbi/v3/sqlobject/customizer/SqlStatementCustomizer.html) (for annotations on types or methods), and [SqlStatementParameterCustomizer](apidocs/org/jdbi/v3/sqlobject/customizer/SqlStatementParameterCustomizer.html) (for annotations on method parameters).

Let’s implement a statement customizer factory for our annotation:

```
public class MaxRowsFactory implements SqlStatementCustomizerFactory {
    @Override
    public SqlStatementCustomizer createForType(Annotation annotation,
                                                Class<?> sqlObjectType) {
        final int maxRows = ((MaxRows)annotation).value(); 
        return stmt -> ((Query)stmt).setMaxRows(maxRows); 
    }

    @Override
    public SqlStatementCustomizer createForMethod(Annotation annotation,
                                                  Class<?> sqlObjectType,
                                                  Method method) {
        return createForType(annotation, sqlObjectType); 
    }

    @Override
    public SqlStatementParameterCustomizer createForParameter(Annotation annotation,
                                                              Class<?> sqlObjectType,
                                                              Method method,
                                                              Parameter param,
                                                              int index,
                                                              Type type) {
        return (stmt, maxRows) -> ((Query)stmt).setMaxRows((Integer) maxRows); 
    }
}
```

|      | Extract the max rows from the annotation                     |
| ---- | ------------------------------------------------------------ |
|      | [SqlStatementCustomizer](apidocs/org/jdbi/v3/sqlobject/customizer/SqlStatementCustomizer.html) can be implemented as a lambda—it receives a [SqlStatement](apidocs/org/jdbi/v3/core/statement/SqlStatement.html) as a parameter, calls whatever method it wants on the statement, and returns void. |
|      | Since the customization for this annotation is the same at the method level as at the type level, we simply delegate to the type-level method for brevity. |
|      | [SqlStatementParameterCustomizer](apidocs/org/jdbi/v3/sqlobject/customizer/SqlStatementParameterCustomizer.html) can also be implemented as a lambda. It accepts a `SqlStatement` and the value that was passed into the method on the annotated parameter. |

Finally, add the [@SqlStatementCustomizingAnnotation](apidocs/org/jdbi/v3/sqlobject/customizer/SqlStatementCustomizingAnnotation.html) annotation the `@MaxRows` annotation type. This tells Jdbi that `MaxRowsFactory` implements the behavior of the `@MaxRows` annotation:

```
@SqlStatementCustomizingAnnotation(MaxRowsFactory.class)
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.PARAMETER})
public @interface MaxRows {
    int value() default -1;
}
```

Your statement customizing annotation is now ready to use on any SQL object:

```
public interface Dao {
  @SqlQuery("select * from contacts")
  @MaxRows(100)
  List<Contact> list();

  @SqlQuery("select * from contacts")
  List<Contact> list(@MaxRows int maxRows);
}
```

|      | We chose `@MaxRows` as an example here because it was easy to understand. In practice, you will get better database performance by using a `LIMIT` clause in your SQL statement than by using `@MaxRows`. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

<a name="147_____9_8_2__Configuration_Annotations"></a>
#### 9.8.2. Configuration Annotations

Configuration annotations are used to apply some change to the [ConfigRegistry](apidocs/org/jdbi/v3/core/config/ConfigRegistry.html) associated with a SQL object or method.

Typically these annotations correlate to a method of [Configurable](apidocs/org/jdbi/v3/core/config/Configurable.html) (`Jdbi`, `Handle`, and `SqlStatement` all implement this interface). For example, `@RegisterColumnMapper` correlates to `Configurable.registerColumnMapper()`.

You can create your own configuration annotations, and attach runtime behavior to them:

- Write a new configuration annotation, with any attributes you need.
- Write an implementation of [Configurer](apidocs/org/jdbi/v3/sqlobject/config/Configurer.html) which performs the configuration associated with your annotation.
- Add the `@ConfiguringAnnotation` annotation to your configuration annotation type.

With the above steps completed, Jdbi will invoke your configurer whenever it encounters the associated annotation.

Let’s re-implement one of Jdbi’s built-in annotations as an example:

The `@RegisterColumnMapper` annotation has an attribute to specify the class of the column mapper to register. Wherever the annotation is used, we want Jdbi to create an instance of that mapper type, and register it with the config registry.

First, let’s create the new annotation type:

```
@Retention(RetentionPolicy.RUNTIME) 
@Target({ElementType.TYPE, ElementType.METHOD}) 
public @interface RegisterColumnMapper{
  Class<? extends ColumnMapper<?>> value();
}
```

|      | All configuration annotations should have a `RUNTIME` retention policy. |
| ---- | ------------------------------------------------------------ |
|      | Configuration annotations only work on types and methods. Strictly speaking, the `@Target` annotation is not required, but it’s a good practice to include it, so that annotations can only be applied where they will actually do something. |

Placing a configuration annotation on a type means "apply this configuration to every method."

Next, we write an implementation of the [Configurer](apidocs/org/jdbi/v3/sqlobject/config/Configurer.html) class, to process the annotation and apply the configuration:

```
public class RegisterColumnMapperImpl implements Configurer {
  @Override
  public void configureForMethod(ConfigRegistry registry,
                                 Annotation annotation,
                                 Class<?> sqlObjectType,
                                 Method method) {
    configure(registry, (RegisterColumnMapper) annotation);
  }

  @Override
  public void configureForType(ConfigRegistry registry,
                               Annotation annotation,
                               Class<?> sqlObjectType) {
    configure(registry, (RegisterColumnMapper) annotation);
  }

  private void configure(ConfigRegistry registry,
                         RegisterColumnMapper registerColumnMapper) { 
    try {
      Class<? extends ColumnMapper> mapperType = registerColumnMapper.value();
      ColumnMapper mapper = mapperType.getConstructor().newInstance();
      registry.get(ColumnMappers.class).register(mapper);
    }
    catch (NoSuchMethodException e) {
      throw new RuntimeException("Cannot construct " + mapperType, e);
    }
  }
}
```

|      | In this example, we’re applying the same configuration, whether the `@RegisterColumnMapper` annotation is used on the SQL object type or method. However this is not a requirement—some annotations may choose to apply configuration differently depending on whether the annotation is placed on the type or the method. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

For configuration annotations with only one target, (e.g. `@KeyColumn` and `@ValueColumn` may only be applied to methods), you need only implement the `Configurer` method appropriate for the annotation target.

Finally, add the [@ConfiguringAnnotation](apidocs/org/jdbi/v3/sqlobject/config/ConfiguringAnnotation.html) annotation to your `@RegisterColumnMapper` annotation type. This tells Jdbi that `RegisterColumnMapperImpl` implements the behavior of the `@RegisterColumnMapper` annotation.

```
@ConfiguringAnnotation(RegisterColumnMapperImpl.class)
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface RegisterColumnMapper {
    Class<? extends TemplateEngine> value();
}
```

Your configuration annotation is now ready to use in any SQL object:

```
public interface AccountDao {
  @SqlQuery("select balance from accounts where id = ?")
  @RegisterColumnMapper(MoneyMapper.class)
  public Money getBalance(long accountId);
}
```

<a name="148_____9_8_3__Method_Decorating_Annotations"></a>
#### 9.8.3. Method Decorating Annotations

Method decorating annotations are used to enhance a SQL Object method with additional (or substitute) behavior.

Internally, SQL Object represents the behavior of each method with an instance of the [Handler](apidocs/org/jdbi/v3/sqlobject/Handler.html) interface. Every time you call a method on a SQL Object instance, the method is executed by executing the handler for the given method.

When you use a decorating annotation (like `@Transaction`), the regular handler for a method is wrapped in another handler which may perform some action before and/or after passing the call to the original handler.

A decorator could even perform some action *instead* of calling the original, e.g. for a caching annotation.

Let’s re-implement the `@Transaction` annotation to see how it works:

First, create the annotation type:

```
@Retention(RetentionPolicy.RUNTIME) 
@Target(ElementType.METHOD) 
public @interface Transaction {
    TransactionIsolationLevel value();
}
```

|      | All decorating annotations should have a `RUNTIME` retention policy. |
| ---- | ------------------------------------------------------------ |
|      | Decorating annotations only work on types and methods. Strictly speaking, the `@Target` annotation is not required, but it’s a good practice to include it, so that annotations can only be applied where they will actually do something. |

Placing a decorating annotation on a type means "apply this decoration to every method."

Next we write an implementation of the [HandlerDecorator](apidocs/org/jdbi/v3/sqlobject/HandlerDecorator.html) interface, to process the annotation and apply the decoration:

```
public class TransactionDecorator implements HandlerDecorator {
  public Handler decorateHandler(Handler base,
                                 Class<?> sqlObjectType,
                                 Method method) {
    Transaction anno = method.getAnnotation(Transaction.class); 
    TransactionIsolationLevel isolation = anno.value(); 

    return (target, args, handleSupplier) -> handleSupplier.getHandle() 
        .inTransaction(isolation, h -> base.invoke(target, args, handleSupplier));
  }
}
```

|      | Get the `@Transaction` annotation                            |
| ---- | ------------------------------------------------------------ |
|      | Extract the transaction isolation level from the annotation  |
|      | The `Handler` interface accepts a target (the SQL Object instance being invoked), an `Object[]` array of arguments passed to the method, and a `HandleSupplier`. |

Finally, add the [@SqlMethodDecoratingAnnotation](apidocs/org/jdbi/v3/sqlobject/SqlMethodDecoratingAnnotation.html) annotation to your `@Transaction` annotation type. This tells Jdbi that `TransactionDecorator` implements the behavior of the `@Transaction` annotation.

```
@SqlMethodDecoratingAnnotation(TransactionDecorator.class)
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Transaction {
    TransactionIsolationLevel value();
}
```

Your decorating annotation is now ready to use in any SQL object:

```
public interface ContactDao {
  @SqlBatch("insert into contacts (id, name) values (:id, :name)")
  @Transaction
  void batchInsert(@BindBean Contact... contacts);
}
```

<a name="149______Decorator_Order"></a>
##### Decorator Order

If a SQL object method has two or more decorating annotations applied, and the order of decorations is important, use the `@DecoratorOrder` annotation. If no order is declared, type decorators apply first, then method decorators, but the order is not further specified.

For example, suppose a method were annotated both with `@Cached` and `@Transaction` (just go with it..). We would probably want the `@Cached` annotation to go first, so that transactions are not created unnecessarily when the cache already contains the entry.

```
public interface ContactDao {
  @SqlQuery("select * from contacts where id = ?")
  @Cached
  @Transaction
  @DecoratorOrder(Cached.class, Transaction.class)
  Contact getById(long id);
}
```

Decorator order is expressed from outermost to innermost.

<a name="150____9_9__TemplateEngine"></a>
### 9.9. TemplateEngine

Jdbi uses a [TemplateEngine](apidocs/org/jdbi/v3/core/statement/TemplateEngine.html) implementation to render templates into SQL. Template engines take a SQL template string and the `StatementContext` as input, and produce a parseable SQL string as output.

Out of the box, Jdbi is configured to use `DefinedAttributeTemplateEngine`, which replaces angle-bracked tokens like `<name>` in your SQL statements with the string value of the named attribute:

```
String tableName = "customers";
Class<?> entityClass = Customer.class;

handle.createQuery("select <columns> from <table>")
      .define("table", "customers")
      .defineList("columns", "id", "name")
      .mapToMap()
      .list() // => "select id, name from customers"
```

|      | The `defineList` method defines a list of elements as the comma-separated splice of String values of the individual elements. In the above example, the `columns` attribute is defined as `"id, name"`. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

Any custom template engine can be used. Simply implement the `TemplateEngine` interface, then call `setTemplateEngine()` on the `Jdbi`, `Handle`, or on a SQL statement like `Update` or `Query`:

```
TemplateEngine templateEngine = (template, ctx) -> {
  ...
};

jdbi.setTemplateEngine(templateEngine);
```

|      | Jdbi also provides `StringTemplateEngine`, which renders templates using the StringTemplate library. See [StringTemplate 4](#_stringtemplate_4). |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

<a name="151____9_10__SqlParser"></a>
### 9.10. SqlParser

After the SQL template has been rendered, Jdbi uses a [SqlParser](apidocs/org/jdbi/v3/core/statement/SqlParser.html) to parse out any named parameters from the SQL statement. This Produces a `ParsedSql` object, which contains all the information Jdbi needs to bind parameters and execute your SQL statement.

Out of the box, Jdbi is configured to use `ColonPrefixSqlParser`, which recognizes colon-prefixed named parameters, e.g. `:name`.

```
handle.createUpdate("insert into characters (id, name) values (:id, :name)")
      .bind("id", 1)
      .bind("name", "Dolores Abernathy")
      .execute();
```

Jdbi also provides `HashPrefixSqlParser`, which recognizes hash-prefixed parameters, e.g. `#hashtag`. Use this parser by calling `setSqlParser()` on the `Jdbi`, `Handle`, or any SQL statement such as `Query` or `Update`.

```
handle.setSqlParser(new HashPrefixSqlParser());
handle.createUpdate("insert into characters (id, name) values (#id, #name)")
      .bind("id", 2)
      .bind("name", "Teddy Flood")
      .execute();
```

|      | The default parsers recognize any Java identifier as a parameter or attribute name. Even some strange cases like emoji are allowed, although the Jdbi authors encourage appropriate discretion 🧐. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

|      | The default parsers try to ignore parameter-like constructions inside of string literals, since JDBC drivers wouldn’t let you bind parameters there anyway. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

For you fearless adventurers who have read the [Dragon book](https://www.amazon.com/Compilers-Principles-Techniques-Tools-2nd/dp/0321486811), any custom SQL parser can be used. Simply implement the `SqlParser` interface, then set it on the Jdbi, Handle, or SQL statement:

```
SqlParser parser = (sql, ctx) -> {
  ...
};

jdbi.setParser(parser);
```

<a name="152____9_11__SqlLogger"></a>
### 9.11. SqlLogger

The [SqlLogger](apidocs/org/jdbi/v3/core/statement/SqlLogger.html) interface is called before and after executing each statement, and given the current `StatementContext`, to log any relevant information desired: mainly the query in various compilation stages, attributes and bindings, and important timestamps.

<a name="153____9_12__ResultProducer"></a>
### 9.12. ResultProducer

A **ResultProducer** takes a lazily supplied **PreparedStatement** and produces a result. The most common producer path, **execute()**, retrieves the **ResultSet** over the query results and then uses a **ResultSetScanner** or higher level mapper to produce results.

An example alternate use is to just return the number of rows modified, as in an UPDATE or INSERT statement:

```
public static ResultProducer<Integer> returningUpdateCount() {
    return (statementSupplier, ctx) -> {
        try {
            return statementSupplier.get().getUpdateCount();
        } finally {
            ctx.close();
        }
    };
}
```

If you acquire the lazy statement, you are responsible for ensuring that the context is closed eventually to release database resources.

Most users will not need to implement the **ResultProducer** interface.

<a name="154____9_13__Generator"></a>
### 9.13. Generator

Jdbi includes an experimental SqlObject code generator. If you include the `jdbi3-generator` artifact as an annotation processor and annotate your SqlObject definitions with `@GenerateSqlObject`, the generator will produce an implementing class and avoid using `Proxy` instances. This may be useful for `graal-native` compilation.

<a name="155___10__Appendix"></a>
## 10. Appendix

<a name="156____10_1__Best_Practices"></a>
### 10.1. Best Practices

- Test your SQL Objects (DAOs) against real databases when possible. Jdbi tries to be defensive and fail eagerly when you hold it wrong.
- Use the `-parameters` compiler flag to avoid all those `@Bind("foo") String foo` redundant qualifiers in SQL Object method parameters. See [使用参数名称编译](#133____9_2__使用参数名称编译).
- Use a profiler! The true root cause of performance problems can often be a surprise. Measure first, *then* tune for performance. And then measure again to be sure it made a difference.
- Don’t forget to bring a towel!

<a name="157____10_2__API_Reference"></a>
### 10.2. API Reference

- [Javadoc](apidocs/index.html)
- [jdbi3-kotlin](apidocs-kotlin/jdbi3-kotlin/index.html)
- [jdbi3-kotlin-sqlobject](apidocs-kotlin/jdbi3-kotlin-sqlobject/index.html)

<a name="158____10_3__Related_Projects"></a>
### 10.3. Related Projects

[Embedded Postgres](https://github.com/opentable/otj-pg-embedded) makes testing against a real database quick and easy.

[dropwizard-jdbi3](https://github.com/arteam/dropwizard-jdbi3) provides integration with DropWizard.

[metrics-jdbi3](https://github.com/arteam/metrics-jdbi3) instruments using DropWizard-Metrics to emit statement timing statistics.

Do you know of a project related to Jdbi? Send us an issue and we’ll add a link here!

<a name="159____10_4__Contributing"></a>
### 10.4. Contributing

**jdbi** uses GitHub for collaboration. Please check out the [project page](https://github.com/jdbi/jdbi) for more information.

If you have a question, we have a [Google Group mailing list](https://groups.google.com/group/jdbi)

Users sometimes hang out on [IRC in #jdbi on Freenode](irc://irc.freenode.net/#jdbi).

<a name="160____10_5__Upgrading_from_v2_to_v3"></a>
### 10.5. Upgrading from v2 to v3

Already using Jdbi v2?

Here’s a quick summary of differences to help you upgrade:

General:

- Maven artifacts renamed and split out:
- Old: `org.jdbi:jdbi`
- New: `org.jdbi:jdbi3-core`, `org.jdbi:jdbi3-sqlobject`, etc.
- Root package renamed: `org.skife.jdbi.v2` → `org.jdbi.v3`

Core API:

- `DBI`, `IDBI` → `Jdbi`
  - Instantiate with `Jdbi.create()` factory methods instead of constructors.
- `DBIException` → `JdbiException`
- `Handle.select(String, …)` now returns a `Query` for further method chaining, instead of a `List<Map<String, Object>>`. Call `Handle.select(sql, …).mapToMap().list()` for the same effect as v2.
- `Handle.insert()` and `Handle.update()` have been coalesced into `Handle.execute()`.
- `ArgumentFactory` is no longer generic.
- `AbstractArgumentFactory` is a generic implementation of `ArgumentFactory` for factories that handle a single argument type.
- Argument and mapper factories now operate in terms of `java.lang.reflect.Type` instead of `java.lang.Class`. This allows Jdbi to handle arguments and mappers for generic types.
- Argument and mapper factories now have a single `build()` method that returns an `Optional`, instead of separate `accepts()` and `build()` methods.
- `ResultSetMapper` → `RowMapper`. The row index parameter was also removed from `RowMapper`--the current row number can be retrieved directly from the `ResultSet`.
- `ResultColumnMapper` → `ColumnMapper`
- `ResultSetMapperFactory` → `RowMapperFactory`
- `ResultColumnMapperFactory` → `ColumnMapperFactory`
- `Query` no longer maps to `Map<String, Object>` by default. Call `Query.mapToMap()`, `.mapToBean(type)`, `.map(mapper)` or `.mapTo(type)`.
- `ResultBearing<T>` was refactored into `ResultBearing` (no generic parameter) and `ResultIterable<T>`. Call `.mapTo(type)` to get a `ResultIterable<T>`.
- `TransactionConsumer` and `TransactionCallback` only take a `Handle` now—the `TransactionStatus` argument is removed. Just rollback the handle now.
- `TransactionStatus` class removed.
- `CallbackFailedException` class removed. The functional interfaces like `HandleConsumer`, `HandleCallback`, `TransactionCallback`, etc can now throw any exception type. Methods like `Jdbi.inTransaction` that take these callbacks use exception transparency to throw only the exception thrown by the callback. If your callback throws no checked exceptions, you don’t need a try/catch block.
- `StatementLocator` interface removed from core. All core statements expect to receive the actual SQL string now. A similar concept, `SqlLocator` was added but is specific to SQL Object.
- `StatementRewriter` refactored into `TemplateEngine`, and `SqlParser`.
- StringTemplate no longer required to process `<name>`-style tokens in SQL.
- Custom SqlParser implementations must now provide a way to transform raw parameter names to names that will be properly parsed out as named params.

SQL Object API:

- SQL Object support is not installed by default. It must be added as a separate dependency, and the plugin installed into the `Jdbi` object:

```
Jdbi jdbi = Jdbi.create(...);
jdbi.installPlugin(new SqlObjectPlugin());
```

- SQL Object types in v3 must be public interfaces—no classes. Method return types must likewise be public. This is due to SQL Object implementation switching from CGLIB to `java.lang.reflect.Proxy`, which only supports interfaces.
- `GetHandle` → `SqlObject`
- `SqlLocator` replaces `StatementLocator`, and only applies to SQL Objects.
- `@RegisterMapper` divided into `@RegisterRowMapper` and `@RegisterColumnMapper`.
- `@Bind` annotations on SQL Object method parameters can be made optional, by compiling your code with the `-parameters` compiler flag enabled.
- `@BindIn` → `@BindList`, and no longer requires StringTemplate
- On-demand SQL objects don’t play well with methods that return `Iterable` or `FluentIterable`. On-demand objects strictly close the handle after each method call, and no longer "hold the door open" for you to finish consuming the interable as they did in v2. This forecloses a major source of connection leaks.
- SQL Objects are no longer closeable — they are either on-demand, or their lifecycle is tied to the lifecycle of the `Handle` they are attached to.
- `@BindAnnotation` meta-annotation removed. Use `@SqlStatementCustomizingAnnotation` instead.
- `@SingleValueResult` → `@SingleValue`. The annotation may be used for method return types, or on `@SqlBatch` parameters.
