# 第一讲 你真的了解 Go 语言中的切片吗？
## 1-1 前言
1. 切片 slice 是 golang 中一个非常经典的数据结构，其定位可以类比于其他编程语言中的数组. 本文介绍的内容会分为 slice 的使用教程、问题讲解以及源码解析，走读的源码为 go v1.19。

## 1-2 几个问题
### 1-2-1 问题 1
1. 初始化切片 s 长度和容量均为 10
2. 在 s 的基础上追加 append 一个元素
3. 请问经过上述操作后，切片 s 的内容、长度以及容量分别是什么？
    ```Go
        func Test_slice(t *testing.T){
            s := make([]int, 10)  
            s = append(s, 10)
            t.Logf("s: %v, len of s: %d, cap of s: %d", s, len(s), cap(s))
        }
    ```

### 1-2-2 问题 2
1. 初始化切片 s 长度为 0，容量为 10
2. 在 s 的基础上追加 append 一个元素
3. 请问经过上述操作后，切片 s 的内容、长度以及容量分别是什么？
    ```Go
        func Test_slice(t *testing.T){
            s := make([]int, 0, 10)  
            s = append(s, 10)
            t.Logf("s: %v, len of s: %d, cap of s: %d", s, len(s), cap(s))
        }
    ```

### 1-2-3 问题 3
1. 初始化切片 s 长度为 10，容量为 11
2. 在 s 的基础上追加 append 一个元素
3. 请问经过上述操作后，切片 s 的内容、长度以及容量分别是什么？
    ```Go
        func Test_slice(t *testing.T){
            s := make([]int, 10, 11)  
            s = append(s, 10)
            t.Logf("s: %v, len of s: %d, cap of s: %d", s, len(s), cap(s))
        }
    ```

### 1-2-4 问题 4
1. 初始化切片 s 长度为 10，容量为 12
2. 截取切片 s index = 8 往后的内容赋给 s1
3. 求问 s1 的内容、长度以及容量分别是什么？
    ```Go
        func Test_slice(t *testing.T){
            s := make([]int, 10, 12)  
            s1 := s[8:]
            t.Logf("s1: %v, len of s1: %d, cap of s1: %d", s1, len(s1), cap(s1))
        }
    ```

### 1-2-5 问题 5
1. 初始化切片 s 长度为 10，容量为 12
2. 截取切片 s index 为 [8, 9) 范围内的元素赋给切片 s1
3. 求问 s1 的内容、长度以及容量分别是什么？
    ```Go
        func Test_slice(t *testing.T){
            s := make([]int, 10, 12)  
            s1 := s[8:9]
            t.Logf("s1: %v, len of s1: %d, cap of s1: %d", s1, len(s1), cap(s1))
        }
    ```

### 1-2-6 问题 6
1. 初始化切片 s 长度为 10，容量为 12
2. 截取切片 s index = 8 往后的内容赋给 s1
3. 修改 s1[0] 的值
4. 请问这个修改是否会影响到 s？ 此时，s 的内容是什么？
    ```Go
        func Test_slice(t *testing.T){
            s := make([]int, 10, 12)  
            s1 := s[8:]
            s1[0] = -1
            t.Logf("s: %v", s)
        }
    ```

### 1-2-7 问题 7
1. 初始化切片 s 长度为 10，容量为 12
2. 请问，访问 s[10] 是否会越界？
    ```GO
        func Test_slice(t *testing.T){
            s := make([]int, 10, 12)  
            s = s[10]
            // 求问，此时数组访问是否会越界
        }
    ```

### 1-2-8 问题 8
1. 初始化切片 s 长度为 10，容量为 12
2. 截取 s 中 index = 8 后面的内容赋给 s1
3. 在 s1 的基础上追加 []int{10, 11, 12} 3 个元素
4. 请问，经过上述操作时候，访问 s[10] 是否会越界？
    ```Go
        func Test_slice(t *testing.T){
            s := make([]int, 10, 12)  
            s1 := s[8:]
            s1 = append(s1, []int{10, 11, 12}...)
            v := s[10]
            // ...
            // 求问，此时数组访问是否会越界
        }
    ```

### 1-2-9 问题 9
1. 初始化切片 s 长度为 10，容量为 12
2. 截取切片 s index = 8 往后的内容赋给 s1
3. 在方法 changeSlice 中，对 s1[0] 进行修改
4. 求问，经过上述操作之后，s 的内容是什么？
    ```Go
        func Test_slice(t *testing.T){
            s := make([]int, 10, 12)  
            s1 := s[8:]
            changeSlice(s1)
            t.Logf("s: %v",s)
        }


        func changeSlice(s1 []int){
          s1[0] = -1
        }
    ```

### 1-2-10 问题 10
1. 初始化切片 s 长度为 10，容量为 12
2. 截取切片 s index = 8 往后的内容赋给 s1
3. 在方法 changeSlice 中，对 s1 进行 apend 追加操作
4. 请问，经过上述操作后，s 以及 s1 的内容、长度和容量分别是什么？
    ```Go
        func Test_slice(t *testing.T){
            s := make([]int, 10, 12)  
            s1 := s[8:]
            changeSlice(s1)
            t.Logf("s: %v, len of s: %d, cap of s: %d", s, len(s), cap(s))
            t.Logf("s1: %v, len of s1: %d, cap of s1: %d", s1, len(s1), cap(s1))
        }


        func changeSlice(s1 []int){
          s1 = append(s1, 10)
        }
    ```

### 1-2-11 问题 11
1. 初始化切片 s，内容为 []int{0, 1, 2, 3, 4}
2. 截取 s 中 index = 2 前面的内容(不含s[2])，并在此基础上追加 index = 3 后面的内容
3. 请问，经过上述操作后，s 的内容、长度和内容分别是什么？此时访问 s[4] 是否会越界？
    ```Go
        func Test_slice(t *testing.T){
            s := []int{0, 1, 2, 3, 4}
            s = append(s[:2], s[3:]...)
            t.Logf("s: %v, len: %d, cap: %d", s, len(s), cap(s))
            v := s[4] 
            // 是否会数组访问越界
        }
    ```

### 1-2-12 问题 12
1. 初始化切片 s 长度和容量均为 5122
2. 在 s 的基础上追加 append 一个元素3
3. 请问经过上述操作后，切片 s 的内容、长度以及容量分别是什么？
    ```Go
        func Test_slice(t *testing.T){
            s := make([]int, 512)  
            s = append(s, 1)
            t.Logf("len of s: %d, cap of s: %d", len(s), cap(s))
        }
    ```
