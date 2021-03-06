---
title: GORM 2.0 发布说明
layout: page
---

GORM 2.0 完全从零开始，引入了一些不兼容的 API 变更和许多改进

**摘要**

* 性能改进
* 代码模块化
* Context，批量插入，预编译模式，DryRun 模式，Join 预加载，Find To Map，Create From Map，FindInBatches
* 支持嵌套事务，SavePoint，Rollback To SavePoint
* SQL 生成器，命名参数，分组条件，Upsert，锁， 支持 Optimizer/Index/Comment Hint，子查询改进，使用SQL表达式、Context Valuer 进行 CRUD
* 支持完整的自引用，改进 Join Table，批量数据的关联模式
* 允许多个字段用于追踪 create、update 时间 ，支持 UNIX （毫/纳）秒
* 支持字段权限：只读、只写、只创建、只更新、忽略
* 新的插件系统，为多个数据库提供了官方插件，读写分离，prometheus 集成...
* 全新的 Hook API：带插件的统一接口
* 全新的 Migrator：允许为关系创建数据库外键，更智能的 AutoMigrate，支持约束、检查器，增强索引支持
* 全新的 Logger：支持 context、改进可扩展性
* 统一命名策略：表名、字段名、连接表名、外键、检查器、索引名称规则
* 更好的自定义类型支持（例如： JSON）

## 如何升级

