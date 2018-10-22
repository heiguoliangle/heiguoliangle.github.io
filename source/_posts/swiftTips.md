---
title: swiftTips
date: 2018-02-08 11:26:37
tags: [swift]
---

# map & flatMap 

## map 
### map函数的定义:

```
public func map<T>(_ transform: (Element) throws -> T) rethrows -> [T]
```
> 这个是一个接受一个闭包参数,并且在闭包中返回一个集合元素类型
> 该函数返回的是一个当前调用集合所有元素经过处理后的集合

### 使用示例

1. 可以直接使用`$0` 来代表每个元素,并作为返回值.

	```
let numbers = 1...3
        let result = numbers.map {
            $0
        }
        print(result)    //输出：[1, 2, 3]
	```

2. 作为闭包参数来返回,可以对每一个元素进行单独处理,并且在闭包内需要`return`,还需要特别的指定`Int`这个泛型的具体类型

	```
let result1 = numbers.map { (element) -> Int in
            if element < 3 {
                return element
            }else{
                return 10
            }
        }
print(result1)    //输出：[1, 2, 10]
	```

## flatmap
### flatmap函数定义


```
extension SequenceType {
    public func flatMap<S : SequenceType>(transform: (Self.Generator.Element) -> S) 
         -> [S.Generator.Element]
}

extension SequenceType {
    public func flatMap<T>(transform: (Self.Generator.Element) -> T?) 
         -> [T]
}
```
> 与map类似的,同样是一个闭包参数,返回单个元素
> 最终返回的类型调用函数数组(可以在闭包中对单个元素进行重新添加)

### 使用示例
1. 使用`$0`


使用的数组`let strs : [AnyObject?] = ["23" as AnyObject,"43rf" as AnyObject,nil,34 as AnyObject]`

	```
let flat2 = strs.flatMap{$0}
print(flat2) // [23, 43rf, 34]
	```

2. 使用闭包处理

	```
let test = strs.flatMap { (e) -> Int? in
            return e as? Int
        }
print(test) // [34]
let flat1 = strs.flatMap { (e) -> String? in
  return e as? String
}
print(flat1) // ["23", "43rf"]
	```
	
3. 将多维数组合并到一个数组中
	
	```
	let arr = [[1, 2, 3], [6, 5, 4]]
	let brr = arr.flatMap {
	    $0
	}
	// brr = [1, 2, 3, 6, 5, 4]
	```

