---
title: go反射构建postgresql&opengauss简易orm
date: 2021.11.29
tags: 
    - Golang
    - 19级
    - 反射
    - 数据库
author: mirrorlied
---



## 前言

本文将通过Go的反射机制，实现对OpenGauss(兼容Postgresql)的一个简单orm。

### 连接数据库

支持Postgresql的驱动一般都是支持OpenGauss的，这里我选择的是`bmizeran/pq`作为驱动。

首先，我们需要先构建配置文件：

```ini
# app.ini
[database]
User = jack
Password = ********
Host = **.**.**.**
Port = 26000
Name = postgres
```

然后构建一个Setup函数，将配置文件进行配置，然后使用sql.open()实现连接操作：

```go
// models.go
package models

import (
   "database/sql"
   "fmt"
   _ "github.com/bmizerany/pq"
   "log"
   "main/pkg/setting"
)

var db *sql.DB

// Setup 初始化数据库
func Setup() {
   var err error
   dsn := fmt.Sprintf("host=%s port=%d user=%s dbname=%s password=%s sslmode=disable TimeZone=Asia/Shanghai",
         setting.DatabaseSetting.Host,
         setting.DatabaseSetting.Port,
         setting.DatabaseSetting.User,
         setting.DatabaseSetting.Name,
         setting.DatabaseSetting.Password,
      )
   db, err = sql.Open("postgres", dsn)
   if err != nil {
      log.Fatalf("数据库配置错误: %v", err)
   }
}
```

这里需要注意，虽然看起来好像没有使用到bmizerany/pg，但是我们仍需要进行导入（需要加个_），导入后会在后台引入一个postgres驱动，否则会报以下错误：

![07$KK__WRX7H3KT}3{3FO.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a79e0661605a4db298515569c7ab50b5~tplv-k3u1fbpfcp-watermark.awebp?)

类比Python库，两者的关系类似于BeautifulSoup和lxml的关系，这里就不展开了。

### 自动建表

这里先展示一段使用sql语句实现的建表操作：

```sql
CREATE TABLE Account (
   id serial PRIMARY KEY,
   name VARCHAR(64) NOT NULL,
   username VARCHAR(64) NOT NULL,
   password VARCHAR(64) NOT NULL,
   role_id INT NOT NULL,
   FOREIGN KEY(role_id) REFERENCES Role(id)
);
```

这里我们一共出现了5条字段和一个额外行与role表用于建立关联。

5条字段我们不难发现创建结构是相同的： `字段名 类型 约束`

因此需要实现自动建表，实践上是想办法从结构体struct中提取出对应部分，想要实现这种功能，理所当然想到使用反射和标签进行实现。

下面是一种比较方便编码实现的方案：

```go
type Account struct {
   Id       string `json:"id" type:"serial" constraint:"PRIMARY KEY"`
   Name     string `json:"name" type:"VARCHAR(64)" constraint:"NOT NULL"`
   Username string `json:"account" type:"VARCHAR(64)" constraint:"NOT NULL"`
   Password string `json:"password" type:"VARCHAR(64)" constraint:"NOT NULL"`
   RoleId   string `json:"role_id" type:"INT" constraint:"NOT NULL"`

   extra string `constraint:"FOREIGN KEY(role_id) REFERENCES Role(id)"`
}
```

各参数提取方式如下：

1. 字段名 => 反射获取标签中的json
2. 类型 => 反射获取标签中的type
3. 约束 => 反射获取标签中的constraint

然后使用extra的constraint保存其他出现的关联、约束条件。

这里也有很多不合理的地方，例如：字段名可以通过反射Field.Name获取，数据类型可以通过Field的数据类型实现自动设置，关联等条件也可以通过Tag自动生成等。但是会增加较大编码量，等后面再优化吧。

上述功能的代码实现大体如下，主要通过字符串拼接的方式生成sql语句，然后交付数据库处理：