* GORM 的开发已经迁移至 [github.com/go-gorm](https://github.com/go-gorm)，import 路径也修改为 `gorm.io/gorm` ，对于以前的项目，您可以继续使用 `github.com/jinzhu/gorm` 和 [GORM V1 文档](http://v1.gorm.io/zh_CN/)
* 数据库驱动被拆分为独立的项目，例如：[github.com/go-gorm/sqlite](https://github.com/go-gorm/sqlite)，且它的 import 路径也变更为 `gorm.io/driver/sqlite`

### 安装

```go
go get gorm.io/gorm
// **注意** GORM `v2.0.0` 发布的 git tag 是 `v1.20.0`
```

### 快速开始

```go
import (
  "gorm.io/gorm"
  "gorm.io/driver/sqlite"
)

func init() {
  db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{})

  // 大部分 CRUD API 都是兼容的
  db.AutoMigrate(&Product{})
  db.Create(&user)
  db.First(&user, 1)
  db.Model(&user).Update("Age", 18)
  db.Model(&user).Omit("Role").Updates(map[string]interface{}{"Name": "jinzhu", "Role": "admin"})
  db.Delete(&user)
}
```

## 主要特性

此发布说明仅涵盖了 GORM V2 中的重大更改，作为快速参考

#### Context 支持

* 通过 `WithContext` 方法提供 `context.Context` 支持
* Logger 也支持用于追踪的 context

```go
DB.WithContext(ctx).Find(&users)
```

#### 批量插入

* 将切片数据传递给 `Create` 方法，GORM 将生成单个 SQL 语句来插入所有数据，并回填主键的值。
* 如果这些数据包含关联，将使用另一个 SQL 语句 upsert 所有关联
* 批量插入的数据依然会调用它的 `钩子` 方法（Before/After Create/Save）

```go
var users = []User{{Name: "jinzhu1"}, {Name: "jinzhu2"}, {Name: "jinzhu3"}}
DB.Create(&users)

for _, user := range users {
  user.ID // 1,2,3
}
```

#### 预编译模式

预编译模式会预编译 SQL 执行语句，以加速后续执行速度

```go
// 全局模式，所有的操作都会创建并缓存预编译语句，以加速后续执行速度
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{PrepareStmt: true})

// 会话模式，当前会话中的操作会创建并缓存预编译语句
tx := DB.Session(&Session{PrepareStmt: true})
tx.First(&user, 1)
tx.Find(&users)
tx.Model(&user).Update("Age", 18)
```

#### DryRun 模式

DarRun 模式会生成但不执行 SQL，可以用于检查、测试生成的 SQL

```go
stmt := DB.Session(&Session{DryRun: true}).Find(&user, 1).Statement
stmt.SQL.String() //=> SELECT * FROM `users` WHERE `id` = $1 // PostgreSQL
stmt.SQL.String() //=> SELECT * FROM `users` WHERE `id` = ?  // MySQL
stmt.Vars         //=> []interface{}{1}
```

#### Joins 预加载

使用 INNER JOIN 预加载关联，并处理 null 数据避免 scan 失败

```go
DB.Joins("Company").Joins("Manager").Joins("Account").Find(&users, "users.id IN ?", []int{1,2})
```

#### Find To Map

Scan 结果到 `map[string]interface{}` 或 `[]map[string]interface{}`

```go
var result map[string]interface{}
DB.Model(&User{}).First(&result, "id = ?", 1)
```

#### Create From Map

根据 `map[string]interface{}` 或 `[]map[string]interface{}` Create

```go
DB.Model(&User{}).Create(map[string]interface{}{"Name": "jinzhu", "Age": 18})

datas := []map[string]interface{}{
  {"Name": "jinzhu_1", "Age": 19},
  {"name": "jinzhu_2", "Age": 20},
}

DB.Model(&User{}).Create(datas)
```

#### FindInBatches

批量查询并处理记录

```go
result := DB.Where("age>?", 13).FindInBatches(&results, 100, func(tx *gorm.DB, batch int) error {
  // 批量处理
  return nil
})
```

#### 嵌套事务

```go
DB.Transaction(func(tx *gorm.DB) error {
  tx.Create(&user1)

  tx.Transaction(func(tx2 *gorm.DB) error {
    tx.Create(&user2)
    return errors.New("rollback user2") // rollback user2
  })

  tx.Transaction(func(tx2 *gorm.DB) error {
    tx.Create(&user3)
    return nil
  })

  return nil // commit user1 and user3
})
```

#### SavePoint，RollbackTo

```go
tx := DB.Begin()
tx.Create(&user1)

tx.SavePoint("sp1")
tx.Create(&user2)
tx.RollbackTo("sp1") // rollback user2

tx.Commit() // commit user1
```

#### 命名参数

GORM 支持使用 `sql.NamedArg`，`map[string]interface{}` 作为命名参数

```go
DB.Where("name1 = @name OR name2 = @name", sql.Named("name", "jinzhu")).Find(&user)
// SELECT * FROM `users` WHERE name1 = "jinzhu" OR name2 = "jinzhu"

DB.Where("name1 = @name OR name2 = @name", map[string]interface{}{"name": "jinzhu2"}).First(&result3)
// SELECT * FROM `users` WHERE name1 = "jinzhu2" OR name2 = "jinzhu2" ORDER BY `users`.`id` LIMIT 1

DB.Raw(
  "SELECT * FROM users WHERE name1 = @name OR name2 = @name2 OR name3 = @name",
  sql.Named("name", "jinzhu1"), sql.Named("name2", "jinzhu2"),
).Find(&user)
// SELECT * FROM users WHERE name1 = "jinzhu1" OR name2 = "jinzhu2" OR name3 = "jinzhu1"

DB.Exec(
  "UPDATE users SET name1 = @name, name2 = @name2, name3 = @name",
  map[string]interface{}{"name": "jinzhu", "name2": "jinzhu2"},
)
// UPDATE users SET name1 = "jinzhu", name2 = "jinzhu2", name3 = "jinzhu"
```

#### 分组条件

```go
db.Where(
  db.Where("pizza = ?", "pepperoni").Where(db.Where("size = ?", "small").Or("size = ?", "medium")),
).Or(
  db.Where("pizza = ?", "hawaiian").Where("size = ?", "xlarge"),
).Find(&pizzas)

// SELECT * FROM pizzas WHERE (pizza = 'pepperoni' AND (size = 'small' OR size = 'medium')) OR (pizza = 'hawaiian' AND size = 'xlarge')
```

#### 子查询

```go
// Where 子查询
db.Where("amount > (?)", db.Table("orders").Select("AVG(amount)")).Find(&orders)

// From 子查询
db.Table("(?) as u", DB.Model(&User{}).Select("name", "age")).Where("age = ?", 18}).Find(&User{})
// SELECT * FROM (SELECT `name`,`age` FROM `users`) as u WHERE age = 18

// Update 子查询
DB.Model(&user).Update(
  "price", DB.Model(&Company{}).Select("name").Where("companies.id = users.company_id"),
)
```

#### Upsert

`clause.OnConflict` 为不同的数据库（SQLite，MySQL，PostgreSQL，SQL Server）提供了兼容的 Upsert 支持

```go
import "gorm.io/gorm/clause"

DB.Clauses(clause.OnConflict{DoNothing: true}).Create(&users)

DB.Clauses(clause.OnConflict{
  Columns:   []clause.Column{{Name: "id"}},
  DoUpdates: clause.Assignments(map[string]interface{}{"name": "jinzhu", "age": 18}),
}).Create(&users)
// MERGE INTO "users" USING *** WHEN NOT MATCHED THEN INSERT *** WHEN MATCHED THEN UPDATE SET ***; SQL Server
// INSERT INTO `users` *** ON DUPLICATE KEY UPDATE name="jinzhu", age=18; MySQL

DB.Clauses(clause.OnConflict{
  Columns:   []clause.Column{{Name: "id"}},
  DoUpdates: clause.AssignmentColumns([]string{"name", "age"}),
}).Create(&users)
// MERGE INTO "users" USING *** WHEN NOT MATCHED THEN INSERT *** WHEN MATCHED THEN UPDATE SET "name"="excluded"."name"; SQL Server
// INSERT INTO "users" *** ON CONFLICT ("id") DO UPDATE SET "name"="excluded"."name", "age"="excluded"."age"; PostgreSQL
// INSERT INTO `users` *** ON DUPLICATE KEY UPDATE `name`=VALUES(name),`age=VALUES(age); MySQL
```

#### Locking

```go
DB.Clauses(clause.Locking{Strength: "UPDATE"}).Find(&users)
// SELECT * FROM `users` FOR UPDATE

DB.Clauses(clause.Locking{
  Strength: "SHARE",
  Table: clause.Table{Name: clause.CurrentTable},
}).Find(&users)
// SELECT * FROM `users` FOR SHARE OF `users`
```

#### Optimizer/Index/Comment Hint

```go
import "gorm.io/hints"

// Optimizer Hint
DB.Clauses(hints.New("hint")).Find(&User{})
// SELECT * /*+ hint */ FROM `users`

// Index Hint
DB.Clauses(hints.UseIndex("idx_user_name")).Find(&User{})
// SELECT * FROM `users` USE INDEX (`idx_user_name`)

// Comment Hint
DB.Clauses(hints.Comment("select", "master")).Find(&User{})
// SELECT /*master*/ * FROM `users`;
```

查看 [Hint](hints.html) 获取详情

#### 使用 SQL 表达式、Context Valuer 进行 CRUD

```go
type Location struct {
    X, Y int
}

func (loc Location) GormDataType() string {
  return "geometry"
}

func (loc Location) GormValue(ctx context.Context, db *gorm.DB) clause.Expr {
  return clause.Expr{
    SQL:  "ST_PointFromText(?)",
    Vars: []interface{}{fmt.Sprintf("POINT(%d %d)", loc.X, loc.Y)},
  }
}

DB.Create(&User{
  Name:     "jinzhu",
  Location: Location{X: 100, Y: 100},
})
// INSERT INTO `users` (`name`,`point`) VALUES ("jinzhu",ST_PointFromText("POINT(100 100)"))

DB.Model(&User{ID: 1}).Updates(User{
  Name:  "jinzhu",
  Point: Point{X: 100, Y: 100},
})
// UPDATE `user_with_points` SET `name`="jinzhu",`point`=ST_PointFromText("POINT(100 100)") WHERE `id` = 1
```

查看 [自定义数据类型](data_types.html#gorm_valuer_interface) 获取详情

#### 字段权限

支持字段权限，权限级别有：只读、只写、只创建、只更新、忽略

```go
type User struct {
  Name string `gorm:"<-:create"` // 允许读和创建
  Name string `gorm:"<-:update"` // 允许读和更新
  Name string `gorm:"<-"`        // 允许读和写（创建和更新）
  Name string `gorm:"->:false;<-:create"` // 只创建
  Name string `gorm:"->"` // 只读
  Name string `gorm:"-"`  // 忽略
}
```

#### 支持多个字段追踪 create/update 时间（ time、unix (毫/纳) 秒）

```go
type User struct {
  CreatedAt time.Time // 在创建时，如果该字段值为零值，则使用当前时间填充
  UpdatedAt int       // 在创建时该字段值为零值或者在更新时，使用当前时间戳的秒数填充
  Updated   int64 `gorm:"autoUpdateTime:nano"` // 使用时间戳的纳秒数填充更新时间
  Updated2   int64 `gorm:"autoUpdateTime:milli"` // 使用时间戳的毫秒数填充更新时间
  Created   int64 `gorm:"autoCreateTime"`      // 使用时间戳的秒数填充创建时间
}
```

#### 多数据库，读写分离

GORM 通过 `DB Resolver` 插件提供了多数据库，读写分离支持。该插件还支持基于当前 struct 和表自动切换数据库和表，自定义负载均衡逻辑的多 source、replica

查看 [Database Resolver](dbresolver.html) 获取详情

#### Prometheus

GORM 提供了 `Prometheus` 插件来收集 `DBStats` 和用户自定义指标

查看 [Prometheus](prometheus.html) 获取详情

#### 命名策略

GORM 允许用户通过覆盖默认的 `命名策略` 更改默认的命名约定，命名策略被用于构建： `TableName`、`ColumnName`、`JoinTableName`、`RelationshipFKName`、`CheckerName`、`IndexName`。查看 [GORM 配置](gorm_config.html) 获取详情

```go
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  NamingStrategy: schema.NamingStrategy{TablePrefix: "t_", SingularTable: true},
})
```

#### Logger

* Context 支持
* 自定义或关闭日志的颜色
* 慢 SQL 日志，慢 SQL 默认阈值是 100ms
* 优化了 SQL 日志格式，可以更方便的复制到数据库控制台中执行

#### 事务模式

默认情况下，GORM 所有的写操作都会在事务中运行，以确保数据的一致性。 如果不需要，您可以在初始化时禁用它来加速写入操作

```go
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  SkipDefaultTransaction: true,
})
```

#### 数据类型（以 JSON 为例）

GORM 优化了对自定义类型的支持，现在您可以定义一个 struct 来支持所有类型的数据库

下面以 JSON 为例（支持 SQLite、MySQL、Postgres。参考自：https://github.com/go-gorm/datamypes/blob/master/json.go）

```go
import "gorm.io/datatypes"

