# rails学习

## 建立工程

```
rails new -M --skip-active-storage cookbook3
```

创建数据对象：

```
rails generate scaffold recipe title:string
```

创建数据库（每当修改数据对象后，都需要执行该语句）：

```
rails db:migrate
```

访问服务器：

```
rails server -b localhost
```

增加字段：

```
rails g migration AddInstructionToRecipe instructions:string
```

增加外键：向Recipe加入category外键

```
rails g migration AddCategoryRefToRecipes category:references
```

删除整个数据库：

```
rails db:drop
```

进入数据库命令行：

```
rails dbconsole
.exit
```

## 基本原理

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202310180903028.png" alt="image-20231018090312814" style="zoom:50%;" />

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/202310180904111.png" alt="image-20231018090451995" style="zoom:50%;" />

## 文件结构

* _form.html.erb
