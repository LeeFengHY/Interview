# Interview

# 面试题集（APM、性能优化、runtime、runloop、设计模式等）
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
答：上面这题修饰词使用不当也会发生，blok里面持有self，self持有block、delegate用strong修饰、timer强制持有target，如果timer到点后不调用invalidate的话也会发生、两个对象相互引用用strong修饰。

8. A、B两个label用autolayout横向布局，如何让文字过长时挤压A而不挤压B？
答：设置A视图最大宽度相对于父视图垂直居中，距左leading = 10，B视图距A视图右边为10px，距父视图右边上下0px。
```