type User struct {
  gorm.Model
  Name       string
  Attributes datatypes.JSON
}

DB.Create(&User{
  Name:       "jinzhu",
  Attributes: datatypes.JSON([]byte(`{"name": "jinzhu", "age": 18, "tags": ["tag1", "tag2"], "orgs": {"orga": "orga"}}`)),
}

// 查询 attributes 中有 role 字段的 user
DB.First(&user, datatypes.JSONQuery("attributes").HasKey("role"))
// 查询 attributes 中有 orgs->orga 字段的 user
DB.First(&user, datatypes.JSONQuery("attributes").HasKey("orgs", "orga"))
```

#### Smart Select

GORM 可以通过 [`Select`](query.html) 选择指定的字段，而在 V2 中，通过一个较小的 struct，可以使用 GORM 提供的 smart select 模式

```go
type User struct {
  ID     uint
  Name   string
  Age    int
  Gender string
  // 假设后面还有几百个字段
}

type APIUser struct {
  ID   uint
  Name string
}

// 查询时会自动 select `id`, `name`
db.Model(&User{}).Limit(10).Find(&APIUser{})
// SELECT `id`, `name` FROM `users` LIMIT 10
```

#### 批量关联模式

关联模式也支持批量处理，例如：

```go
// 查询所有用户的所有角色
db.Model(&users).Association("Role").Find(&roles)

// 将 userA 移出所有的 Team
db.Model(&users).Association("Team").Delete(&userA)

// 获取所有 Team 成员的不重复计数
db.Model(&users).Association("Team").Count()

// 对于 `Append`、`Replace` 的批量处理，参数与数据的长度必须相等，否则会返回错误
var users = []User{user1, user2, user3}
// 例如：我们有 3 个 user，将 userA 添加到 user1 的 Team，将 userB 添加到 user2 的 Team，将 userA、userB、userC 添加到 user3 的 Team
db.Model(&users).Association("Team").Append(&userA, &userB, &[]User{userA, userB, userC})
// 将 user1 的 Team 重置为 userA，将 user2的 team 重置为 userB，将 user3 的 team 重置为 userA、userB 和 userC
db.Model(&users).Association("Team").Replace(&userA, &userB, &[]User{userA, userB, userC})
```

#### Delete Associations when deleting

You are allowed to delete selected has one/has many/many2many relations with `Select` when deleting records, for example:

```go
// delete user's account when deleting user
db.Select("Account").Delete(&user)

// delete user's Orders, CreditCards relations when deleting user
db.Select("Orders", "CreditCards").Delete(&user)

// delete user's has one/many/many2many relations when deleting user
db.Select(clause.Associations).Delete(&user)

// delete users's account when deleting users
db.Select("Account").Delete(&users)
```

## 破坏性变更

We are trying to list big breaking changes or those changes can't be caught by the compilers, please create an issue or pull request [here](https://github.com/go-gorm/gorm.io) if you found any unlisted breaking changes

#### Tags

* GORM V2 使用 `camelCase` 风格的 tag 名。`snake_case` 风格的 tag 已经失效，例如： `auto_increment`、`unique_index`、`polymorphic_value`、`embeded_prefix`，查看 [Model Tag](models.html#tags) 获取详情
* 用于指定外键的 tag 已变更为 `foreignKey`，`references`，查看 [Association Tag](associations.html#tags) 获取详情

#### Table Name

`TableName` will *not* allow dynamic table name anymore, the result of `TableName` will be cached for future

```go
func (User) TableName() string {
  return "t_user"
}
```

Please use `Scopes` for dynamic tables, for example:

```go
func UserTable(u *User) func(*gorm.DB) *gorm.DB {
  return func(db *gorm.DB) *gorm.DB {
    return db.Table("user_" + u.Role)
  }
}

DB.Scopes(UserTable(&user)).Create(&user)
```

#### Method Chain Safety/Goroutine Safety

To reduce GC allocs, GORM V2 will share `Statement` when using method chains, and will only create new `Statement` instances for new initialized `*gorm.DB` or after a `New Session Method`, to reuse a `*gorm.DB`, you need to make sure it just after a `New Session Method`, for example:

```go
db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})

