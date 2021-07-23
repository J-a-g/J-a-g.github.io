---
title: '学习Android Jetpack:Room'
date: 2021-06-16 15:39:28
catalog:    true
header-img: "post-bg-unix-linux.jpg"
tags: Android JetPack
---

## 官方文档

项目地址：[https://github.com/android/architecture-components-samples](https://github.com/android/architecture-components-samples)
文档地址：[https://developer.android.com/training/data-storage/room](https://developer.android.com/training/data-storage/room)

## Sameple展示
它的效果大概是这样：
![java-javascript](ezgif.com-gif-maker.gif)
后续的文档也是根据 Sameple 项目进行说明，可能不能涵盖 Room 的所有知识点，但是尽量将最常用的功能进行说明。
这里大致说明一下 Sameple 的需求功能：可以注册账号，并通过账号进行登录；主页有两个导航商品和收藏。商品展示的所有商品信息，并且可以通过按钮展示指定系列的商品信息，且可以用过收藏按钮关联的用户信息上；收藏页中主要是展示该账号用户选在了收藏的商品信息。

## 概述

Room 在 SQLite 上提供了一个抽象层，以便在充分利用 SQLite 的强大功能的同时，能够流畅地访问数据库。

如需在应用中使用 Room，请将以下依赖项添加到应用的 build.gradle 文件：
```js
dependencies {
    def room_version = "2.3.0"

    implementation("androidx.room:room-runtime:$room_version")
    annotationProcessor "androidx.room:room-compiler:$room_version"

    // To use Kotlin annotation processing tool (kapt)
    kapt("androidx.room:room-compiler:$room_version")
    // To use Kotlin Symbolic Processing (KSP)
    ksp("androidx.room:room-compiler:$room_version")

    // optional - Kotlin Extensions and Coroutines support for Room
    implementation("androidx.room:room-ktx:$room_version")

    // optional - RxJava2 support for Room
    implementation "androidx.room:room-rxjava2:$room_version"

    // optional - RxJava3 support for Room
    implementation "androidx.room:room-rxjava3:$room_version"

    // optional - Guava support for Room, including Optional and ListenableFuture
    implementation "androidx.room:room-guava:$room_version"

    // optional - Test helpers
    testImplementation("androidx.room:room-testing:$room_version")
}
```

Room 包含 3 个主要组件：

* **数据库：**包含数据库持有者，并作为应用已保留的持久关系型数据的底层连接的主要接入点。
使用 `@Database` 注释的类应满足以下条件：
     * 是扩展 `RoomDatabase` 的抽象类。
     * 在注释中添加与数据库关联的实体列表。
     * 包含具有 0 个参数且返回使用 `@Dao` 注释的类的抽象方法。

  在运行时，您可以通过调用 `Room.databaseBuilder()` 或 `Room.inMemoryDatabaseBuilder()` 获取 `Database` 的实例。
* **Entity：**表示数据库中的表。
* **DAO：**包含用于访问数据库的方法。

应用使用 Room 数据库来获取与该数据库关联的数据访问对象 (DAO)。然后，应用使用每个 DAO 从数据库中获取实体，然后再将对这些实体的所有更改保存回数据库中。 最后，应用使用实体来获取和设置与数据库中的表列相对应的值。
![java-javascript](1.jpg)
<small class="img-hint">Room 架构图</small>

## 插件推荐

为了方便数据库调试，推荐在 Android studio 中安装 Database Inspector 插件。可以通过这个插件直观的看到数据库表结构、数据库中的数据。
![java-javascript](3.jpg)
![java-javascript](2.jpg)

## RoomDatabase

```js
@Database(entities = [Product::class, User::class, Like::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun productDao(): ProductDao

    abstract fun userDao(): UserDao

    abstract fun likeDao(): LikeDao

    companion object {
        @Volatile
        private var instance: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase {
            return instance ?: synchronized(this) {
                instance ?: buildDataBase(context).also { instance = it }
            }
        }

        private fun buildDataBase(context: Context): AppDatabase {
            return Room.databaseBuilder(
                context,
                AppDatabase::class.java, "my_database.db"
            ).allowMainThreadQueries()
                .addCallback(sRoomDatabaseCallback)
                .build()
        }

        private val sRoomDatabaseCallback: RoomDatabase.Callback =
            object : RoomDatabase.Callback() {
                override fun onCreate(db: SupportSQLiteDatabase) {
                    super.onCreate(db)
                    Log.w("scj", "sRoomDatabaseCallback onCreate")

                }
            }
    }
}
```

## 定义实体 Entity
使用 `Room` 持久性库时，您可以将相关字段集定义为实体。对于每个实体，系统会在关联的 `Database` 对象中创建一个表，以存储这些项。您必须通过 `Database` 类中的 `entities` 数组引用实体类。

**如何定义实体**
```js
@Entity(tableName = "products") //表名
data class Product(
    @PrimaryKey(autoGenerate = true)//主键，且自增
    var pd_id: Long = 0,
    @ColumnInfo(name = "pd_name") var name: String ="",
    @ColumnInfo(name = "pd_price") var price: Double = 0.0,
    @ColumnInfo(name = "pd_brand") var brand: String = "",
    var likes: Int = 0,
    @ColumnInfo(name = "pd_description") var description: String = "",
    @ColumnInfo(name = "pd_imgUrl") var imageUrl: String = ""
//    @Ignore val picture: Bitmap?  忽略字段，不会再表中生成列

)
```
`@Entity(tableName = "products")` 设置在数据库中的表名 `products`
`@PrimaryKey(autoGenerate = true)` 定义 `pd_id` 为主键，且自增
`@ColumnInfo(name = "pd_name")` 定义字段在表中的列名
`@Ignore val picture: Bitmap` 如果类中有些字段不需要在表中有对应列，使用 `@Ignore` 进行注释

项目除了上面的实体，还有两个实体分别是：
```js
@Entity(primaryKeys = ["pd_id", "user_id"])
data class Like(
    val pd_id: Long,
    val user_id: Long

) 

@Entity(tableName = "user")
data class User (
    @PrimaryKey(autoGenerate = true)
    var user_id: Long = 0,
    @ColumnInfo(name = "tb_username")
    var username: String? = null,
    @ColumnInfo(name = "tb_password")
    var password: String? = null,
    var email: String? = null,
    var login: Boolean? = false
)
```

## 使用 Room DAO 访问数据

如需使用 `Room 持久性库`访问应用的数据，您可以使用数据访问对象 (DAO)。这些 `Dao` 对象构成了 Room 的主要组件，因为每个 DAO 都包含一些方法，这些方法提供对应用数据库的抽象访问权限。

通过使用 DAO 类（而不是查询构建器或直接查询）访问数据库，您可以拆分数据库架构的不同组件。此外，借助 DAO，您可以在测试应用时轻松模拟数据库访问。    

DAO 既可以是接口，也可以是抽象类。如果是抽象类，则该 DAO 可以选择有一个以 `RoomDatabase` 为唯一参数的构造函数。Room 会在编译时创建每个 DAO 实现。


##### 插入

当您创建 DAO 方法并使用 `@Insert` 对其进行注释时，Room 会生成一个实现，该实现在单个事务中将所有参数插入数据库中。
```js
@Dao
interface ProductDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)//冲突策略
    fun insertProduct(product: Product)

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    fun insertProducts(product: List<Product>)

    @Insert
    fun insertBothProducts(product1: Product, product2: Product)

    @Insert
    fun insertProductAndLoveProducts(product: Product, loveProduct: List<Product>)
}
```
如果 `@Insert` 方法只接收 1 个参数，则它可以返回 long，这是插入项的新 rowId。如果参数是数组或集合，则应返回 long[] 或 List<Long>。

对数据库设计时，不允许重复数据的出现。否则，必然造成大量的冗余数据。实际上，难免会碰到这个问题：冲突。当我们像数据库插入数据时，该数据已经存在了，必然造成了冲突。该冲突该怎么处理呢？

* **OnConflictStrategy.REPLACE**：冲突策略是取代旧数据同时继续事务 
* **OnConflictStrategy.ROLLBACK**：冲突策略是回滚事务
* **OnConflictStrategy.ABORT**：冲突策略是终止事务
* **OnConflictStrategy.FAIL**：冲突策略是事务失败
* **OnConflictStrategy.IGNORE**：冲突策略是忽略冲突

##### 更新

`@Update` 便捷方法会修改数据库中以参数形式给出的一组实体。**它使用与每个实体的主键匹配的查询。**
```js
@Dao
interface ProductDao {
    @Update
    fun updateProduct(vararg product: Product)
}
```
虽然通常没有必要，但是您可以让此方法返回一个 int 值，以指示数据库中更新的行数。

##### 删除

`@Delete` 便捷方法会从数据库中删除一组以参数形式给出的实体。**它使用主键查找要删除的实体。**
```js
@Dao
interface ProductDao {
    @Delete
    fun deleteProducts(vararg product: Product)
}
```
虽然通常没有必要，但是您可以让此方法返回一个 int 值，以指示从数据库中删除的行数。

##### 查询

`@Query` 是 DAO 类中使用的主要注释。它允许您对数据库执行读/写操作。每个 `@Query` 方法都会在编译时进行验证，因此如果查询出现问题，则会发生编译错误，而不是运行时失败。

上述的 **更新、删除** 都是快捷方法，一些复杂的更新、删除逻辑则可以通过 `@Query（"sql语句")` 进行更新或者删除操作

**查询**

下面罗列一下常见的几种查询语句
```js
@Dao
interface ProductDao {
    @Query("SELECT * FROM products ORDER BY pd_id DESC")
    fun loadAllProducts(): Flow<List<Product>>//返回所有商品信息并对pd_id进行降序排序

    @Query("SELECT * FROM products WHERE pd_brand LIKE :brand ORDER BY pd_id DESC")
    fun loadBrandProducts(brand: String): Flow<List<Product>>//根据品牌类型查询商品信息，并根据pd_id进行降序排序

    @Query("SELECT pd_name, pd_price FROM products")
    fun getNamePrices(): List<NameTuple>//列子集查询

    @Query("SELECT * FROM products ORDER BY pd_id DESC LIMIT 1")
    fun getTopProduct(): Product?//返回所有商品信息并对pd_id进行降序排序且只返回一条商品信息

    @Query("select * from `like` inner join products on `like`.pd_id = products.pd_id inner join user on user.user_id = `like`.user_id")
    fun getUserAndProduct(): List<Result>//多表查询
}
```

**返回列的子集**

大多数情况下，您只需获取实体的几个字段。例如，您的界面可能仅显示用户的名字和姓氏，而不是用户的每一条详细信息。通过仅提取应用界面中显示的列，您可以节省宝贵的资源，并且您的查询也能更快完成。
```js
data class NameTuple(
    @ColumnInfo(name = "pd_name") val name: String?,
    @ColumnInfo(name = "pd_price") val price: Double?
) 

@Dao
interface ProductDao {
    @Query("SELECT pd_name, pd_price FROM products")
    fun getNamePrices(): List<NameTuple>//列子集查询
}
```

**多表查询**

下面案例的意思：查看所有用户收藏的所有商品信息，并返回列子集结果

```js
class Result(
    @ColumnInfo(name = "tb_username") val userName: String = "",
    @ColumnInfo(name = "pd_name") var name: String = "",
    @ColumnInfo(name = "pd_price") var price: Double = 0.0
) 

@Dao
interface ProductDao {
    @Query("select * from `like` inner join products on `like`.pd_id = products.pd_id inner join user on user.user_id = `like`.user_id")
    fun getUserAndProduct(): List<Result>//多表查询
}
```

**更新、删除**

通过 `@Query` 执行更新或删除语句

```js
@Dao
interface ProductDao {
    @Query("update products set likes = case when products.pd_id in (select pd_id from `like` where user_id Like :user_id) then 1 else 0 end")
    fun updateFavoritesProducts(user_id: Long)
}

@Dao
abstract class LikeDao {
    @Query("DELETE FROM `like` WHERE pd_id Like :pro_id AND user_id Like :user_id")
    abstract fun deleteLikes(pro_id: Long, user_id: Long)
}
```

**使用 Kotlin 协程进行异步查询**

您可以将 suspend Kotlin 关键字添加到 DAO 方法中，以使用 Kotlin 协程功能使这些方法成为异步方法。这样可确保不会在主线程上执行这些方法。

```js
@Dao
abstract class LikeDao {

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    abstract  fun insertLike(like: Like)

    //1.删除数据，根据主键来删除数据的！！！
    @Delete
    abstract suspend fun deleteLike(like: Like)

    @Query("DELETE FROM `like` WHERE pd_id Like :pro_id AND user_id Like :user_id")
    abstract suspend fun deleteLikes(pro_id: Long, user_id: Long)

    @Query("SELECT * FROM `like` WHERE user_id Like :user_id")
    abstract suspend fun queryLikesByUserId(user_id: Long): Array<Like>
}
```

适用于带有 `@Transaction` 注释的 DAO 方法。您可以使用此功能通过其他 DAO 方法构建暂停数据库方法。然后，这些方法会在单个数据库事务中运行。
```js
@Dao
abstract class UserDao {

    @Transaction
    open suspend fun regist(user: User): Boolean {
        val users = user.username?.let { user.email?.let { it1 -> loadUsersByColumn(it, it1) } }
        return if (users != null && users.isNotEmpty()) {
            false
        } else {
            insertUsers(user)
            true
        }
    }

    @Query("SELECT * FROM user WHERE tb_username Like :username OR email Like :email")
    abstract fun loadUsersByColumn(username: String, email: String): Array<User>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    abstract suspend fun insertUsers(vararg users: User): List<Long>
}
```

## 定义对象之间的关系

##### 创建嵌套对象

有时，您可能希望在数据库逻辑中将某个实体或数据对象表示为一个紧密的整体，即使该对象包含多个字段也是如此。在这些情况下，您可以使用 `@Embedded` 注释表示要分解为表格中的子字段的对象。然后，您可以像查询其他各个列一样查询嵌套字段。

例如，您的 User 类可以包含一个 Address 类型的字段，它表示名为 street、city、state 和 postCode 的字段的组合。若要在表中单独存储组合列，请在 User 类（带有 `@Embedded` 注释）中添加 Address 字段，如以下代码段所示：
```js
    data class Address(
        val street: String?,
        val state: String?,
        val city: String?,
        @ColumnInfo(name = "post_code") val postCode: Int
    )

    @Entity
    data class User(
        @PrimaryKey val id: Int,
        val firstName: String?,
        @Embedded val address: Address?
    )
    
```
然后，表示 User 对象的表将包含具有以下名称的列：id、firstName、street、state、city 和 post_code。

如果某个实体具有相同类型的多个嵌套字段，您可以通过设置 `prefix` 属性确保每个列的唯一性。然后，Room 会将提供的值添加到嵌套对象中每个列名称的开头。

##### 多对多关系

对象之间的关系主要有：[嵌套关系](https://developer.android.com/training/data-storage/room/relationships#nested-relationships)、[一对一关系](https://developer.android.com/training/data-storage/room/relationships#one-to-one)、[一对多关系](https://developer.android.com/training/data-storage/room/relationships#one-to-many)、[多对多关系](https://developer.android.com/training/data-storage/room/relationships#many-to-many),案例中说明了多对多的关系，其他关系并没有说明可以直接看文档。

两个实体之间的多对多关系是指这样一种关系：父实体的每个实例对应于子实体的零个或多个实例，反之亦然。

在案例中一个用户账号可能收藏多个商品，同样一个商品也可能被多个用户账号所收藏。

首先需要两个实体分别代表用户和商品，再创建第三个类来表示两个实体之间的**关联实体（即交叉引用表）**，交叉引用表中必须包含表中表示的多对多关系中每个实体的主键列。在案例中就是用户的主键和商品的主键
```js
@Entity(tableName = "user")
data class User (
    @PrimaryKey(autoGenerate = true)
    var user_id: Long = 0,
    @ColumnInfo(name = "tb_username")
    var username: String? = null,
    @ColumnInfo(name = "tb_password")
    var password: String? = null,
    var email: String? = null,
    var login: Boolean? = false
)

@Entity(tableName = "products")
data class Product(
    @PrimaryKey(autoGenerate = true)
    var pd_id: Long = 0,
    @ColumnInfo(name = "pd_name") var name: String ="",
    @ColumnInfo(name = "pd_price") var price: Double = 0.0,
    @ColumnInfo(name = "pd_brand") var brand: String = "",
    var likes: Int = 0,
    @ColumnInfo(name = "pd_description") var description: String = "",
    @ColumnInfo(name = "pd_imgUrl") var imageUrl: String = ""
//    @Ignore val picture: Bitmap?  忽略字段

) 


@Entity(primaryKeys = ["pd_id", "user_id"])
data class Like(
    val pd_id: Long,
    val user_id: Long

)
```
下一步取决于您想如何查询这些相关实体。
* 如果您想查询用户和每个用户所收藏的商品信息，则应创建一个新的数据类 `UserWithProducts`，其中包含单个 User 对象，以及该用户所收藏的所有 Product 对象的列表。

* 如果您想查询商品和每个商品被那些用户收藏的信息，则应创建一个新的数据类 `ProductWithUsers`，其中包含单个 Product 对象，以及该商品被那些用户收藏的所有 User 对象的列表。
在这两种情况下，都可以通过以下方法在实体之间建立关系：在上述每个类中的 `@Relation` 注释中使用 `associateBy` 属性来确定提供 Product 实体与 User 实体之间关系的交叉引用实体。
```js
class ProductWithUsers(
    @Embedded
    val product: Product,
    @Relation(
        parentColumn = "pd_id",
        entityColumn = "user_id",
        associateBy = Junction(Like::class)
    )
    val users: List<User>

)


class UserWithProducts(
    @Embedded
    val user: User,
    @Relation(
        parentColumn = "user_id",
        entityColumn = "pd_id",

        associateBy = Junction(Like::class)
    )
    val products: List<Product>
    ) 
```
最后，向 DAO 类添加一个方法，用于提供您的应用所需的查询功能。
```js
    @Transaction
    @Query("SELECT * FROM user WHERE user_id Like :user_id")
    abstract fun getFavoritesProducts(user_id: Long): LiveData<List<UserWithProducts>>

    @Transaction
    @Query("SELECT * FROM products")
    fun getProductsWithUsers(): List<ProductWithUsers>
```
这两个方法都需要 Room 运行两次查询，因此应为这两个方法添加 @Transaction 注释，以确保整个操作以原子方式执行。