```go
// CreateTable 创建表
func CreateTable(tables []interface{})  {
   for _, table := range tables {
      t := reflect.TypeOf(table)
      tableName := strings.Split(t.String(), ".")[1]
      sql := "CREATE TABLE " + tableName + " (\n"
      for i := 0; i < t.Elem().NumField(); i++ {
         field := t.Elem().Field(i)
         if i == t.Elem().NumField() - 1 {
            extra := field.Tag.Get("constraint")
            if extra != "" {
               sql += ",\n\t" + extra
            }
            break
         }
         if i != 0 {
            sql += ",\n\t"
         }
         sql += fmt.Sprintf(
            "%s %s %s",
            field.Tag.Get("json"),
            field.Tag.Get("type"),
            field.Tag.Get("constraint"),
         )
      }

      sql += "\n);"
      logrus.Debugln(sql)
      _, err := db.Exec(sql)
      if err != nil {
         logrus.Errorln(tableName, "创建失败:", err.Error())
      } else {
         logrus.Println(tableName, "创建成功")
      }

   }
}
```

其中，出现了一个循环，通过反射的方式，获取我们编写的结构体的各field：

```go
t := reflect.TypeOf(table)
for i := 0; i < t.Elem().NumField(); i++ {
    field := t.Elem().Field(i)
}
```

获取到了每个field，实现标签读取就简单了：

```go
name := field.Tag.Get("json")
dType := field.Tag.Get("type")
constraint := field.Tag.Get("constraint")
```

#### 使用方法

```go
CreateTable([]interface{}{
   &Role{},
   &Account{},
})
```

由于存在外键关系，我们需要先传教Role表，然后构建Account表，在数据库创建后，执行此代码，结果如下：

![MI72D.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6cd4fa6f075d493e93ec292a964b325e~tplv-k3u1fbpfcp-watermark.awebp?)

![LG.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7545b0360cb0454aa0aa382cc12ce218~tplv-k3u1fbpfcp-watermark.awebp?)

运行结果符合开始的预期。



### 单数据查询

#### 生成sql语句

查询的基础是构建一条正确的sql语句，对于查询操作，我们直接使用SELECT *来获取行各字段，我们需要构建下面一条语句：

```sql
SELECT * FROM table WHERE id = 1;
```

可以看出，我们需要实现对所有表进行查询操作，需要两个变量：

1. 查询表名称
2. 约束条件

对于表名，我们直接使用reflect.typeOf(model)反射出表名：

```go
   t := reflect.TypeOf(model)
   tableName := strings.Split(t.String(), ".")[1]
```

对于约束条件来说，作为参数直接输出即可，完成代码如下：

```go
// First 按字段名查询单个
// odds => WHERE id = 1 || LIKE name = '%cj%' || ......
func First(model interface{}, odds string) error {
   // 反射获取表名
   t := reflect.TypeOf(model)
   tableName := strings.Split(t.String(), ".")[1]
   // 构建查询语句并交付数据库查询
   sql := "SELECT * FROM " + tableName + " " + odds
   return getFirst(model, sql)
}
```

#### 读取返回数据

上面构建sql语句还是挺容易的，但是要实现返回的将赋值给输出的model对象就难了，下面分几个部分进行实现：

##### 一、将sql语句交付数据库查询

```go
   logrus.Debugln(sql)
   rows, err := db.Query(sql)
   if err != nil {
      return err
   }
   defer rows.Close()
```

首先，我们先使用db.Query(sql)，将sql交付给openGauss数据库，然后返回一个rows对象，此时，如果没有报错的话，数据库返回的数据会被存储在rows内。

##### 二、存储返回数据

想要读取rows中的数据，我们需要先执行rows.Next()，该函数会返回一个bool类型数据，如果可以读取到数据，则会返回true，否则返回false。

另外，换行操作也封装在rows.Next()中，因此在读取行之前，需要先执行一次rows.Next()，然后才能读到数据。

构建代码如下：

```go
   if !rows.Next() {
      return errors.New("sql: Scan called without calling Next")
   }
```

这里我们只读首行，多行读取可以通过for循环进行，但是在赋值方面差异较大，我们后面再讨论。

然后我们在需要读取出rows中的数据之前，需要先读取rows.Columns，即行的字段，通过len()读取出字段数量，然后用该长度构建一个slice：

```go
   columns, err := rows.Columns()
   if err != nil {
      return err
   }
```

至于为什么要该数据，是因为我们需要使用rows.Scan()来读取数据，Scan函数参数如下：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f3dadc25766e4f88835000e0df83140a~tplv-k3u1fbpfcp-watermark.awebp?)