// Safe for new initialized *gorm.DB
for i := 0; i < 100; i++ {
  go db.Where(...).First(&user)
}

tx := db.Where("name = ?", "jinzhu")
// NOT Safe as reusing Statement
for i := 0; i < 100; i++ {
  go tx.Where(...).First(&user)
}

ctxDB := db.WithContext(ctx)
// Safe after a `New Session Method`
for i := 0; i < 100; i++ {
  go ctxDB.Where(...).First(&user)
}

ctxDB := db.Where("name = ?", "jinzhu").WithContext(ctx)
// Safe after a `New Session Method`
for i := 0; i < 100; i++ {
  go ctxDB.Where(...).First(&user) // `name = 'jinzhu'` will apply to the query
}

tx := db.Where("name = ?", "jinzhu").Session(&gorm.Session{WithConditions: true})
// Safe after a `New Session Method`
for i := 0; i < 100; i++ {
  go tx.Where(...).First(&user) // `name = 'jinzhu'` will apply to the query
}
```

Check out [Method Chain](method_chaining.html) for details

#### Default Value

GORM V2 won't auto-reload default values created with database function after creating, checkout [Default Values](create.html#default_values) for details

#### Soft Delete

GORM V1 will enable soft delete if the model has a field named `DeletedAt`, in V2, you need to use `gorm.DeletedAt` for the model wants to enable the feature, e.g:

```go
type User struct {
  ID        uint
  DeletedAt gorm.DeletedAt
}

