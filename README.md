# Interview-收集身边朋友大厂面试题持续更新

# 面试题集-2017年5月
```swift
1. 请写出下面代码执行顺序以及每次执行前等待了多长时间？并解释下原因？
DispatchQueue.main.async {
            DispatchQueue.main.async {
                sleep(1)
                print("1"+"\(Thread.current)")
            }
            print("2" + "\(Thread.current)")
            DispatchQueue.main.async {
                print("3" + "\(Thread.current)")
            }
}
sleep(1)
答：等待1秒输出2，等待两秒输出1，再输出3，main是一个串行队列，每次按顺序执行，由于mian 异步的原因所以不会阻塞线程。
但是输出2和输出1、3是在两个不同loop周期完成的。
哈哈嵌套async的方式可以很好的把任务分散到多个周期执行，是一种优化的方案。

2. 如果把上面的DispatchQueue.main.async都改成DispatchQueue.global().async是怎么输出呢？并解释下原因？
答：在当今计算机多核情况下，DispatchQueue.global().async都是异步并行队列输出2 再下个loop周期 3 睡眠1秒输出1.
注意：如果都没启动runloop的话，是不会执行的，直到启动loop为止。

3. 如果下面这种情况请输出print输出顺序？并解释原因，如果maxConcurrentOperationCount = 1结果会是什么样子？
let queue = OperationQueue()
        queue.maxConcurrentOperationCount = 2
        queue.addOperation {
            queue.addOperation {
                sleep(2)
                print("1"+"\(Thread.current)")
            }
            print("2"+"\(Thread.current)")
            queue.addOperation {
                print("3"+"\(Thread.current)")
            }
}
sleep(2)
答： addOperation一旦添加到队列中，任务就会被自动异步执行，所以当maxConcurrentOperationCount = 2时输出顺序为2、3睡眠2秒后输出1。
当maxConcurrentOperationCount = 1时输出顺序为2、睡眠2秒输出1、再输出3，因为maxConcurrentOperationCount = 1，1和3用的都是同一个线程。

4. CGPoint在内存中的分配是如何的？
CGPoint在OC中是一个结构体，结构体是采用是内存对齐的方式分配，比如：结构体内有char、float、int、double几种数据类型：
char1个字节、float2个字节int4个字节、double8个字节。
在分配内存的时候按照变量顺序，变量存放的起始地址相对于结构体的起始地址的偏移量必须为该变量的类型所占用的字节数的倍数，不够时填充。
即结构体的size必然是最大变量类型字节数倍数。

5. 编写一个函数，不管调用多少次只执行一次？再写一个函数，在time时间内不论调用多少次，它只执行最后一次函数（debounce）？
答：我快速想到的是定义一个static var flag条件判断来选择执行函数。类似在一个时间内控制Button被恶意点击发生BUG控制思路，一个判断条件bool中间变量和一个时间变量，当时间为>0时把bool变量设置为NO，不调用方法，直到时间为<=0时才调用，可以通过信号量保证时间变量安全操作。

6. 为什么xib连接的property要用weak？用strong会有什么问题？
答：因为xib创建的ViewController或者View，xib是强制持有的，xib连接的属性用weak修饰的话是为了防止相互持有导致谁都释放不了发生内存泄露（定义的属性ViewController或者View weak持有xib对象再则保证了xib生命周期和ViewController和View一样改用strong对象就变为强引用，谁都不能释放内存泄露。

7. 请写出一段导致内存泄露的代码（越多越好）
答：a.上面这题修饰词使用不当也会发生，blok里面持有self，self持有block、delegate用strong修饰、timer强制持有target，如果timer到点后不调用invalidate的话也会发生、两个对象相互引用用strong修饰。
b.下面这段代码：
```objc
+ (instancetype)initWithActionBlock:(LLGestureBlock)block {
  return [[self alloc] initWithTarget: [self _gestureRecognizerBlockTarget:block] selector:@selector(invoke:)];
}

