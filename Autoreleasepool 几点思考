# Autoreleasepool

> 关于Autoreleasepool，解读的文章很多，对此，本文提出几点思考。

### 1. Autoreleasepool 是什么？

 Autoreleasepool，自动释放池，自动管理对象的释放。它没有单独的内存结构，通过以AutoreleasePoolPage 为结点的双向链表来实现。AutoreleasePoolPage 是一个C++类，里面最主要的是三个变量：
 
```
class AutoreleasePoolPage 
{
    id *next;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    .....
}
```
parent 和 child 构成双向链表；next 依次指向最新添加进来的变量，也可以理解成一个栈。一个空的AutoreleasePoolPage内存结构，表示如下：
![一个空的AutoreleasePoolPage的内存结构](http://blog.leichunfeng.com/images/AutoreleasePoolPage.png)
一个添加了变量的AutoreleasePoolPage的内存结构，表示如下：
![一个添加了变量的AutoreleasePoolPage内存结构](http://blog.leichunfeng.com/images/AutoreleasePoolPage2.png)

### 2. AutoreleasePoolPage 保存在了什么地方？ 

- AutoreleasePoolPage 指针，是保存在了 当前线程的TLS里面（Thread Local Storage 线程局部存储）；
- TLS 里面，通过key，获取page指针；
- TLS 总会保存最新的 page，即 hotPage。
- 查询之前的 page，即 coldPage时，是通过 hotPage 的 parent 指针查找的。

```
  //当添加一个变量到AutoreleasePool时
 static inline id *autoreleaseFast(id obj)
 {
    //先查找当前的 AutoreleasePoolPage
     AutoreleasePoolPage *page = hotPage();
     if (page && !page->full()) {
         return page->add(obj);
     } else if (page) {
         return autoreleaseFullPage(obj, page);
     } else {
         return autoreleaseNoPage(obj);
     }
 }
 
 static pthread_key_t const key = AUTORELEASE_POOL_KEY;

  static inline AutoreleasePoolPage *hotPage() 
 {
    //通过key，从TLS中读取hotPage指针
     AutoreleasePoolPage *result = (AutoreleasePoolPage *)
         tls_get_direct(key);
     if ((id *)result == EMPTY_POOL_PLACEHOLDER) return nil;
     if (result) result->fastcheck();
     return result;
 }
 
 //coldPage也是通过hotPage的parent指针，一步步查询的
  static inline AutoreleasePoolPage *coldPage() 
 {
     AutoreleasePoolPage *result = hotPage();
     if (result) {
         while (result->parent) {
             result = result->parent;
             result->fastcheck();
         }
     }
     return result;
 }
```

### 3. 什么样的变量，会被加入到自动释放池？

- MRC 下需要对象调用 autorelease 才会加入；
- ARC 下可以通过 __autoreleasing 修饰符，否则的话看方法名，通过调用 alloc/new/copy/mutablecopy 以外的方法取得的对象，编译器会自动加入 autoreleasepool (使用 alloc/new/copy/mutablecopy 方法进行初始化时，由系统管理对象，在适当的位置 release，不加入 autoreleasepool )；
- __weak 修饰的对象，为了保证在引用时不被废弃，会注册到 autoreleasepool中；
- id 的指针或对象的指针，在没有显式指定时会被注册到 autoreleasepool 中。

> 番外：如何证明一个变量，是否加入到了自动释放池？

### 4. 主线程的自动释放池什么时候创建 ？什么时候释放？

在主线程的runloop中，AutoreleasePool 注册了三个observer：kCFRunLoopEntry、kCFRunLoopBeforeWaiting、kCFRunLoopExit。当runloop进入的时候（即kCFRunLoopEntry），在observer回调中，执行：_objc_autoreleasePoolPush，创建AutoreleasePoolPage，并插入一个边界对象（以前叫哨兵对象，都是nil的别名而已）。当runloop即将休眠（kCFRunLoopBeforeWaiting）和退出（kCFRunLoopExit）的时候，在observer回调中，执行：_objc_autoreleasePoolPop，向next和边界对象之间的所有变量发送release消息（如果对象引用计数为0，就释放掉了，如果不为0，则继续持有）。

### 5. 子线程的runloop默认不开启，那子线程有自动释放池吗？ 子线程中有需要加入自动释放池的变量怎么办？

在子线程中，没有开启runloop的情况下：如果手动创建了 pool 的话，产生的 Autorelease 对象就会交给 pool 去管理；如果没有手动创建 Pool ，但是产生了 Autorelease 对象，就会调用 autoreleaseNoPage 方法，在这个方法中，会自动创建一个AutoreleasePoolPage：即 hotpage，并调用 page->add(obj)将对象添加到 AutoreleasePoolPage 的栈中。

### 6. 子线程中，没有开启runloop时，自动创建的 AutoreleasePool，什么时候释放？

在子线程销毁的时候（再具体点就是 Thread-Local Storage，即线程局部存储 销毁的时候），会同时销毁 AutoreleasePoolPage。

```
static void tls_dealloc(void *p) 
{
    if (p == (void*)EMPTY_POOL_PLACEHOLDER) {
        // No objects or pool pages to clean up here.
        return;
    }

    // reinstate TLS value while we work
    setHotPage((AutoreleasePoolPage *)p);

    if (AutoreleasePoolPage *page = coldPage()) {
        if (!page->empty()) pop(page->begin());  // pop all of the pools
        if (DebugMissingPools || DebugPoolAllocation) {
            // pop() killed the pages already
        } else {
            page->kill();  // free all of the pages
        }
     }
     
     // clear TLS value so TLS destruction doesn't loop
     setHotPage(nil);
 }
```

### 7. AutoreleasePool 和 线程、runloop 之间的关系 ？

- AutoreleasePool 是由 AutoreleasePoolPage 组成的双向链表。第一个page叫 codePage，最后一个page叫 hotPage；
- hotPage 是保存在 线程的 TLS 中；
- 线程销毁的时候，所有 page 都会一同销毁；
- runloop 的‘进入’状态，会创建 page，runloop 的‘休眠之前’状态 和 ‘退出’状态，会释放 page；
- 主线程的runloop默认开启，会自动创建 AutoreleasePool；
- 子线程的runloop默认不开启，不会有 AutoreleasePool；
- 在runloop开启状态下：一个线程，只会有一个runloop，是一对一的关系；一个线程，也只会有一个 AutoreleasePool（即只有一个双向链表，具体多少个page，就不一定了），也是一对一的关系。



