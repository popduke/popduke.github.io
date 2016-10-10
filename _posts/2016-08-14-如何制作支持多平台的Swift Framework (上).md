---
layout: post
title: "如何制作支持多平台的Swift Framework (上)"
comments: true
---
Xcode从6.0开始正式支持Framework类型的工程，之前只能创建Static Library。有些时候某些功能是全平台（iOS，macOS，watchOS，tvOS）通用的，分别为不同平台创建工程、维护代码当然可以，但显然不是最省力的方式。

本文通过制作一个简单的纯Swift编写的Framework，教你如何在一个项目中维护一套代码，并同时可以构建多个平台的版本，最终通过Cocoapods和Carthage发布。

演示环境：Xcode 7.3.1， Swift 2.2，Carthage 0.17，Cocoapods 1.0.0。

### 第一步，创建Framework工程：

选择工程模版，这里选择iOS\Cocoa Touch Framework作为开始，其实选择其他平台下的Framework模版也可以，因为最终我们是要在一个工程内支持所有平台；第二步先别勾选Include Unit Tests，之后会添加；第三步选择同时创建Git。

![创建工程](/images/2016-08-14/创建工程.gif){: .imgpoper }

创建之后的目录结构如下:

![自动生成的目录结构](/images/2016-08-14/自动生成的目录结构.png){: .imgpoper }

先稍作下改动以符合即将发布的Swift Package Manager的要求。将包含*Info.plist*和*RandomArithmetics.h*的文件夹重命名成*Sources*，并删除自动生成的*RandomArithmetics.h*文件:

![修改后的目录结构](/images/2016-08-14/修改后的目录结构.png){: .imgpoper }

