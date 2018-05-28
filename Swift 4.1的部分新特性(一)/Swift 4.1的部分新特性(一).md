2018-02-06苹果更新了[Swift官方文档-修订历史][1]，加入了关于Swift 4.1的一些新特性：  
**1\. 有条件地遵守协议**  
**2\. 递归的协议约束**  
**3\. 增加了条件编译判断项：`canImport()` 和 `targetEnvironment()`**

下面详细介绍三项特性，所有代码基于**Xcode9.3** beta 4 (9Q127n)调试运行。

## 1\.有条件地遵守协议(Conditionally Conforming to a Protocol)

按字面意思就是在满足一定条件时，才认为一个类遵循了某个协议，不满足时就认为该类没有遵循该协议。一个简单的例子：

    extension Array:TextRepresentable where Element: ProtocolForElement {
        //......
    }
    

在上面例子中，我们扩展了Array类，让其遵循`TextRepresentable`协议，但只有在这个数组的所有元素都遵循`ProtocolForElement`协议时，才有效。就是说，当我们在数组中存入了Int类型的元素后，是无法访问`TextRepresentable`协议所确定的内容的(认为其并没有遵循该协议)。这里where子句所描述的就是条件。where子句描述的并不只有继承关系，更多表达请参考[泛型where子句][2]。完整的例子和运行结果如下：

    //定义一个协议，提供一个文本描述属性
    protocol TextRepresentable { 
        var textualDescription: String { get }
    }
    
    //定义一个协议，用于限定子元素。内容不重要，暂为空
    protocol ProtocolForElement { 
    }
    
    //定义一个遵循了上面元素协议的类。用于存放到数组中。
    class ClassForElement: ProtocolForElement { 
    }
    
    //扩展Array类，按条件遵循了TextRepresentable协议
    extension Array:TextRepresentable where Element: ProtocolForElement { 
        var textualDescription: String {
            return "此数组的元素遵循了ProtocolForElement协议"
        }
    }
    
    let arr1 = [ClassForElement(), ClassForElement()] //建立数组1，存入遵循ProtocolForElement协议的元素
    print(arr1.textualDescription) //正确输出：“此数组的元素遵循了TextRepresentable协议”
    
    let arr2 = [1,4,7] //建立数组2，只存入Int元素
    print(arr2.textualDescription) //错误： type 'Int' does not conform to protocol 'ProtocolForElement'
    

通过以上例子，我们看到：数组1由于其元素都遵循`ProtocolForElement`协议，所以扩展`extension Array:TextRepresentable`就发挥了作用，我们可以调用`TextRepresentable`协议所明确的`textualDescription`方法。而数组2其元素类型为Int, 并没有遵循`ProtocolForElement`协议，上面的扩展就对其无效，无法访问`TextRepresentable`协议所明确的内容。

## 2\.递归的协议约束(Recursive protocol constraints)

递归在我们平时的编程中通常是指函数对自己的调用，所以，当这个概念用在协议定义时，可以理解为协议中对自己的访问。它赋予了协议可以强制要求属性、方法参数和方法返回值也遵循本协议的能力。举个例子：

    protocol ExampleProtocol {
        associatedtype T: ExampleProtocol //关联类型为协议本身
        func doSomething() -> T //要求方法返回的对象也遵循本协议
    }
    

以上例子中，我们使用`associatedtype`关键字定义关联类型时，关联的类型为协议本身，并将它作为一个方法的返回类型。以此便限定了方法的实现者必须返回遵循该协议的对象。协议实现者例子：

    struct Example:ExampleProtocol {
        func doSomething() -> Example { //方法的返回对象也必须遵循了ExampleProtocol协议。
            return Example()
        }
    }
    

## 3\.增加了条件编译判断项

新增的`canImport()`可以判定是否可以引入某个模块。比如通过CocoaPods安装Alamofire后，使用该语句判断是否可以正确引入：

    #if canImport(Alamofire) //判断是否可引入Alamofire模块
        class classWithAlamofire{
            //......
        }
    #endif
    

新增的`targetEnvironment()`可以判定运行环境是否为虚拟机，当处于虚拟机环境时返回true, 其他情况返回false。它目前有效的参数只有simulator：

    #if targetEnvironment(simulator)
        class classForSimulator{
            //......
        }
    #endif
    

关于目前Swift 4.1官方文档更新的内容就介绍到这里。下一篇介绍Xcode9.3 beta release文档中提及的部分Swift编译器特性更新。

 [1]: https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/RevisionHistory.html#//apple_ref/doc/uid/TP40014097-CH40-ID459
 [2]: https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Generics.html#//apple_ref/doc/uid/TP40014097-CH26-ID192