type User struct {
  ID      uint
  // field with different name
  Deleted gorm.DeletedAt
}
```

{% note warn %}
**NOTE:** `gorm.Model` is using `gorm.DeletedAt`, if you are embedding it, nothing needs to change
{% endnote %}

#### BlockGlobalUpdate

GORM V2 enabled `BlockGlobalUpdate` mode by default, to trigger a global update/delete, you have to use some conditions or use raw SQL or enable `AllowGlobalUpdate` mode, for example:

```go
DB.Where("1 = 1").Delete(&User{})

DB.Raw("delete from users")

DB.Session(&gorm.Session{AllowGlobalUpdate: true}).Delete(&User{})
```

#### ErrRecordNotFound

GORM V2 only returns `ErrRecordNotFound` when you are querying with methods `First`, `Last`, `Take` which is expected to return some result, and we have also removed method `RecordNotFound` in V2, please use `errors.Is` to check the error, e.g:

```go
err := DB.First(&user).Error
errors.Is(err, gorm.ErrRecordNotFound)
```

#### Hooks Method

Before/After Create/Update/Save/Find/Delete must be defined as a method of type `func(tx *gorm.DB) error` in V2, which has unified interfaces like plugin callbacks, if defined as other types, a warning log will be printed and it won't take effect, check out [Hooks](hooks.html) for details

```go
func (user *User) BeforeCreate(tx *gorm.DB) error {
  // Modify current operation through tx.Statement, e.g:
  tx.Statement.Select("Name", "Age")
  tx.Statement.AddClause(clause.OnConflict{DoNothing: true})

  // Operations based on tx will runs inside same transaction without clauses of current one
  var role Role
  err := tx.First(&role, "name = ?", user.Role).Error
  // SELECT * FROM roles WHERE name = "admin"
  return err
}
```

#### Update Hooks support `Changed` to check fields changed or not

When updating with `Update`, `Updates`, You can use `Changed` method in Hooks `BeforeUpdate`, `BeforeSave` to check a field changed or not

```go
func (user *User) BeforeUpdate(tx *gorm.DB) error {
  if tx.Statement.Changed("Name", "Admin") { // if Name or Admin changed
    tx.Statement.SetColumn("Age", 18)
  }

  if tx.Statement.Changed() { // if any fields changed
    tx.Statement.SetColumn("Age", 18)
  }
  return nil
}