>Swift Package Manager[规范](https://swift.org/package-manager/)要求，代码文件默认放在Sources目录下。

>关于自动生成的头文件（*RandomArithmetics.h*）的作用。如项目中包含Objective-C的代码，那么所有Public的头文件都需要在该头文件中声明，然后通过这个头文件间接暴露给使用者（如下图所示，*RandomArithmetics.h*被包含在了Build Phases/Headers/Public里）。但因为Swift这门语言代码本身不使用头文件.h与实现文件.m来控制“能见度”，接口的[开放程度](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/AccessControl.html#//apple_ref/doc/uid/TP40014097-CH41-ID3)完全由关键字（public, internal, private）控制，因此编译器有足够的信息生成访问控制代码，所以不需要这个文件。

![默认生成的头文件](/images/2016-08-14/默认生成的头文件.png){: .imgpoper }

回到Xcode，因为我们直接修改了目录结构，所以有些文件变红了，需要删除后重新添加一下：删除*Info.plist*和*RandomArithmetics.h*，*RandomArithmetics* group重命名成*Sources*，之后再把*Info.plist*添加回来，最后别忘了更正Build Settings/Info.plist File的值，使其重新指向正确的相对路径。


最后编译（⌘+B）一下工程，正常的话Products下的*RandomArithmetics.framework*将会由红变黑说明编译成功。

修复前的工程布局:

![修复前的工程布局](/images/2016-08-14/修复前的工程布局.png){: .imgpoper }

修复后的工程布局:

![修复后的工程布局](/images/2016-08-14/修复后的工程布局.png){: .imgpoper }

### 第二步，添加代码
我们要实现的功能很简单，随机生成20以内的自然数加减法运算和乘法口诀问题。这里对不同平台的功能作以下规定:

* iOS: 支持生成全部类型的问题
* macOS：只支持生成20以内的加法运算问题
* watchOS：只支持生成乘法口诀问题
* tvOS：只支持生成20以内的减法运算问题

向工程中添加第一个源文件*ArithmeticProblem.swift*:

```swift
//ArithmeticProblem.swift
import Foundation

public enum Operator:String, CustomStringConvertible{
    case Add = "+"
    case Minus = "-"
    case Multiply = "*"
    
    public var description: String{
        get{
            return rawValue
        }
    }
}

public struct ArithmeticProblem: CustomStringConvertible{
    let leftOperand: UInt32
    let op: Operator
    let rightOperand: UInt32
    
    public func answer(with:UInt32) -> Bool{
        switch op {
        case .Add:
            return leftOperand + rightOperand == with
        case .Minus:
            return leftOperand - rightOperand == with
        case .Multiply:
            return leftOperand * rightOperand == with
        }
    }
    
    public var description: String{
        get{
            return "\(leftOperand) \(op) \(rightOperand) = ?"
        }
    }
}
```

文件中定义了一个表示运算符的enum：*Operator*和一个表示运算问题的struct：*ArithmeticProblem*，二者都是public的，因为要对外可见。

添加后文件与*RandomArithmetics* Target关联

![包含ArithmeticProblem.swift的工程布局](/images/2016-08-14/包含ArithmeticProblem.swift的工程布局.png){: .imgpoper }

添加第二个源文件*GenerateProblem.swift*，同样也与*RandomArithmetics* Target关联

```swift
//GenerateProblem.swift
import Foundation

func randomNaturalNum(under: UInt32) -> UInt32{
    return arc4random_uniform(under - 1) + 1
}

public func nextAddition() -> ArithmeticProblem{
    return ArithmeticProblem(leftOperand: randomNaturalNum(20), op: .Add, rightOperand: randomNaturalNum(20))
}

public func nextSubtraction() -> ArithmeticProblem{
    return ArithmeticProblem(leftOperand: randomNaturalNum(20), op: .Minus, rightOperand: randomNaturalNum(20))
}

public func nextMultiplication() -> ArithmeticProblem{
    return ArithmeticProblem(leftOperand: randomNaturalNum(10), op: .Multiply, rightOperand: randomNaturalNum(10))
}
```

文件中定义了三个public函数分别随机返回20以内加法、乘法口诀、20以内减法问题。

![包含GenerateProblem.swift的工程布局](/images/2016-08-14/包含GenerateProblem.swift的工程布局.png){: .imgpoper}

编译（⌘+B）一下工程，看是否可以编译成功。接下来添加些Unit Tests验证代码逻辑。

### 第三步，添加Unit Tests

创建工程的时候我们故意没有选择包含Unit Tests，之后只需添加Unit Test Target即可，我们选择iOS\iOS Unit Testing Bundle，因为目前我们只有一个Cocoa Touch Framework Target等待测试。这是我们的第一个Testing Bundle，先暂且起一个通用的名字“Tests”，因为Xcode会用这个名字帮我们生成目录和文件，然而我们希望Test的代码也要复用，那么这样做能省事一些。

![添加第一个Unit Tests Bundle](/images/2016-08-14/添加第一个UnitTestsBundle.gif){: .imgpoper }

之后工程布局和目录结构将会如下:

![添加Unit Test Target之后的工程布局](/images/2016-08-14/添加UnitTestTarget之后的工程布局.png){: .imgpoper }

添加Unit Test Target之后的目录结构:

![添加Unit Test Target之后的目录结构](/images/2016-08-14/添加UnitTestTarget之后的目录结构.png){: .imgpoper }

接下来把*Tests* Target重命名成更容易辨识的名称，如：*RandomArithmetics iOSTest*。

![重命名Tests Target](/images/2016-08-14/重命名TestsTarget.gif){: .imgpoper }

切换到“Test navigator”，试着跑以下看看能不能通过。

![Unit Tests试运行](/images/2016-08-14/UnitTests试运行.gif){: .imgpoper }

接下先添加一个第三方Framework：[Quick/Nimble](https://github.com/Quick/Nimble)用于方便做Assertion，同时也演示如何用[Carthage](https://github.com/Carthage/Carthage)管理项目依赖。

首先在项目根目录下添加一个文本文件Cartfile.private，内容如下:

>github "Quick/Nimble" ~> 4.1.0

一般我们将测试用依赖声明在Cartfile.private里，这样别人在使用我们的Framework时，Cartfile.private里的依赖不会被下载。

打开Terminal, cd到项目根目录，运行以下命令：

>$ carthage update

完成后，在项目根目录下会出现个Carthage文件夹，包含了Checkouts和Build两个字目录：

![Carthage Update后的目录结构](/images/2016-08-14/CarthageUpdate后的目录结构.png){: .imgpoper }

Checkouts目录里是Quick/Nimble的工程源码，Build目录里是Nimble在三个不同平台的编译输出。接下来我们需要手动的将Nimble添加到工程当中。有两种方式：

第一种，直接使用Build目录下的已生成的Framework，具体如何操作不同平台略有差别，请移步[官方文档](https://github.com/Carthage/Carthage)。

第二种，将第三方Framework的源码直接纳入统一个xcode workspace，共同编译。这样在调试的时候可以直接步入源码。我们使用第二种方式。

回到Xcode工程，在"*File*"菜单下选择"*Save As Workspace...*"，在弹出的对话框中给workspace取项目同名：*RandomArithmetics*, 并在项目根目录保存。

![Save As Workspace](/images/2016-08-14/SaveAsWorkspace.gif){: .imgpoper }

然后关闭工程，打开workspace，将Nimble.xcodeproj拖入到Xcode中，使之与*RandomArithmetics*成为并列项目。

![向workspace中添加Nimble](/images/2016-08-14/向workspace中添加Nimble.gif){: .imgpoper }

接下来在*RandomArithmetics iOSTests* Target的“Build Phases/Link Binary With Libraries”中添加对iOS版Nimble.Framework的引用。

![添加对Nimble的编译链接](/images/2016-08-14/添加对Nimble的编译链接.gif){: .imgpoper }

设置完毕后就可以添加测试代码了，如下图所示，代码很简单共三个测试方法分别测试三个函数的逻辑，这里我们使用了Nimble的Assertion写法:

```swift
\\ GenerateProblemTests.swift
import XCTest
import Nimble
@testable import RandomArithmetics

class Tests: XCTestCase {
    
    func testNextAddtion() {
        let additionProblem = nextAddition()
        additionProblem
        expect(additionProblem.leftOperand).to(beLessThanOrEqualTo(20))
        expect(additionProblem.leftOperand).to(beGreaterThan(0))
        expect(additionProblem.rightOperand).to(beLessThanOrEqualTo(20))
        expect(additionProblem.rightOperand).to(beGreaterThan(0))
        expect(additionProblem.op).to(equal(Operator.Add))
    }
    
    func testNextMultiplication() {
        let multiplicationProblem = nextMultiplication()
        multiplicationProblem
        expect(multiplicationProblem.leftOperand).to(beLessThanOrEqualTo(9))
        expect(multiplicationProblem.leftOperand).to(beGreaterThan(0))
        expect(multiplicationProblem.rightOperand).to(beLessThanOrEqualTo(9))
        expect(multiplicationProblem.leftOperand).to(beGreaterThan(0))
        expect(multiplicationProblem.op).to(equal(Operator.Multiply))
    }
    
    func testNextSubtraction() {
        let minusProblem = nextSubtraction()
        minusProblem
        expect(minusProblem.leftOperand).to(beLessThanOrEqualTo(20))
        expect(minusProblem.leftOperand).to(beGreaterThan(0))
        expect(minusProblem.rightOperand).to(beLessThanOrEqualTo(20))
        expect(minusProblem.rightOperand).to(beGreaterThan(0))
        expect(minusProblem.op).to(equal(Operator.Minus))
    }
}
```

切换到Test Navigator后运行测试（⌘+U），如果你跟着做到现在的话会看到测试都通过了，Great!

![运行测试结果](/images/2016-08-14/运行测试结果.png){: .imgpoper }

且慢！虽然测试通过但出现了一个⚠️，点开看看是什么情况:

![⚠️](/images/2016-08-14/warning.png){: .imgpoper }

问题出现在Link阶段，给Linker指定的某个搜索链接对象的目录不存在，这个路径是在"Build Settings/Search Paths/Framework Search Paths"里指定的，是在我们手动添加Nimble.Framework到"Build Phases/Link Binary With Libraries"时Xcode帮我们自动加上的，Xcode根据Nimble的"Build Settings/Build Locations/Per-configuration Build Products Path"中的值结合Nimble项目的位置生成了这个路径：

![自动添加的Framework Search Path](/images/2016-08-14/自动添加的FrameworkSearchPath.png){: .imgpoper }

Xcode这么做无可厚非，它假设我们可能会把编译的结果输出到这个目录，但实际最终编译输出到哪还受Xcode "Derived Data Location"设置的影响。默认情况每个Xcode打开窗口（Xcode Session，可以是单个xcodeproj也可以是xcworkspace）会被随机分配一个唯一的输出目录（Derived Data Location，Window->Projects可以打开查看这个目录）。

![查看Derived Data Location](/images/2016-08-14/查看DerivedDataLocation.gif){: .imgpoper }

可以看到所有Target的编译结果都输出到同一个目录下了，因此即便这个路径不存在，Linker还是可以在当前目录找到链接对象。所以解决这个⚠️最简单的办法就是删掉这个路径，另外一个方法是指向一个真实存在的输出路径，做个双保险。正巧我们使用的Carthage会把编译结果输出到固定的相对路径：$(PROJECT_DIR)/Carthage/Build/*PLATFORM*，*PLATFORM*可以是 iOS、Mac、watchOS、tvOS四者之一，所以把这个路径添加到“Build Settings/Search Paths/Framework Search Paths”下正合适。

设置正确的Search Path:

![设置正确的Search Path](/images/2016-08-14/设置正确的SearchPath.png){: .imgpoper }

接着再运行测试（⌘+U）就一切正常了。

到目前为止，我们已经实现了iOS版RandomArithmetics Framework的功能，在下一篇中我们将介绍如何添加对多平台版本的支持，以及最后如何通过Cocoapods和Carthage发布。