# Jdbi 3 开发指南

> 翻译: <span style="font-size:1.5rem; background:yellow;">**白石**</span>
>
> 开源地址: https://github.com/wjw465150/Jdbi3-Developer-Guide

<a name=" _introduction_to_jdbi"></a>
## [1. Jdbi 简介](#_introduction_to_jdbi)

Jdbi提供了对Java中关系数据的方便、惯用的访问。Jdbi 3是第三个主要版本，它引入了对Java 8的增强支持，对设计和实现的无数改进，以及对模块化插件的增强支持。

> **💡提示:** 不想升级了吗?[v2文档](jdbi2/index.html)仍然可用。

Jdbi构建在JDBC之上。如果您的数据库有JDBC驱动程序，则可以使用Jdbi。Jdbi改进了JDBC的粗糙接口，提供了更自然的Java数据库接口，易于绑定到域数据类型。与ORM不同的是，我们的目标不是提供一个完整的对象关系映射框架—与隐藏的复杂性不同，我们提供的构建块允许您根据您的应用程序构建关系和对象之间的映射。

Jdbi的API有两种形式:

### [1.1. 流式 API](#_fluent_api)

Core API 提供了一个流畅的命令式接口。 使用 Builder 样式对象将 SQL 连接到丰富的 Java 数据类型。

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

### [1.2. 声明式 API](#_declarative_api)

SQL Object扩展位于Core之上，并提供了一个声明式接口。通过声明一个带注解的Java“接口”，告诉Jdbi要执行什么SQL以及您喜欢的结果的形状，并且提供实现。

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

## [2. 开始](#_getting_started)

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

Gradle里这样引入依赖:

```groovy
dependencies {
  //Jdbi3
  implementation group: 'org.jdbi', name: 'jdbi3-core', version: '3.16.0'
  implementation group: 'org.jdbi', name: 'jdbi3-sqlobject', version: '3.16.0'
}
```

### [2.1. Java兼容性](#_java_compatibility)
`Jdbi` 在所有 Java 版本 8 或更高版本上运行。 构建需要 11 或更高版本。
> **☢警告:** Java 8的支持被认为是不赞成的，目前将会尽最大努力维护，但很快就会消失!为了在8上运行，你可能需要依赖你的咖啡因版本回到2.x。3.x需要在新的jdk上运行，但不能在8上运行。

### [2.2. 模块](#_modules)

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

### [2.3. 事先申明](#_forethoughts)

> **💡提示:** 您可能想要添加我们的注解`org.jdbi.v3.meta.Beta` 将被列入IDE的“不稳定API使用”黑名单。

> **☢警告:** 我们的`org.jdbi.*.internal`包不被认为是公共API;它们的内容可能会在没有警告的情况下发生根本变化。

## [3. 核心 API](#_core_api)

### [3.1. Jdbi](#_jdbi)

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

**`Jdbi`实例是线程安全的，不拥有任何数据库资源。**

