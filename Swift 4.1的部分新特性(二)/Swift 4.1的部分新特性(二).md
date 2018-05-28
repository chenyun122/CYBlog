在Xcode_9.3_beta_4的发布说明中，给出了一些新编译器所支持的Swfit变化。此文对其中两点进行说明：

* * *

### Equatable和Hashable协议

对于`Equatable`协议所规定的`==`方法实现将会由编译器自动添加，只要类或者结构的所有存储属性都遵循`Equatable`协议，比如类型`String`, `Int`, `Double`, `Array<String>`等（这些类型是模式遵循了`Equatable`协议的）。原来这样的代码：

    struct Person:Equatable {
        var id:String
    
        static func ==(lhs: Person, rhs: Person) -> Bool {
            return lhs.id == rhs.id
        }
    }
    

在Xcode9.3以后可以省略`==`方法，变为下面这样也是可以的：

    struct Person:Equatable {
        var id:String
    }
    

类似的，`Hashable`协议所要求的`hashValue`方法也会由编译器自动添加默认实现，同样需要所有存储属性都支持`Hashable`协议。当我们提供了自己的`==`和`hashValue`方法时，它们将会覆盖编译器提供的默认版本。  
得益于对`Equatable`协议更好的支持，现在包含可选类型元素的数组和字典也能直接比较了，只要它们的元素类型同样支持了`Equatable`协议。以下代码在Xcode9.2无法编译通过，但在9.3版本中已经可以了：

    let arr1:[Int?]? = [1,2,3]
    let arr2:[Int?]? = [1,2,3]
    let isSame = arr1 == arr2  // isSame: true
    

### 隐式可选类型

目前把"隐式可选类型"传递给inout关键字修饰地"可选类型"参数是不允许的，比如：

    func takesOptional(i: inout Int?) {
        //......
    }
    
    var x: Int! = 1
    takesOptional(i: &x) //error: Cannot pass immutable value of type 'Int?' as inout argument
    

以上代码中，变量`x`是一个隐式可选类型，`takesOptional`方法要求的参数是一个可选类型，这在Xcode9.2中会报错。这符合我们对隐式可选类型的预期，因为它一旦初始化后就不可能为nil。但在Xcode9.3中这样做是允许的，这就存在以上代码中`x`重新变为`nil`的可能, 比如：

    func takesOptional(i: inout Int?) {
        i = nil
    }
    
    var x: Int! = 1
    takesOptional(i: &x)
    print("x:\(x)")  // in Xcode9.3 beta4, output: x:nil
    

我个人猜想这与将来隐式可选类型**可能被废除**有关。