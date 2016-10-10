---
layout: post
title: "如何制作支持多平台的Swift Framework (下)"
comments: true
---

上一篇我们已经完成了创建项目，添加依赖，编码，测试等工作，本篇着重介绍如何支持构建多平台版本，以及最终通过Cocoapods和Carthage发布的一些细节。

### 第四步，支持多平台版本

首先修改代码，在代码中添加[编译开关](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/BuildingCocoaApps/InteractingWithCAPIs.html#//apple_ref/doc/uid/TP40014216-CH8-ID17)，使同一份源文件支持多个平台:

```swift
import Foundation

func randomNaturalNum(under: UInt32) -> UInt32{
    return arc4random_uniform(under - 1) + 1
}

#if os(iOS) || os(OSX)
public func nextAddition() -> ArithmeticProblem{
    return ArithmeticProblem(leftOperand: randomNaturalNum(20), op: .Add, rightOperand: randomNaturalNum(20))
}
#endif

#if os(iOS) || os(tvOS)
public func nextSubtraction() -> ArithmeticProblem{
    return ArithmeticProblem(leftOperand: randomNaturalNum(20), op: .Minus, rightOperand: randomNaturalNum(20))
}
#endif

#if os(iOS) || os(watchOS)
public func nextMultiplication() -> ArithmeticProblem{
    return ArithmeticProblem(leftOperand: randomNaturalNum(10), op: .Multiply, rightOperand: randomNaturalNum(10))
}
#endif
```

注意，Swift中使用的编译开关相对C/C++/Objective-C严格的多，它要求所有被*#if* ... *＃elseif* ... *#else* ... *#endif*包围的代码块一定要保证语法结构上完整，单独拿出语法解析可以通过，否则编译不通过。

同时，在Unit Tests中也加入对应的编译开关

```swift
import XCTest
import Nimble
@testable import RandomArithmetics

class Tests: XCTestCase {
    
    #if os(iOS) || os(OSX)
    func testNextAddtion() {
        let additionProblem = nextAddition()
        additionProblem
        expect(additionProblem.leftOperand).to(beLessThanOrEqualTo(20))
        expect(additionProblem.leftOperand).to(beGreaterThan(0))
        expect(additionProblem.rightOperand).to(beLessThanOrEqualTo(20))
        expect(additionProblem.rightOperand).to(beGreaterThan(0))
        expect(additionProblem.op).to(equal(Operator.Add))
    }
    #endif
    
    #if os(iOS) || os(watchOS)
    func testNextMultiplication() {
        let multiplicationProblem = nextMultiplication()
        multiplicationProblem
        expect(multiplicationProblem.leftOperand).to(beLessThanOrEqualTo(9))
        expect(multiplicationProblem.leftOperand).to(beGreaterThan(0))
        expect(multiplicationProblem.rightOperand).to(beLessThanOrEqualTo(9))
        expect(multiplicationProblem.leftOperand).to(beGreaterThan(0))
        expect(multiplicationProblem.op).to(equal(Operator.Multiply))
    }
    #endif
    
    #if os(iOS) || os(tvOS)
    func testNextSubtraction() {
        let minusProblem = nextSubtraction()
        minusProblem
        expect(minusProblem.leftOperand).to(beLessThanOrEqualTo(20))
        expect(minusProblem.leftOperand).to(beGreaterThan(0))
        expect(minusProblem.rightOperand).to(beLessThanOrEqualTo(20))
        expect(minusProblem.rightOperand).to(beGreaterThan(0))
        expect(minusProblem.op).to(equal(Operator.Minus))
    }
    #endif
}
```

接着添加一个Cocoa Framework Target用来生成macOS版的Framework，并取名*Random Arithmetics macOS*，注意同样不选择同时包含Unit Tests:

![添加Cocoa Framework Target](/images/2016-08-16/添加CocoaFrameworkTarget.gif){: .imgpoper }

删除自动生成“RandomArithmetics macOS”目录以及里面的代码，

![删除自动生成的目录和代码](/images/2016-08-16/删除自动生成的目录和代码.gif){: .imgpoper }

然后更正下Build Settings：

“Packaging/Info.plist File” 设成 “Sources/Info.Plist”  
“Packaging/Bundle Identifier” 设成 “com.example.RandomArithmetics”  
“Packaging/Product Name” 设成 “RandomArithmetics”，统一Framework名称  

为了方便区分，也可以将iOS版的target重命名成*RandomArithmetics iOS*，同时别忘了“Packaging/Product Name”也设成*RandomArithmetics*。

接下来向“Build Phases/Compile Sources”中添加已有代码：

![添加代码](/images/2016-08-16/添加代码.gif){: .imgpoper }

激活Mac版的Build Scheme，开始编译（⌘+B），编译成功后可以打开Finder查看编译结果。

![编译并察看结果](/images/2016-08-16/编译并察看结果.gif){: .imgpoper }

与之前添加“iOS Unit Testing Bundle”类似，添加“OS X Unit Testing Bundle”后需要做些Clean&Fix的工作：
1，删除自动生成的目录和代码  
2，修正“Build Settings/Info.plist File”, “Build Settings/Bundle Identifier”  
3，添加Mac版Nimble.Framework到“Build Phases/Link Binary With Libraries”中，并更新Framework Search Paths  
4，添加已有的测试代码到“Build Phases/Compile Sources”  

![设置Compile Sources和Nimble.Framework](/images/2016-08-16/设置CompileSources和NimbleFramework.gif){: .imgpoper }

5，激活RandomArithmetics macOSTests，运行测试（⌘+U）

![macOS版Framework的测试结果](/images/2016-08-16/macOS版Framework的测试结果.png){: .imgpoper }

可以看到，编译开关起了作用，只执行了macOS上对应测试方法。  添加对watchOS和tvOS版的支持过程类似，这里就不再赘述了。

### 第五步，准备发布

至此，我们已经可以在一个工程中用同一套代码同时生成iOS、macOS、watchOS和tvOS各自版本的Framework，接下来的工作是发布出去，这里主要介绍Cocoapods和Carthage两种发布方式，Swift Package Manager先占个位，等到正式发布后再补充。

#### Cocoapods

1, 注册

本地[安装](https://guides.cocoapods.org/using/getting-started.html)好Cocoapods后先向Cocoapods注册当前使用的电脑，如果之前注册过可以跳过这步。打开Terminal运行以下命令：

>$ pod trunk register EMAIL [NAME] --description='XXXXXX'

把EMAIL、NAME替换成自己的邮箱和名字，命令完成后会有一封确认信发送到你提供的邮箱，点击邮件中包含的确认链接完成注册。同一邮箱可以在多台Mac上重复使用，以后注册时邮箱不变的情况下NAME可以忽略。注册完成后就可以用当前的电脑发布pod（Cocoapods对共享库的叫法）到Cocoapods Public Repository了。

2, 准备Pod

除了代码外，Cocoapods要求每个项目必须提供后缀是.[podspec](https://guides.cocoapods.org/syntax/podspec.html)文件和LICENSE信息。.podspec是文本文件，一般放在项目根目录下，内容包含了关于项目的一些元数据（metadata）；LICENSE信息可以通过单独文件指定，如：LICENSE.md，或直接写在.podspec里。下面是为RandomArithmetis Framework准备的RandomArithmetics.podspec文件:

```
\\RandomArithmetics.podspec
Pod::Spec.new do |spec|
    spec.name = "RandomArithmetics"
    spec.version = "1.0.0"
    spec.summary = "A companion project for online tutorial"
    spec.description = "A companion project for an online tutorial which is demostrating how to create a single Swift Framework targeting multiple platform"
    spec.homepage = "https://github.com/popduke/RandomArithmatics"
    spec.license = { type: 'MIT', file: 'LICENSE.md' }
    spec.authors = { "Yonny Hao" => 'popduke@gmail.com' }

    spec.ios.deployment_target = '8.0'
    spec.osx.deployment_target = '10.10'
    spec.watchos.deployment_target = '2.0'
    spec.tvos.deployment_target = '9.0'

    spec.frameworks  = "Foundation"
    spec.source = { git: "https://github.com/popduke/RandomArithmetics.git", tag: spec.version.to_s }
    spec.source_files = 'Sources/*.{h,swift}'
end
```

podspec里最关键的部分是设定Cocoapods如何编译代码，因为Cocoapods没有直接使用xcodeproj或xcworkspace中保存的Build Settings，而是规定了一套自己的编译设置写法，所以需要我们把Xcode项目中的Build Settings重新用Cocoapods的方式表示出来，重复劳动总是显得很麻烦而且容易出错。另外，这个podspec还指定了license的类型（MIT）和文件地址（项目根目录下的LICENSE.md）。

3, 本地验证  

在向Cocoapods提交之前，我们最好先在本地验证一下通过Cocoapods是否使用正常。随便创建一个Xcode工程，如新建一个iOS Single View Application，起名叫RandomArithmeticsPodTest，之后在项目根目录中加入一个叫“Podfile”的文本文件，并包含以下内容：

```
use_frameworks!

    target 'RandomArithmeticsPodTest' do

    pod 'RandomArithmetics', :path=>'PATH_TO_YOUR_PROJECT_ROOT'

end
```

把PATH_TO_YOUR_PROJECT_ROOT替换成*RandomArithmetics*项目在你本机上的路径。

然后cd到项目根目录，运行以下命令：

>$ pod install

接着，关闭打开的Xcode的工程后，打开生成的RandomArithmeticsPodTest.xcworkspace可以看到RandomArithmetics已经添加成功了。

![RandomArithmetics Framework添加成功](/images/2016-08-16/RandomArithmeticsFramework添加成功.png){: .imgpoper }

4, 发布到Cocoapod Public Repository

发布之前打好Release Tag，然后Push到Github。一切就绪后运行以下命令（本例只用于演示，请勿发布到Cocoapods上）：

>$ pod trunk push RandomArithmetics.podspec

该命令可能会运行较长时间，完成后需等几分钟才可以在cocoapods.org上检索到发布的pod。

#### Carthage

Carthage没有像Cocoapods那样中心化的Public Repository，发布比较简单，只需提交代码到Github或其他在线托管服务就可以了，也不需要特殊的描述文件（Carthage直接运行xcodebuild编译工程文件），但是还是要先在本地验证一下。

随便建一个空目录，如：RandomArithmeticsCarthageTest，然后在其中添加一个叫Cartfile的文本文件，包含以下内容：

> git "file:///PATH/TO/RANDOM_ARITHMETICS_PROJECT" "master"

路径指向你本地的Framework项目根目录，然后在Terminal中cd到这个目录运行以下命令：

>$ carthage update

如果运行后出现以下错误信息

![No shared schemes error](/images/2016-08-16/NoSharedSchemesError.png){: .imgpoper }

说明工程没有公开的Build Scheme，打开Manage Schemes对话框，把需要公开的Scheme标记成Shared即可:

![公开Build Schemes](/images/2016-08-16/公开BuildSchemes.gif){: .imgpoper }

之后commit代码时提交“RandomArithmetics.xcodeproj/Shared Data/Schemes”下的所有内容:

![提交Shared Data目录](/images/2016-08-16/提交SharedData目录.png){: .imgpoper }

重新运行命令，这回应该没问题了:

![编译成功](/images/2016-08-16/编译成功.png){: .imgpoper }

可以看到公开的Scheme都编译成功，Great！打开Finder查看下编译结果。

![Carthage编译输出结果](/images/2016-08-16/Carthage编译输出结果.png){: .imgpoper }

本地验证通过后，剩下的就是Push代码到Github等类似托管服务，不在赘述。

Swift Package Manager
等正式发布后补充...

### 后记

本文内容总结自一个本人开发的开源Framework：[RxLocationManager](https://github.com/popduke/RxLocationManager)，基于RxSwift，如果你正在尝试RP开发App，没准可以帮你更方便的处理跟CoreLocation有关的逻辑，欢迎使用。本人水平有限，文中如有纰漏、不准确之处，望轻拍。另外演示用代码已提交到Github。
