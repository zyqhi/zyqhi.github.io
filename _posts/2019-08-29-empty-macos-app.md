---
title: 纯代码创建macOS App
tags: macOS AppKit Swift
typora-root-url: /Users/mutsu/project/blog/zyqhi.github.io
---

无论是开发iOS还是macOS App，我都喜欢使用纯代码来创建整个App。对我而言，纯代码的好处是，我能清楚地『看到』App做了什么，而不是要忍受Xcode给我的魔盒。采用纯代码开发，能让我更好地理解这门技术的实现原理，从而在以后开发中遇到问题时，能够更快地定位到问题的原因。

image-20190829175325569.png

![image-20190829200718389](/../../../../../../../assets/image-20190829175325569.png)

废话不多说，来看下如何用Xcode创建一个Cocoa App。

- 新建一个Cocoa App项目，语言选Swift，不使用Storyboards。

![image-20190829200718389](/../../../../../../../_media/2019-08-29-empty-macos-app/image-20190829175325569.png)

![image-20190829200718389](/../../../../../../../_media/2019-08-29-empty-macos-app/image-20190829200718389.png)

- 删除`MainMenu.xib`和Info.plist文件中对应的属性。

![image-20190829200803372](/../../../../../../../_media/2019-08-29-empty-macos-app/image-20190829200803372.png)



- 新建`ViewController，继承自NSViewController。

```swift
import Cocoa

class ViewController: NSViewController {
    
    // 必须重写 loadView 方法，并且创建view，否则运行时会出错
    override func loadView() {
        view = NSView()
    }

    override func viewDidLoad() {
        super.viewDidLoad()
        // Do view setup here.
    }
    
}
```

注意此处必须重写`loadView`方法，并在方法中创建view，否则运行时会出现如下错误：

{:.warning}

2019-08-29 19:21:15.988264+0800 EmptyApp[64760:6216502] -[NSNib _initWithNibNamed:bundle:options:] could not load the nibName: EmptyApp.ViewController in bundle (null).

原因在于ViewController默认从xib文件加载view，而此处我们没有提供对应的xib文件。



-  新建`main.swift`文件，此文件是整个App的入口。

```swift
import Foundation
import AppKit

let app: NSApplication = NSApplication.shared
let appDelegate = AppDelegate()  // Instantiates the class the @NSApplicationMain was attached to
app.delegate = appDelegate
_ = NSApplicationMain(CommandLine.argc, CommandLine.unsafeArgv)
```



- 编辑`AppDelegate.swift`文件，创建初始window和view controller，建立初始UI。

```swift
import Cocoa
// 删除此处的 @NSApplicationMain
class AppDelegate: NSObject, NSApplicationDelegate {

    var window: NSWindow!
    var vc: ViewController!

    func applicationDidFinishLaunching(_ aNotification: Notification) {
        window = NSWindow(contentRect: NSRect(x: 100, y: 100, width: 640, height: 480), styleMask: .resizable, backing: .buffered, defer: false)
        vc = ViewController()
        window?.contentView?.addSubview(vc.view)
        
        
        window.makeKeyAndOrderFront(nil)
    }

    func applicationWillTerminate(_ aNotification: Notification) {
    }
}
```

此时运行项目，可以看到一个非常简陋的App，连关闭按钮和基本菜单都没有。随后可以便可以在此基础上创建自己的菜单和UI。

![image-20190829200827801](/../../../../../../../_media/2019-08-29-empty-macos-app/image-20190829200827801.png)