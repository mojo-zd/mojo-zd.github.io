---
title: golang实用小工具
date: 2018-04-18 14:17:02
categories: math
mathjax: true
tags: golang
---

平时遇到的一些有趣小工具记录, 增添点Gopher乐趣.
#### 包含中文字符串的倒置处理
```
func Reverse(s string) string {
    b := []rune(s)
    fmt.Println(len(s)/2, len(s))
    for i := 0; i < len(s)/2; i++ {
        j := len(b) - i - 1
        b[i], b[j] = b[j], b[i]
    }
    return string(b)
}
```

#### 通过反射机制修改对应结构的字段
```
func EvaluationTime(v interface{}, fields []string) {
    ty := GetType(reflect.TypeOf(v))
    for i := 0; i < ty.NumField(); i++ {
        if isContains(ty.Field(i).Name, fields) {
            GetValue(reflect.ValueOf(v)).Field(i).SetInt(time.Now().Unix())
        }
    }
}

func isContains(name string, fields []string) (contains bool) {
    for _, field := range fields {
        if field == name {
            contains = true
            break
        }
    }
    return
}

func GetType(ty reflect.Type) (t reflect.Type) {
    if ty.Kind() == reflect.Ptr {
        t = ty.Elem()
        return
    }
    t = ty
    return
}

func GetValue(value reflect.Value) (v reflect.Value) {
    if value.Kind() == reflect.Ptr {
        v = value.Elem()
        return
    }
    v = value
    return
}
```