DB.Model(&user).Update("Name", "Jinzhu") // update field `Name` to `Jinzhu`
DB.Model(&user).Updates(map[string]interface{}{"name": "Jinzhu", "admin": false}) // update field `Name` to `Jinzhu`, `Admin` to false
DB.Model(&user).Updates(User{Name: "Jinzhu", Admin: false}) // Update none zero fields when using struct as argument, will only update `Name` to `Jinzhu`

DB.Model(&user).Select("Name", "Admin").Updates(User{Name: "Jinzhu"}) // update selected fields `Name`, `Admin`，`Admin` will be updated to zero value (false)
DB.Model(&user).Select("Name", "Admin").Updates(map[string]interface{}{"Name": "Jinzhu"}) // update selected fields exists in the map, will only update field `Name` to `Jinzhu`

// Attention: `Changed` will only check the field value of `Update` / `Updates` equals `Model`'s field value, it returns true if not equal and the field will be saved
DB.Model(&User{ID: 1, Name: "jinzhu"}).Updates(map[string]interface{"name": "jinzhu2"}) // Changed("Name") => true
DB.Model(&User{ID: 1, Name: "jinzhu"}).Updates(map[string]interface{"name": "jinzhu"}) // Changed("Name") => false, `Name` not changed
DB.Model(&User{ID: 1, Name: "jinzhu"}).Select("Admin").Updates(map[string]interface{"name": "jinzhu2", "admin": false}) // Changed("Name") => false, `Name` not selected to update

