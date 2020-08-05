



var name = "Python编程时光"


name := "Python编程时光"



```
package main

import "fmt"

func main()  {
    var age int = 28
    var ptr = &age  // &后面接变量名，表示取出该变量的内存地址
    fmt.Println("age: ", age)
    fmt.Println("ptr: ", ptr)
}
```


```
// 第一种方法
var scores map[string]int = map[string]int{"english": 80, "chinese": 85}

// 第二种方法
scores := map[string]int{"english": 80, "chinese": 85}

// 第三种方法
scores := make(map[string]int)
scores["english"] = 80
scores["chinese"] = 85
```