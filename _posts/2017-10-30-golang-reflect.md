---
layout: post 
author: shalou
title:  "golang反射的应用" 
category: golang
tag: [golang, reflect]
---

golang的反射主要包括两大类型，Value和Type，interface{}是桥梁，反射三大定律

* 反射可以将“接口类型变量”转换为“反射类型对象”
* 反射可以将“反射类型对象”转换为“接口类型变量”。
* 如果要修改“反射类型对象”，其值必须是“可写的”（settable）

<!-- more -->

## 1. 查看结构体信息

```golang
package main

import (
    "fmt"
    "reflect"
)

type Person struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

func (p *Person) GetName() string {
    return p.Name
}

func (p *Person) GetAget() int {
    return p.Age
}

func (p *Person) SetName(name string) {
    p.Name = name
}

func (p *Person) SetAget(name string) {
}

func main() {
    p := Person{
        Name: "Jim",
        Age:  27, 
    }   

    t1 := reflect.TypeOf(p)
    t2 := reflect.TypeOf(&p)

    fmt.Println("Person Type", t1) 
    fmt.Println("&Person Type", t2) 

    //结构体类型Kind
    fmt.Println("Person Kind:", t1.Kind())
    //结构体指针类型Kind
    fmt.Println("&Person Kind:", t2.Kind())

    //结构体名称
    fmt.Println("Person Name:", t1.Name())
    //指针为空
    fmt.Println("Person Name:", t2.Name())

    fmt.Printf("\n======\n\n")

    //只有结构体才有Field，假如t2.NumField()会报错
    fmt.Println("Person Num Fields", t1.NumField())
    for i := 0; i < t1.NumField(); i++ {
        //获取各个域的名称，类型，json tag
        fmt.Printf("Person Field %d: [Name: %s, Type: %s, Tag: %s]\n", i, t1.Field(i).Name, t1.Field(i).Type, t1.Field(i).Tag.Get("json"))
    }

    fmt.Printf("\n======\n\n")

    // 上面四个方法均为结构体指针的方法，而不是结构体的方法，t1.NumMethod会报错
    fmt.Println("Person Num Methods", t2.NumMethod())
    for i := 0; i < t2.NumMethod(); i++ {
        //方法的名称，类型
        fmt.Printf("Person Method %d: [Name: %s, Type: %s]\n", i, t2.Method(i).Name, t2.Method(i).Type)
        //方法的输入参数，其中第一个参数为结构体指针本身
        fmt.Printf("  NumIn: %d\n", t2.Method(i).Type.NumIn())
        for j := 0; j < t2.Method(i).Type.NumIn(); j++ {
            param := t2.Method(i).Type.In(j)
            fmt.Printf("    param %d: [kind: %s]\n", j, param.Kind())
        }

        //方法的输出参数
        fmt.Printf("  NumOut: %d\n", t2.Method(i).Type.NumOut())
        for j := 0; j < t2.Method(i).Type.NumOut(); j++ {
            param := t2.Method(i).Type.Out(j)
            fmt.Printf("    param %d: [kind: %s]\n", j, param.Kind())
        }
    }
}

输出
Person Type main.Person
&Person Type *main.Person
Person Kind: struct
&Person Kind: ptr
Person Name: Person
Person Name: 

======

Person Num Fields 2
Person Field 0: [Name: Name, Type: string, Tag: name]
Person Field 1: [Name: Age, Type: int, Tag: age]

======

Person Num Methods 4
Person Method 0: [Name: GetAget, Type: func(*main.Person) int]
  NumIn: 1
    param 0: [kind: ptr]
  NumOut: 1
    param 0: [kind: int]
Person Method 1: [Name: GetName, Type: func(*main.Person) string]
  NumIn: 1
    param 0: [kind: ptr]
  NumOut: 1
    param 0: [kind: string]
Person Method 2: [Name: SetAget, Type: func(*main.Person, string)]
  NumIn: 2
    param 0: [kind: ptr]
    param 1: [kind: string]
  NumOut: 0
Person Method 3: [Name: SetName, Type: func(*main.Person, string)]
  NumIn: 2
    param 0: [kind: ptr]
    param 1: [kind: string]
  NumOut: 0
```

## 2. 直接修改结构体

