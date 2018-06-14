# 自动释放池与@autoreleasepool
## 一、Autoreleasepool概念

* 自动释放池(autorelease pool)是OC的一种内存自动回收机制. 具有延迟释放的特性，即当我们创建了一个对象，并把他加入到了自动释放池中时，他不会立即被释放，会等到一次runloop结束或者作用域超出{}或者超出[pool release]之后再被释放
* 当你向一个对象发送一个autorelease消息的时候,Cocoa就会将对象的一个引用放入 
到最新的自动释放池中(当前线程栈顶位置),它仍然是一个对象,因此自动释放池定义的作用域内的其他对象都可以向他发送消息.一个自动释放池存储的对象当自己被销毁的时会向其中的对象发送 release 消息。

## 二、Autoreleasepool创建和销毁时机
### MRC
* 通过手动创建的方式来创建自动释放池，这种方式创建的自动释放池需要手动调用release（引用计数环境下，调用release和drain的效果相同，但是在CG下，drain会触发GC,release不会），方法如下：

```
NSAutoreleasePool *pool = [[ NSAutoreleasePool alloc]init ];//创建一个自动释放池
    Person *person = [[Person alloc]init];
    //调autorelease方法将对象加入到自动释放池
    //注意使用该方法，对象不会自己加入到自动释放池，需要人为调用autorelease方法加入
    [person autorelease];
    //,手动释放自动释放池执行完这行代码是，自动释放池会对加入他中的对象做一次release操作
    [pool release];

```
自动释放池销毁时机：[pool release]代码执行完后
* 通过@autoreleasepool来创建
    - 对象的创建在自动释放池里面

```
@autoreleasepool {
        //在这个{}之内的变量默认被添加到自动释放池
         Person *p = [[Person alloc] init];
      }//除了这个括号，p被释放
```
*   - 如果一个变量在自动释放池之外创建，如下，需要通过__autoreleasing该修饰符将其加入到自动释放池。
    
```
@autoreleasepool {

}
Person *   __autoreleasing p = [
[Person alloc]init];
 self.person = p;
```
系统就是通过@autoreleasepool {}这种方式来为我们创建自动释放池的，一个线程对应一个runloop，系统会为每一个runloop隐式的创建一个自动释放池，所有的autoreleasePool构成一个栈式结构，在每个runloop结束时，当前栈顶的autoreleasePool会被销毁，而且会对其中的每一个对象做一次release（严格来说，是你对这个对象做了几次autorelease就会做几次release，不一定是一次)，特别指出，使用容器的block版本的枚举器的时候，系统会自动添加一个autoreleasePool 
[array enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) { 
// 这里被一个局部@autoreleasepool包围着 
}];

### ARC

ARC下除了NSAutoreleasePool不可用以外，其他的同MRC

## 三、Autoreleasepool 应用场景

MRC： 
* 对象作为函数返回值 
    - 当一个对象要作为函数返回值的时候，因为要遵循谁申请谁释放的思想，所以应该在返回之前释放，但要是返回之前释放了，就会造成野指针错误，但是要是不释放，那么就违背了谁申请谁释放的原则，所以就可以使用autorelease延迟释放的特性，将其在返回之前做一次autorelease，加入到自动释放池中，保证可以被返回，一次runloop之>>后系统会帮我们释放他
* 自动释放池可以延长对象的声明周期，如果一个事件周期很长，比如有一个很长的循环逻辑，那么一个临时变量可能很长时间都不会被释放，一直在内存中保留，那么内存的峰值就会一直增加，但是其实这个临时变量是我们不再需要的。这个时候就通过创建新的自动释放池来缩短临时变量的生命周期来降低内存的峰值。
* 临时生成大量对象,一定要将自动释放池放在for循环里面，要释放在外面，就会因为大量对象得不到及时释放，而造成内存紧张，最后程序意外退出

```
for (int i = 0; i<10000; i++) {
        @autoreleasepool {
            UIImageView *imegeV = [[UIImageView alloc]init];
            imegeV.image = [UIImage imageNamed:@"efef"];
            [self.view addSubview:imegeV];
        }
    }
    
```

