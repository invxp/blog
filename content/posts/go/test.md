---
title: "[Go]单测打桩最佳实践"
date: 2022-09-30T17:55:52+08:00

tags: ["go"]

author: "阿猫"
showToc: false
TocOpen: false
hidemeta: false
comments: true
description: ""

disableShare: true
ShowReadingTime: true
ShowBreadCrumbs: true

ShowPostNavLinks: true

ShowWordCount: true
ShowRssButtonInSectionTermList: true

cover:
    image: ""
    alt: ""
    caption: ""
---
### 单元测试是最好的文档

Go默认是带单元测试框架的，只需要新建一个xxx_test.go文件并引入test包即可
```go
package test

import (
    "testing"
    "math/rand"
)

func TestRandomFail(t *testing.T) {
    if rand.Int31() % 2 == 0 { t.Fail() }	
}
```

但实际开发生产环境中，我们的业务代码没有想象中那么简单

举例一个典型场景，用户登录成功后查询积分，无论登陆成功或失败都记录一条日志

这里将动作分解成三个步骤
1. 用户登录接口（User.Login）
2. 查询积分接口（User.QueryCredit）
3. 记录日志接口（StoreLog）

以下是伪代码
```go
func (u *User) Login() (err error) {
    //生成登录Token
    u.Token = uuid.New().String()
    return nil
}

func (u *User) QueryCredit() (credit uint, err error) {
    //从三方接口获取积分
    thirdPart := &finace.Object{}
    return thirdPart.GetCredit(u.Token)
}

func StoreLog(log string) (err error) {
    //写入SQL日志 
    return gormdb.Exec("INSERT INTO U_Login_Log VALUES (?)", log)
}

func TestQueryCredit(t *testing.T) {
    loginErr := user.Login()
    credit, creditErr := user.QueryCredit()
    storeErr := StoreLog("UserLoginResult:", loginErr, "Credit:", credit, "CreditResult:", creditErr)

    if loginErr != nil || creditErr != nil || storeErr != nil {
        t.Fatalf("GotError:", loginErr, creditErr, storeErr)
    }
}
```
这段代码可能存在的两个问题
1. 从三方接口获取积分，但三方接口没有准备好，影响开发进度
2. 写入SQL日志依赖数据库，但本地或测试环境没有数据库，流程跑不通

我们要核心解决的问题是伪造接口（Mock）
1. 伪造一个成功/失败的三方接口来保证开发进度（MockFunction）
2. 伪造一个数据库实例进行增删改查（MockSQL）

### Go的Mock库（GoMonkey）

Monkey的具体介绍不在此展开，其主要用到的功能为

* 打桩某个函数（ApplyFunc）
* 打桩成员方法（ApplyMethod）
* 打桩成员变量（ApplyFuncVar）

已知thirdPart.GetCredit是成员方法，这里需要Mock掉，伪代码为
```go
func (u *User) QueryCredit() (credit uint, err error) {
    //从三方接口获取积分
    thirdPart := &finace.Object{}
    //第一个参数为成员类型指针
    //第二个参数为成员方法名
    //第三个参数为成员方法函数（第一个参数固定为成员类型指针，之后在写入参）
    patch := monkey.ApplyMethod(
        //成员类型指针
        reflect.TypeOf(thirdPart),
        //成员方法名（字符串）
        "GetCredit",
        //成员方法函数,第一个参数与成员类型指针对应
        func (_ *finace.Object, token string) (uint, error) {
            return 66666, nil
        }
    ) 
    //测试完毕后需要释放Mock
    defer patch.Reset()
    return thirdPart.GetCredit(u.Token)
}
```
这段代码最后执行的结果就是为finace对象的GetCredit打桩

从代码上看结果永远返回66666, nil

### Go的SQLMock库（go-sqlmock）

在数据库应用开发过程中，会在数据库上执行各种SQL语句

但在做单测时，一般不会与实际数据库交互，这时就需要Mock

即在不建立真实连接的情况下，模拟sql driver中的各种操作

go-sqlmock只会生成sql.db对象，如果使用了gorm则需特殊处理如
```go
//db是sql.db对象指针
//mock是需要打桩的函数列表
func Mock() {
    db, mock, _ := sqlmock.New()

    ormdb, err := gorm.Open(
    sql.New(
        sql.Config{
            //传入SQL.DB连接池（Mock） 
            Conn: db,
            //跳过获取SQL版本，不开会报错
            SkipInitializeWithVersion: true
        }
		)
    )

    //打桩INSERT函数
    mock.ExpectExec("INSERT").WillReturnResult(sqlmock.NewResult(0, 0))
    //打桩QUERY函数
    mock.ExpectQuery("SHOW").WillReturnRows(sqlmock.NewRows([]string{""}))
}
```

最终Mock的代码为

```go
func StoreLog(log string) (err error) {
    db, mock, _ := sqlmock.New()

    ormdb, err := gorm.Open(sql.New(
        sql.Config{
            Conn: db, 
            SkipInitializeWithVersion: true
        }
        )
    )

    //Mock掉INSERT语句，注意区分大小写
    //WillReturnResult是与其返回的结果，如果是SELECT语句需要Mock数据结构
    //具体不在展开
    mock.ExpectExec("INSERT").WillReturnResult(sqlmock.NewResult(0, 0))
	
    return gormdb.Exec("INSERT INTO U_Login_Log VALUES (?)", log)
}
```
### 参考文章

* [Go单元测试指引](https://zhuanlan.zhihu.com/p267341653)
* [GoMonkey介绍](https://blog.csdn.net/weixin_30337227/article/details/121316675)
* [GoSQLMock介绍](https://blog.csdn.net/lanyang123456/article/details/115290490)
