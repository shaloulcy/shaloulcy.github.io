---
layout: post 
author: shalou
title:  "gorm的关联问题" 
category: golang
tag: [gorm, golang, mysql]
---

最近正在做一个项目，需要与数据库打交道，因此选用了gorm这个golang的orm框架，gorm比较完善，支持多种关联，包括属于、包含一个、包含多个、以及多对多
## 1. 属于
以用户、角色为例，这里规定，一个用户属且属于一个角色，而同一个角色可以赋予多个用户，因此用户与角色的关系是 n:1

```golang
type User struct {
	ID 			string  `gorm:"primary_key"`
	Name 		string  `gorm:"not null"`
	RoleId    string  `gorm:"not null"`
	Role		Role
}

type Role struct {
	ID 			string `gorm:"primary_key"`
	Name 		string `gorm:"not null"`
}
```

<!-- more -->

我们读取User的时候如何关联到Role呢

```golang
var user1 User
db.Where("id = ?", "1").First(&user1)

// 关联的关键代码
db.Model(&user1).Related(&user1.Role)
```

之所以可以达到这个效果，是因为约定俗成了很多东西

* User有一个成员是RoleId，在这里只能是RoleId，也就是关联的类名加上ID，就是外键，否则，gorm会找到不到关联关系
* User有一个成员类型为Role，且名为Role，这里只能为Role，否则，Related时，需要指定其名称


## 1.1
讨论第一种情况，假如User并没有RoleId这个变量并没有命名为RoleId，而是命名为RoleRefer，这时该如何建立关联管理呢，可以采用以下方式


```golang
type User struct {
	ID 			string  `gorm:"primary_key"`
	Name 		string  `gorm:"not null"`
	RoleRefer    string  `gorm:"not null"`
	Role		Role	`gorm:"ForeignKey:RoleRefer"`
}

type Role struct {
	ID 			string `gorm:"primary_key"`
	Name 		string `gorm:"not null"`
}
```

这里我们需要在Role这个成员那里加上一个gorm tag，即制定外键为RoleRefer，这样做gorm就是到外键成员为RoleRefer，这样做假如还是采用Related这种关联方式还是关联不上，后台会报错，因此需要采用另外一种管理方式，即Association

```golang
var user1 User
db.Where("id = ?", "1").First(&user1)

// 关联的关键代码
db.Model(&user1).Association("Role").First(&user1.Role)
```

在这样我们学习了两种关联方法，即Related和Association，Related当没有使用默认约定时，会关联出错，而Association通过标记ForeignKey tag，可以正常关联，因此推荐使用Association

## 2.2
假如Role成员并不是命名为Role，而是MasterRole

```golang
type User struct {
	ID 			string  `gorm:"primary_key"`
	Name 		string  `gorm:"not null"`
	RoleId    string  `gorm:"not null"`
	MasterRole		Role
}

type Role struct {
	ID 			string `gorm:"primary_key"`
	Name 		string `gorm:"not null"`
}
```

可以通过以下方式关联

```golang
var user1 User
db.Where("id = ?", "1").First(&user1)

// 关联的关键代码
db.Model(&user1).Related(&user1.MasterRole, "MasterRole")
// 或者
db.Model(&user1).Association("MasterRole").First(&user1.MasterRole)
```

也就是关联的时候，需要指定是哪个成员变量关联，这里是MasterRole

以上两个问题在包含一个、包含多个、多对多情况下也存在，解决办法一样

## 2. 包含一个
这里以用户和邮箱为例，规定用户有且只有一个邮箱，用户于邮箱的关系是 1:1

```golang
type User struct {
	ID 			string  `gorm:"primary_key"`
	Name 		string  `gorm:"not null"`
	Email		Email
}

type Email struct {
	ID		string	`gorm:"primary_key"`
	Address	string	`gorm:"not null"`
	UserId		string	`gorm:"not null"`
}
```

关联方式如下

```golang
var user1 User
db.Where("id = ?", "1").First(&user1)

// 关联的关键代码
db.Model(&user1).Related(&user1.Email, "Email")
// 或者
db.Model(&user1).Association("Email").First(&user1.Email)
```

## 3. 包含多个
这里以用户和邮箱为例，规定用户可以有多个邮箱，一个邮箱只能属于一个用户，用户与邮箱的关系是 1:n

```golang
type User struct {
	ID 			string  `gorm:"primary_key"`
	Name 		string  `gorm:"not null"`
	Emails		[]Email
}

type Email struct {
	ID		string	`gorm:"primary_key"`
	Address	string	`gorm:"not null"`
	UserId		string	`gorm:"not null"`
}
```

关联方式如下

```golang
var user1 User
db.Where("id = ?", "1").First(&user1)

// 关联的关键代码
db.Model(&user1).Related(&user1.Emails, "Emails")
// 或者
db.Model(&user1).Association("Emails").Find(&user1.Emails)
```

## 4. 多对多
这里以用户与语言为例，一个用户可以掌握多门语言，而语言可以被多个用户掌握，也就是用户与语言的关系是 n:m

```golang
type User struct {
	ID 			string  `gorm:"primary_key"`
	Name 		string  `gorm:"not null"`
	Languages	[]Language	`gorm:"many2many:user_languages;"`
}

type Language struct {
	ID		string	`gorm:"primary_key"`
	Name	string	`gorm:"not null"`
}
```

即需要使用many2many这个gorm tag，这个tag的值设置为user\_languages，用户与语言的关联表的名字为user\_languages，当使用auto-migration功能时，gorm会自动帮我们创建这样关联表，这张关联表只有两个columm，即user\_id和language\_id

```golang
var user1 User
db.Where("id = ?", "1").First(&user1)

// 关联的关键代码
db.Model(&user1).Association("Languages").Find(&user1.Languages)
```





