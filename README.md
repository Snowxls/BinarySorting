二进制重排 

使用.order文件

工程路径下生成.order文件

![](https://upload-images.jianshu.io/upload_images/8416233-c734231fba45dcff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

工程中设置order文件

![](https://upload-images.jianshu.io/upload_images/8416233-b5e8f9fa43cde3ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Clang插桩设置

![](https://upload-images.jianshu.io/upload_images/8416233-7a8ef39d564bb06b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
-fsanitize-coverage=trace-pc-guard
```

工程中第一个ViewController中添加以下代码

head

```
#include <stdint.h>
#include <stdio.h>
#include <sanitizer/coverage_interface.h>
#import <dlfcn.h>
#import <libkern/OSAtomic.h>
```

结构体和原子列表设置

```
//原子队列
static OSQueueHead symbolList = OS_ATOMIC_QUEUE_INIT;
//定义符号结构体
typedef struct {
    void *pc;
    void *next; //用来取下一个数据
}SYNode;
```

Clang相关

```
void __sanitizer_cov_trace_pc_guard_init(uint32_t *start,
                                                    uint32_t *stop) {
  static uint64_t N;  // Counter for the guards.
  if (start == stop || *start) return;  // Initialize only once.
  printf("INIT: %p %p\n", start, stop);
    printf("总计：%x",*(stop -1));
  for (uint32_t *x = start; x < stop; x++)
    *x = ++N;  // Guards should start from 1.
}

//所有的OC方法 C函数 Block 都Hook到了
void __sanitizer_cov_trace_pc_guard(uint32_t *guard) {
  if (!*guard) return;
  void *PC = __builtin_return_address(0); //内存地址
    printf("PC____:%p",PC);
    SYNode *node = malloc(sizeof(SYNode));
    *node = (SYNode){PC,NULL};
    //进入
    OSAtomicEnqueue(&symbolList, node, offsetof(SYNode, next));// offsetof(SYNode, next) 计算出next在SYNode的偏移值
    
    
    /*
     const char      *dli_fname; // 函数所在的文件路径
     void            *dli_fbase; // 函数所在的文件地址
     const char      *dli_sname; // 符号
     void            *dli_saddr; // 符号地址
     */
    Dl_info info;
    dladdr(PC, &info); //将地址赋值到info结构体上
    printf(" dli_fname:%s\n dli_fbase:%p\n dli_sname:%s\n dli_saddr:%p",info.dli_fname,info.dli_fname,info.dli_sname,info.dli_saddr);
    
  char PcDescr[1024];
//  __sanitizer_symbolize_pc(PC, "%p %F %L", PcDescr, sizeof(PcDescr));
  printf("guard: %p %x PC %s\n", guard, *guard, PcDescr);
}
```

通过点击屏幕获取列表

```
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    NSMutableArray <NSString *> * symbolNames = [NSMutableArray array];
    
    while (YES) {
        SYNode * node = OSAtomicDequeue(&symbolList, offsetof(SYNode, next));
        if (node == NULL) {
            break;
        }
        Dl_info info;
        dladdr(node->pc, &info);
        NSString * name = @(info.dli_sname);//取出符号
        //判断是否为OC方法
        BOOL  isObjc = [name hasPrefix:@"+["] || [name hasPrefix:@"-["];
        NSString * symbolName = isObjc ? name: [@"_" stringByAppendingString:name];//如果不是OC方法添加前缀"_"
        [symbolNames addObject:symbolName];
    }
    //取反
    NSEnumerator * emt = [symbolNames reverseObjectEnumerator];
    //去重
    NSMutableArray<NSString *> *funcs = [NSMutableArray arrayWithCapacity:symbolNames.count];
    NSString * name;
    while (name = [emt nextObject]) {
        if (![funcs containsObject:name]) {
            [funcs addObject:name];
        }
    }
    [funcs removeObject:[NSString stringWithFormat:@"%s",__FUNCTION__]];
    //将数组变成字符串
    NSString * funcStr = [funcs  componentsJoinedByString:@"\n"];
    
    //可以将数据存在沙盒中
    NSString * filePath = [NSTemporaryDirectory() stringByAppendingPathComponent:@"snow.order"];
    NSData * fileContents = [funcStr dataUsingEncoding:NSUTF8StringEncoding];
    [[NSFileManager defaultManager] createFileAtPath:filePath contents:fileContents attributes:nil];
    
    //也可以将答应出的文本直接复制到工程目录下的 snow.order 文件中
    NSLog(@"%@",funcStr);
}
```