```golang
package main

import (
    "fmt"
    "reflect"
)

type Person struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

func (p *Person) GetName() string {
    return p.Name
}

func (p *Person) GetAget() int {
    return p.Age
}

func (p *Person) SetName(name string) {
    p.Name = name
}

func (p *Person) SetAget(name string) {
}

func main() {
    p := Person{
        Name: "Jim",
        Age:  27, 
    }   

    v1 := reflect.ValueOf(p)
    v2 := reflect.ValueOf(&p)

    fmt.Println("reflect.ValueOf(p)", v1)
    fmt.Println("reflect.ValueOf(&p)", v2)
    fmt.Println("Person Value Kind", v1.Kind())
    fmt.Println("&Person Value Kind", v2.Kind())
    fmt.Println("Person Value Type", v1.Type())
    fmt.Println("&Person Value Type", v2.Type())

    fmt.Println("reflect.ValueOf(p).Interface()", v1.Interface())
    fmt.Println("reflect.ValueOf(&p).Interface()", v2.Interface())

    fmt.Printf("\n======\n\n")

    //结构体是值传递，不可CanSet
    fmt.Println("reflect.ValueOf(p) CanSet: ", v1.CanSet())
    //结构体指针也是值传递，不可CanSet
    fmt.Println("reflect.ValueOf(&p) CanSet: ", v2.CanSet())
    //将结构体指针指向的结构出来，可CanSet
    fmt.Println("reflect.ValueOf(&p).Elem() CanSet: ", v2.Elem().CanSet())

    //可以直接修改CanSet的Object
    fmt.Println("Before Set: ", p.GetName())
    v2.Elem().FieldByName("Name").SetString("Kate")
    fmt.Println("After Set: ", p.GetName())

    fmt.Printf("\n======\n\n")

    //调用函数
    fmt.Println("Call GetName Method: ", v2.MethodByName("GetName").Call([]reflect.Value{}))
    fmt.Println("Call SetName Method: ", v2.MethodByName("SetName").Call([]reflect.Value{reflect.ValueOf("Tom")}))
    fmt.Println("Call GetName Method: ", v2.MethodByName("GetName").Call([]reflect.Value{}))
}

输出
reflect.ValueOf(p) {Jim 27}
reflect.ValueOf(&p) &{Jim 27}
Person Value Kind struct
&Person Value Kind ptr
Person Value Type main.Person
&Person Value Type *main.Person
reflect.ValueOf(p).Interface() {Jim 27}
reflect.ValueOf(&p).Interface() &{Jim 27}

======

reflect.ValueOf(p) CanSet:  false
reflect.ValueOf(&p) CanSet:  false
reflect.ValueOf(&p).Elem() CanSet:  true
Before Set:  Jim
After Set:  Kate

======

Call GetName Method:  [Kate]
Call SetName Method:  []
Call GetName Method:  [Tom]
```

## 3. "传入"[]interface{}

假如需要传入一个切片到函数内部，切片的具体类型不确定。

```golang
package main

import (
    "encoding/json"
    "fmt"
    "reflect"
)

type Response struct {
    Length int         `json:"length"`
    Data   interface{} `json:"data"`
}

type Instance struct {
    Id      int    `json:"id"`
    OrderId int    `json:"order_id"`
    Name    string `json:"instance_name"`
}

type Order struct {
    Id   int    `json:"id"`
    Name string `json:"order_name"`
}

func MarshalData(data interface{}) []byte {
    var isSlice = false
    var resp *Response
    var ret []byte
    var err error

    v := reflect.ValueOf(data)
    if v.Kind() == reflect.Slice {
        isSlice = true
    }   
    if isSlice {
        resp = &Response{
            Length: v.Len(),
            Data:   data,
        }   
        ret, err = json.Marshal(resp)
    } else {
        ret, err = json.Marshal(data)
    }

    if err != nil {
        return []byte("data error")
    } else {
        return ret
    }
}

func main() {
    orders := []*Order{
        &Order{
            Id:   1,
            Name: "first",
        },
        &Order{
            Id:   2,
            Name: "second",
        },
    }
    
    instances := []*Instance{
        &Instance{
            Id:      1,
            OrderId: 1,
            Name:    "first",
        },
        &Instance{
            Id:      2,
            OrderId: 2,
            Name:    "second",
        },
    }

    one := &Order{
        Id:   10,
        Name: "only",
    }

    fmt.Println(string(MarshalData(instances)))
    fmt.Println(string(MarshalData(orders)))
    fmt.Println(string(MarshalData(one)))
}

输出
{"length":2,"data":[{"id":1,"order_id":1,"instance_name":"first"},{"id":2,"order_id":2,"instance_name":"second"}]}
{"length":2,"data":[{"id":1,"order_name":"first"},{"id":2,"order_name":"second"}]}
{"id":10,"order_name":"only"}
```