可以看出需要传入的是可变参数，而传入的参数数量需要与SELECT返回回来的个数相同，否则会报错，因此我们使用slice的一个特性，使用`...`也传一个可变的入参。

代码实现如下，由于Scan输入为指针类型，需要在输入前对values进行初始化：

```go
   values := make([]interface{}, len(columns))
   for i := range values {
      values[i] = new(string)
   }
   if err := rows.Scan(values...); err != nil {
      return err
   }
```

然后，然后返回值就会保存在values数值中了。

##### 三、构建map，转移数据

由于在slice中的数据是没有字段数据的，因此我们需要使用上面的columns和values共同构建一个map，用于存储读取到的数据，详情见代码即可。

代码实现如下：

```go
   // 4、构建map，将数组数据转移到map缓存中
   m := make(map[string]interface{})
   for i, column := range columns {
      m[strings.ToUpper(column[:1]) + column[1:]] = *values[i].(*string)
   }
```

##### 四、反射数据到原对象

然后就是查询操作中最重要的一个环节：将数据反射到原对象中。

这里使用的是反射中的reflect.ValueOf()函数，获得原对象的Value类型数据，然后通过其下的.Elem().FieldByName函数，通过字段名获取到我们需要修改字段的值位置，最后使用SetString()函数实现赋值操作。而所使用的字段名和之前一样，通过reflect.TypeOf()函数，然后遍历获取。

代码实现如下：

```go
func getFirst(model interface{}, sql string) error {
   // 5、使用反射将map映射到原结构体中
   t := reflect.TypeOf(model)
   for i := 0; i < t.Elem().NumField(); i++ {
      field := t.Elem().Field(i)
      if m[field.Name] != nil {
         v := reflect.ValueOf(model).Elem().FieldByName(field.Name)
         v.SetString(m[field.Name].(string))
      }
   }
   return nil
}
```

#### 调用函数

构建一个空结构体，然后调用函数对齐赋值：

```go
r := &models.Role{}
err := models.First(r, "WHERE id = 1")
if err != nil {
   fmt.Println(err)
}
fmt.Println(r)
复制代码
```

输出如下：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/afe26dd00d14402d8884d6bedc1f2762~tplv-k3u1fbpfcp-watermark.awebp?)

### 数据插入

#### 一、使用Insert函数

```go
r := &models.Role {
   Name:         "Admin",
   Introduction: "管理员",
}
err := models.Insert(r)
if err != nil {
   fmt.Println(err)
}
```

我们的最终目的就是构建这样一个Insert函数，将我们之前创建的数据库结构体对象此函数实现数据插入数据库的操作，且返回可能出现的错误。

#### 二、编写Insert函数

Insert()函数分为两个部分，一是生成sql语句，二是将sql语句交付数据库执行并捕获错误，其中的重难点在于前者。

下面是一个简单的插入函数，是我们上面Insert函数想要达到的效果(其中主键id由serial数据类型实现自增)：

```sql
INSERT INTO Role(Name,Introduction)
VALUES ('Admin','管理员')
```

看过前面两部分应该已经知道，这里我们需要使用反射reflect.TypeOf()提取出Role对象的所有字段。

但是，仅仅这样是不够的，因为很多时候Role对象内部分字段值为空，表示用户不希望对这些字段进行赋值。

由于对象值的获取是使用reflect.ValueOf()存放的，因此我们需要在读取到值后做进一步的判断，放弃空值字段。

而字段和值恰好是一一对应的关系，可使用map很方便的进行存储，并在后续过程中用于拼接sql语句。

#### 三、交付执行sql语句

字符串拼接这块比较简单（但是要主要行结尾逗号需匹配），就不展开了，最后使用db.Exec(sql)将sql语句交付数据库即可实现插入操作。

#### 四、完整代码实现

