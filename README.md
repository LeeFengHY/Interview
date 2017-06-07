# Interview

# 面试题集
```objc
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
答：等待1秒输出2，等待两秒输出1，再输出3，main是一个串行队列，每次按顺序执行，由于mian 异步的原因所以不会阻塞线程，但是输出2和输出1、3是在两个不同loop
周期完成的。哈哈嵌套async的方式可以很好的把任务分散到多个周期执行，是一种优化的方案。
1. CGPoint在内存中的分配是如何的？
CGPoint是一个在OC中是一个结构体，结构体是采用内存对齐的方式分配，比如：结构体内有char、float、int、double几种数据类型，char一个字节、float二个字节
int四个字节、double8个字节。在分配内存的时候按照顺序，变量存放的起始地址相对于结构体的起始地址的偏移量必须为该变量的类型所占用的字节数的倍数，不够时填充。
即结构体的size必然是最大变量size的倍数。
```
