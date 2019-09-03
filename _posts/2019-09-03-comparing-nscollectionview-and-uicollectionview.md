---
title: NSCollectionView vs UICollectionView
tags: macOS AppKit
article_header:
  type: cover
  image:
    src: /cover/cover1.jpg
typora-root-url: /Users/mutsu/project/blog/zyqhi.github.io
---

NSCollectionView是AppKit框架提供的集合视图控件，用于构建macOS App。如今的NSCollectionView和UICollectionView在设计思想上有非常多的类似，但是又有诸多不同。本文将对二者的差异进行简单的对比，并给出使用纯代码创建NSCollectionView的方法。

我们先来看下，二者提供的API，如下表所示：

|                 | **NSCollectionView**       | **UICollectionView**       |
| --------------- | -------------------------- | -------------------------- |
| **Super Class** | NSView                     | UIScrollView               |
| **Cell**        | NSCollectionViewItem       | UICollectionViewCell       |
| **Layout**      | NSCollectionViewLayout     | UICollectionViewLayout     |
| **Delegate**    | NSCollectionViewDelegate   | UICollectionViewDelegate   |
| **Data Source** | NSCollectionViewDataSource | UICollectionViewDataSource |

从表中可以看出，二者的API在设计上有非常多的共同之处，比如都是通过data source来配置数据源，delegate来处理事件，layout来管理布局。下图展示了Collection View的设计思想。

![image](/../../../../../../../media/2019-09-03-comparing-nscollectionview-and-uicollectionview/cv_objects_2x_37439bb5-da91-4db6-a7f0-03c4046877f4.png)



尽管二者有诸多类似之处，但是在使用时，有些细节点仍然需要注意。对于熟悉UICollectionView的开发人员而言，在使用NSCollectionView需要注意以下几点：

1. NSCollectionView不是NSScrollView的子类，而是NSView的子类，这一点和UICollectionView不同，后者为UIScrollView的子类；
2. 正因为NSCollectionView不是NSScrollView的子类，我们在项目使用NSCollectionView时一般不单独使用，而是和NSScrollView配合使用；
3. NSCollectionView使用NSCollectionViewItem表示其所要显示的条目，NSCollectionViewItem和UICollectionViewCell也不尽相同，前者是NSViewController的子类，而后者为UIView的子类。

除此之外，二者的编程思路非常类似。

# 使用NSCollectionView

前文中介绍了NSCollectionView的API，本节将以一个例子来展示如何以纯代码的方式构建NSCollectionView。

- 打开Xcode，新建一个Cocoa App工程，并添加两个空白View，如图所示。我们将在左侧的View中添加一个NSCollectionView：

![image-20190903123817405](/../../../../../../../media/2019-09-03-comparing-nscollectionview-and-uicollectionview/image-20190903123817405.png)

- 创建Collection View：

```swift
class ViewController: NSViewController, DragDropImageViewDelegate {

    @IBOutlet var leftView: BaseView! // BaseView继承自NSView，只是isFlipped为true
    @IBOutlet var rightView: BaseView!

    var scrollView: NSScrollView!    // collection view 容器
    var collectionView: NSCollectionView! 
    
    override func viewDidLoad() {
        super.viewDidLoad()

        loadSubviews()
    }

    func loadSubviews() {
        // 创建layout对象
        let flowLayout = NSCollectionViewFlowLayout()
        flowLayout.itemSize = NSSize(width: 48, height: 48)
        flowLayout.sectionInset = NSEdgeInsets(top: 10.0, left: 20.0, bottom: 10.0, right: 20.0)
        flowLayout.minimumInteritemSpacing = 20.0
        flowLayout.minimumLineSpacing = 20.0
       
        // 创建collection view，不要提供具体的frame，否则item无法正常展出
        collectionView = NSCollectionView()
        collectionView.collectionViewLayout = flowLayout
        collectionView.wantsLayer = true
        collectionView.layer?.backgroundColor = NSColor.yellow.cgColor
        collectionView.layer?.cornerRadius = 4
        collectionView.register(CollectionViewItem.self, forItemWithIdentifier: NSUserInterfaceItemIdentifier(rawValue: "CollectionViewItem"))
        collectionView.dataSource = self
        collectionView.delegate = self
        
        // 创建scroll view，作为collection view的容器
        scrollView = NSScrollView(frame: NSRect(x: 20, y: 108 + 2 * 48 + 3 * 15, width: 48 * 5 + 60, height: 120))
        // 将collection view作为scroll view的容器
        scrollView.documentView = collectionView
        // 将scroll view添加到视图层级
        leftView.addSubview(scrollView)
        
        leftView.wantsLayer = true
        leftView.layer?.backgroundColor = NSColor.white.cgColor
        
        rightView.wantsLayer = true
        rightView.layer?.backgroundColor = NSColor.white.cgColor
    }
    
    override func viewWillLayout() {
        // 视图布局发生变化时，更新scroll view的布局（这一步可选）
        scrollView.frame = leftView.bounds
    }
}
```

- 创建Item类：

``` swift
class CollectionViewItem: NSCollectionViewItem {
    // 因为不适用xib，此处必须重载该方法，创建对应的view
    override func loadView() {
        view = NSView()
        view.wantsLayer = true
        view.layer?.backgroundColor = NSColor.red.cgColor
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
    }
    
    override func viewWillLayout() {
        print("view is: \(view), frame is: \(view.frame)")
    }
    
}
```

- 设置Collection View的delegate和datasource：

``` swift
extension ViewController: NSCollectionViewDelegate, NSCollectionViewDataSource {
    func numberOfSections(in collectionView: NSCollectionView) -> Int {
        return 2
    }

    func collectionView(_ collectionView: NSCollectionView, numberOfItemsInSection section: Int) -> Int {
        return 2
    }
    
    func collectionView(_ collectionView: NSCollectionView, itemForRepresentedObjectAt indexPath: IndexPath) -> NSCollectionViewItem {
        let item = collectionView.makeItem(withIdentifier: NSUserInterfaceItemIdentifier(rawValue: "CollectionViewItem"), for: indexPath) as! CollectionViewItem
        
        return item
    }   
}
```

添加上述代码以后，此时编译运行，便可以看到Collection View已经创建成功了。😀

![image-20190903124747550](/../../../../../../../media/2019-09-03-comparing-nscollectionview-and-uicollectionview/image-20190903124747550.png)

# 参考

[NSCollectionView](https://developer.apple.com/documentation/appkit/nscollectionview)

[NSCollectionViewItem](https://developer.apple.com/documentation/appkit/nscollectionviewitem)