```go
// Insert 插入
func Insert(model interface{}) error {
   // 把存在的字段插入
   t := reflect.TypeOf(model)
   mp := make(map[string]string)
   for i := 0; i < t.Elem().NumField(); i++ {
      field := t.Elem().Field(i)
      name := field.Name
      value := reflect.ValueOf(model).Elem().FieldByName(name).String()
      if value == "" {
         continue
      }
      mp[name] = value
   }
   tableName := strings.Split(t.String(), ".")[1]
   var kStr string
   var vStr string
   for k, v := range mp {
      kStr += k + ","
      vStr += "'" + v + "',"
   }
   kStr = kStr[:len(kStr)-1]
   vStr = vStr[:len(vStr)-1]
   sql := fmt.Sprintf("INSERT INTO %s(%s)\nVALUES (%s)", tableName, kStr, vStr)
   logrus.Debugln(sql)
   _, err := db.Exec(sql)
   return err
}
```

### 数据修改

很数据插入操作类似，我们需要实现将数据中值为空的数据对应的字段提取出来，先构建一个map对象存储，然后通过字符串拼接的方式构建需要的sql语句，因此在这部分，代码和之前完全一致：

```go
t := reflect.TypeOf(model)
mp := make(map[string]string)
for i := 0; i < t.Elem().NumField(); i++ {
   field := t.Elem().Field(i)
   name := field.Name
   value := reflect.ValueOf(model).Elem().FieldByName(name).String()
   if value == "" {
      continue
   }
   mp[name] = value
}
```

和数据插入不同，我们在执行插入操作是，主键是由我们自己决定的，但是修改和删除操作，需要获取到主键，通过添加WHERE查询条件，才能定位数据，实现修改和删除。

当然，由于之前建表的时候我们已经将主键的信息存在Tags内了，因此我们只需要反射使用field.Tag.Get读取标签，遍历搜索主键名，然后再使用reflect.ValueOf，实现主键值的获取。

此部分代码实现如下：
```go
// getPrimaryKey 获取结构体主键名和对应值
func getPrimaryKey(model interface{}) (name string, value string, err error) {
   // 反射获取主键名和对应值
   t := reflect.TypeOf(model)
   for i := 0; i < t.Elem().NumField(); i++ {
      field := t.Elem().Field(i)
      constraint := field.Tag.Get("constraint")
      if strings.Contains(constraint, "PRIMARY KEY") {
         name = field.Tag.Get("json")
         value = reflect.ValueOf(model).Elem().FieldByName(name).String()
      }
   }
   if name != "" && value != "" {
      return
   }
   err = errors.New("PRIMARY KEY NO FOUND")
   return
}
```

另外，数据修改和数据插入操作一大不同是sql语句结构的不同，因此，在字符串拼接实现上，存在较大差异。

数据修改操作，完整代码实现如下：
```go
name, value, err := getPrimaryKey(model)
if err != nil {
   return err
}
tableName := strings.Split(t.String(), ".")[1]
sql := "UPDATE " + tableName + " SET\n"
tmp := 0
for k, v := range mp {
   sql += fmt.Sprintf("\t%s = '%s'", k, v)
   tmp++
   if tmp != len(mp) {
      sql += ",\n"
   }
}
```

使用方法如下：
```go
err := models.Update(&models.Role{
   Id:           "1",
   Name:         "Admin666",
   Introduction: "管理员666",
})
fmt.Println(err)
```

生成的sql语句：
```sql
UPDATE Role SET
    Id = '1',
    Name = 'Admin666',
    Introduction = '管理员666'
WHERE Id = 1;
```

### 数据删除

数据删除比较简单，一个删除语句如下：
```sql
DELETE FROM Role
WHERE id = 1;
```

可以看到，除表名外，我们只需要主键和主键名，就可以实现删除了。

凑巧的是，这两参数，我们在实现上面数据修改的时候已经完成了，因此直接写出代码：

```go
func Delete(model interface{}) error {
   t := reflect.TypeOf(model)
   tableName := strings.Split(t.String(), ".")[1]
   sql := "DELETE FROM " + tableName
   name, value, err := getPrimaryKey(model)
   if err != nil {
      return err
   }
   sql += fmt.Sprintf("\nWHERE %s = %s;", name, value)
   logrus.Debugln(sql)
   _, err = db.Exec(sql)
   return err
}
```

使用方式如下：
```go
err := models.Delete(&models.Role{
   Id:           "1",
})
fmt.Println(err)
```

生成的sql语句就是上面展示的那个。