+ (_LLGestureRecognizerBlockTarget *)_gestureRecognizerBlockTarget:(LLGestureBlock)block{
  _LLGestureRecognizerBlockTarget *target = objc_getAssociatedObject(self, &target_key);
  if (!target) {
    target = [[_LLGestureRecognizerBlockTarget alloc]initWithBlock:block];
    objc_setAssociatedObject(self, &target_key, target, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
  }
  return target;
}
```
这样导致的结果就是，变量被绑到了这个类对象上，因为在类方法里面 self 指的是这个类对象，而类对象是常驻内存直到程序退出才释放的，这也就导致了这个类上始终绑着第一次的target，之后无论怎样都不会改变。如果恰好你在 block 中有强制持有了 block 外的其他对象，那就会导致这些对象都不会释放，造成内存泄露。在实例方法中动态绑定即可解决。
```objc
//准确代码
static const int target_key;  
@implementation UIGestureRecognizer (Block)

+(instancetype)nvm_gestureRecognizerWithActionBlock:(LLGestureBlock)block {
  return [[self alloc]initWithActionBlock:block];
}

- (instancetype)initWithActionBlock:(LLGestureBlock)block {
  self = [self init];
  [self addActionBlock:block];
  [self addTarget:self action:@selector(invoke:)];
  return self;
}

- (void)addActionBlock:(LLGestureBlock)block {
  if (block) {
    objc_setAssociatedObject(self, &target_key, block, OBJC_ASSOCIATION_COPY_NONATOMIC);
  }
}

- (void)invoke:(id)sender {
  LLGestureBlock block = objc_getAssociatedObject(self, &target_key);
  if (block) {
    block(sender);
  }
}

@end
```


8. A、B两个label用autolayout横向布局，如何让文字过长时挤压A而不挤压B？
答：设置A、B视图相对于父视图纵向居中，A距左为10px，B距父视图右边10px，A、B相距10px。由于label、imageView、UIButton遵循intrinsicContentSize在不设置大小的情况，指定了位置约束不会出错，现在A、B设置了间距但是由于文字过长的时候谁挤压谁这个是个问题，可以通过设置Content Hugging Priority 和 Content Compression Resistance Priority的优先级来使谁变大谁缩小。
Content Hugging Priority: 该优先级表示一个控件抗被拉伸的优先级。优先级越高，越不容易被拉伸，默认是250。
Content Compression Resistance Priority: 该优先级和上面那个优先级相对应，表示一个控件抗压缩的优先级。优先级越高，越不容易被压缩，默认是750。
所以要实现挤压A不挤压B，就让B的抗压缩优先级大于A的抗压缩优先级即可。

9. 编写一个函数，接受一个数组array作为参数，array中包含N个长度不等的升序数组，请将这N个数组合并，并保证合并后的数组也是升序。
答：归并排序。
    //MARK: 归并排序
    /**
     8 4 5 3 1 2
     8 4 5   3 1 2
     8  4 5  3  1 2
       4   5    1   2 ---> [1 2]-->[3] [1 2]
     第一步：递归将一个大数组切割成N个数组，直到N个元素为1，典型分治思想。
     第二步：再将N个数组递归合并一个大序列，合并过程中做好排序。
     缺点：额外空间和N成正比。
     */
    func mergeSort(_ list: Array<Int>) -> [Int]
    {
        if list.count == 1 { return list}
        let middleIndex = list.count / 2
        let leftArray: Array<Int> = self.mergeSort(Array(list[0..<middleIndex]))
        let rightArray: Array<Int> = self.mergeSort(Array(list[middleIndex..<list.count]))
        return merge(leftArray: leftArray, rightArray: rightArray)
    }
    
    func merge(leftArray: Array<Int>, rightArray: Array<Int>) -> [Int]
    {
        var leftIndex = 0
        var rightIndex = 0
        var orderArray: Array<Int> = Array()
        while leftIndex < leftArray.count && rightIndex < rightArray.count {
            if leftArray[leftIndex] > rightArray[rightIndex] {
                orderArray.append(leftArray[leftIndex])
                leftIndex = leftIndex + 1
            }else if leftArray[leftIndex] < rightArray[rightIndex] {
                orderArray.append(rightArray[rightIndex])
                rightIndex = rightIndex + 1
            }else {
                orderArray.append(leftArray[leftIndex])
                leftIndex = leftIndex + 1
                orderArray.append(rightArray[rightIndex])
                rightIndex = rightIndex + 1
            }
        }
        
        while leftIndex < leftArray.count {
            orderArray.append(leftArray[leftIndex])
            leftIndex = leftIndex + 1
        }
        
        while rightIndex < rightArray.count {
            orderArray.append(rightArray[rightIndex])
            rightIndex = rightIndex + 1
        }
        return orderArray
    }
    
10. 写出快速排序、冒泡排序、选择排序、插入排序。
    // MARK: 快速排序 
    /**
     值类型
     传递的是参数的一个副本，这样在调用参数的过程中不会影响原始数据。
     
     引用类型
     把参数本身引用(内存地址)传递过去，在调用的过程会影响原始数据。
     在Swift众多数据类型中，只有class是引用类型，
     其余的如Int,Float,Bool,Character,Array,Set,enum,struct全都是值类型.
     inout:关键字修饰可以将一个值类型参数以引用方式传递。
     数内部实现改变外部参数
     传入参数时(调用函数时)，在变量名字前面用&符号修饰表示。表明这个变量在参数内部是可以被改变的（可将改变传递到原始数据）
     注意：
     inout修饰的参数是不能有默认值的(比如list = [1, 2, 3]被赋予默认值)，有范围的参数集合也不能被修饰；
     一个参数一旦被inout修饰，就不能再被var和let修饰了。
     */
    func quickSort(list: inout Array<Int>)
    {
        quickRecursive(list: &list, leftIndex: 0, rightIndex: list.count - 1)
    }
    
    func quickRecursive(list: inout Array<Int>, leftIndex: Int, rightIndex: Int)
    {
        if leftIndex >= rightIndex {
            return
        }
        var i = leftIndex + 1
        var j = rightIndex
        let pivot = list[leftIndex]
        while i < j {
            while i < list.count && list[i] > pivot {
                i = i + 1
            }
            while j >= 0 && list[j] < pivot {
                j = j - 1
            }
            if i < j {
                swap(object1: &list[i], object2: &list[j])
            }
        }
        swap(object1: &list[j], object2: &list[leftIndex])
        quickRecursive(list: &list, leftIndex: leftIndex, rightIndex: j - 1)
        quickRecursive(list: &list, leftIndex: j + 1, rightIndex: rightIndex)
    }
    
    func swap(object1: inout Int, object2: inout Int)
    {
        let temp = object1
        object1 = object2
        object2 = temp
    }
    
    // MARK: 冒泡排序
    func bubbleSort(list: inout Array<Int>)  {
        if list.count == 1 {
            return
        }
        //左归排序
        for i in 0..<list.count - 1 {
            for j in 0..<list.count - i - 1 {
                if list[j] > list[j + 1] && j < list.count {
                    swap(object1: &list[j], object2: &list[j+1])
                }
            }
        }
        //右归排序
        for  i in 0..<list.count - 1
        {
            for j in i..<list.count
            {
                if list[i] < list[j] {
                    swap(object1: &list[i], object2: &list[j])
                }
            }
        }
    }
    
    //MARK: 选择排序
    func selectionSort(list: inout Array<Int>) {
        if list.count == 1 {
            return
        }
        for i in 0..<list.count {
            var minItem = list[i]
            var index = i
            for j in (i + 1)..<list.count {
                if list[j] < minItem {
                    minItem = list[j]
                    index = j
                }
            }
            if index != i {
                swap(object1: &list[index], object2: &list[i])
            }
        }
    }

    
    //MARK: 插入排序[8 4 3 5 2 1]
    func insertionSort(list: inout Array<Int>)
    {
        if list.count == 1 { return }
        for i in 1..<list.count {
            var j = i
            while j > 0 && list[j - 1] > list[j] {
                swap(object1: &list[j - 1], object2: &list[j])
                j = j - 1
            }
        }
    }

10. 设计一个可扩展性较强的的缓存池，可以通过传入不同算法实现FIFO、LRU等调度方式。请描述思路并附上伪代码。
答：扩展性强可采用protocol，数据实现了这个协议即可。
    protocol Cacheable {
    var priority: Int {get}
    var node: Any {get}
    var key: String {get}
  }
   class LinkedMapNode: Cacheable {
    var prev: LinkedMapNode?
    var next: LinkedMapNode?
    var cost: Int?
    var time: TimeInterval?
}
    a. 定义一组枚举FIFO、LRU、或者Mix，采用双向链表的方式根据不同的算法，操作缓存池里面的数据。
    b. 比如LRU（使用最少移出），每次新数据直接插入到链表头部，每次数据被使用时移动到链表头部，每次缓存满了或者内存不足时移除链表尾部节点。
    c. FIFO（先进先出）原则，每次把数据插入到链表头部，每次移除尾部节点。
    d. 当然实际中还需要考虑更多更详细的细节，比如数据量过大缓存污染的问题、缓存策略要更加人性化、智能化。
    
11. 存在下面三个接口：
fetchStory(StoryId) -> List<ChapterId>
fetchChapter(ChapterId) -> Chapter
output(Chapter)
设置一套文章请求框架，画图加伪代码
答：a. output应该根据章节缓存顺序执行。
   b. try-catch之后应该取消后续操作。
   c. 请求并发最多为2。

12. 你为代码改进做了哪些工作
答： 主要从性能架构方面入手，毕竟移动设备内存才是重点，加载时间、启动时间的优化；模块解藕、类的设计是否合理等；属性的访问方式是否合理等！一切工作就是为了APP低消耗、高性能。
```