DB.Model(&User{ID: 1, Name: "jinzhu"}).Updates(User{Name: "jinzhu2"}) // Changed("Name") => true
DB.Model(&User{ID: 1, Name: "jinzhu"}).Updates(User{Name: "jinzhu"})  // Changed("Name") => false, `Name` not changed
DB.Model(&User{ID: 1, Name: "jinzhu"}).Select("Admin").Updates(User{Name: "jinzhu2"}) // Changed("Name") => false, `Name` not selected to update
```

#### Plugins

Plugin callbacks also need be defined as a method of type `func(tx *gorm.DB) error`, check out [Write Plugins](write_plugins.html) for details

#### Updating with struct

When updating with struct, GORM V2 allows to use `Select` to select zero-value fields to update them, for example:

```go
DB.Model(&user).Select("Role", "Age").Update(User{Name: "jinzhu", Role: "", Age: 0})
```

#### Associations

GORM V1 allows to use some settings to skip create/update associations, in V2, you can use `Select` to do the job, for example:

```go
DB.Omit(clause.Associations).Create(&user)
DB.Omit(clause.Associations).Save(&user)

DB.Select("Company").Save(&user)
```

and GORM V2 doesn't allow preload with `Set("gorm:auto_preload", true)` anymore, you can use `Preload` with `clause.Associations`, e.g:

```go
// preload all associations
db.Preload(clause.Associations).Find(&users)
```

Also, checkout field permissions, which can be used to skip creating/updating associations globally

#### Join Table

In GORM V2, a `JoinTable` can be a full-featured model, with features like `Soft Delete`，`Hooks`, and define other fields, e.g:

```go
type Person struct {
  ID        int
  Name      string
  Addresses []Address `gorm:"many2many:person_addresses;"`
}

type Address struct {
  ID   uint
  Name string
}

type PersonAddress struct {
  PersonID  int
  AddressID int
  CreatedAt time.Time
  DeletedAt gorm.DeletedAt
}

func (PersonAddress) BeforeCreate(db *gorm.DB) error {
  // ...
}

// PersonAddress must defined all required foreign keys, or it will raise error
err := DB.SetupJoinTable(&Person{}, "Addresses", &PersonAddress{})
```

After that, you could use normal GORM methods to operate the join table data, for example:

```go
var results []PersonAddress
DB.Where("person_id = ?", person.ID).Find(&results)

DB.Where("address_id = ?", address.ID).Delete(&PersonAddress{})

DB.Create(&PersonAddress{PersonID: person.ID, AddressID: address.ID})
```

#### Count

Count only accepts `*int64` as the argument

#### Transactions

some transaction methods like `RollbackUnlessCommitted` removed, prefer to use method `Transaction` to wrap your transactions

```go
db.Transaction(func(tx *gorm.DB) error {
  // do some database operations in the transaction (use 'tx' from this point, not 'db')
  if err := tx.Create(&Animal{Name: "Giraffe"}).Error; err != nil {
    // return any error will rollback
    return err
  }

  if err := tx.Create(&Animal{Name: "Lion"}).Error; err != nil {
    return err
  }

  // return nil will commit the whole transaction
  return nil
})
```

Checkout [Transactions](transactions.html) for details

#### Migrator

* Migrator 默认会创建数据库外键
* Migrator 更加独立，重命名了很多 API，以便使用统一 API 接口为每个数据库提供更好的支持
* 如果大小、精度、是否为空可以更改，则 AutoMigrate 会改变列的类型
* 通过 `check` 标签支持检查器
* 增强 `index` 标签的设置

Checkout [Migration](migration.html) for details

```go
type UserIndex struct {
  Name  string `gorm:"check:named_checker,(name <> 'jinzhu')"`
  Name2 string `gorm:"check:(age > 13)"`
  Name4 string `gorm:"index"`
  Name5 string `gorm:"index:idx_name,unique"`
  Name6 string `gorm:"index:,sort:desc,collate:utf8,type:btree,length:10,where:name3 != 'jinzhu'"`
}
```

## Happy Hacking!

<style>
li.toc-item { list-style: none; }
li.toc-item.toc-level-4 { display: none; }
</style>