通常，应用程序创建一个单例的、共享的`Jdbi`实例，并在那里设置任何公共配置。更多细节请参见[Configuration](#_configuration)。

`Jdbi `本身不提供连接池或其他[高可用性](#_high_availability)特性，但它可以与其他提供这些特性的软件结合使用。

在一个更有限的范围内(比如HTTP请求或事件回调)，您将从您的`Jdbi`实例请求一个`Handle`对象。

### [3.2. Handle(句柄)](#_handle)

句柄表示一个活动的[数据库连接](https://docs.oracle.com/javase/8/docs/api/java/sql/Connection.html).

[Handle](apidocs/org/jdbi/v3/core/Handle.html)用于准备和运行针对数据库的SQL语句，并管理数据库事务。它提供对流式语句API的访问，这些API可以绑定参数、执行语句，然后将任何结果映射到Java对象。

`Handle`在创建时从`Jdbi`继承配置。更多细节请参见[Configuration](#_configuration)。

> **⚠小心:** 因为`Handle`持有一个打开的连接，所以必须小心确保当您使用完它时，每个Handle都是关闭的。如果关闭句柄失败，将会导致数据库被打开的连接淹没，或者耗尽连接池。

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

或者，如果你想自己管理句柄的生命周期，使用`jdbi.open()`:

```java
try (Handle handle = jdbi.open()) {
    handle.execute("insert into contacts (id, name) values (?, ?)", 3, "Chuck");
}
```

> **⚠小心:** 当使用`jdbi.open()`时，应该始终使用try-with-resources或try-finally块来确保数据库连接被释放。不释放Handle将泄漏连接。我们建议在可能的情况下使用`withHandle`或`useHandle`而不是`open`。

### [3.3. 参数](_arguments)

Arguments是Jdbi对JDBC语句参数的表示(the `?` in `select * from Foo where bar = ?`).

在JDBC `PreparedStatement`上设置参数`?`，您可以使用`ps.setString(1, "Baz")`。 使用Jdbi，当你绑定字符串`"Baz"`时，它会搜索所有注册的*ArgumentFactory*实例，直到找到一个愿意将String转换为*Argument*的实例。参数负责为占位符设置字符串，就像`setString`所做的那样。

参数可以执行比简单JDBC支持的更高级的绑定:BigDecimal可以绑定为SQL decimal，java.time.Year可以绑定为SQL int，或者一个复杂对象可以序列化为字节数组并绑定为SQL blob。

> **🏷注意:** Jdbi参数的使用仅限于JDBC `prepared statement`语句参数。 值得注意的是，arguments通常不能用于改变查询的结构(例如表或列名，`SELECT`或`INSERT`等)，也不能将参数插入到字符串字面量中。 更多信息请参见[Templating](#36____3_6__Templating) 和 [TemplateEngine](#150____9_9__TemplateEngine)。

#### [3.3.1. 位置参数](#_positional_arguments)

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

#### [3.3.2. Named Arguments(命名参数)](#_named_arguments)

当SQL语句使用`:name`这样的冒号前缀标记时，Jdbi可以通过名称绑定参数:

```java
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

#### [3.3.3. Supported Argument Types(支持的参数类型)](#_supported_argument_types)

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

#### [3.3.4. Binding Arguments(绑定参数)](#_binding_arguments)

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

#### [3.3.5. Custom Arguments(自定义参数)](#_custom_arguments)

有时，您的数据模型将使用 Jdbi 本身不支持的数据类型（请参阅 [支持的参数类型](#_supported_argument_types)）。

幸运的是，通过实现一些简单的接口，Jdbi 可以配置为将自定义数据类型绑定为参数。

> **🏷注意:** JDBC的核心特性通常得到所有数据库供应商的良好支持。然而，更高级的用法，如数组支持或几何类型，往往很快就会变成特定于供应商的。

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
        statement.setString(position, uuid.toString()); //<1>
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

> **<1>** 由于 Argument 通常直接调用 JDBC，因此在应用时会给出从 1 开始的索引（正如 JDBC 所期望的）。

这里我们使用 **Argument** 直接绑定一个 UUID。 在这种特殊情况下，最明显的方法是将 UUID 作为字符串发送到数据库。 如果您的 JDBC 驱动程序直接支持自定义类型或高效的二进制传输，您可以在此处轻松利用它们。

##### ArgumentFactory(参数工厂)

[ArgumentFactory](apidocs/org/jdbi/v3/core/argument/ArgumentFactory.html) 接口为它知道的任何数据类型提供 [Argument](#16______Argument) 实例。 通过实现和注册一个参数工厂，可以绑定自定义数据类型，而不必将它们显式地包装在 `Argument` 对象中。

Jdbi 提供了一个 `AbstractArgumentFactory` 类，它简化了 `ArgumentFactory` 接口的实现：

```java
static class UUIDArgumentFactory extends AbstractArgumentFactory<UUID> {
    UUIDArgumentFactory() {
        super(Types.VARCHAR); //<1>
    }

    @Override
    protected Argument build(UUID value, ConfigRegistry config) {
        return (position, statement, ctx) -> statement.setString(position, value.toString()); //<2>
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

> **<1>** 绑定 UUID 时使用的 JDBC [SQL 类型常量](https://docs.oracle.com/javase/8/docs/api/java/sql/Types.html)。 Jdbi 需要这个来绑定 `null` 的 UUID 值。 参见 [PreparedStatement.setNull(int,int)](https://docs.oracle.com/javase/8/docs/api/java/sql/PreparedStatement.html#setNull-int-int-)
> **<2>** 由于 `Argument` 是一个函数式接口，它可以实现为一个简单的 lambda 表达式。                                                                                                                                                                                                               |


##### Prepared Arguments(准备参数)

传统的参数工厂根据绑定的类型和实际值来决定绑定。 这是非常灵活的，但是当绑定一个大的 `PreparedBatch` 时，它会导致严重的性能损失，因为必须为每批添加的参数咨询整个参数工厂链。 为了解决这个问题，实现 `ArgumentFactory.Preparable`，它承诺处理给定 `Type` 的所有值。 大多数内置参数工厂现在都实现了 Preparable 接口。

##### Arguments Registry(参数注册表)

当您注册一个 `ArgumentFactory` 时，注册信息存储在 Jdbi 持有的 [Arguments](apidocs/org/jdbi/v3/core/argument/Arguments.html) 实例中。 `Arguments` 是一个配置类，它存储所有注册的参数工厂（包括内置参数的工厂）。

实际上，当您将参数绑定到语句时，Jdbi会查询`Arguments`配置对象，并搜索`ArgumentFactory`，该对象知道如何将绑定对象转换为`Argument`。

稍后，当语句执行时，绑定期间定位的每个`Argument`都会应用到JDBC [PreparedStatement](https://docs.oracle.com/javase/8/docs/api/java/sql/PreparedStatement.html) .

> **🏷注意:**  有时，两个或多个参数工厂将支持相同数据类型的参数。 当这种情况发生时，最后注册的工厂获胜。 可准备参数工厂总是优先于基本参数工厂。 这意味着您可以覆盖任何数据类型的绑定方式，包括开箱即用支持的数据类型。

### [3.4. Queries(查询)](#_queries)

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

当您希望结果至少包含一行时，请调用`first()`。 如果第一行映射到 `null`，则返回 `null`。 如果结果有零行，则抛出异常。

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

### [3.5. Mappers(映射器)](#_mappers)

Jdbi 使用映射器将结果数据转换为 Java 对象。 有两种类型的映射器：

- [Row Mappers](#22_____3_5_1__Row_Mappers), 映射整行结果集数据。
- [Column Mappers](#25_____3_5_2__Column_Mappers), 映射结果集行的单个列。

#### [3.5.1. Row Mappers(行映射器)](#_row_mappers)

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
> **💡提示:** `翻译者WJW`提示: 也可以handle调用registerRowMapper方法

可以在不指定映射类型的情况下注册使用显式映射类型（例如上一节中的 UserMapper 类）实现 `RowMapper` 的映射器：

```java
handle.registerRowMapper(new UserMapper());
```

使用此方法时，Jdbi 检查映射器的泛型类签名以自动发现映射类型。

可以为任何给定类型注册多个映射器。 发生这种情况时，给定类型的最后注册的映射器优先。 这允许优化，比如为某种类型注册一个“默认”映射器，同时允许在适当的时候用不同的映射器覆盖默认映射器。

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

#### [3.5.2. Column Mappers(列映射器)](#_column_mappers)

[ColumnMapper](apidocs/org/jdbi/v3/core/mapper/ColumnMapper.html) 是一个函数接口，从一个JDBC [ResultSet](https://docs.oracle.com/javase/8/docs/api/java/sql/ResultSet.html) 到映射类型。

由于 `ColumnMapper` 是一个函数式接口，所以可以使用 lambda 表达式将它们内联提供给查询：

```java
List<Money> amounts = handle
    .select("select amount from transactions where account_id = ?", accountId)
    .map((rs, col, ctx) -> Money.parse(rs.getString(col))) //<1>
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
- java.time: `Instant`, `LocalDate`, `LocalDateTime`, `LocalTime`, `OffsetDateTime`, `ZonedDateTime`, and `ZoneId` (`翻译者WJW`注: 不包括`java.util.Date`)
- java.util: `UUID`
- `java.util.Collection` 和 Java 数组（用于数组列）。 根据数组元素的类型，可能需要一些额外的设置——请参阅 [SQL 数组](#37____3_7__SQL_Arrays).

> **🏷注意:** 枚举值的绑定和映射方法可以通过 [Enums](apidocs/org/jdbi/v3/core/enums/Enums.html) 配置以及 [EnumByName](apidocs/org/jdbi/v3 /core/enums/EnumByName.html) 和 [EnumByOrdinal](apidocs/org/jdbi/v3/core/enums/EnumByOrdinal.html) 注解。

> `翻译者WJW`添加: 如何注册`java.util.Date`的列映射器
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

#### [3.5.3. Primitive Mapping(基本类型映射)](#_primitive_mapping)

所有 Java 基本类型都有到它们相应的 JDBC 类型的默认映射。 一般情况下，Jdbi 在遇到包装器类型时会自动进行适当的装箱和拆箱。

默认情况下，映射到原始类型的 SQL `null` 将采用 Java 默认值。 这可以通过配置`jdbi.getConfig(ColumnMappers.class).setCoalesceNullPrimitivesToDefaults(false)`来禁用。

#### [3.5.4. Immutables Mapping(不可变映射)](#_immutables_mapping)

`Immutables` 值对象可能会被映射，详情参见 [Immutables](#_Immutables) 部分。

#### [3.5.5. Freebuilder Mapping(自由建造器映射)](#_freebuilder_mapping)

`Freebuilder` 值对象可能会被映射，详情参见 [Freebuilder](#_Freebuilder) 部分。

#### [3.5.6. Reflection Mappers(反射映射器)](#_reflection_mappers)

Jdbi 提供了一些开箱即用的基于反射的映射器。

反射映射器将列名称视为 bean 属性名称 (BeanMapper)、构造函数参数名称 (ConstructorMapper) 或字段名称 (FieldMapper)。

反射映射器可以识别蛇形大小写，并且会自动将这些列与驼峰式字段/参数/属性名称匹配。

> **💡提示:** 要指示 Jdbi 忽略其他可映射的方法，请将其注解为 `@Unmappable`。

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

结果集中可能会省略带有`@Nullable`注解的参数，其中`ConstructorMapper`会将`null`传递给该参数的构造函数。

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

##### BeanMapper(Bean映射器)

我们还提供映射 bean 的基本支持：

```java
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

使用 `factory()` 方法为你的映射类注册一个 bean 映射器：

```java
handle.registerRowMapper(BeanMapper.factory(UserBean.class));

List<UserBean> users = handle
        .createQuery("select id, name from user")
        .mapTo(UserBean.class)
        .list();
```

或者，调用 `mapToBean()` 而不是注册 bean 映射器：

```java
List<UserBean> users = handle
        .createQuery("select id, name from user")
        .mapToBean(UserBean.class)
        .list();
```

Bean 映射器可以为每个映射属性配置一个列名前缀。 这可以帮助消除映射连接的歧义，例如 当两个映射类具有相同的属性名称（如 `id` 或 `name`）时：

```java
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

对于与属性名称不匹配的列名称，请使用 `@ColumnName` 批注来提供准确的列名称。

```java
public class User {
  private int id;

  @ColumnName("user_id")
  public int getId() { return id; }

  public void setId(int id) { this.id = id; }
}
```

`@ColumnName` 注解可以放置在 getter 或 setter 方法上。

> **🏷注意:** `@ColumnName` 注解仅在将 SQL 数据映射到 Java 对象时适用。 当绑定对象属性时（例如使用`bindBean()`），绑定属性名（`:id`）而不是列名（`:user_id`）。

嵌套的 Java Bean 类型可以使用 `@Nested` 注解进行映射：

```java
public class User {
  private int id;
  private String name;
  private Address address;

  ... (getters and setters)

  @Nested //<1>
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

> **<1>** `@Nested` 注解可以放置在 getter 或 setter 方法上。

```java
handle.registerRowMapper(BeanMapper.factory(User.class));

List<User> users = handle
    .select("select id, name, street, city, state, zip from users")
    .mapTo(User.class)
    .list();
```

`@Nested` 注解有一个可选的 `value()` 属性，可用于将列名前缀应用于每个嵌套的 bean 属性：

```java
@Nested("addr")
public Address getAddress() { ... }
handle.registerRowMapper(BeanMapper.factory(User.class));

List<User> users = handle
    .select("select id, name, addr_street, addr_city, addr_state, addr_zip from users")
    .mapTo(User.class)
    .list();
```

> **🏷注意:** 如果结果集没有与嵌套对象的任何属性匹配的列，则 `@Nested` 属性保持不变（即 null）。


##### FieldMapper(字段映射器)

[FieldMapper](apidocs/org/jdbi/v3/core/mapper/reflect/FieldMapper.html) 使用反射将数据库列直接映射到对象字段（包括私有字段）。

```java
public class User {
  public int id;

  public String name;
}
```

使用 `factory()` 方法为你的映射类注册一个字段映射器：

```java
handle.registerRowMapper(FieldMapper.factory(User.class));

List<UserBean> users = handle
        .createQuery("select id, name from user")
        .mapTo(User.class)
        .list();
```

字段映射器可以为每个映射字段配置一个列名前缀。 这可以帮助消除映射连接的歧义，例如 当两个映射类具有相同的属性名称（如 `id` 或 `name`）时：

```java
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

对于与字段名称不匹配的列名称，请使用 `@ColumnName` 注解提供准确的列名称：

```java
public class User {
  @ColumnName("user_id")
  public int id;

  public String name;
}
```

> **🏷注意:** `@ColumnName` 注解仅在将 SQL 数据映射到 Java 对象时适用。 当绑定对象属性时（例如使用`bindBean()`），绑定属性名（`:id`）而不是列名（`:user_id`）。

嵌套的字段映射类型可以使用 `@Nested` 注解进行映射：

```java
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
```

```java
handle.registerRowMapper(FieldMapper.factory(User.class));

List<User> users = handle
    .select("select id, name, street, city, state, zip from users")
    .mapTo(User.class)
    .list();
```

`@Nested` 注解有一个可选的 `value()` 属性，可用于将列名前缀应用于每个嵌套字段：

```java
public class User {
  public int id;
  public String name;

  @Nested("addr")
  public Address address;
}
```

```java
handle.registerRowMapper(FieldMapper.factory(User.class));

List<User> users = handle
    .select("select id, name, addr_street, addr_city, addr_state, addr_zip from users")
    .mapTo(User.class)
    .list();
```

> **🏷注意:** 如果结果集没有与嵌套对象的任何字段匹配的列，则 `@Nested` 字段保持不变（即 null）。

##### Map.Entry mapping(Map条目映射)

开箱即用，Jdbi 注册了一个 `RowMapper<Map.Entry<K,V>>`。 由于结果集中的每一行都是一个`Map.Entry<K,V>`，整个结果集可以很容易地收集到一个`Map<K,V>`（或Guava的`Multimap<K,V>`） .

> **🏷注意:** 必须为键和值类型注册映射器。

通过指定通用映射签名，可以将连接行收集到map结果中：

```java
String sql = "select u.id u_id, u.name u_name, p.id p_id, p.phone p_phone "

    + "from user u left join phone p on u.id = p.user_id";
Map<User, Phone> map = h.createQuery(sql)
        .registerRowMapper(ConstructorMapper.factory(User.class, "u"))
        .registerRowMapper(ConstructorMapper.factory(Phone.class, "p"))
        .collectInto(new GenericType<Map<User, Phone>>() {});
```

在前面的示例中，`User` 映射器使用“u”列名称前缀，`Phone` 映射器使用“p”。 由于每个映射器仅读取具有预期前缀的列，因此相应的 `id` 列是明确的。

可以通过设置键列名来获得唯一索引（例如通过 ID 列）：

```java
Map<Integer, User> map = h.createQuery("select * from user")
        .setMapKeyColumn("id")
        .registerRowMapper(ConstructorMapper.factory(User.class))
        .collectInto(new GenericType<Map<Integer, User>>() {});
```

设置键和值列名称以将两列查询收集到map结果中：

```java
Map<String, String> map = h.createQuery("select key, value from config")
        .setMapKeyColumn("key")
        .setMapValueColumn("value")
        .collectInto(new GenericType<Map<String, String>>() {});
```

以上所有示例都假设是一对一的键/值关系。 如果存在一对多关系怎么办？

Google Guava 提供了一个 `Multimap` 类型，它支持每个键映射多个值。

首先，按照 [Google Guava](#102____7_1__Google_Guava) 部分中的说明将 `GuavaPlugin` 安装到 Jdbi 中。

然后，简单地请求一个 `Multimap` 而不是 `Map`：

```java
String sql = "select u.id u_id, u.name u_name, p.id p_id, p.phone p_phone "
    + "from user u left join phone p on u.id = p.user_id";
Multimap<User, Phone> map = h.createQuery(sql)
        .registerRowMapper(ConstructorMapper.factory(User.class, "u"))
        .registerRowMapper(ConstructorMapper.factory(Phone.class, "p"))
        .collectInto(new GenericType<Multimap<User, Phone>>() {});
```

`collectInto()`方法值得解释。当你调用它时，有几件事情会在幕后发生:

- 参考`JdbiCollectors`注册表获取[CollectorFactory](apidocs/org/jdbi/v3/core/collector/CollectorFactory.html)，它支持给定的容器类型。
- 接下来，要求`CollectorFactory` 从容器类型签名中提取元素类型。 在上面的例子中，`Multimap<User,Phone>` 的元素类型是`Map.Entry<User,Phone>`。
- 从映射注册表中获取该元素类型的映射器。
- 从`CollectorFactory`获取容器类型的[Collector](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collector.html)。
- 最后，返回`map(elementMapper).collect(collector)`。

> **🏷注意:** 如果对`收集器(collector )`工厂、元素类型或元素映射器的查找失败，则会引发异常。

可以增强Jdbi以支持任意容器类型。 有关更多信息，请参阅 [[JdbiCollectors\]](#JdbiCollectors)。


### [3.6. Codecs](#_codecs)

> **🏷注意:** Codec API 仍然不稳定，可能会发生变化。 `翻译者WJW`提示: 先不要使用此功能!

编解码器是为类型注册参数和列映射器的替代品。 它负责将类型化的值序列化为数据库列并从数据库列创建类型。

编解码器收集在编解码器工厂中，该工厂可以使用 `registerCodecFactory` 便捷方法进行注册。

```java
// register a single codec
jdbi.registerCodecFactory(CodecFactory.forSingleCodec(type, codec));

// register a few codecs
jdbi.registerCodecFactory(CodecFactory.builder()
  // register a codec by qualified type
  .addCodec(QualifiedType.of(Foo.class), codec1)
  // register a codec by direct java type
  .addCodec(Foo.class, codec2)
  // register a codec by generic type
  .addCodec(new GenericType<Set<Foo>>() {}. codec3)
  .build());

// register many codecs
Map<QualifiedType<?>, Codec<?>> codecs = ...
jdbi.registerCodecFactory(new CodecFactory(codecs));
```

编解码器示例：

```java
public class Counter {

    private int count = 0;

    public Counter() {}

    public int nextValue() {
        return count++;
    }

    private Counter setValue(int value) {
        this.count = value;
        return this;
    }

    private int getValue() {
        return count;
    }

    /**
     * Codec to persist a counter to the database and restore it back.
     */
    public static class CounterCodec implements Codec<Counter> {

        @Override
        public ColumnMapper<Counter> getColumnMapper() {
            return (r, idx, ctx) -> new Counter().setValue(r.getInt(idx));
        }

        @Override
        public Function<Counter, Argument> getArgumentFunction() {
            return counter -> (idx, stmt, ctx) -> stmt.setInt(idx, counter.getValue());
        }
    }
}
```

JDBI 核心 API:

```java
// register the codec with JDBI
jdbi.registerCodecFactory(CodecFactory.forSingleCodec(COUNTER_TYPE, new CounterCodec()));


// store object
int result = jdbi.withHandle(h -> h.createUpdate("INSERT INTO counters (id, value) VALUES (:id, :value)")
    .bind("id", counterId)
    .bindByType("value", counter, COUNTER_TYPE)
    .execute());


// load object
Counter restoredCounter = jdbi.withHandle(h -> h.createQuery("SELECT value from counters where id = :id")
    .bind("id", counterId)
    .mapTo(COUNTER_TYPE).first());
```

SQL Object API 透明地使用编解码器：

```java
// SQL object dao
public interface CounterDao {
    @SqlUpdate("INSERT INTO counters (id, value) VALUES (:id, :value)")
    int storeCounter(@Bind("id") String id, @Bind("value") Counter counter);

    @SqlQuery("SELECT value from counters where id = :id")
    Counter loadCounter(@Bind("id") String id);
}


    // register the codec with JDBI
    jdbi.registerCodecFactory(CodecFactory.forSingleCodec(COUNTER_TYPE, new CounterCodec()));


    // store object
    int result = jdbi.withExtension(CounterDao.class, dao -> dao.storeCounter(counterId, counter));


    // load object
    Counter restoredCounter = jdbi.withExtension(CounterDao.class, dao -> dao.loadCounter(counterId));
```

#### [3.6.1. Resolving Types(解析类型)](#_resolving_types)

通过使用 [guava](apidocs/org/jdbi/v3/guava/package-summary.html) 模块中的 `TypeResolvingCodecFactory`，可以使用为具体类的子类或接口类型注册的编解码器。这是必要的，例如 将 [Auto Value](https://github.com/google/auto/blob/master/value/userguide/index.md) 生成的类映射到数据库列。

在下面的例子中，只注册了一个用于 `Value<String>` 的编解码器，但是代码使用了 bean 和具体实现的类（`StringBean` 包含一个 `StringValue`，`StringValue` 实现了 `Value<String> `接口）。 如果找不到完美匹配，`TypeResolvingCodecFactory` 将检查类型以查找接口或超类的编解码器。

```java
// SQL object dao using concrete types
public interface DataDao {

    @SqlUpdate("INSERT INTO data (id, value) VALUES (:bean.id, :bean.value)")
    int storeData(@BindBean("bean") StringBean bean);

    @SqlUpdate("INSERT INTO data (id, value) VALUES (:id, :value)")
    int storeData(@Bind("id") String id, @Bind("value") StringValue data);

    @SqlQuery("SELECT value from data where id = :id")
    StringValue loadData(@Bind("id") String id);
}


// generic type representation
public static final QualifiedType<Value<String>> DATA_TYPE = QualifiedType.of(new GenericType<Value<String>>() {});


public static class DataCodec implements Codec<Value<String>> {

    @Override
    public ColumnMapper<Value<String>> getColumnMapper() {
        return (r, idx, ctx) -> {
            return new StringValue(r.getString(idx));
        };
    }

    @Override
    public Function<Value<String>, Argument> getArgumentFunction() {
        return data -> (idx, stmt, ctx) -> {
            stmt.setString(idx, data.getValue());
        };
    }
}


// value interface
public interface Value<T> {

    T getValue();
}


// bean using concrete types, not interface types.
public static class StringBean implements Bean<Value<String>> {

    private final String id;

    private final StringValue value;

    public StringBean(String id, StringValue value) {
        this.id = id;
        this.value = value;
    }

    @Override
    public String getId() {
        return id;
    }

    @Override
    public StringValue getValue() {
        return value;
    }
}
```


### [3.7. Templating(模板)](#_templating)

如上所述，绑定查询参数非常适合向数据库引擎发送一组静态参数。 绑定确保参数化查询字符串（`... where foo = ?`）被传输到数据库，而不允许恶意参数值注入 SQL。

绑定参数并不总是足够的。 有时，查询在执行之前需要进行复杂的或结构化的更改，而参数并不能解决问题。 模板化（使用`TemplateEngine`）允许你通过一般的字符串操作来改变查询的内容。

模板的典型用途是可选或重复段（条件和循环）、复杂变量（如 IN 子句的逗号分隔列表）以及不可绑定 SQL 元素（如表名）的变量替换。 与*参数绑定*不同，由 TemplateEngines 执行的属性呈现**不是** SQL 感知的。 由于它们执行通用字符串操作，如果您不小心使用它们，TemplateEngines很容易产生可怕的混乱或微妙的缺陷查询。

> **⚠小心:** [查询模板是一种常见的攻击载体！](https://www.xkcd.com/327/) 在可能的情况下，始终更喜欢将参数绑定到静态 SQL 而不是动态 SQL。

```java
handle.createQuery("select * from <TABLE> where name = :n")

    // -> "select * from Person where name = :n"
    .define("TABLE", "Person")

    // -> "select * from Person where name = 'MyName'"
    .bind("n", "MyName");
```

> **💡提示:** 使用 TemplateEngine 对查询执行粗略的字符串操作。 查询参数应该由 Arguments 处理。

> **⚠小心:** TemplateEngines 和 SqlParsers 按顺序依次操作：初始字符串将由 TemplateEngine 使用属性呈现，然后由 SqlParser 与 Argument 绑定解析。

如果TemplateEngine输出与SqlParser的参数格式匹配的文本，解析器将尝试将Argument绑定到它。这可能是有用的，例如有命名参数的名称本身也是一个变量，但也可能导致令人困惑的错误:

```java
String paramName = "arg";

handle.createQuery("select * from Foo where bar = :<attr>")
    .define("attr", paramName)
    ...
    .bind(paramName, "baz"); // <- does not need to know the parameter's name ("arg")!
```

```java
handle.createQuery("select * from emotion where emoticon = <sticker>")
    .define("sticker", ":-)") // -> "... where emoticon = :-)"
    .mapToMap()
    // exception: no binding/argument named "-)" present
    .list();
```

绑定和定义通常是分开的。 您可以使用 `stmt.defineNamedBindings()` 或 `@DefineNamedBindings` 定制器以有限的方式链接它们。 对于每个绑定参数（包括 bean 属性），这将定义一个布尔值，如果绑定存在则为“true”而不是“null”。 您可以使用它来制作条件更新和查询子句。

例如:

```java
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

另请参阅有关 [TemplateEngine](#_TemplateEngine)的部分。

#### [3.7.1. ClasspathSqlLocator](#_classpathsqllocator)

您可能会发现，将SQL模板存储在类路径上的单个文件中，而不是存储在Java代码中的字符串中是很有帮助的。

`ClasspathSqlLocator` 将 Java 类型和方法名称转换为类路径位置，然后读取、解析和缓存加载的语句。

```java
// reads classpath resource com/foo/BarDao/query.sql
ClasspathSqlLocator.create().locate(com.foo.BarDao.class, "query");

// same resource as above
ClasspathSqlLocator.create().locate("com.foo.BarDao.query");
```

默认情况下，加载文件中的任何注释都保持不变。 可以通过使用 `removingComments()` 方法实例化 `ClasspathSqlLocator` 来删除注释：

```java
// reads classpath resource com/foo/BarDao/query.sql, stripping all comments
ClasspathSqlLocator.removingComments().locate(com.foo.BarDao.class, "query");

// same resource as above
ClasspathSqlLocator.removingComments().locate("com.foo.BarDao.query");
```

支持多种注释样式：

- C-风格 (`/* … */` 和 `//` 到行尾)
- SQL 风格 (`--` 到行尾)
- shell 风格 (`#` 到行尾; 除非紧跟 `>` 字符； 这是 Postgres `#>` 和 `#>>` 运算符所必需的).

### [3.8. SQL Arrays(SQL数组)](#_sql_arrays)

Jdbi 可以绑定/映射 Java 数组到/从 SQL 数组：

```java
handle.createUpdate("insert into groups (id, user_ids) values (:id, :userIds)")
      .bind("id", 1)
      .bind("userIds", new int[] { 10, 5, 70 })
      .execute();

int[] userIds = handle.createQuery("select user_ids from groups where id = :id")
      .bind("id", 1)
      .mapTo(int[].class)
      .one();
```

你也可以使用集合来代替数组，但如果你使用fluent API，你需要提供元素类型，因为它被擦除了:

```java
handle.createUpdate("insert into groups (id, user_ids) values (:id, :userIds)")
      .bind("id", 1)
      .bindArray("userIds", int.class, Arrays.asList(10, 5, 70))
      .execute();

List<Integer> userIds = handle.createQuery("select user_ids from groups where id = :id")
      .bind("id", 1)
      .mapTo(new GenericType<List<Integer>>() {})
      .one();
```

使用`@SingleValue`来映射SqlObject API的数组结果:

```java
public interface GroupsDao {
  @SqlQuery("select user_ids from groups where id = ?")
  @SingleValue
  List<Integer> getUserIds(int groupId);
}
```

#### [3.8.1. Registering array types(注册数组类型)](#_registering_array_types)

你想要绑定支持的任何 Java 数组元素类型都需要在 Jdbi 的 `SqlArrayTypes` 注册表中注册。 可以使用以下方式注册 JDBC 驱动程序直接支持的数组类型：

```java
jdbi.registerArrayType(int.class, "integer");
```

这里，`"integer"` 是 JDBC 驱动程序本身支持的 SQL 类型名称。

> **🏷注意:** `PostgresPlugin`和`H2DatabasePlugin`等插件会自动为其各自的数据库注册最常见的数组元素类型。

> **💡提示:** Postgres 支持枚举数组类型，因此您可以使用 `jdbi.registerArrayType(Colors.class, "colors")` 为 `enum Colors { red, blue }` 注册数组类型，其中 `"colors"` 是用户定义的枚举 在您的数据库中键入名称。

#### [3.8.2. Binding custom array types(绑定自定义数组类型)](#_binding_custom_array_types)

您还可以提供您自己的 SqlArrayType 实现，它将自定义 Java 元素类型转换为 JDBC 驱动程序支持的类型：

```java
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

您现在可以将 `User[]` 的实例绑定到数据类型 `integer[]` 的参数：

```java
User user1 = new User(1, "bob")
User user2 = new User(42, "alice")

handle.registerArrayType(new UserArrayType());
handle.createUpdate("insert into groups (id, user_ids) values (:id, :users)")
      .bind("id", 1)
      .bind("users", new User[] { user1, user2 })
      .execute();
```

> **🏷注意:** 和[Arguments Registry](#_arguments_registry)一样，如果有多个`SqlArrayType`为同一个数据类型注册，最后注册的获胜。

#### [3.8.3. Mapping array types(映射数组类型)](#_mapping_array_types)

`SqlArrayType` only allows you to bind Java array/collection arguments to their SQL counterparts. To map SQL array columns back to Java types, you can register a regular `ColumnMapper`:

```java
public class UserIdColumnMapper implements ColumnMapper<UserId> {
    @Override
    public UserId map(ResultSet rs, int col, StatementContext ctx) throws SQLException {
        return new UserId(rs.getInt(col));
    }
}
```

```java
handle.registerColumnMapper(new UserIdColumnMapper());
List<UserId> userIds = handle.createQuery("select user_ids from groups where id = :id")
      .bind("id", 1)
      .mapTo(new GenericType<List<UserId>>() {})
      .one();
```

> **🏷注意:** 数组列可以映射到任何在“JdbiCollectors”注册表中注册的容器类型。 例如。 如果安装了 guava 插件，则 `VARCHAR[]` 可以映射到 `ImmutableList<String>`。

### [3.9. Results(结果)](#_results)

执行数据库查询后，您需要解释结果。 JDBC 提供了 **ResultSet** 类，它可以简单地映射到 Java 基本类型和内置类，但 API 使用起来往往很麻烦。 **Jdbi** 提供可配置的映射，包括为行和列注册自定义映射器的能力。

**RowMapper** 将 **ResultSet** 的一行转换为结果对象。

**ColumnMapper** 将单个列的值转换为 Java 对象。 如果只有一列，它可以用作 **RowMapper**，或者它可以用于构建更复杂的 **RowMapper** 类型。

映射器是根据查询的声明结果类型选择的。

**jdbi** 遍历 ResultSet 中的行，并在容器（例如 **List**、**Stream**、**Optional** 或 **Iterator**）中向您呈现映射的结果。

```java
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

#### [3.9.1. ResultBearing(结果承载)](#_resultbearing)

[ResultBearing](apidocs/org/jdbi/v3/core/result/ResultBearing.html) 接口代表一个数据库操作的结果集，它没有映射到任何特定的结果类型。

TODO(要做):

- Query 实现了 ResultBearing
- Update.executeAndReturnGeneratedKeys() 返回 ResultBearing
- PreparedBatch.executeAndReturnGeneratedKeys() 返回 ResultBearing
- 可以映射 ResultBearing 对象，它返回映射类型的 ResultIterable。
  - mapTo(Type | Class | GenericType) 如果映射器已注册类型
  - map(RowMapper | ColumnMapper)
  - mapToBean() 用于bean类型
  - mapToMap() 返回 Map<String,Object> 将小写列名映射到值
- reduceRows
  - RowView
- reduceResultSet
- **collectInto** 例如 带有 GenericType 标记。 在一个操作中隐含一个 mapTo() 和一个 collect() 。 例如 `collectInto(new GenericType<List<User>>(){})` 与 `mapTo(User.class).collect(toList())` 相同
- 提供开箱即用支持的容器类型列表

#### [3.9.2. ResultIterable(结果可迭代)](#_resultiterable)

[ResultIterable](apidocs/org/jdbi/v3/core/result/ResultIterable.html) 表示已映射到特定类型的结果集，例如 `ResultIterable<用户>`。

TODO(要做):

- ResultIterable.forEach
- ResultIterable.iterator()
  - 必须显式关闭，以释放数据库资源。
  - 使用 try-with-resources 确保数据库资源得到清理。

##### Find a Single Result(查找单个结果)

`ResultIterable.one()` 返回结果集中的唯一行。 如果遇到零行或多行，则会抛出`IllegalStateException`。

`ResultIterable.findOne()` 返回结果集中唯一行的 `Optional<T>`，如果没有返回行，则返回 `Optional.empty()`。

`ResultIterable.first()` 返回结果集中的第一行。 如果遇到零行，则抛出“IllegalStateException”。

`ResultIterable.findFirst()` 返回第一行的 `Optional<T>`，如果有的话。

##### Stream(流)

**Stream** 集成允许您使用 RowMapper 将 ResultSet 适配到新的 Java 8 Streams 框架中。 只要您的数据库支持流式结果（例如，只要您在事务中并设置提取大小，PostgreSQL 就会这样做），流将根据需要从数据库中延迟提取行。

**#stream** 返回 **Stream<T>**。 然后您应该处理流并产生结果。 必须关闭此流以释放持有的任何数据库资源，因此我们建议使用 **useStream**、**withStream** 或 **try-with-resources** 块以确保没有资源泄漏。

```java
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

**#withStream** 和 **#useStream** 为您处理关闭流。 您分别提供产生结果的 **StreamCallback** 或不产生结果的 **StreamConsumer**。

##### List(列表)

**#list** 发出 **List<T>**。 这必然会在内存中缓冲所有结果。

```java
List<User> users =
    handle.createQuery("SELECT id, name FROM user")
        .map(new UserMapper())
        .list();
```

##### Collectors(收集者)

**#collect** 需要一个 **Collector<T, ? , R>** 构建结果集合 **R<T>**。 **java.util.stream.Collectors** 类有许多有趣的 **Collector** 实现。

您还可以编写自己的自定义收集器。 例如，要将找到的行累积到一个 **Map** 中：

```java
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

##### Reduction(规约)

**#reduce** 提供了一个简化的 **Stream#reduce**。 给定一个单位起始值和一个 **BiFunction<U, T, U>** 它将反复组合 **U** 直到只剩下一个，然后返回那个。

##### ResultSetScanner(结果集扫描器)

**ResultSetScanner** 接口接受延迟提供的 **ResultSet** 并生成 Jdbi 从语句执行返回的结果。

上面的大多数操作都是通过**ResultSetScanner**实现的。扫描器拥有ResultSet的所有权，可以前进或寻找它。

返回值最终是语句执行的最终结果。

大多数用户应该更喜欢使用上面描述的更高级别的结果收集器，但总得有人做脏活。

#### [3.9.3. Joins(连接)](#_joins)

将多个表连接在一起是一项非常常见的数据库任务。 这也是关系模型和 Java 对象模型之间的不匹配开始抬头的地方。

在这里，我们提出了几种从更复杂的行中检索结果的策略。

以联系人列表应用程序为例。 联系人列表包含任意数量的联系人。 联系人有姓名和任意数量的电话号码。 电话号码有一个类型（例如家庭、工作）和一个电话号码：

```java
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

为简洁起见，我们省略了 getter、setter 和访问修饰符。

由于我们将重用相同的查询，我们现在将它们定义为常量：

```java
static final String SELECT_ALL = "select contacts.id c_id, name c_name, "
    + "phones.id p_id, type p_type, phones.phone p_phone "
    + "from contacts left join phones on contacts.id = phones.contact_id "
    + "order by c_name, p_type ";

static final String SELECT_ONE = SELECT_ALL + "where phones.id = :id";
```

请注意，我们提供了别名（例如`c_id`、`p_id`）来区分不同表中的相同名称（`id`）的列。

Jdbi 提供了一些不同的 API 来处理连接数据。


##### ResultBearing.reduceRows()

[ResultBearing.reduceRows(U, BiFunction)](apidocs/org/jdbi/v3/core/result/ResultBearing.html#reduceRows-U-java.util.function.BiFunction-) 方法接受一个累加器种子值和一个 lambda 函数。 对于结果集中的每一行，Jdbi 使用当前累加器值和结果集当前行上的 [RowView](apidocs/org/jdbi/v3/core/result/RowView.html) 调用 lambda。 每行返回的值成为为下一行传入的输入累加器。 在处理完最后一行后，`reducedRows()` 返回从 lambda 返回的最后一个值。

```java
List<Contact> contacts = handle.createQuery(SELECT_ALL)
    .registerRowMapper(BeanMapper.factory(Contact.class, "c"))
    .registerRowMapper(BeanMapper.factory(Phone.class, "p")) //<1>
    .reduceRows(new LinkedHashMap<Long, Contact>(), //<2>
                (map, rowView) -> {
      Contact contact = map.computeIfAbsent( //<3>
          rowView.getColumn("c_id", Long.class),
          id -> rowView.getRow(Contact.class));

      if (rowView.getColumn("p_id", Long.class) != null) { //<4>
        contact.addPhone(rowView.getRow(Phone.class));
      }

      return map; //<5>
    })
    .values() //<6>
    .stream()
    .collect(Collectors.toList()); //<7>
```

> **<1>** 为`Contact` 和 `Phone`注册行映射器。 注意使用的 `"c"` 和 `"p"` 参数——这些是列名前缀。 通过使用前缀注册映射器，`Contact`映射器将只映射`c_id`和`c_name`列，而`Phone`映射器将仅映射`p_id`、`p_type`和`p_phone`。
> **<2>** 使用一个空的[LinkedHashMap](https://docs.oracle.com/javase/8/docs/api/java/util/LinkedHashMap.html)作为累加器种子，按联系人ID映射。当选择多个主记录时，`LinkedHashMap`是一个很好的累加器，因为它有快速的存储和查找，同时保留插入顺序(这有助于尊重`order BY`子句)。如果排序不重要，那么HashMap也足够了。
> **<3>** 如果我们已经有了它，则从累加器中加载`ontact`； 否则，通过`RowView`初始化它。
> **<4>** 如果 `p_id` 列不为空，则从当前行加载电话号码并将其添加到当前联系人。
> **<5>** 返回输入map（现在有一个额外的联系人和/或电话）作为下一行的累加器。
> **<6>** 此时，所有行都已读入内存，我们不需要联系人 ID 键。 所以我们调用`Map.values()`来获得一个`Collection<Contact>`。
> **<7>** 将联系人收集到一个 `List<Contact>` 中。

或者，[ResultBearing.reduceRows(RowReducer)](apidocs/org/jdbi/v3/core/result/ResultBearing.html#reduceRows-org.jdbi.v3.core.result.RowReducer-) 接受一个 [RowReducer](apidocs/org/jdbi/v3/core/result/RowReducer.html) 并返回一个被简化的元素流。

对于简单的主从连接，[ResultBearing.reduceRows(BiConsumer,RowView>)](apidocs/org/jdbi/v3/core/result/ResultBearing.html#reduceRows-java.util.function.BiConsumer-) 方法可以轻松地将这些连接简化为主元素流。

修改上面的例子：

```java
List<Contact> contacts = handle.createQuery(SELECT_ALL)
    .registerRowMapper(BeanMapper.factory(Contact.class, "c"))
    .registerRowMapper(BeanMapper.factory(Phone.class, "p"))
    .reduceRows((Map<Long, Contact> map, RowView rowView) -> { //<1>
      Contact contact = map.computeIfAbsent(
          rowView.getColumn("c_id", Long.class),
          id -> rowView.getRow(Contact.class));

      if (rowView.getColumn("p_id", Long.class) != null) {
        contact.addPhone(rowView.getRow(Phone.class));
      }
      //<2>
    })
    .collect(Collectors.toList()); //<3>
```

> **<1>** lambda接收一个map，其中结果对象将被存储，和一个`RowView`。该map是一个`LinkedHashMap`，因此结果流将以插入结果对象的相同顺序生成结果对象。
> **<2>** 不需要 `return` 语句。 在每一行上重复使用相同的 `map`。
> **<3>** 这个`reduceRows()`调用产生一个`Stream<contact>`(即来自`map.values(). Stream()`)。</contact>在这个例子中，我们将元素收集到一个列表中，但是我们可以在这里调用任何`Stream`方法。

你可能想知道 `getRow()` 和 `getColumn()` 对 `rowView` 的调用。 当你调用 `rowView.getRow(SomeType.class)` 时，`RowView` 会为 `SomeType` 查找注册的行映射器，并使用它来将当前行映射到一个 `SomeType` 对象。

同样，当你调用 `rowView.getColumn("my_value", MyValueType.class)` 时，`RowView` 会为 `MyValueType` 查找注册的列映射器，并使用它来将当前行的 `my_value` 列映射到一个 `MyValueType` 对象。

现在让我们做同样的事情，但对于单个 contact：

```java
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

##### ResultBearing.reduceResultSet()

[ResultBearing.reduceResultSet()](apidocs/org/jdbi/v3/core/result/ResultBearing.html#reduceResultSet-U-org.jdbi.v3.core.result.ResultSetAccumulator-) 是一个类似于` reduceRows()`，除了它提供对 JDBC `ResultSet` 的直接访问，而不是每行的 `RowView`。

与“reduceRows()”相比，此方法可以提供更出色的性能，但代价是冗长：

```java
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

##### JoinRowMapper(连接行映射器)

`JoinRowMapper` 需要从每一行中提取一组类型。 它使用映射注册表来确定如何映射每个给定类型，并向您提供一个 `JoinRow`，其中包含所有结果值。

让我们考虑两个简单的类型，User 和 Article，有一个名为 Author 的连接表。 Guava 提供了一个 Multimap 类，它对于表示这样的连接表非常方便。 假设我们已经注册了映射器：

```java
h.registerRowMapper(ConstructorMapper.factory(User.class));
h.registerRowMapper(ConstructorMapper.factory(Article.class));
```

然后，我们可以轻松地用数据库中的映射填充Multimap:

```java
Multimap<User, Article> joined = HashMultimap.create();
h.createQuery("SELECT * FROM user NATURAL JOIN author NATURAL JOIN article")
    .map(JoinRowMapper.forTypes(User.class, Article.class))
    .forEach(jr -> joined.put(jr.get(User.class), jr.get(Article.class)));
```
> **💡提示:** `翻译者WJW`提示: NATURAL JOIN即自然连接，`natural join`等同于`inner join`或`inner using`，其作用是将两个表中具有相同名称的列进行匹配.

> **🏷注意:** 虽然这种方法易于读写，但对于某些数据模式可能效率低下。 在决定是使用高级映射还是使用手写映射器进行更直接的低级访问时，请考虑性能要求。

您还可以将它与 SqlObject 一起使用：

```java
public interface UserArticleDao {
    @RegisterJoinRowMapper({User.class, Article.class})
    @SqlQuery("SELECT * FROM user NATURAL JOIN author NATURAL JOIN article")
    Stream<JoinRow> getAuthorship();
}
```

```java
Multimap<User, Article> joined = HashMultimap.create();

handle.attach(UserArticleDao.class)
        .getAuthorship()
        .forEach(jr -> joined.put(jr.get(User.class), jr.get(Article.class)));

assertThat(joined).isEqualTo(JoinRowMapperTest.getExpected());
```

### [3.10. Updates(更新)](#_updates)

更新是返回整数行修改的操作，例如数据库 **INSERT**、**UPDATE** 或 **DELETE**。

您可以使用`Handle` 的`int execute(String sql, Object... args)` 方法执行简单的更新，该方法绑定了简单的位置参数。

```java
count = handle.execute("INSERT INTO user(id, name) VALUES(?, ?)", 4, "Alice");
assertThat(count).isEqualTo(1);
```

要进一步自定义，请使用 `createUpdate`：

```java
int count = handle.createUpdate("INSERT INTO user(id, name) VALUES(:id, :name)")
    .bind("id", 3)
    .bind("name", "Charlie")
    .execute();
assertThat(count).isEqualTo(1);
```

更新可能返回[Generated Keys](#58____3_12__Generated_Keys)而不是一个结果计数。

### [3.11. Batches(批处理)](#_batches)

**Batch** 向服务器批量发送许多命令。

打开批处理后，重复添加语句，并调用**add**。

```java
Batch batch = handle.createBatch();

batch.add("INSERT INTO fruit VALUES(0, 'apple')");
batch.add("INSERT INTO fruit VALUES(1, 'banana')");

int[] rowsModified = batch.execute();
```

语句被批量发送到数据库，但每个语句是单独执行的。 没有参数。 每个语句都返回一个修改计数，就像更新一样，然后这些计数在一个 `int[]` 数组中返回。 在常见情况下，所有元素都将为“1”。


### [3.12. Prepared Batches(准备好了的批处理)](#_prepared_batches)

**PreparedBatch** 向服务器发送一个带有多个参数集的语句。 该语句被重复执行，每批 **添加** 的参数执行一次。

结果仍然是修改后的行数的`int[]`。

```java
PreparedBatch batch = handle.prepareBatch("INSERT INTO user(id, name) VALUES(:id, :name)");
for (int i = 100; i < 5000; i++) {
    batch.bind("id", i).bind("name", "User:" + i).add();
}
int[] counts = batch.execute();
```

SqlObject 也支持批量插入：

```java
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

> **💡提示:** 与重复执行单条语句相比，批处理显着提高了效率，但许多数据库也不能很好地处理非常大的批处理。 使用您的数据库配置进行测试，但通常应将极大的数据集分割并提交——否则可能会使您的数据库瘫痪。

#### [3.12.1. Exception Rewriting(异常重写)](#_exception_rewriting)

`JDBC SQLException` 类非常古老并且比更现代的异常工具如 Throwable 的抑制异常早。 当一个批次失败时，可能会报告多个失败，这无法用当天的基本异常类型来表示。

所以 SQLException 有一个定制的 [getNextException](https://docs.oracle.com/javase/8/docs/api/java/sql/SQLException.html#getNextException--) 链来表示批处理失败的原因。 不幸的是，默认情况下，大多数日志库不会打印出这些异常，而是将它们的处理推入您的代码中。 忘记处理这种情况并最终得到的日志除了

```log
java.sql.BatchUpdateException: Batch entry 1 insert into something (id, name) values (0, '') was aborted. Call getNextException to see the cause.
```

**jdbi** 将尝试将此类 nextExceptions 重写为“被抑制的异常”（Java 8 中的新功能），以便您的日志更有帮助：

```log
java.sql.BatchUpdateException: Batch entry 1 insert into something (id, name) values (0, 'Keith') was aborted. Call getNextException to see the cause.
Suppressed: org.postgresql.util.PSQLException: ERROR: duplicate key value violates unique constraint "something_pkey"
  Detail: Key (id)=(0) already exists.
```

### [3.13. Generated Keys(生成的键)](#_generated_keys)

Update 或 PreparedBatch 可以自动生成键。 这些键与正常结果分开处理。 根据您的数据库和配置，整个插入行可能可用。

> **☢警告:** 不幸的是，支持该特性的数据库之间有很多差异，所以请彻底测试该特性与数据库的交互。

在 PostgreSQL 中，整行都可用，因此您可以立即将插入的名称映射回完整的 User 对象！ 这避免了插入完成后单独查询的开销。

考虑下表：

```java
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

您可以以fluent的方式获取生成的键：

```java
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

### [3.14. Stored Procedure Calls(存储过程调用)](#_stored_procedure_calls)

**Call** 调用数据库存储过程。

让我们假设一个现有的存储过程作为例子：

```sql
CREATE FUNCTION add(a IN INT, b IN INT, sum OUT INT) AS $$
BEGIN
  sum := a + b;
END;
$$ LANGUAGE plpgsql
```

下面是调用存储过程的方法：

```java
OutParameters result = handle
        .createCall("{:sum = call add(:a, :b)}") //<1>
        .bind("a", 13) //<2>
        .bind("b", 9) //<2>
        .registerOutParameter("sum", Types.INTEGER)   //<3> <4>
        .invoke(); //<5>
```

> **<1>** 使用 SQL 语句调用 `Handle.createCall()`。 请注意，JDBC 在调用存储过程时具有特殊的 SQL 格式，我们必须遵循该格式。
> **<2>** 将输入参数绑定到过程调用。
> **<3>** 注册输出参数，即将从存储过程调用返回的值。 这告诉 JDBC 期望每个输出参数的数据类型。
> **<4>** 如果 SQL 使用位置参数，则输出参数可以按名称（如示例所示）或从零开始的索引进行注册。 可以注册多个输出参数，具体取决于存储过程本身的输出。
> **<5>** 最后，调用 `invoke()` 来执行该过程。

调用存储过程会返回一个 [OutParameters](apidocs/org/jdbi/v3/core/statement/OutParameters.html) 对象，其中包含从存储过程调用返回的值。

现在我们可以从 `OutParameters` 中提取结果：

```java
int sum = result.getInt("sum");
```

通过将打开的游标声明为`Types.REF_CURSOR`，然后通过`OutParameters.getRowSet()`检查它，可以将打开的游标作为类似结果的对象返回。 通常这必须在事务中完成，并且必须在关闭语句之前通过使用 `Call.invoke(Consumer)` 或 `Call.invoke(Function)` 回调样式处理它来消耗结果。

> **☢警告:** 由于 JDBC 中的设计限制，通过 `OutParameters` 可用的参数数据类型仅限于 JDBC 直接支持的那些类型。 这不能通过例如扩展 映射器注册。

### [3.15. Scripts(脚本)](#_scripts)

**Script** 将 String 解析为分号终止的语句。 这些语句可以在单个 **Batch** 中执行，也可以单独执行。

```java
int[] results = handle.createScript(
        "INSERT INTO user VALUES(3, 'Charlie');"
        + "UPDATE user SET name='Bobby Tables' WHERE id=2;")
    .execute();

assertThat(results).containsExactly(1, 1);
```

### [3.16. Transactions(事务)](#_transactions)

**jdbi** 完全支持 JDBC 事务。

**Handle** 对象提供了两种开启事务的方式 —— **inTransaction** 允许你返回结果，而**useTransaction** 没有返回值。

两者都允许您选择指定事务隔离级别。

```java
public Optional<User> findUserById(long id) {
    return handle.inTransaction(h ->
            h.createQuery("SELECT * FROM users WHERE id=:id")
                    .bind("id", id)
                    .mapTo(User.class)
                    .findFirst());
}
```

在这里，我们（可能是不必要地）用事务保护一个简单的 *SELECT* 语句。

此外，Handle 有许多用于直接事务管理的方法：begin()、savepoint()、rollback()、commit() 等。通常，您不需要使用这些方法。 如果您没有明确提交手动打开的事务，它将被回滚。


#### [3.16.1. Serializable Transactions(可序列化事务)](#_serializable_transactions)

对于更高级的查询，有时需要可序列化的事务。 **jdbi** 包括一个事务运行器，它能够重试由于序列化失败而中止的事务。 重要的是您的事务没有副作用，因为它可能会被执行多次。

```java
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

上面的测试旨在在锁定步骤中运行两个事务。 每个尝试读取表中所有行的总和，然后插入具有该总和的新行。 我们用值 10 和 20 为表设置种子。

如果没有可序列化隔离，每个事务读取10和20，然后返回30。最终结果是30 + 30 = 60，这并不对应于事务的任何串行执行!

使用可序列化隔离，两个事务中的一个将被迫中止并重试。在第二次循环中，它计算出10 + 20 + 30 = 60。加上另一个的30，我们得到30 + 60 = 90，断言成功。

### [3.17. ClasspathSqlLocator(类路径SqlLocator)](#_metadata)

您可能会发现将 SQL 模板存储在类路径上的单个文件中而不是 Java 代码中的字符串中很有帮助。

The `ClasspathSqlLocator` converts Java type and method names into classpath locations, and then reads, parses, and caches the loaded statements.

```java
// reads classpath resource com/foo/BarDao/query.sql
ClasspathSqlLocator.findSqlOnClasspath(com.foo.BarDao.class, "query");

// same resource as above
ClasspathSqlLocator.findSqlOnClasspath("com.foo.BarDao.query");
```

## [4. Configuration(配置)](#_configuration)

`Jdbi` 旨在以最少的配置开箱即用。 有时您需要更改默认行为，或添加扩展以处理其他数据库类型。每一个希望参与配置的核心或扩展都定义了一个配置类，例如`SqlStatements`类存储了SqlStatement相关的配置。 然后，在任何`Configurable`上下文（如`Jdbi` 或 `Handle`）上，您都可以以类型安全的方式更改配置：

```java
jdbi.getConfig(SqlStatements.class).setUnusedBindingAllowed(true);
jdbi.getConfig(Arguments.class).register(new MyTypeArgumentFactory());
jdbi.getConfig(Handles.class).setForceEndTransactions(true);

// 或者，如果您有很多工作要做：
jdbi.configure(RowMappers.class, rm -> {
    rm.register(new TypeARowMapperFactory();
    rm.register(new TypeBRowMapperFactory();
});
```

通常，您应该在与数据库交互之前完成所有配置更改。

创建新上下文时，它会在创建时继承父上下文配置的副本。 因此，`Handle` 从创建的 `Jdbi` 初始化其配置，但更改永远不会传播回来。

有关更高级的实现细节，请参阅 [JdbiConfig](#141____9_5__JdbiConfig)

### [4.1. Qualified Types(限定类型)](#_qualified_types)

有时，同一个 Java 对象可以对应数据库中的多种数据类型。 例如，`String` 可以是 `varchar` 纯文本、`nvarchar` 文本、`json` 数据等，所有这些都有不同的处理要求。

[QualifiedType](apidocs/org/jdbi/v3/core/qualifier/QualifiedType.html)允许您添加这样的上下文到Java类型:

```java
QualifiedType.of(String.class).with(Json.class);
```

这个 `QualifiedType` 仍然代表 `String` *类型*，但使用 `@Json` 注解进行限定。 它可以以类似于 [GenericType](#138_____9_3_1__GenericType) 的方式使用，使组件处理值（主要是 `ArgumentFactories` 和 `ColumnMapperFactories`）以不同的方式执行它们的工作，并让不同的实现完全处理这些值：

```java
@Json
public class JsonArgumentFactory extends AbstractArgumentFactory<String> {
    @Override
    protected Argument build(String value, ConfigRegistry config) {
        // do something specifically for json data
    }
}
```

一旦注册，这个`@Json` 限定工厂将只接收`@Json String` 值。 其他不限定的工厂将不会收到此值：

```java
QualifiedType<String> json = QualifiedType.of(String.class).with(Json.class);
query.bindByType("jsonValue", "{\"foo\":1}", json);
```

> **💡提示:** Jdbi通过**精确匹配**它们的**限定符**来选择工厂来处理值。这取决于工厂实现是否区分值的*type*。

> **🏷注意:** 限定符实现为“注解”。 这允许工厂在源头独立检查限定符的值，例如在他们的“类”上，以改变他们自己的行为或*重新限定*一个值并让它由 Jdbi 的查找链重新评估。

> **☢警告:** 限定符是注释**并不**意味着它们在放置在源类中时会固有地激活它们的功能。 每个功能都有自己的使用规则。

> **⚠小心:** 参数只能通过`bindByType` 调用进行绑定，而不是常规的`bind` 或`update.execute(Object...)`。 此外，数组不能被限定。

这些功能目前使用限定类型：

- `@NVarchar` 和 `@MacAddr`（后者在 `jdbi3-postgres` 中）分别将字符串绑定和映射为 `nvarchar` 和 `macaddr`，而不是通常的 `varchar`。
- `jdbi3-postgres` 提供 [HStore](#_hstore).
- [JSON](#jdbi3-json)
- `BeanMapper`、`@BindBean`、`@RegisterBeanMapper`、`mapTobean()` 和 `bindBean()` 尊重 getter、setter 和 setter 参数的限定符。
- `ConstructorMapper` 和 `@RegisterConstructorMapper` 尊重构造函数参数的限定符。
- `@BindMethods` 和 `bindMethods()` 尊重方法的限定符。
- `@BindFields`、`@RegisterFieldMapper`、`FieldMapper` 和 `bindFields()` 尊重字段的限定符。
- `SqlObject` 尊重方法的限定符（将它们应用于返回类型）和参数。
  - 在 `Consumer<T>` 类型的参数上，限定符应用于 `T`。
- `@MapTo`
- `@BindJpa` 和 `JpaMapper` 尊重 getter 和 setter 的限定符。
- `@BindKotlin`、`bindKotlin()` 和 `KotlinMapper` 尊重构造函数参数、getter、setter 和 setter 参数的限定符。

## [5. SQL Objects(SQL对象)](#_sql_objects)

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

### [5.1. Annotated Methods(注解方法)](#_annotated_methods)

使用Jdbi的SQL方法注解 ([@SqlBatch](apidocs/org/jdbi/v3/sqlobject/statement/SqlBatch.html), [@SqlCall](apidocs/org/jdbi/v3/sqlobject/statement/SqlCall.html), [@SqlQuery](apidocs/org/jdbi/v3/sqlobject/statement/SqlQuery.html), or [@SqlUpdate](apidocs/org/jdbi/v3/sqlobject/statement/SqlUpdate.html))注解的方法将基于方法上的注解及其参数自动生成实现。方法的参数用作语句的参数，SQL语句结果映射到方法返回类型。

#### [5.1.1. @SqlUpdate](#_sqlupdate)

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

#### [5.1.2. 绑定参数](#_binding_arguments_2)

在我们继续使用其他 `@Sql__` 注解之前，让我们讨论如何将方法参数作为参数绑定到 SQL 语句。

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

绑定值列表是通过`@BindList` 注解完成的。 这将以“a,b,c,d,...”形式展开列表。 请注意，此注解要求您使用 `<绑定>` 符号，这与 `@Bind`（使用 `:绑定`）不同：

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

#### [5.1.3. @SqlQuery](#_sqlquery)

使用 `@SqlQuery` 注解进行选择操作。

查询方法可能返回单行或多行结果，具体取决于方法返回类型是否类似于集合。

```java
public interface UserDao {
  @SqlQuery("select name from users")
  List<String> listNames(); //<1>

  @SqlQuery("select name from users where id = ?")
  String getName(long id);   //<2> <3>

  @SqlQuery("select name from users where id = ?")
  Optional<String> findName(long id); //<4>
}
```

> **<1>** 当多行方法返回空结果集时，将返回空集合。
> **<2>** 如果单行方法从查询中返回多行，则该方法只返回结果集中的第一行。
> **<3>** 如果单行方法返回空结果集，则返回 `null`。
> **<4>** 方法可能返回“可选”值。 如果查询没有返回任何行（或者如果行中的值为 null），则返回 `Optional.empty()` 而不是 null。 如果查询返回多于一行，SQL 对象将引发异常。

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
public class UserMapper implements RowMapper<User> {   //<1> <2>
  public UserMapper() { //<3>
    ...
  }

  public T map(ResultSet rs, StatementContext ctx) throws SQLException {
    ...
  }
}
```

> **<1>** 必须是一个公共类。
> **<2>** 必须使用显式类型参数（例如，`RowMapper<User>`）而不是类型变量（例如`RowMapper<T>`）来实现`RowMapper`。
> **<3>** 必须有一个公共的、无参数的构造函数（或一个默认构造函数）。

> **💡提示:** `@RegisterRowMapper` 注解可以在同一类型或方法上重复多次以注册多个映射器。

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
public class UserMapperFactory implements RowMapperFactory { //<1>
  public UserMapperFactory() { //<2>
    ...
  }

  public Optional<RowMapper<?>> build(Type type, ConfigRegistry config) {
    ...
  }
}
```

> **<1>** 必须是一个公共类。
> **<2>** 必须有一个公共的、无参数的构造函数（或一个默认构造函数）。

> **💡提示:** `@RegisterRowMapperFactory` 注解可以在同一类型或方法上重复多次以注册多个工厂。

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
public class MoneyMapper implements ColumnMapper<Money> {   //<1> <2>
  public MoneyMapper() { //<3>
    ...
  }

  public T map(ResultSet r, int columnNumber, StatementContext ctx) throws SQLException {
    ...
  }
}
```

> **<1>** 必须是一个公共类。
> **<2>** 必须使用显式类型参数（例如 `ColumnMapper<User>`）而不是类型变量（例如 `ColumnMapper<T>`）来实现 `ColumnMapper`。
> **<3>** 必须有一个公共的、无参数的构造函数（或一个默认构造函数）。

> **💡提示:** `@RegisterColumnMapper` 注解可以在同一类型或方法上重复多次以注册多个映射器。

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
public class UserMapperFactory implements ColumnMapperFactory { //<1>
  public UserMapperFactory() { //<2>
    ...
  }

  public Optional<ColumnMapper<?>> build(Type type, ConfigRegistry config) {
    ...
  }
}
```

> **<1>** 必须是一个公共类。
> **<2>** 必须有一个公共的、无参数的构造函数（或一个默认构造函数）。

> **💡提示:** `@RegisterColumnMapperFactory` 注解可以在同一类型或方法上重复多次以注册多个工厂。

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

##### @SingleValue

有时，在使用诸如数组之类的高级 SQL 功能时，诸如 `int[]` 或 `List<Integer>` 之类的容器类型可能会含糊不清地表示“单个 SQL int[]”或“一个 ResultSet of int”。

由于数组在规范化模式中不常用，因此 SQL 对象默认假定您将 **ResultSet(表示数据库结果集的当前行)** 收集到容器对象中。 您可以将返回类型注解为 `@SingleValue` 以覆盖它。

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

通常，Jdbi 会将 `List<String>` 解释为表示映射类型为 `String`，并将所有结果行收集到一个列表中。 `@SingleValue` 注解导致 Jdbi 将 `List<String>` 视为映射类型。

> **🏷注意:** `@SingleValue Optional<String>` 很诱人，但通常不需要。 `Optional` 被实现为一个包含零个或一个元素的容器。 添加`@SingleValue` 意味着数据库本身有一个类似`optional<varchar>` 类型的列。


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

首先，按照 [Google Guava](#102____7_1__Google_Guava) 部分中的说明安装 `GuavaPlugin`。

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

默认情况下，SQL 对象将`Map` 返回类型视为`Map.Entry` 值的集合。 使用 `@SingleValue` 注解覆盖它，以便将返回类型视为单个值而不是集合：

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


##### @UseRowReducer

`@SqlQuery` 方法使用连接查询可以将主从连接减少到一个或多个主级对象。 请参阅 [ResultBearing.reduceRows()](#51______ResultBearing_reduceRows__) 以了解行减行器的介绍。

考虑一个包含文件夹和文档的文件系统比喻。 在连接中，我们将使用 `f_` 作为文件夹列的前缀，并使用 `d_` 作为文档列的前缀。

```java
@RegisterBeanMapper(value = Folder.class, prefix = "f") //<1>
@RegisterBeanMapper(value = Document.class, prefix = "d")
public interface DocumentDao {
    @SqlQuery("select " +
            "f.id f_id, f.name f_name, " +
            "d.id d_id, d.name d_name, d.contents d_contents " +
            "from folders f left join documents d " +
            "on f.id = d.folder_id " +
            "where f.id = :folderId" +
            "order by d.name")
    @UseRowReducer(FolderDocReducer.class) //<2>
    Optional<Folder> getFolder(int folderId); //<3>

    @SqlQuery("select " +
            "f.id f_id, f.name f_name, " +
            "d.id d_id, d.name d_name, d.contents d_contents " +
            "from folders f left join documents d " +
            "on f.id = d.folder_id " +
            "order by f.name, d.name")
    @UseRowReducer(FolderDocReducer.class) //<2>
    List<Folder> listFolders(); //<3>

    class FolderDocReducer implements LinkedHashMapRowReducer<Integer, Folder> { //<4>
        @Override
        public void accumulate(Map<Integer, Folder> map, RowView rowView) {
            Folder f = map.computeIfAbsent(rowView.getColumn("f_id", Integer.class), //<5>
                                           id -> rowView.getRow(Folder.class));

            if (rowView.getColumn("d_id", Integer.class) != null) { //<6>
                f.getDocuments().add(rowView.getRow(Document.class));
            }
        }
    }
}
```

> **<1>** 在此示例中，我们使用前缀注册文件夹和文档映射器，以便每个映射器仅查看具有该前缀的列。 这些映射器由 `getRow(Folder.class)` 和 `getRow(Document.class)` 调用中的行缩减器间接使用。
> **<2>** 用`@UseRowReducer`注解该方法，并指定`RowReducer`实现类。
> **<3>** 同样的' RowReducer '实现可以用于单主记录和多主记录查询。
> **<4>** [LinkedHashMapRowReducer](apidocs/org/jdbi/v3/core/result/LinkedHashMapRowReducer.html) 是一个抽象的`RowReducer` 实现，它使用`LinkedHashMap` 作为结果容器，并返回`values()` 集合作为结果。
> **<5>** 通过 ID 从map中获取此行的`Folder`，如果不在map中，则创建它。
> **<6>** 在映射文档并将其添加到文件夹之前，确认该行有一个文档（这是左联接）。

#### [5.1.4. @SqlBatch](#_sqlbatch)

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
  void bulkInsert(@Bind("tenantId") long tenantId, //<1>
                  @BindBean("user") User... users);
}
```

> **<1>** 使用相同的`tenant_id`插入每个用户记录。


> **☢警告:** `@SqlBatch` 方法必须至少有一个可迭代参数。

默认情况下，`@SqlBatch` 方法可能会返回一些类型：

- `void`: 不返回任何内容（显然）
- `int[]` 或者 `long[]`: 返回批处理中每次执行的更新计数。根据数据库供应商和JDBC驱动程序，这可能是语句更改的行数，也可能是查询匹配的行数(不管是否更改了任何数据)。
- `boolean[]`: 如果更新计数大于零，则返回true，批处理中每次执行一个值。

##### @GetGeneratedKeys

与`@SqlUpdate` 类似，`@GetGeneratedKeys` 注解告诉 SQL 对象返回值应该是每个 SQL 语句生成的键，而不是更新计数。 有关更深入的讨论，请参阅 [@GetGeneratedKeys](#__getgeneratedkeys)。

```java
public interface UserDao {
  @SqlBatch("insert into users (id, name) values (nextval('user_seq'), ?)")
  @GetGeneratedKeys("id")
  long[] bulkInsert(List<String> names); //<1>
}
```

> **<1>** 为每个插入的名称返回生成的 ID。

可以通过这种方式生成和返回多个列：

```java
public interface UserDao {
  @SqlBatch("insert into users (id, name, created_on) values (nextval('user_seq'), ?, now())")
  @GetGeneratedKeys({"id", "created_on"})
  @RegisterBeanMapper(IdCreateTime.class)
  List<IdCreateTime> bulkInsert(String... names);
}
```

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

#### [5.1.5. @SqlCall](#_sqlcall)

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

#### [5.1.6. @SqlScript](#_sqlscript)

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

#### [5.1.7. @GetGeneratedKeys](#_getgeneratedkeys_3)

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

#### [5.1.8. SqlLocator](#_sqllocator)

当 SQL 语句变得越来越复杂时，在 `@Sql__` 方法注解中将语句作为 Java 字符串提供可能会很麻烦。

Jdbi提供注解，允许您配置外部位置以加载SQL语句。

- @UseAnnotationSqlLocator (默认的行为;使用@Sql__(…)注解值)
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

#### [5.1.9. @CreateSqlObject](#_createsqlobject)

使用@CreateSqlObject注解在另一个SqlObject中重用一个SqlObject。例如，您可以构建一个事务方法，该方法执行在其他SqlObject中定义的SQL更新，作为事务的一部分。Jdbi不会为对子SqlObject的调用打开新的句柄。

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

#### [5.1.10. @Timestamped](#_timestamped)

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

### [5.2. Consumer Methods](#_consumer_methods)

作为一种特殊情况，除了其他绑定参数之外，您还可以额外在最后提供一个 `Consumer<T>` 参数。 提供的使用者对结果集中的每一行执行一次。 参数 T 的静态类型决定了行类型。

```java
@SqlQuery("select id, name from users")
void forEachUser(Consumer<User> consumer);
```

### [5.3. Default Methods](#_default_methods)

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

<a name="_transaction_management"></a>
### [5.4. Transaction Management](#_transaction_management)

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

### [5.5. Using SQL Objects(使用 SQL 对象)](#_using_sql_objects)

定义接口后，有几种方法可以获取它的实例：

#### [5.5.1. Attached to Handle(附加到Handle)](#_attached_to_handle)

您可以获得附加到打开Handle的 SQL 对象。

```java
try (Handle handle = jdbi.open()) {
  ContactPhoneDao dao = handle.attach(ContactPhoneDao.class);
  dao.insertFullContact(contact);
}
```

附加的 `SQL对象`与句柄具有相同的生命周期——当句柄关闭时，`SQL对象`将变得不可用。

#### [5.5.2. Temporary SQL Objects(临时SQL对象)](#_temporary_sql_objects)

还可以通过传递回调(通常是lambda)，从Jdbi对象获得临时SQL对象。 使用[Jdbi.withExtension](apidocs/org/jdbi/v3/core/Jdbi.html#withExtension-java.lang.Class-org.jdbi.v3.core.extension.ExtensionCallback-)操作返回结果
, 或者[useExtension](apidocs/org/jdbi/v3/core/Jdbi.html#useExtension-java.lang.Class-org.jdbi.v3.core.extension.ExtensionConsumer-)用于没有结果的操作。

```java
jdbi.useExtension(ContactPhoneDao.class, dao -> dao.insertFullContact(alice));
long bobId = jdbi.withExtension(ContactPhoneDao.class, dao -> dao.insertFullContact(bob));
```

临时 `SQL对象` 仅在传递给方法的回调中有效。 当回调返回时，`SQL对象`（和关联的临时句柄）将关闭。


#### [5.5.3. On-Demand(按需)](#_on_demand)

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
> 例如:  `public interface StoreDetailInfoDao extends Transactional<StoreDetailInfoDao> {`


接口的 `default` 方法，以及[SqlObject](apidocs/org/jdbi/v3/sqlobject/SqlObject.html) 和 [Transactional](apidocs/org/jdbi/v3/sqlobject/transaction/Transactional.html)等的混入(mix-ins)，允许您在按需句柄保持打开状态的情况下运行代码。 同一线程上的重入调用将收到相同的“句柄”。 当最外面的按需调用完成时，句柄将关闭。

> **☢警告:** 在最外层的按需调用之外返回类似游标的类型，例如 `Stream<T>` 或 `Iterable<T>` 不起作用。 由于`Handle`关闭，数据库游标被释放，读取将失败。

### [5.6. Additional Annotations](#_additional_annotations)

Jdbi provides dozens of annotations out of the box:

- [org.jdbi.v3.sqlobject.config](apidocs/org/jdbi/v3/sqlobject/config/package-summary.html) 为可以在`Jdbi` 或`Handle` 级别配置的事物提供注解。 这包括映射器和参数的注册，以及用于配置 SQL 语句呈现和解析。
- [org.jdbi.v3.sqlobject.customizer](apidocs/org/jdbi/v3/sqlobject/customizer/package-summary.html) 为绑定参数、定义属性和控制语句结果集的获取行为提供了注解。
- [org.jdbi.v3.jpa](apidocs/org/jdbi/v3/jpa/package-summary.html) 提供了`@BindJpa`注解，用于根据JPA`@Column`注解将属性绑定到列。
- [org.jdbi.v3.sqlobject.locator](apidocs/org/jdbi/v3/sqlobject/locator/package-summary.html) 提供注解，配置Jdbi从其他源加载SQL语句，例如类路径上的文件。
- [org.jdbi.v3.sqlobject.statement](apidocs/org/jdbi/v3/sqlobject/statement/package-summary.html) 提供了`@MapTo`注解，用于在调用方法时动态指定映射类型。
- [org.jdbi.v3.stringtemplate4](apidocs/org/jdbi/v3/stringtemplate4/package-summary.html) 提供配置 Jdbi 以从类路径上的 StringTemplate 4 `.stg` 文件加载 SQL 和/或使用 ST4 模板引擎解析 SQL 模板的注解。
- [org.jdbi.v3.sqlobject.transaction](apidocs/org/jdbi/v3/sqlobject/transaction/package-summary.html) 为 SQL 对象中的事务管理提供注解。 详见 [Transaction Management](#_transaction_management)。

Jdbi被设计为支持用户定义的注解。请参阅[自定义注解](#145____9_8__User_Defined_Annotations)以获得创建自己的注解的指南。

### 5.7. [Annotations and Inheritance(注解 和 继承)](#_additional_annotations)

SQL 对象从它们扩展的接口继承方法和注解：

```java
package com.app.dao;

@UseClasspathSqlLocator  //<1> <2>
public interface CrudDao<T, ID> {
  @SqlUpdate //<3>
  void insert(@BindBean T entity);

  @SqlQuery //<3>
  Optional<T> findById(ID id);

  @SqlQuery
  List<T> list();

  @SqlUpdate
  void update(@BindBean T entity);

  @SqlUpdate
  void deleteById(ID id);
}
```

> **<1>** 参见 [SqlLocator](#88_____5_1_8__SqlLocator).
> **<2>** 类注解由子类型继承。
> **<3>** 方法和参数注解由子类型继承，除非子类型覆盖了方法。

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

假设`Account` 使用`name()` 样式的访问器而不是`getName()`。 在这种情况下，我们希望 `AccountDao` 使用 `@BindMethods` 而不是 `@BindBean`。

让我们用正确的注解覆盖这些方法：

```java
package com.app.account;

@RegisterConstructorMapper(Account.class)
public interface AccountDao extends CrudDao<Account, UUID> {
  @Override
  @SqlUpdate //<1>
  void insert(@BindMethods Account entity);

  @Override
  @SqlUpdate //<1>
  void update(@BindMethods Account entity);
}
```

> **<1>** 方法注解不会在`override`上继承，因此必须复制想要保留的注解。

## [6. Testing(测试)](#_testing)

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

## [7. Third-Party Integration(第三方集成)](#_third_party_integration)

### [7.1. Google Guava(谷歌Guava)](#_google_guava)

这个插件增加了对以下类型的支持：

- `Optional<T>` - 注册一个参数和映射器。 对于注册映射器/参数工厂的任何包装类型`T`，支持`Optional`。
- 大多数 Guava 集合和Map类型 - 请参阅 [GuavaCollectors](apidocs/org/jdbi/v3/guava/GuavaCollectors.html) 以获取支持类型的完整列表。

要使用此插件，请添加 Maven 依赖项：

```xml
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-guava</artifactId>
</dependency>
```

然后将插件安装到你的 `Jdbi` 实例中：

```java
jdbi.installPlugin(new GuavaPlugin());
```

安装插件后，可以从 SQL 对象方法返回任何受支持的 Guava 集合类型：

```java
public interface UserDao {
    @SqlQuery("select * from users order by name")
    ImmutableList<User> list();

    @SqlQuery("select * from users where id = :id")
    com.google.common.base.Optional<User> getById(long id);
}
```

### [7.2. H2 Database(H2数据库)](#_h2_database)

该插件配置 Jdbi 以正确处理 H2 数据库中的 `integer[]` 和 `uuid[]` 数据类型。

这个插件包含在核心 jar 中（但将来可能会被提取到单独的模块中）。 通过将插件安装到您的`Jdbi`实例中来使用它：

```java
jdbi.installPlugin(new H2DatabasePlugin());
```

### [7.3. JSON](#jdbi3-json)

`jdbi3-json` 模块添加了一个 `@Json` 类型限定符，允许将任意 Java 对象作为 JSON 数据存储在数据库中。

不包括实际的 JSON（反）序列化代码。 为此，您必须安装一个支持插件（见下文）。

> **💡提示:** 支持插件将为您安装`JsonPlugin`。 您**无需**自行安装或直接包含 `jdbi3-json` 依赖项。

该功能已经在 H2 和 Sqlite 中使用 Postgres `json` 列和 `varchar` 列进行了测试。

#### [7.3.1. Jackson 2](#_jackson_2)

这个插件通过 Jackson 2 提供 JSON 支持。

```xml
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-jackson2</artifactId>
</dependency>
```

```java
jdbi.installPlugin(new Jackson2Plugin());
// 可选配置您的 ObjectMapper（推荐）
jdbi.getConfig(Jackson2Config.class).setMapper(myObjectMapper);

// 如果要过滤属性，现在对 Json 视图提供简单支持：
jdbi.getConfig(Jackson2Config.class).setView(ApiProperty.class);
```

#### [7.3.2. Gson 2](#_gson_2)

这个插件通过 Gson 2 提供 JSON 支持。

```xml
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-gson2</artifactId>
</dependency>
```

```java
jdbi.installPlugin(new Gson2Plugin());
// optional
jdbi.getConfig(Gson2Config.class).setGson(myGson);
```

#### [7.3.3. Moshi](#_moshi)

这个插件通过 Moshi 提供 JSON 支持。

```xml
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-moshi</artifactId>
</dependency>
```

```java
jdbi.installPlugin(new MoshiPlugin());
// optional
jdbi.getConfig(MoshiConfig.class).setMoshi(myMoshi);
```

#### [7.3.4. Operation(操作)](#_operation)

任何限定为 [@Json](apidocs/org/jdbi/v3/json/Json.html) 的绑定对象 - 除了 `String` - 将被 [registered](apidocs/org/jdbi/v3/json/JsonConfig .html) [JsonMapper](apidocs/org/jdbi/v3/json/JsonMapper.html) 并重新限定为 `@Json String`。 然后将调用相应限定的`ArgumentFactory`来存储 JSON 数据，从而允许为您的数据库实现特殊的 JSON 处理。 如果没有找到，则将使用纯字符串工厂，以将 JSON 处理为纯文本。

映射的工作方式相同，但反过来：限定为 `@Json T` 的输出类型将从 `@Json String` 或 `String` 列映射器 中获取，然后通过 `JsonMapper`传递。

> **💡提示:** 我们的 PostgresPlugin 提供了合格的工厂，可以将 `@Json String` 绑定/映射到/从 `json` 或 `jsonb` 类型的列。


#### [7.3.5. Usage(用法)](#_usage)

```java
handle.execute("create table myjsons (id serial not null, value json not null)");
```

SqlObject:

```java
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

使用 Fluent API，你可以在通常提供 `Class<T>` 或 `GenericType<T>` 的任何地方提供 `QualifiedType<T>`：

```java
QualifiedType<MyJson> qualifiedType = QualifiedType.of(MyJson.class).with(Json.class);

h.createUpdate("insert into myjsons(json) values(:json)")
    .bindByType("json", new MyJson(), qualifiedType)
    .execute();

MyJson result = h.createQuery("select json from myjsons")
    .mapTo(qualifiedType)
    .one();
```

### [7.4. Immutables(不可变的)](#_immutables)

[Immutables](https://immutables.github.io/) 是一个注解处理器，根据简单的接口描述生成值类型。 值类型自然地很好地映射到`Jdbi` 属性绑定和行映射。

> **☢警告:** 不可变支持仍处于试验阶段，尚不支持自定义命名方案。 我们确实支持可配置的 `get`、`is` 和 `set` 前缀。

只需通过安装插件并配置您的`Immutables`类型来告诉我们您的类型：

```java
jdbi.getConfig(JdbiImmutables.class).registerImmutable(MyValueType.class)
```

该配置既会注册适当的`RowMapper`，也会配置新的`bindPojo`(或`@BindPojo`)绑定器:

```java
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

### [7.5. Freebuilder](#_freebuilder)

[Freebuilder](https://https://freebuilder.inferred.org/) 是一个注解处理器，它根据简单的接口或抽象类描述生成值类型。 Jdbi 支持 Freebuilder 的方式与它支持 Immutables 的方式大致相同。

> **☢警告:** Freebuilder 支持仍处于试验阶段，可能不支持所有 Freebuilder 实现的功能。 我们支持 JavaBean 风格的 getter 和 setter 以及不带前缀的 getter 和 setter。

只需通过安装插件并配置您的`Freebuilder`类型来告诉我们您的 Freebuilder 类型：

```java
jdbi.getConfig(JdbiFreebuilder.class).registerFreebuilder(MyFreeBuilderType.class)
```

该配置既会注册适当的`RowMapper`，也会配置新的`bindPojo`(或`@BindPojo`)绑定器:

```java
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

### [7.6. JodaTime](#_jodatime)

这个插件增加了对使用 joda-time 的`DateTime` 类型的支持。

要使用此插件，请添加 Maven 依赖项：

```xml
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-jodatime2</artifactId>
</dependency>
```

然后将插件安装到你的 `Jdbi` 实例中：

```java
jdbi.installPlugin(new JodaTimePlugin());
```

### [7.7. JPA(Java持久化框架)](#_jpa)

使用JPA插件是欺骗你的老板让你尝试Jdbi的好方法。“没问题，老板，它已经支持JPA注解了，很简单!”

此插件为 JPA 实体注解的一小部分添加了映射支持：

- Entity
- MappedSuperclass
- Column

要使用此插件，请添加 Maven 依赖项：

```xml
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-jpa</artifactId>
</dependency>
```

然后将插件安装到你的 `Jdbi` 实例中：

```java
jdbi.installPlugin(new JpaPlugin());
```

老实说虽然. .只要扯掉绷带，切换到正确的Jdbi。

### [7.8. Kotlin](#_kotlin)

[Kotlin](https://kotlinlang.org/) 支持由 **jdbi3-kotlin** 和 **jdbi3-kotlin-sqlobject** 模块提供。

Kotlin API 文档：

- [jdbi3-kotlin](apidocs-kotlin/jdbi3-kotlin/index.html)
- [jdbi3-kotlin-sqlobject](apidocs-kotlin/jdbi3-kotlin-sqlobject/index.html)

#### [7.8.1. ResultSet mapping](#_resultset_mapping)

**jdbi3-kotlin** 插件添加到 Kotlin 数据类的映射。 它支持所有字段都存在于构造函数中的数据类以及具有可写属性的类。 构造函数中不存在的任何字段将在构造函数调用后设置。 映射器支持可为空类型。 如果参数类型不可为空且结果集中不存在该值，它还会在构造函数中使用默认参数值。

要使用此插件，请添加 Maven 依赖项：

```xml
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-kotlin</artifactId>
</dependency>
```

确保 Kotlin 编译器的 [JVM 目标版本](https://kotlinlang.org/docs/reference/using-maven.html#attributes-specific-for-jvm) 设置为至少 1.8：

```xml
<kotlin.compiler.jvmTarget>1.8</kotlin.compiler.jvmTarget>
```

然后将插件安装到你的 `Jdbi` 实例中：

```java
jdbi.installPlugin(KotlinPlugin());
```

Kotlin 映射器还支持允许显式指定属性或参数名称的`@ColumnName`注解，以及允许映射嵌套 Kotlin 对象的`@Nested`注解。

> **🏷注意:** 不要使用`@BindBean`， `bindBean()`和`@RegisterBeanMapper`，而是使用`@BindKotlin`， `bindKotlin()`和`KotlinMapper`'来修饰Kotlin类的构造器参数、getter、setter和setter参数。

> **🏷注意:** `@ColumnName` 注解仅在将 SQL 数据映射到 Java 对象时适用。 当绑定对象属性时（例如使用`bindBean()`），绑定属性名（`:id`）而不是列名（`:user_id`）。

如果你通过 `Jdbi.installPlugins()` 加载所有 Jdbi 插件，这个插件将被自动发现和注册。 否则，您可以使用 `Jdbi.installPlugin(KotlinPlugin())` 附加它。

测试类的一个例子：

```java
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

有两个扩展可以提供帮助：

- `<reified T : Any>ResultBearing.mapTo()`
- `<T : Any>ResultIterable<T>.useSequence(block: (Sequence<T>) → Unit)`

允许代码如下：

```java
val qry = handle.createQuery("select id, name from something where id = :id")
val things = qry.bind("id", brian.id).mapTo<Thing>.list()
```

以及使用自动关闭的序列:

```java
qryAll.mapTo<Thing>.useSequence {
    it.forEach(::println)
}
```

#### [7.8.2. SqlObject](#_sqlobject)

**jdbi3-kotlin-sqlobject** 插件通过名称为 SqlObjects 中的 Kotlin 方法添加了自动参数绑定以及对 Kotlin 默认方法的支持。

```xml
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-kotlin-sqlobject</artifactId>
</dependency>
```

然后将插件安装到你的 `Jdbi` 实例中：

```java
jdbi.installPlugin(KotlinSqlObjectPlugin());
```

参数绑定支持单个原始类型以及 Kotlin 或 JavaBean 样式对象作为参数（在绑定中引用为`:paramName.propertyName`）。 不再需要注解。

如果你通过 `Jdbi.installPlugins()` 加载所有 Jdbi 插件，这个插件将被自动发现和注册。 否则，您可以通过以下方式附加插件：`Jdbi.installPlugin(KotlinSqlObjectPlugin())`。

测试类的一个例子：

```java
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

### [7.9. Lombok](#_lombok)

Lombok是一个很好的工具，可以从POJO类中删除冗余样板代码。

```java
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

Lombok和Jdbi在开箱即用时表现得很好:

- 使用 `BeanMapper` 或者 `@RegisterBeanMapper` 来映射 `@Data` 类.
- 使用 `ConstructorMapper` 或者 `@RegisterConstructorMapper` 来映射 `@Value` 类.
- 使用 `bindBean()` 或者 `@BindBean` 来绑定 `@Data` 或者 `@Value` 类.

我们之所以这么说，主要是因为一旦您开始使用 Jdbi 注解（如“@Nested”、“@ColumnMapper”）或类型限定注解（如“@HStore”）来注解字段，就会出现问题。

- BeanMapper 在 getter、setter 或 setter 参数上查找这些注解。
- ConstructorMapper 在构造函数参数上查找它们。
- 默认情况下，Lombok 不会将它们移动到那里。

从Lombok 1.18.4版本开始，可以将Lombok配置为将指定的任何注解复制到生成的getter、setter、setter参数和构造函数参数。

在您的项目 src 树中创建一个文件 `lombok.config`（或编辑现有的），并为每个应该复制的注解类型添加一行，如下例所示：

```java
lombok.copyableAnnotations += org.jdbi.v3.core.mapper.Nested
lombok.copyableAnnotations += org.jdbi.v3.core.mapper.reflect.ColumnName
lombok.copyableAnnotations += org.jdbi.v3.postgres.HStore
```

### [7.10. Oracle 12](#_oracle_12)

该模块添加了对 Oracle `RETURNING` DML 表达式的支持。

要使用此功能，请添加 Maven 依赖项：

```xml
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-oracle12</artifactId>
</dependency>
```

然后，使用带有 `Update` 或 `PreparedBatch` 的 `OracleReturning` 类来获取返回的 DML。

### [7.11. PostgreSQL](#_postgresql)

**jdbi3-postgres** 插件提供了与 [PostgreSQL JDBC 驱动程序](https://jdbc.postgresql.org/) 的增强集成。

要使用此功能，请添加 Maven 依赖项：

```xml
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-postgres</artifactId>
</dependency>
```

然后将插件安装到您的`Jdbi`实例中。

```java
Jdbi jdbi = Jdbi.create("jdbc:postgresql://host:port/database")
                .installPlugin(new PostgresPlugin());
```

该插件为 Java 8 **java.time** 类型配置映射，如 **Instant** 或 **Duration**、**InetAddress**、**UUID**、类型枚举和 **hstore** .

它还为 `int`、`long`、`float`、`double`、`String` 和 `UUID` 配置 SQL 数组类型支持。

有关详尽列表，请参阅 [javadoc](apidocs/org/jdbi/v3/postgres/package-summary.html)。

> **🏷注意:** 一些 Postgres 操作符，例如 `?` 查询操作符，会与 `jdbi` 或 `JDBC` 特殊字符发生冲突。 在这种情况下，您可能需要将操作符转义到例如 `??` 或 `\:`。

#### [7.11.1. hstore](#_hstore)

Postgres 插件提供了一个 `hstore` 到 `Map<String, String>` 列映射器，反之亦然：

```java
Map<String, String> accountAttributes = handle
    .select("select attributes from account where id = ?", userId)
    .mapTo(new GenericType<Map<String, String>>() {})
    .one();
```

使用 `@HStore` 限定类型：

```java
QualifiedType<> HSTORE_MAP = QualifiedType.of(new GenericType<Map<String, String>>() {})
    .with(HStore.class);

Map<String, String> caps = handle.createUpdate("update account set attributes = :hstore")
    .bindByType("hstore", mapOfStrings, HSTORE_MAP)
    .execute();
```

默认情况下，SQL 对象将`Map` 返回类型视为`Map.Entry` 值的集合。 使用 `@SingleValue` 注解覆盖它，以便将返回类型视为单个值而不是集合：

```java
public interface AccountDao {
  @SqlQuery("select attributes from account where id = ?")
  @SingleValue
  Map<String, String> getAccountAttributes(long accountId);
}
```

> **🏷注意:** 安装插件的默认变体添加了来自和到 `hstore` Postgres 数据类型的原始 `Map` 类型的非限定映射。 在某些情况下，这会干扰Map的其他映射。 建议始终使用带有 `@HStore` 限定类型的变体。

为了避免绑定不合格的 Argument 和 ColumnMapper 绑定，请使用静态工厂方法安装插件：

```java
Jdbi jdbi = Jdbi.create("jdbc:postgresql://host:port/database")
                .installPlugin(PostgresPlugin.noUnqualifiedHstoreBindings());
```

#### [7.11.2. @GetGeneratedKeys](#_getgeneratedkeys_4)

在 Postgres 中，如果您在不命名任何列的情况下请求生成的键，`@GetGeneratedKeys` 可以返回整个修改后的行。

```java
public interface UserDao {
  @SqlUpdate("insert into users (id, name, created_on) values (nextval('user_seq'), ?, now())")
  @GetGeneratedKeys
  @RegisterBeanMapper(User.class)
  User insert(String name);
}
```

If a database operation modifies multiple rows (e.g. an update that will modify several rows), your method can return all the modified rows in a collection:

```java
public interface UserDao {
  @SqlUpdate("update users set active = false where id = any(?)")
  @GetGeneratedKeys
  @RegisterBeanMapper(User.class)
  List<User> deactivateUsers(long... userIds);
}
```

#### [7.11.3. Large Objects](#_large_objects)

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

### [7.12. Spring5](#_spring5)

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

  //<1>
  <bean id="db" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
    <property name="url" value="jdbc:h2:mem:testing"/>
  </bean>

  //<2>
  <bean id="transactionManager"
    class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="db"/>
  </bean>
  <tx:annotation-driven transaction-manager="transactionManager"/>

  //<3>
  <bean id="jdbi"
    class="org.jdbi.v3.spring4.JdbiFactoryBean">
    <property name="dataSource" ref="db"/>
  </bean>

  //<4>
  <bean id="service"
    class="com.example.service.MyService">
    <constructor-arg ref="jdbi"/>
  </bean>
</beans>
```

> **<1>** The SQL data source that Jdbi will connect to. In this example we use an H2 database, but it can be any JDBC-compatible database. 
> **<2>** Enable configuration of transactions via annotations.
> **<3>** Configure `JdbiFactoryBean` using the data source configured earlier.
> **<4>** Inject `Jdbi` into a service class. Alternatively, use standard JSR-330 `@Inject` annotations on the target class instead of configuring it in your `beans.xml`.

#### [7.12.1. Installing plugins](#_installing_plugins)

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

#### [7.12.2. Global Attributes](#_global_attributes)

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

### [7.13. SQLite](#_sqlite)

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

### [7.14. StringTemplate 4](#_stringtemplate_4)

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
> **☢警告:** Since StringTemplate by default uses the `<` character to mark ST expressions, you might need to escape some SQL: `String datePredSql = "<if(datePredicate)> <dateColumn> \\< :dateFilter <endif>"`

Alternatively, SQL templates can be loaded from StringTemplate group files on the classpath:

`com/foo/AccountDao.sql.stg`

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

### [7.15. Vavr](#_vavr)

Vavr 插件提供了 **Jdbi** 与 Vavr 函数库的深度集成：

- 支持服务器 Vavr 值类型的参数解析，例如 `Option<T>`、`Either<L, T>`、`Lazy<T>`、`Try<T>` 和 `Validation<T>`。 对于包装类型 `T` 注册了一个 Mapper映射器。
- 从查询中返回 Vavr 集合类型。 支持的是 `Seq<T>`、`Set<T>`、`Map<K, T>` 和 `Multimap<K, T>` 及其所有子类型。 可以收集到一个 `Traversable<T>`，在这种情况下，将返回一个 `List<T>`。 对于所有接口类型，将使用合理的默认实现（例如，`List<T>` 为 `Seq<T>`）。 此外，`Multimap<K, T>` 为 `Seq<T>` 作为默认值容器支持。
- 列可以映射到 Vavr 的 `Option<T>` 类型。
- **Jdbi** 的元组投影！ 耶！ Vavr 提供最多 8 个元组的元组。您可以映射查询结果，例如 到`Tuple3<Integer, String, Long>`。 如果您选择的列多于投影的数量，则将使用该索引之前的列。

要使用该插件，请添加一个 Maven 依赖项：

```XML
<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-vavr</artifactId>
</dependency>
```

目前支持和测试 Vavr >= 0.9.0。 该插件会提取受支持的 Vavr 版本并可以使用。 与其他插件一样：通过 `Jdbi` 实例安装或使用自动安装。

```java
jdbi.installPlugin(new VavrPlugin());
```

以下是上面列出的功能的一些使用示例：

```java
String query = "select * from users where :name is null or name = :name";
Option<String> param = Option.of("eric");

// will fetch first user with given name or first user with any name (Option.none)
return handle.createQuery(query)
        .bind("name", param)
        .mapToBean(User.class)
        .findFirst();
```

其中 `param` 可以是 `Option<T>`、`Either<L, T>`、`Lazy<T>`、`Try<T>` 或 `Validation<T>` 之一。 请注意，在这些类型的情况下，嵌套值必须是“存在”，否则使用`null`值（例如，用于`Either.Left`或`Validation.Invalid`）。

```java
handle.createQuery("select name from users")
        .collectInto(new GenericType<Seq<String>>() {});
```

这适用于所有支持的集合类型。 对于已经安装在 **Jdbi** 中的嵌套值行和列映射器，将被使用。 因此，如果该列可以为空，则以下内容将起作用并且可能有意义：

```java
handle.createQuery("select middle_name from users") // nulls incoming!
        .collectInto(new GenericType<Seq<Option<String>>>() {});
```

该插件将遵循为 `Map<K, T>` 和 `Multimap<K, T>` 返回类型配置的键和值列。 在下一个示例中，我们将按用户名作为key，这不一定是唯一的。

```java
Multimap<String, User> usersByName = handle.createQuery("select * from users")
        .setMapKeyColumn("name")
        .collectInto(new GenericType<Multimap<String, User>>() {});
```

最后但同样重要的是，我们现在可以将简单的查询投射到 Vavr 元组，如下所示：

```java
// given a 'tuples' table with t1 int, t2 varchar, t3 varchar, ...
List<Tuple3<Integer, String, String>> tupleProjection = handle
        .createQuery("select t1, t2, t3 from tuples")
        .mapTo(new GenericType<Tuple3<Integer, String, String>>() {})
        .list();
```

只要注册了行映射器，您还可以将复杂类型投影到元组中。

```java
// given that there are row mappers registered for both complex types
Tuple2<City, Address> tupleProjection = handle
        .createQuery("select cityname, zipcode, street, housenumber from " +
            "addresses where user_id = 1")
        .mapTo(new GenericType<Tuple2<City, Address>>() {})
        .one();
```

如果您想混合使用复杂类型和简单类型，我们也可以满足您的需求。 使用 `TupleMappers` 类，您可以配置您的投影。（事实上，您必须 - 阅读下文！）

```java
handle.configure(TupleMappers.class, c ->
        c.setColumn(2, "street").setColumn(3, "housenumber"));

Tuple3<City, String, Integer> result = handle
        .createQuery("select cityname, zipcode, street, housenumber from " +
             "addresses where user_id = 1")
        .mapTo(new GenericType<Tuple3<City, String, Integer>>() {})
        .one();
```

记住：

- 列的配置是基于 1 的，因为它们反映了元组的值（您可以通过例如 `._1` 进行查询）。
- 元组总是完全按列映射或完全通过行映射器映射。 如果您想混合行映射类型和单列映射，则必须正确配置“TupleMappers”，即必须为所有非行映射元组索引提供列配置！

## [8. Cookbook(烹饪书)](#_cookbook)

本节包括您可能喜欢用 `Jdbi` 做的各种事情的示例。

### [8.1. 简单的依赖注入](#_simple_dependency_injection)

`Jdbi`试图独立于使用依赖项注入框架，但它很容易集成您的框架中。只需在一个简单的自定义配置类型上进行字段注入:

```java
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

### [8.2. LIKE clauses with Parameters(带参数的 LIKE 子句)](#_like_clauses_with_parameters)

由于 JDBC（因此`Jdbi`）不允许将参数绑定到字符串文字的中间，你不能将绑定插入到`LIKE` 子句（`LIKE '%:param%'`）中。

Incorrect usage:

```java
handle.createQuery("select name from things where name like '%:search%'")
    .bind("search", "foo")
    .mapTo(String.class)
    .list()
```

此查询将尝试按**字面意思**选择 `where name like '%:search%'`，而不绑定任何参数。 这是因为 JDBC 驱动程序不会在**字符串文字中**绑定参数。

但是，它永远不会到达那一步——这个查询将抛出一个异常，因为默认情况下我们不允许未使用的参数绑定。

解决方案是使用SQL字符串连接:

```java
handle.createQuery("select name from things where name like '%' || :search || '%'")
    .bind("search", "foo")
    .mapTo(String.class)
    .list()
```

现在，可以将 `search` 作为参数正确绑定到语句，并且一切都按预期工作。

> **🏷注意:** 在执行此操作之前，请检查数据库的字符串连接语法。

## [9. Advanced Topics(高级主题)](#_advanced_topics)

### [9.1. High Availability(高可用性)](#_high_availability)

Jdbi可以与数据库驱动程序中的连接池和高可用性特性结合使用。我们已经成功地将[HikariCP](https://brettwooldridge.github.io/HikariCP/)与[PgJDBC连接负载平衡](https://jdbc.postgresql.org/documentation/head/connect.html)结合使用。

```java
PGSimpleDataSource ds = new PGSimpleDataSource();
ds.setServerName("host1,host2,host3");
ds.setLoadBalanceHosts(true);
HikariConfig hc = new HikariConfig();
hc.setDataSource(ds);
hc.setMaximumPoolSize(6);
Jdbi jdbi = Jdbi.create(new HikariDataSource(hc)).installPlugin(new PostgresPlugin());
```

每个Jdbi可以由任意数量的主机池支持，但是连接应该都是相同的。确切地说，哪些参数必须保持不变，哪些参数可能有所不同，这取决于数据库和驱动程序。

如果您希望有两个单独的池，例如一个连接读副本的只读集和一个较小的只访问单个主机的写入池，那么当前应该有单独的`Jdbi`实例，每个实例都指向单独的`DataSource`。


### [9.2. 使用参数名称编译](#_compiling_with_parameter_names)

默认情况下，Java编译器不会将构造函数和方法的参数名写入类文件。在运行时，反射式请求参数名称会给出“arg0”、“arg1”等值。

开箱即用，Jdbi 使用注解来了解每个参数的名称，例如：

- `ConstructorMapper` 使用 `@ConstructorProperties` 注解.
- SQL 对象方法参数使用 `@Bind` 注解。

```java
@SqlUpdate("insert into users (id, name) values (:id, :name)")
void insert(@Bind("id") long id, @Bind("name") String name); //<1>
```

> **<1>** 如此冗长，非常样板。哇。

如果你使用 `-parameters` 编译器标志编译你的代码，那么就不需要这些注解 — Jdbi 自动使用方法参数名称：

```java
@SqlUpdate("insert into users (id, name) values (:id, :name)")
void insert(long id, String name);
```

#### [9.2.1. Maven 设置](#_maven_setup)

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

#### [9.2.2. IntelliJ IDEA 设置](#_intellij_idea_setup)

- File → Settings
- Build, Execution, Deployment → Compiler → Java Compiler
- Additional command-line parameters: `-parameters`
- Click Apply, then OK.
- Build → Rebuild Project

#### [9.2.3. Eclipse 设置](#_eclipse_setup)

- Window → Preferences
- Java → Compiler
- Under "Classfile Generation," check the option "Store information about method parameters (usable via reflection)."

### 9.3. [Working with Generic Types(使用泛型类型)](#_working_with_generic_types)

Jdbi 提供了实用程序类，以便更轻松地使用 Java 泛型类型。

#### [9.3.1. GenericType(泛型类型)](#_generictype)

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

#### [9.3.2. GenericTypes(泛型类型帮助类)](#_generictypes)

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

### [9.4. NamedArgumentFinder(命名参数查找器)](#_namedargumentfinder)

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

### 9.5. [JdbiConfig(Jdbi配置)](#_jdbiconfig)

配置由 [ConfigRegistry](apidocs/org/jdbi/v3/core/config/ConfigRegistry.html) 类管理。每个代表不同数据库上下文的 Jdbi 对象（例如，**Jdbi** 本身、**Handle** 实例或附加的 SqlObject 类）都有自己的配置注册表。大多数上下文实现了 [Configurable](apidocs/org/jdbi/v3/core/config/Configurable.html) 接口，它允许修改其配置以及检索当前上下文的配置以供 Jdbi 核心或扩展使用。

创建新的可配置上下文时，它会在创建时继承其父配置的副本 - 对原始配置的进一步修改不会影响已创建的配置上下文。 当从 Jdbi 生成Handle、从Handle打开 **SqlStatement** 以及附加或创建按需扩展（如 **SqlObject**）时，会发生配置上下文复制。

配置本身存储在 [JdbiConfig](apidocs/org/jdbi/v3/core/config/JdbiConfig.html) 接口的各种实现中。 每个实现都必须遵守接口的约定； 特别是它必须有一个提供有用默认值的公共无参数构造函数和一个在配置注册表被克隆时调用的 **createCopy** 方法。

通常，应该在使用上下文之前在上下文上设置配置，并且以后不要更改。 一些配置类可能是线程安全的，但大多数不是。

Jdbi 的许多核心功能，例如参数或映射器注册表，都是 **JdbiConfig** 的简单实现，用于存储已注册的映射以供以后在查询执行期间使用。

```java
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

#### [9.5.1. Creating a custom JdbiConfig type(创建自定义 JdbiConfig 类型)](#_creating_a_custom_jdbiconfig_type)

- 创建一个实现 JdbiConfig 的公共类。
- 添加一个公共的、无参数的构造函数
- 添加私有的复制构造函数。
- 实现 `createCopy()` 来调用复制构造函数。
-添加配置属性，并为每个属性提供合理的默认值。
- 确保所有配置属性都被复制到复制构造函数中的新实例中。
- 如果您的配置类希望能够使用注册表中的其他配置类，请重写`setConfig(ConfigRegistry)`。例如，如果RowMappers注册表心没有为给定类型注册一个映射器，则它将委托给ColumnMappers注册表。
- 从其他感兴趣的类中使用该配置对象。
  - 例如 BeanMapper、FieldMapper 和 ConstructorMapper 都使用 ReflectionMappers 配置类来保持通用配置。

### [9.6. JdbiPlugin(Jdbi插件)](#_jdbiplugin)

JdbiPlugin 可用于捆绑批量配置。 插件可以通过`Jdbi.installPlugin(JdbiPlugin)`显式安装，也可以通过`installPlugins()`使用ServiceLoader机制从类路径自动安装。

Jars 可能会在`META-INF/services/org.jdbi.v3.core.spi.JdbiPlugin` 中提供一个文件，其中包含你插件的完全限定类名。

一般来说，Jdbi 的单独模块每个都提供一个相关的插件（例如`jdbi3-sqlite`），并且这些模块将是可自动加载的。 不提供（例如`jdbi3-commons-text`）或多个（例如`jdbi3-core`）插件的模块通常不会。

> **💡提示:** 开发人员鼓励您显式地安装插件。在所使用的模块上声明依赖项的代码对于重构来说更加健壮，并为静态分析工具提供关于哪些代码被使用，哪些代码没有被使用的有用数据。

### [9.7. StatementContext(Statement上下文)](#_statementcontext)

[StatementContext](apidocs/org/jdbi/v3/core/statement/StatementContext.html) 类是与创建和执行语句相关的各种状态的载体，这些状态不适合放在 **Query** 或 其他特定的语句类本身。 除其他外，它拥有开放的**JDBC** 资源、处理过的 SQL 语句和累积的绑定。 它暴露于大多数用户扩展点的实现，例如 **RowMapper、\*ColumnMapper\*s 或 \*CollectorFactory**。

**StatementContext**本身不打算被扩展，通常扩展不需要改变上下文。请阅读JavaDoc以获得更多关于高级用法的信息。

### [9.8. User-Defined Annotations(用户自定义的注解)](#_user_defined_annotations)

SQL Object被设计为使用用户定义的注解进行扩展。事实上，Jdbi中提供的大多数注解都与下面概述的方法相关联。

在SQL Object中有一些不同类别的注解，理解它们之间的区别是很重要的:

- [Statement Customizing Annotations](#_statement_customizing_annotations) - 在执行之前配置方法的底层 [SqlStatement](apidocs/org/jdbi/v3/core/statement/SqlStatement.html)。 这些只能与诸如`@SqlQuery`、`@SqlUpdate` 等注解一起使用，并且不适用于默认方法。
- [Configuration Annotations](#_configuration_annotations) - 在 SQL 对象或其方法之一的范围内修改 [ConfigRegistry](apidocs/org/jdbi/v3/core/config/ConfigRegistry.html) 中的配置。
- [Method Decorating Annotations](#_method_decorating_annotations) - 用一些额外的行为装饰一个方法调用，例如 `@Transaction` 注解将方法调用包装在一个 `handle.inTransaction()` 调用中。

一旦您知道您想要哪种类型的注解，请继续下面的相应部分并按照指南进行设置。

#### [9.8.1. Statement 自定义注解](#_statement_customizing_annotations)

SQL statement 自定义注解用于对与 SQL 方法关联的 [SqlStatement](apidocs/org/jdbi/v3/core/statement/SqlStatement.html) 应用一些更改。

通常，这些注解与核心中的 API 方法相关。 例如 `@Bind`对应`SqlStatement.bind()`，`@MaxRows`对应`Query.setMaxRows()`等。

只有在创建 [SqlStatement](apidocs/org/jdbi/v3/core/statement/SqlStatement.html) 后才应用自定义注解。

您可以创建自己的 SQL statement 自定义注解并将运行时行为附加到它们。

首先，创建一个想要附加语句定制的注解:

```java
@Retention(RetentionPolicy.RUNTIME) //<1>
@Target({ElementType.TYPE, ElementType.METHOD, ElementType.PARAMETER}) //<2>
public @interface MaxRows {
  int value();
}
```

> **<1>** 所有的 statement 自定义注解都应该有一个`RUNTIME`保留策略。
> **<2>** Statement 自定义注解仅适用于类型、方法或参数。 严格来说，`@Target`注解释不是必需的，但包含它是一个很好的做法，这样注解只能应用在它们实际执行某些操作的地方。

在类型上放置自定义注解意味着“将此自定义应用于每个方法”。

当用于参数时，注解可以在处理注解时使用传递给方法的参数。

接下来，我们编写 [SqlStatementCustomizerFactory](apidocs/org/jdbi/v3/sqlobject/customizer/SqlStatementCustomizerFactory.html) 类的实现，以处理注解并将自定义应用于语句。

`SqlStatementCustomizerFactory` 生成两种不同类型的“语句定制器”命令对象：[SqlStatementCustomizer](apidocs/org/jdbi/v3/sqlobject/customizer/SqlStatementCustomizer.html)(用于类型或方法的注解)和 [SqlStatementParameterCustomizer](apidocs/org/jdbi/v3/sqlobject/customizer/SqlStatementParameterCustomizer.html)(用于方法参数的注解)。

让我们为注解实现一个语句定制工厂:

```java
public class MaxRowsFactory implements SqlStatementCustomizerFactory {
    @Override
    public SqlStatementCustomizer createForType(Annotation annotation,
                                                Class<?> sqlObjectType) {
        final int maxRows = ((MaxRows)annotation).value(); //<1>
        return stmt -> ((Query)stmt).setMaxRows(maxRows); //<2>
    }

    @Override
    public SqlStatementCustomizer createForMethod(Annotation annotation,
                                                  Class<?> sqlObjectType,
                                                  Method method) {
        return createForType(annotation, sqlObjectType); //<3>
    }

    @Override
    public SqlStatementParameterCustomizer createForParameter(Annotation annotation,
                                                              Class<?> sqlObjectType,
                                                              Method method,
                                                              Parameter param,
                                                              int index,
                                                              Type type) {
        return (stmt, maxRows) -> ((Query)stmt).setMaxRows((Integer) maxRows); //<4>
    }
}
```

> **<1>** 从注解中提取最大行
> **<2>** [SqlStatementCustomizer](apidocs/org/jdbi/v3/sqlobject/customizer/SqlStatementCustomizer.html) 可以实现为一个 lambda——它接收一个 [SqlStatement](apidocs/org/jdbi/v3/core/statement/SqlStatement.html) 作为参数，在语句上调用它想要的任何方法，并返回 `void`。
> **<3>** 由于此注解的定制在方法级别和类型级别是相同的，为了简洁起见，我们简单地委托给类型级别的方法。
> **<4>** [SqlStatementParameterCustomizer](apidocs/org/jdbi/v3/sqlobject/customizer/SqlStatementParameterCustomizer.html) 也可以实现为 lambda。 它接受一个 `SqlStatement` 和传递给带注解参数的方法的值。

最后，添加[@SqlStatementCustomizingAnnotation](apidocs/org/jdbi/v3/sqlobject/customizer/SqlStatementCustomizingAnnotation.html)注解到`@MaxRows`注解类型。 这告诉 Jdbi `MaxRowsFactory` 实现了 `@MaxRows` 注解的行为：

```java
@SqlStatementCustomizingAnnotation(MaxRowsFactory.class)
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.PARAMETER})
public @interface MaxRows {
    int value() default -1;
}
```

您的statement 自定义注解现在可以在任何 SQL 对象上使用：

```java
public interface Dao {
  @SqlQuery("select * from contacts")
  @MaxRows(100)
  List<Contact> list();

  @SqlQuery("select * from contacts")
  List<Contact> list(@MaxRows int maxRows);
}
```

> **💡提示:** 我们在这里选择了 `@MaxRows` 作为示例，因为它很容易理解。 实际上，在 SQL 语句中使用 LIMIT 子句比使用 @MaxRows 可以获得更好的数据库性能。

#### [9.8.2. Configuration 注解](#_configuration_annotations)

Configuration注解用于对与 SQL 对象或方法关联的 [ConfigRegistry](apidocs/org/jdbi/v3/core/config/ConfigRegistry.html) 应用一些更改。

通常这些注解与[Configurable](apidocs/org/jdbi/v3/core/config/Configurable.html)的方法相关(`Jdbi`、`Handle`和`SqlStatement`都实现了这个接口)。 例如，`@RegisterColumnMapper` 与`Configurable.registerColumnMapper()` 相关。

您可以创建自己的配置注解，并将运行时行为附加到它们：

- 使用您需要的任何属性编写新的配置注解。
- 编写 [Configurer](apidocs/org/jdbi/v3/sqlobject/config/Configurer.html) 的实现，它执行与您的注解相关的配置。
- 将 `@ConfiguringAnnotation` 注解添加到您的配置注解类型上。

完成上述步骤后，Jdbi 将在遇到相关注解时调用您的配置器。

让我们重新实现 Jdbi 的内置注解之一作为示例：

`@RegisterColumnMapper` 注解有一个属性来指定要注册的列映射器的类。 无论在哪里使用注解，我们都希望 Jdbi 创建该映射器类型的实例，并将其注册到配置注册表。

首先，让我们创建新的注解类型：

```java
@Retention(RetentionPolicy.RUNTIME) //<1>
@Target({ElementType.TYPE, ElementType.METHOD}) //<2>
public @interface RegisterColumnMapper{
  Class<? extends ColumnMapper<?>> value();
}
```

> **<1>** 所有配置注解都应该有一个 `RUNTIME` 保留策略。
> **<2>** 配置注解仅适用于类型和方法。 严格来说，`@Target` 注解不是必需的，但包含它是一个很好的做法，这样注解只能应用在它们实际执行某些操作的地方。

在类型上放置配置注解意味着“将此配置应用于每个方法”。

接下来，我们编写 [Configurer](apidocs/org/jdbi/v3/sqlobject/config/Configurer.html) 类的实现，来处理注解并应用配置：

```java
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
                         RegisterColumnMapper registerColumnMapper) { //<1>
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

> **<1>** 在这个例子中，我们应用了相同的配置，无论是在 SQL 对象类型还是方法上使用了 `@RegisterColumnMapper` 注解。 然而，这不是必需的——一些注解可能会根据注解是放在类型上还是方法上来选择应用不同的配置。

对于只有一个目标的配置注解（例如`@KeyColumn` 和`@ValueColumn` 可能只应用于方法），您只需要实现适合该注解目标的`Configurer` 方法。

最后，将 [@ConfiguringAnnotation](apidocs/org/jdbi/v3/sqlobject/config/ConfiguringAnnotation.html) 注解添加到您的 `@RegisterColumnMapper` 注解类型上。 这告诉 Jdbi `RegisterColumnMapperImpl` 实现了 `@RegisterColumnMapper` 注解的行为。

```java
@ConfiguringAnnotation(RegisterColumnMapperImpl.class)
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface RegisterColumnMapper {
    Class<? extends TemplateEngine> value();
}
```

您的配置注解现在可以在任何 SQL 对象中使用：

```java
public interface AccountDao {
  @SqlQuery("select balance from accounts where id = ?")
  @RegisterColumnMapper(MoneyMapper.class)
  public Money getBalance(long accountId);
}
```

#### [9.8.3. Method 装饰注解](#_method_decorating_annotations)

方法装饰注解用于通过附加（或替代）行为增强 SQL 对象方法。

在内部，SQL Object 用 [Handler](apidocs/org/jdbi/v3/sqlobject/Handler.html) 接口的实例来表示每个方法的行为。 每次调用SQL Object实例上的方法时，该方法都通过执行给定方法的处理程序来执行。

当您使用装饰注解（如`@Transaction`）时，方法的常规处理程序被包装在另一个处理程序中，该处理程序可能会在将调用传递给原始处理程序之前和/或之后执行某些操作。

装饰器甚至可以执行一些操作*而不是*调用原始操作，例如 用于缓存注解。

让我们重新实现 `@Transaction` 注解，看看它是如何工作的：

首先，创建注解类型：

```java
@Retention(RetentionPolicy.RUNTIME) //<1>
@Target(ElementType.METHOD) //<2>
public @interface Transaction {
    TransactionIsolationLevel value();
}
```

> **<1>** 所有装饰注解都应该有一个 `RUNTIME` 保留策略。
> **<2>** 装饰注解仅适用于类型和方法。 严格来说，`@Target` 注解不是必需的，但包含它是一个很好的做法，这样注解只能应用在它们实际执行某些操作的地方。

在类型上放置装饰注解意味着“将此装饰应用于每个方法”。

接下来我们编写一个[HandlerDecorator](apidocs/org/jdbi/v3/sqlobject/HandlerDecorator.html)接口的实现，来处理注解并应用装饰：

```java
public class TransactionDecorator implements HandlerDecorator {
  public Handler decorateHandler(Handler base,
                                 Class<?> sqlObjectType,
                                 Method method) {
    Transaction anno = method.getAnnotation(Transaction.class); //<1>
    TransactionIsolationLevel isolation = anno.value(); //<2>

    return (target, args, handleSupplier) -> handleSupplier.getHandle() //<3>
        .inTransaction(isolation, h -> base.invoke(target, args, handleSupplier));
  }
}
```

> **<1>** 获取`@Transaction`注解
> **<2>** 从注解中提取事务隔离级别
> **<3>** `Handler` 接口接受一个目标（被调用的 SQL Object  实例）、一个传递给该方法的参数的 `Object[]` 数组和一个 `HandleSupplier`。

最后，将 [@SqlMethodDecoratingAnnotation](apidocs/org/jdbi/v3/sqlobject/SqlMethodDecoratingAnnotation.html) 注解添加到您的 `@Transaction` 注解类型。 这告诉 Jdbi `TransactionDecorator` 实现了 `@Transaction` 注解的行为。

```java
@SqlMethodDecoratingAnnotation(TransactionDecorator.class)
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Transaction {
    TransactionIsolationLevel value();
}
```

您的装饰注解现在可以在任何 SQL Oject中使用了：

```java
public interface ContactDao {
  @SqlBatch("insert into contacts (id, name) values (:id, :name)")
  @Transaction
  void batchInsert(@BindBean Contact... contacts);
}
```

##### 装饰顺序

如果 SQL Object 方法应用了两个或多个装饰注解，并且装饰的顺序很重要，请使用 `@DecoratorOrder` 注解。 如果没有声明顺序，首先应用类型装饰器，然后应用方法装饰器，但不进一步指定顺序。

例如，假设一个方法同时用`@Cached` 和`@Transaction` 进行了注解（随它去......）。 我们可能希望首先使用 `@Cached` 注释，这样当缓存已经包含该条目时，不会不必要地创建事务。

```
public interface ContactDao {
  @SqlQuery("select * from contacts where id = ?")
  @Cached
  @Transaction
  @DecoratorOrder(Cached.class, Transaction.class)
  Contact getById(long id);
}
```

装饰器顺序从最外层到最内层表示。

### [9.9. 模板引擎](#_templateengine)

Jdbi 使用 [TemplateEngine](apidocs/org/jdbi/v3/core/statement/TemplateEngine.html) 实现将模板呈现为 SQL。 模板引擎将 SQL 模板字符串和`StatementContext`作为输入，并生成可解析的 SQL 字符串作为输出。

开箱即用，Jdbi 被配置为使用 `DefinedAttributeTemplateEngine`，它用命名属性的字符串值替换 SQL 语句中的尖括号标记，如 `<name>`：

```java
String tableName = "customers";
Class<?> entityClass = Customer.class;

handle.createQuery("select <columns> from <table>")
      .define("table", "customers")
      .defineList("columns", "id", "name")
      .mapToMap()
      .list() // => "select id, name from customers"
```

> **🏷注意:** `defineList` 方法将元素列表定义为单个元素的字符串值的逗号分隔拼接。 在上面的例子中，`columns` 属性被定义为 `"id, name"`。

可以使用任何自定义模板引擎。 只需实现`TemplateEngine`接口，然后在`Jdbi`、`Handle`或像`Update`或`Query`这样的SQL语句上调用`setTemplateEngine()`：

```java
TemplateEngine templateEngine = (template, ctx) -> {
  //...
};

jdbi.setTemplateEngine(templateEngine);
```

> **💡提示:** Jdbi 还提供了 `StringTemplateEngine`，它使用 StringTemplate 库呈现模板。 参见 [StringTemplate 4](#_stringtemplate_4)。

### [9.10. SqlParser(Sql解析器)](#_sqlparser)

渲染 SQL 模板后，Jdbi 使用 [SqlParser](apidocs/org/jdbi/v3/core/statement/SqlParser.html) 从 SQL 语句中解析出任何命名参数。 这会产生一个 `ParsedSql` 对象，其中包含 Jdbi 绑定参数和执行 SQL 语句所需的所有信息。

开箱即用，Jdbi 被配置为使用`ColonPrefixSqlParser`，它识别以冒号为前缀的命名参数，例如 `:name`。

```java
handle.createUpdate("insert into characters (id, name) values (:id, :name)")
      .bind("id", 1)
      .bind("name", "Dolores Abernathy")
      .execute();
```

Jdbi 还提供了`HashPrefixSqlParser`，它识别带有哈希前缀的参数，例如 `#hashtag`。 通过在 `Jdbi`、`Handle` 或任何 SQL 语句（如 `Query` 或 `Update`）上调用 `setSqlParser()` 来使用此解析器。

```java
handle.setSqlParser(new HashPrefixSqlParser());
handle.createUpdate("insert into characters (id, name) values (#id, #name)")
      .bind("id", 2)
      .bind("name", "Teddy Flood")
      .execute();
```

> **🏷注意:**默认解析器将任何Java标识符识别为参数或属性名。即使是一些奇怪的情况，如表情符号，也被允许，尽管Jdbi的作者鼓励适当的谨慎 🧐.
> **🏷注意:** 默认的解析器尝试忽略字符串字面量中的类似参数的结构，因为JDBC驱动程序无论如何都不允许在那里绑定参数。

对于读过[Dragon book](https://www.amazon.com/Compilers-Principles-Techniques-Tools-2nd/dp/0321486811)的无畏冒险者，任何自定义SQL解析器都可以使用。 只需实现`SqlParser`接口，然后在Jdbi、Handle或SQL语句上设置：

```java
SqlParser parser = (sql, ctx) -> {
  ...
};

jdbi.setParser(parser);
```

### [9.11. SqlLogger(Sql日志记录器)](#_sqllogger)

[SqlLogger](apidocs/org/jdbi/v3/core/statement/SqlLogger.html) 接口在执行每个语句之前和之后被调用，并给定当前的`StatementContext`，记录所需的任何相关信息：主要是在各个编译阶段的查询，属性和绑定，以及重要的时间戳。

### [9.12. ResultProducer(Result生产者)](#_resultproducer)

**ResultProducer** 采用延迟提供的 **PreparedStatement** 并产生结果。 最常见的生产者路径 **execute()** 检索查询结果上的 **ResultSet**，然后使用 **ResultSetScanner** 或更高级别的映射器来生成结果。

另一个例子是只返回修改的行数，如在UPDATE或INSERT语句中:

```java
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

如果您获得了lazy语句，则负责确保上下文最终关闭以释放数据库资源。

大多数用户将不需要实现**ResultProducer**接口。

### [9.13. Generator(生成器)](#_generator)

Jdbi 包括一个实验性的 SqlObject 代码生成器。 如果你包含 `jdbi3-generator` 工件作为注释处理器并使用 `@GenerateSqlObject` 注释你的 SqlObject 定义，生成器将生成一个实现类并避免使用 `Proxy` 实例。 这对于 `graal-native` 编译可能很有用。

## [10. Appendix(附录)](#_appendix)

### [10.1. 最佳实践](#_best_practices)

- 如果可能的话，针对真实的数据库测试SQL对象(dao)。Jdbi试图采取防御态度，当你错误地把握它时，它会急切地失败。
- 使用 `-parameters` 编译器标志来避免 SQL 对象方法参数中的所有那些 `@Bind("foo") String foo` 冗余限定符。参见 [使用参数名称编译](#133____9_2__使用参数名称编译).
- 使用一个分析器!性能问题的真正根本原因往往出人意料。首先测量，然后调整性能。然后再次测量，以确保它有所不同。
- 别忘了带毛巾！

### [10.2. API 参考](#_api_reference)

- [Javadoc](apidocs/index.html)
- [jdbi3-kotlin](apidocs-kotlin/jdbi3-kotlin/index.html)
- [jdbi3-kotlin-sqlobject](apidocs-kotlin/jdbi3-kotlin-sqlobject/index.html)

### [10.3. 相关项目](#_related_projects)

[Embedded Postgres](https://github.com/opentable/otj-pg-embedded) 使针对真实数据库的测试变得快速而简单。

[dropwizard-jdbi3](https://github.com/arteam/dropwizard-jdbi3) 提供与 DropWizard 的集成。

[metrics-jdbi3](https://github.com/arteam/metrics-jdbi3) 工具使用DropWizard-Metrics发出语句计时统计数据。

你知道一个与Jdbi相关的项目吗?给我们发送一个issue，我们会在这里添加一个链接!

### [10.4. 捐献](#_contributing)

**jdbi** 使用 GitHub 进行协作。 请查看[项目页面](https://github.com/jdbi/jdbi) 了解更多信息。

如果您有问题，我们有 [Google Group 邮件列表](https://groups.google.com/group/jdbi)

用户有时会在 [IRC in #jdbi on Freenode](irc://irc.freenode.net/#jdbi) 上闲逛。

### [10.5. 从 v2 升级到 v3](#_upgrading_from_v2_to_v3)

已经在使用 Jdbi v2？

以下是一个快速的差异总结，以帮助您升级:

总体:

- Maven 工件重命名和拆分：
- 旧的: `org.jdbi:jdbi`
- 新的: `org.jdbi:jdbi3-core`, `org.jdbi:jdbi3-sqlobject`, etc.
- 根 package 重命名: `org.skife.jdbi.v2` 到 `org.jdbi.v3`

Core API:

- `DBI`, `IDBI` → `Jdbi`
  - 使用`Jdbi.create()`工厂方法而不是构造函数进行实例化。
- `DBIException` → `JdbiException`
- `Handle.select(String, ...)` 现在返回一个 `Query` 用于进一步的方法链接，而不是 `List<Map<String, Object>>`。 调用 `Handle.select(sql, ...).mapToMap().list()` 以获得与 v2 相同的效果。
- `Handle.insert()` 和 `Handle.update()` 已合并为 `Handle.execute()`。
- `ArgumentFactory` 不再是通用的。
- `AbstractArgumentFactory` 是用于处理单个参数类型的工厂的 `ArgumentFactory` 的通用实现。
- 参数和映射器工厂现在根据 `java.lang.reflect.Type` 而不是 `java.lang.Class` 运行。 这允许 Jdbi 处理泛型类型的参数和映射器。
- 参数和映射器工厂现在有一个单独的 `build()` 方法，它返回一个 `Optional`，而不是单独的 `accepts()` 和 `build()` 方法。
- `ResultSetMapper` → `RowMapper`. 行索引参数也从 `RowMapper` 中删除——当前行号可以直接从 `ResultSet` 中检索。
- `ResultColumnMapper` → `ColumnMapper`
- `ResultSetMapperFactory` → `RowMapperFactory`
- `ResultColumnMapperFactory` → `ColumnMapperFactory`
- 默认情况下，`Query` 不再映射到 `Map<String, Object>`。 调用`Query.mapToMap()`、`.mapToBean(type)`、`.map(mapper)` 或`.mapTo(type)`。
- `ResultBearing<T>` 被重构为 `ResultBearing(无通用参数)` 和  `ResultIterable<T>`。 调用 `.mapTo(type)` 以获得一个 `ResultIterable<T>`。
- `TransactionConsumer` 和 `TransactionCallback` 现在只接受一个 `Handle`——移除了 `TransactionStatus` 参数。 现在只需rollback句柄即可。
- 删除了`TransactionStatus` 类。
- 删除了`CallbackFailedException` 类。 像`HandleConsumer`、`HandleCallback`、`TransactionCallback` 等功能接口现在可以抛出任何异常类型。 像`Jdbi.inTransaction` 这样的接受这些回调的方法使用异常透明性来只抛出回调抛出的异常。 如果你的回调没有抛出任何检查异常，你就不需要 try/catch 块。
- 从core中删除了`StatementLocator` 接口。 所有core statements 现在都希望接收实际的 SQL 字符串。 添加了一个类似的概念，`SqlLocator`，但它是特定于 SQL 对象的。
- `StatementRewriter` 重构为 `TemplateEngine` 和 `SqlParser`。
- StringTemplate 不再需要在 SQL 中处理 `<name>` 样式的标记。
- 自定义 SqlParser 实现现在必须提供一种方法来将原始参数名称转换为将被正确解析为命名参数的名称。

SQL Object API:

- 默认情况下不安装 SQL 对象支持。 它必须作为单独的依赖项添加，并将插件安装到 `Jdbi` 对象中：

```java
Jdbi jdbi = Jdbi.create(...);
jdbi.installPlugin(new SqlObjectPlugin());
```

- v3 中的 SQL 对象类型必须是公共接口——没有类。 方法返回类型同样必须是公共的。 这是由于 SQL 对象实现从 CGLIB 切换到`java.lang.reflect.Proxy`，它只支持接口。
- `GetHandle` → `SqlObject`
- `SqlLocator` 取代了 `StatementLocator`，并且只适用于 SQL 对象。
- `@RegisterMapper` 被分为`@RegisterRowMapper` 和`@RegisterColumnMapper`。
- 通过在启用 `-parameters` 编译器标志的情况下编译代码，可以将 SQL 对象方法参数上的 `@Bind` 注释设为可选。
- `@BindIn` → `@BindList`，不再需要 StringTemplate
- 按需 SQL 对象不适用于返回`Iterable`或`FluentIterable`的方法。 按需对象在每次方法调用后都严格关闭句柄，不再像 v2 中那样“让门打开”让您完成对可交互对象的消费。 这排除了连接泄漏的主要来源。
- SQL 对象不再是可关闭的 — 它们要么是按需的，要么它们的生命周期与它们所附加的`Handle`的生命周期相关联。
- `@BindAnnotation` 元注解已删除。 改用`@SqlStatementCustomizingAnnotation`。
- `@SingleValueResult` → `@SingleValue`. 此注解可用于方法返回类型，也可用于`@SqlBatch` 参数。