### 嵌套的autoreleasepool

```
@autoreleasepool {//p1被放在该自动释放池里面
        Person *p1 = [[Person alloc]init];
        @autoreleasepool {//p2被放在该自动释放池里面
            Person *p2 = [[Person alloc]init];
        }
    }
```
以上说明autorelease对象被添加进离他最近的自动释放池，多层的pool会有多个哨兵对象。

## 四、Autoreleasepool 底层实现

### Autoreleasepool实现原理
在了解自动释放池的原理之前，我们先来看一下以下几个概念
#### AutoreleasePoolPage
* ARC下，我们使用@autoreleasepool{}来使用一个AutoreleasePool，随后编译器将其改写成下面的样子：

```
void *context = objc_autoreleasePoolPush();
// {}中的代码
objc_autoreleasePoolPop(context);
```
* 而这两个函数都是对AutoreleasePoolPage的简单封装，所以自动释放机制的核心就在于这个类。

AutoreleasePoolPage是一个C++实现的类

![image](https://upload-images.jianshu.io/upload_images/1197643-46bde8074d9a11e8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/541)

* AutoreleasePoolPage：每一个自动释放池没有单独的结构，每一个autorealeasePool对象都是由若干个autoreleasePoolPage通过双向链表连接而成，类的定义如下

```
class AutoreleasePoolPage 
{
    magic_t const magic;
    id *next;//指向栈顶最新被添加进来的autorelease对象的下一个位置
    pthread_t const thread;//指向当前线程
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t 
  }
```
* AutoreleasePool是按线程一一对应的（结构中的thread指针指向当前线程）
* AutoreleasePoolPage每个对象会开辟4096字节内存（也就是虚拟内存一页的大小），除了上面的实例变量所占空间，剩下的空间全部用来储存autorelease对象的地址
* 上面的id *next指针作为游标指向栈顶最新add进来的autorelease对象的下一个位置
* 一个AutoreleasePoolPage的空间被占满时，会新建一个AutoreleasePoolPage对象，连接链表，后来的autorelease对象在新的page加入

所以，若当前线程中只有一个AutoreleasePoolPage对象，并记录了很多autorelease对象地址时内存如下图：

![image](https://upload-images.jianshu.io/upload_images/1197643-512c06daee6f18e0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/502)

图中的情况，这一页再加入一个autorelease对象就要满了（也就是next指针马上指向栈顶），这时就要执行上面说的操作，建立下一页page对象，与这一页链表连接完成后，新page的next指针被初始化在栈底（begin的位置），然后继续向栈顶添加新对象。

所以，向一个对象发送- autorelease消息，就是将这个对象加入到当前AutoreleasePoolPage的栈顶next指针指向的位置

#### 哨兵对象
* 本质是一个值为nil的对象，由被定义在* NSAutoreleasePool类中的_token指针来保存。

#### hotPage
* 最新创建的AutoreleasePoolPage.
 
#### 原理
系统通过一个栈来管理所有的自动释放池，每当创建了一个新的自动释放池，系统就会把它压入栈顶，并且传入一个哨兵对象,将哨兵对象插入hotPage，这里分三种情况
* 若hotPage未满，则直接插入哨兵对象，
* 要是满了，新建一个NSAutoreleasePoolPage，并将其作为hotPage，然后将哨兵对象插入
* 如果没有NSAutoreleasePoolPage,则新建一个NSAutoreleasePoolPage，并将其作为hotPage，插入哨兵对象，注意。这里的hotPage是没有父节点的。

每当有一个自动释放池要被释放的时候，哨兵对象就会作为参数被传入，找到该哨兵对象所在的位置后，将所有晚于哨兵对象的autorelease弹出，并对他们做一次release，然后将next指针一到合适的位置。



### Autoreleasepool底层实现

* 首先我们去可以查看一下，clang 转成 c++ 的 autoreleasepool 的源码：

```
extern "C" __declspec(dllimport) void * objc_autoreleasePoolPush(void);
extern "C" __declspec(dllimport) void objc_autoreleasePoolPop(void *);
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};

```
可以发现objc_autoreleasePoolPush() 和 objc_autoreleasePoolPop() 这两个方法。

* 再看一下runtime 中 Autoreleasepool 的结构，通过阅读源码可以看出 Autoreleasepool 是一个由 AutoreleasepoolPage 双向链表的结构，其中 child 指向它的子 page，parent 指向它的父 page。

![image](https://upload-images.jianshu.io/upload_images/1197643-96db24a6be7d1796.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

* 并且每个 AutoreleasepoolPage 对象的大小都是 4096 个字节。

```
#define PAGE_MAX_SIZE           PAGE_SIZE
#define PAGE_SIZE       I386_PGBYTES
#define I386_PGBYTES        4096        /* bytes per 80386 page */

```
* AutoreleasepoolPage 通过压栈的方式来存储每个需要自动释放的对象。

```
//入栈方法
    static inline void *push() 
    {
        id *dest;
        if (DebugPoolAllocation) {
            // Each autorelease pool starts on a new pool page.
            //在 Debug 情况下每一个自动释放池 都以一个新的 poolPage 开始
            dest = autoreleaseNewPage(POOL_BOUNDARY);
        } else {
//正常情况下，调用 push 方法会先插入一个 POOL_BOUNDARY 标志位
            dest = autoreleaseFast(POOL_BOUNDARY);
        }
        assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
        return dest;
    }

```
* 然后我们来看看 runtime 中的源码

```
/***********************************************************************
 自动释放池的实现：
 一个线程的自动释放池是一个指针堆栈
 每一个指针或者指向被释放的对象，或者是自动释放池的 POOL_BOUNDARY，POOL_BOUNDARY 是自动释放池的边界。
 一个池子的 token 是指向池子 POOL_BOUNDARY 的指针。当池子被出栈的时候，每一个高于标准的对象都会被释放掉。
 堆栈被分成一个页面的双向链表。页面按照需要添加或者删除。
 本地线程存放着指向当前页的指针，在这里存放着新创建的自动释放的对象。
**********************************************************************/
// Set this to 1 to mprotect() autorelease pool contents
//将这个设为 1 可以通过mprotect修改映射存储区的权限来更改自动释放池的内容
#define PROTECT_AUTORELEASEPOOL 0
#define CHECK_AUTORELEASEPOOL (DEBUG)
BREAKPOINT_FUNCTION(void objc_autoreleaseNoPool(id obj));
BREAKPOINT_FUNCTION(void objc_autoreleasePoolInvalid(const void *token));
namespace {
//对AutoreleasePoolPage进行完整性校验
struct magic_t {
    static const uint32_t M0 = 0xA1A1A1A1;
#   define M1 "AUTORELEASE!"
    static const size_t M1_len = 12;
    uint32_t m[4];    
    magic_t() {
        assert(M1_len == strlen(M1));
        assert(M1_len == 3 * sizeof(m[1]));
        m[0] = M0;
        strncpy((char *)&m[1], M1, M1_len);
    }
    ~magic_t() {
        m[0] = m[1] = m[2] = m[3] = 0;
    }
    bool check() const {
        return (m[0] == M0 && 0 == strncmp((char *)&m[1], M1, M1_len));
    }
    bool fastcheck() const {
#if CHECK_AUTORELEASEPOOL
        return check();
#else
        return (m[0] == M0);
#endif
    }
#   undef M1
};
//自动释放页
class AutoreleasePoolPage 
{
  //EMPTY_POOL_PLACEHOLDER 被存放在本地线程存储中当一个池入栈并且没有存放任何对象的时候。这样在栈顶入栈出栈并且没有使用它们的时候会节省内存。
#   define EMPTY_POOL_PLACEHOLDER ((id*)1)
#   define POOL_BOUNDARY nil
    static pthread_key_t const key = AUTORELEASE_POOL_KEY;
    static uint8_t const SCRIBBLE = 0xA3;  // 0xA3A3A3A3 after releasing
    static size_t const SIZE = 
#if PROTECT_AUTORELEASEPOOL
        PAGE_MAX_SIZE;  // must be multiple of vm page size
#else
        PAGE_MAX_SIZE;  // size and alignment, power of 2
#endif
  //对象数量
    static size_t const COUNT = SIZE / sizeof(id);
//校验完整性
    magic_t const magic;
//页中对象的下一位索引
    id *next;
//线程
    pthread_t const thread;
//父页
    AutoreleasePoolPage * const parent;
//子页
    AutoreleasePoolPage *child;
//深度
    uint32_t const depth;
    uint32_t hiwat;
    // SIZE-sizeof(*this) bytes of contents follow
//创建
    static void * operator new(size_t size) {
        return malloc_zone_memalign(malloc_default_zone(), SIZE, SIZE);
    }
//删除
    static void operator delete(void * p) {
        return free(p);
    }
//设置当前内存可读
    inline void protect() {
#if PROTECT_AUTORELEASEPOOL
        mprotect(this, SIZE, PROT_READ);
        check();
#endif
    }
//设置当前内存可读可写
    inline void unprotect() {
#if PROTECT_AUTORELEASEPOOL
        check();
        mprotect(this, SIZE, PROT_READ | PROT_WRITE);
#endif
    }
  //初始化
    AutoreleasePoolPage(AutoreleasePoolPage *newParent) 
        : magic(), next(begin()), thread(pthread_self()),
          parent(newParent), child(nil), 
          depth(parent ? 1+parent->depth : 0), 
          hiwat(parent ? parent->hiwat : 0)
    { 
        if (parent) {
            parent->check();
            assert(!parent->child);
            parent->unprotect();
            parent->child = this;
            parent->protect();
        }
        protect();
    }
//析构
    ~AutoreleasePoolPage() 
    {
        check();
        unprotect();
        assert(empty());
        // Not recursive: we don't want to blow out the stack 
        // if a thread accumulates a stupendous amount of garbage
        assert(!child);
    }
//被破坏的
    void busted(bool die = true) 
    {
        magic_t right;
        (die ? _objc_fatal : _objc_inform)
            ("autorelease pool page %p corrupted\n"
             "  magic     0x%08x 0x%08x 0x%08x 0x%08x\n"
             "  should be 0x%08x 0x%08x 0x%08x 0x%08x\n"
             "  pthread   %p\n"
             "  should be %p\n", 
             this, 
             magic.m[0], magic.m[1], magic.m[2], magic.m[3], 
             right.m[0], right.m[1], right.m[2], right.m[3], 
             this->thread, pthread_self());
    }
//校验
    void check(bool die = true) 
    {
        if (!magic.check() || !pthread_equal(thread, pthread_self())) {
            busted(die);
        }
    }
//快速校验
    void fastcheck(bool die = true) 
    {
#if CHECK_AUTORELEASEPOOL
        check(die);
#else
        if (! magic.fastcheck()) {
            busted(die);
        }
#endif
    }
//页的开始位置
    id * begin() {
        return (id *) ((uint8_t *)this+sizeof(*this));
    }
//页的结束位置
    id * end() {
        return (id *) ((uint8_t *)this+SIZE);
    }
//页是否是空的
    bool empty() {
        return next == begin();
    }
//页是否是满的
    bool full() { 
        return next == end();
    }
//是否少于一半
    bool lessThanHalfFull() {
        return (next - begin() < (end() - begin()) / 2);
    }
//添加对象
    id *add(id obj)
    {
        assert(!full());
        unprotect();
        id *ret = next;  // faster than `return next-1` because of aliasing
        *next++ = obj;
        protect();
        return ret;
    }
//释放所有对象
    void releaseAll() 
    {
        releaseUntil(begin());
    }
//释放到 stop 的位置之前的所有对象
    void releaseUntil(id *stop) 
    {
        // Not recursive: we don't want to blow out the stack 
        // if a thread accumulates a stupendous amount of garbage
            while (this->next != stop) {
            // Restart from hotPage() every time, in case -release 
            // autoreleased more objects
            AutoreleasePoolPage *page = hotPage();
            // fixme I think this `while` can be `if`, but I can't prove it
            while (page->empty()) {
                page = page->parent;
                setHotPage(page);
            }
            page->unprotect();
            id obj = *--page->next;
//将页索引内容置为 SCRIBBLE 表示已经被释放
            memset((void*)page->next, SCRIBBLE, sizeof(*page->next));
            page->protect();
            if (obj != POOL_BOUNDARY) {
                objc_release(obj);
            }
        }
        setHotPage(this);
#if DEBUG
        // we expect any children to be completely empty
        for (AutoreleasePoolPage *page = child; page; page = page->child) {
            assert(page->empty());
        }
#endif
    }
//杀死
    void kill() 
    {
        // Not recursive: we don't want to blow out the stack 
        // if a thread accumulates a stupendous amount of garbage
        AutoreleasePoolPage *page = this;
        while (page->child) page = page->child;
        AutoreleasePoolPage *deathptr;
        do {
            deathptr = page;
            page = page->parent;
            if (page) {
                page->unprotect();
                page->child = nil;
                page->protect();
            }
            delete deathptr;
        } while (deathptr != this);
    }
//释放本地线程存储空间
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
//获取 AutoreleasePoolPage
    static AutoreleasePoolPage *pageForPointer(const void *p) 
    {
        return pageForPointer((uintptr_t)p);
    }
    static AutoreleasePoolPage *pageForPointer(uintptr_t p) 
    {
        AutoreleasePoolPage *result;
        uintptr_t offset = p % SIZE;
        assert(offset >= sizeof(AutoreleasePoolPage));
        result = (AutoreleasePoolPage *)(p - offset);
        result->fastcheck();
        return result;
    }
//是否有空池占位符
    static inline bool haveEmptyPoolPlaceholder()
    {
        id *tls = (id *)tls_get_direct(key);
        return (tls == EMPTY_POOL_PLACEHOLDER);
    }
//设置空池占位符
    static inline id* setEmptyPoolPlaceholder()
    {
        assert(tls_get_direct(key) == nil);
        tls_set_direct(key, (void *)EMPTY_POOL_PLACEHOLDER);
        return EMPTY_POOL_PLACEHOLDER;
    }
//获取当前页
    static inline AutoreleasePoolPage *hotPage() 
    {
        AutoreleasePoolPage *result = (AutoreleasePoolPage *)
            tls_get_direct(key);
        if ((id *)result == EMPTY_POOL_PLACEHOLDER) return nil;
        if (result) result->fastcheck();
        return result;
    }
//设置当前页
    static inline void setHotPage(AutoreleasePoolPage *page) 
    {
        if (page) page->fastcheck();
        tls_set_direct(key, (void *)page);
    }
//获取 coldPage
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
//快速释放
    static inline id *autoreleaseFast(id obj)
    {
        AutoreleasePoolPage *page = hotPage();
        if (page && !page->full()) {
            return page->add(obj);
        } else if (page) {
            return autoreleaseFullPage(obj, page);
        } else {
            return autoreleaseNoPage(obj);
        }
    }
//添加自动释放对象，当页满的时候调用这个方法
    static __attribute__((noinline))
    id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
    {
        // The hot page is full. 
        // Step to the next non-full page, adding a new page if necessary.
        // Then add the object to that page.
        assert(page == hotPage());
        assert(page->full()  ||  DebugPoolAllocation);
        do {
            if (page->child) page = page->child;
            else page = new AutoreleasePoolPage(page);
        } while (page->full());
        setHotPage(page);
        return page->add(obj);
    }
//添加自动释放对象，当没页的时候使用这个方法
    static __attribute__((noinline))
    id *autoreleaseNoPage(id obj)
    {
        // "No page" could mean no pool has been pushed
        // or an empty placeholder pool has been pushed and has no contents yet
        assert(!hotPage());
        bool pushExtraBoundary = false;
        if (haveEmptyPoolPlaceholder()) {
            // We are pushing a second pool over the empty placeholder pool
            // or pushing the first object into the empty placeholder pool.
            // Before doing that, push a pool boundary on behalf of the pool 
            // that is currently represented by the empty placeholder.
            pushExtraBoundary = true;
        }
        else if (obj != POOL_BOUNDARY  &&  DebugMissingPools) {
            // We are pushing an object with no pool in place, 
            // and no-pool debugging was requested by environment.
            _objc_inform("MISSING POOLS: (%p) Object %p of class %s "
                         "autoreleased with no pool in place - "
                         "just leaking - break on "
                         "objc_autoreleaseNoPool() to debug", 
                         pthread_self(), (void*)obj, object_getClassName(obj));
            objc_autoreleaseNoPool(obj);
            return nil;
        }
        else if (obj == POOL_BOUNDARY  &&  !DebugPoolAllocation) {
            // We are pushing a pool with no pool in place,
            // and alloc-per-pool debugging was not requested.
            // Install and return the empty pool placeholder.
            return setEmptyPoolPlaceholder();
        }
        // We are pushing an object or a non-placeholder'd pool.
        // Install the first page.
        AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
        setHotPage(page);
        // Push a boundary on behalf of the previously-placeholder'd pool.
        if (pushExtraBoundary) {
            page->add(POOL_BOUNDARY);
        }
        // Push the requested object or pool.
        return page->add(obj);
    }
    static __attribute__((noinline))
    id *autoreleaseNewPage(id obj)
    {
        AutoreleasePoolPage *page = hotPage();
        if (page) return autoreleaseFullPage(obj, page);
        else return autoreleaseNoPage(obj);
    }
//公开方法
public:
//自动释放
    static inline id autorelease(id obj)
    {
        assert(obj);
        assert(!obj->isTaggedPointer());
        id *dest __unused = autoreleaseFast(obj);
        assert(!dest  ||  dest == EMPTY_POOL_PLACEHOLDER  ||  *dest == obj);
        return obj;
    }
//入栈
    static inline void *push() 
    {
        id *dest;
        if (DebugPoolAllocation) {
            // Each autorelease pool starts on a new pool page.
            dest = autoreleaseNewPage(POOL_BOUNDARY);
        } else {
            dest = autoreleaseFast(POOL_BOUNDARY);
        }
        assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
        return dest;
    }
//兼容老的 SDK 出栈方法
    static void badPop(void *token)
    {
        // Error. For bincompat purposes this is not 
        // fatal in executables built with old SDKs.
        if (DebugPoolAllocation  ||  sdkIsAtLeast(10_12, 10_0, 10_0, 3_0)) {
            // OBJC_DEBUG_POOL_ALLOCATION or new SDK. Bad pop is fatal.
            _objc_fatal
                ("Invalid or prematurely-freed autorelease pool %p.", token);
        }
        // Old SDK. Bad pop is warned once.
        static bool complained = false;
        if (!complained) {
            complained = true;
            _objc_inform_now_and_on_crash
                ("Invalid or prematurely-freed autorelease pool %p. "
                 "Set a breakpoint on objc_autoreleasePoolInvalid to debug. "
                 "Proceeding anyway because the app is old "
                 "(SDK version " SDK_FORMAT "). Memory errors are likely.",
                     token, FORMAT_SDK(sdkVersion()));
        }
        objc_autoreleasePoolInvalid(token);
    }
//出栈
    static inline void pop(void *token) 
    {
        AutoreleasePoolPage *page;
        id *stop;
        if (token == (void*)EMPTY_POOL_PLACEHOLDER) {
            // Popping the top-level placeholder pool.
            if (hotPage()) {
                // Pool was used. Pop its contents normally.
                // Pool pages remain allocated for re-use as usual.
                pop(coldPage()->begin());
            } else {
                // Pool was never used. Clear the placeholder.
                setHotPage(nil);
            }
            return;
        }
        page = pageForPointer(token);
        stop = (id *)token;
        if (*stop != POOL_BOUNDARY) {
            if (stop == page->begin()  &&  !page->parent) {
                // Start of coldest page may correctly not be POOL_BOUNDARY:
                // 1. top-level pool is popped, leaving the cold page in place
                // 2. an object is autoreleased with no pool
            } else {
                // Error. For bincompat purposes this is not 
                // fatal in executables built with old SDKs.
                return badPop(token);
            }
        }
//打印 hiwat
        if (PrintPoolHiwat) printHiwat();
        page->releaseUntil(stop);
        // memory: delete empty children
        if (DebugPoolAllocation  &&  page->empty()) {
            // special case: delete everything during page-per-pool debugging
            AutoreleasePoolPage *parent = page->parent;
            page->kill();
            setHotPage(parent);
        } else if (DebugMissingPools  &&  page->empty()  &&  !page->parent) {
            // special case: delete everything for pop(top) 
            // when debugging missing autorelease pools
            page->kill();
            setHotPage(nil);
        } 
        else if (page->child) {
            // hysteresis: keep one empty child if page is more than half full
            if (page->lessThanHalfFull()) {
                page->child->kill();
            }
            else if (page->child->child) {
                page->child->child->kill();
            }
        }
    }
    static void init()
    {
        int r __unused = pthread_key_init_np(AutoreleasePoolPage::key, 
                                             AutoreleasePoolPage::tls_dealloc);
        assert(r == 0);
    }
//打印
    void print() 
    {
        _objc_inform("[%p]  ................  PAGE %s %s %s", this, 
                     full() ? "(full)" : "", 
                     this == hotPage() ? "(hot)" : "", 
                     this == coldPage() ? "(cold)" : "");
        check(false);
        for (id *p = begin(); p < next; p++) {
            if (*p == POOL_BOUNDARY) {
                _objc_inform("[%p]  ################  POOL %p", p, p);
            } else {
                _objc_inform("[%p]  %#16lx  %s", 
                             p, (unsigned long)*p, object_getClassName(*p));
            }
        }
    }
//打印所有
    static void printAll()
    {        
        _objc_inform("##############");
        _objc_inform("AUTORELEASE POOLS for thread %p", pthread_self());
        AutoreleasePoolPage *page;
        ptrdiff_t objects = 0;
        for (page = coldPage(); page; page = page->child) {
            objects += page->next - page->begin();
        }
        _objc_inform("%llu releases pending.", (unsigned long long)objects);
        if (haveEmptyPoolPlaceholder()) {
            _objc_inform("[%p]  ................  PAGE (placeholder)", 
                         EMPTY_POOL_PLACEHOLDER);
            _objc_inform("[%p]  ################  POOL (placeholder)", 
                         EMPTY_POOL_PLACEHOLDER);
        }
        else {
            for (page = coldPage(); page; page = page->child) {
                page->print();
            }
        }

        _objc_inform("##############");
    }
//打印 hiwat 
    static void printHiwat()
    {
        // Check and propagate high water mark
        // Ignore high water marks under 256 to suppress noise.
        AutoreleasePoolPage *p = hotPage();
        uint32_t mark = p->depth*COUNT + (uint32_t)(p->next - p->begin());
        if (mark > p->hiwat  &&  mark > 256) {
            for( ; p; p = p->parent) {
                p->unprotect();
                p->hiwat = mark;
                p->protect();
            }
            _objc_inform("POOL HIGHWATER: new high water mark of %u "
                         "pending releases for thread %p:", 
                         mark, pthread_self());
            void *stack[128];
            int count = backtrace(stack, sizeof(stack)/sizeof(stack[0]));
            char **sym = backtrace_symbols(stack, count);
            for (int i = 0; i < count; i++) {
                _objc_inform("POOL HIGHWATER:     %s", sym[i]);
            }
            free(sym);
        }
    }
#undef POOL_BOUNDARY
};

```

## Autoreleasepool的优化

### ARC下autorelease的优化
ARC下，runtime提供了一套对autorelease返回值的优化策略TLS(线程局部存储),就是为每个线程单独分配一块栈控件。以key-value的形式对数据进行存储（ARC对autorelease对象的优化标记）。

#### 先看优化中涉及到的几个函数
1、__builtin_return_address
> 是一个内建函数，作用是返回函数的地址，参数是层级，如果是0，则表示是当前函数体返回地址；如果是1：则表示调用这个函数的外层函数。当我们知道了一个函数体的返回地址的时候，就可以根据反汇编，利用某些固定的偏移量，被调方可以定位到主调放在返回值后>面的汇编指令，来确定该条函数指令调用完成后下一个调用的函数

* 这个内建函数原型是char 
* __builtin_return_address(int level)，作用是得到函数的返回地址，参数表示层数，如__builtin_return_address(0)表示当前函数体返回地址，传1是调用这个函数的外层函数的返回值地址，以此类推。

```
- (int)foo {
    NSLog(@"%p", __builtin_return_address(0)); // 根据这个地址能找到下面ret的地址
    return 1;
}
// caller
int ret = [sark foo];
```


2、Thread Local Storage

Thread Local Storage（TLS）线程局部存储，目的很简单，将一块内存作为某个线程专有的存储，以key-value的形式进行读写，比如在非arm架构下，使用pthread提供的方法实现：

```
void* pthread_getspecific(pthread_key_t);
int pthread_setspecific(pthread_key_t , const void *);
```
在返回值身上调用objc_autoreleaseReturnValue方法时，runtime将这个返回值object储存在TLS中，然后直接返回这个object（不调用autorelease）；同时，在外部接收这个返回值的objc_retainAutoreleasedReturnValue里，发现TLS中正好存了这个对象，那么直接返回这个object（不调用retain）。
于是乎，调用方和被调方利用TLS做中转，很有默契的免去了对返回值的内存管理。

* objc_autoreleaseReturnValue
> 通过__builtin_return_address（int level）检测外部是不是ARC环境，可以替代autorelease，是的话直接返回object，不是的话调用objc_autorelease()将对象注册到自动释放池里面，最后通过objc_retain来获取对象。

* objc_retainAutoreleasedReturnValue 
> 与objc_autoreleaseReturnValue配合使用，如果在执行完objc_autoreleaseReturnValue（）这个函数的下一个调用函数是objc_retainAutoreleasedReturnValue，那么就走最优化（在TLS中查询关于这个对象，如果有，直接返回对象，不再对对象做retain）。

* 在调用objc_autoreleaseReturnValue的时候，会先通过__builtin_return_address这个内建函数return address，然后根据这个地址判断主调方在调用完objc_autoreleaseReturnValue这个函数以后是否紧接着调用了objc_retainAutoreleasedReturnValue函数，如果是，那么objc_autoreleaseReturnValue()就不将返回的对象注册到自动释放池里面（不做autorelease），runtime会将这个返回值存储在TLS中，做一个优化标记，然后直接返回这个对象给函数的调用方，在外部接收这个返回值的objc_retainAutoreleasedReturnValue（）会先在TLS中查询有没有这个对象，如果有，那么就直接返回这个对象（不调用retain），所以通过objc_autoreleaseReturnValue和objc_retainAutoreleasedReturnValue的相互配合，利用TSL做一个中转，在ARC下省去了autorelease和retain的步骤，在一定程度上达到了最优化.

## 六、自动释放池常见面试题代码

```
for (int i = 0; i < 100000; ++i) {
        NSString *str = @"Hello World";
        str = [str stringByAppendingFormat:@"- %d",i];  //字符串拼接
        str = [str uppercaseString];   //将字符串替换成大写
    }
```
如果循环的次数过大，会出现什么问题？该怎么解决？

答案：

会出现内存溢出，循环内部创建大量的临时对象，没有被释放

每次循环都将上一次创建的对象release

```
for (int i = 0; i < 100000; ++i) {
       @autoreleasepool{
        NSString *str = @"Hello World";
        str = [str stringByAppendingFormat:@"- %d",i];
        str = [str uppercaseString];
    }
    }
```
