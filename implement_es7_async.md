
#### 在iOS平台实现新的异步解决方案async,await
async,await是ES7提出的异步解决方案，对比回调链和Promise.then链的异步编程模式，基于async,await可以同步风格编写异步代码，程序逻辑清晰明了.
如顺序读取三个文件:
```JS
function readFile(name) {
  return new Promise((resolve, reject) => {
    //异步读取文件
    fs.readFile(name, (err, data) => {
        if (err) reject(err);
        else resolve(data);
    });
  });
}

async function read3Files() {
  try {
      //读取第1个文件
      let data1 = await readFile('file1.txt');
      //读取第2个文件
      let data2 = await readFile('file2.txt');
      //读取第3个文件
      let data3 = await readFile('file3x.txt');
      //3个文件读取完毕
    } catch (error) {
       //读取出错
    }
}
```
读取文件本身是异步操作，而在要求顺序读取的前提下，基于callback实现将造成很深的回调嵌套:
```JS

function readFile(name, callback) {
  //异步读取文件
  fs.readFile(name, (err, data) => {
     callback(err, data);
  });
}

function read3Files() {
  //读取第1个文件
  readFile('file1.txt', (err, data) => {
    //读取第2个文件
    readFile('file2.txt', (err, data) => {
      //读取第3个文件
      readFile('file3.txt', (err, data) => {
        //3个文件读取完毕
      });
    });
  });
}
```
基于Promise.then链需要将逻辑分散在过多的代码块:
```JS
function readFile(name) {
  return new Promise((resolve, reject) => {
    //异步读取文件
    fs.readFile(name, (err, data) => {
       if (err) reject(err);
       else resolve(data);
    });
  });
}

function read3Files() {
  //读取第1个文件
  readFile('file1.txt')
  .then(data => {
    //读取第2个文件
    return readFile('file2.txt');
  })
  .then(data => {
    //读取第3个文件
    return readFile('file3.txt');
  })
  .then(data => {
    //3个文件读取完毕
  })
  .catch(error => {
    //读取出错
  });
}
```
对比可见aync,await模式的优雅与简洁。接触完毕后，深感如果在iOS项目中也能像JS这般编写异步代码也是极好。经过研究发现要在iOS平台实现这些特性其实并不是很困难，因此本文主旨便是描述async,await在iOS平台的一次实现过程，并给出了一个成果项目.

### 暂时继续讨论JavaScript

#### 1.生成器与迭代器
要明白async,await的机制及运用,需从生成器与迭代器逐步说起.在ES6中，生成器是一个函数，和普通函数的区别是:

##### (1)生成器函数function关键字后多了个*:
```JS
function *numbers() {
}
```

##### (2)生成器函数内可以yield语法多次返回值:
```JS
function *numbers() {
    yield 1;
    yield 2;
    yield 3;
}
```

##### (3)直接调用生成器函数得到的是一个迭代器，通过迭代器的next方法控制生成器的执行:
```JS
let iterator = numbers();

let result = iterator.next();
```
每一次next调用将得到结果result, result对象包含两个属性:`value`和`done`.  value表示此次迭代得到的结果值,done表示是否迭代结束.
比如:
```JS
function *numbers() {
    yield 1;
    yield 2;
    yield 3;
}

let iterator = numbers();
//第1次迭代
let result = iterator.next();
console.log(result);
//输出 => { value: 1, done: false }

//第2次迭代
result = iterator.next();
console.log(result);
//输出 => { value: 2, done: false }

//第3次迭代
result = iterator.next();
console.log(result);
//输出 => { value: 3, done: false }

//第4次迭代
result = iterator.next();
console.log(result);
//输出 => { value: undefined, done: true }
```

第1次调用next,生成器numbers开始执行,执行到第一个yield语句时，numbers将中断，并将结果值`1`返回给迭代器，由于numbers并没有执行完，所以done为false.

第2次调用next,生成器numbers从上次中断的位置恢复执行,继续执行到下一个yield语句时，numbers再次中断，并将结果值`2`返回给迭代器，由于numbers并没有执行完，所以done为false.

第3次调用next,生成器numbers从上次中断的位置恢复执行,继续执行到下一个yield语句时，numbers再次中断，并将结果值`3`返回给迭代器，由于numbers并没有执行完，所以done为false.

第4次调用next,生成器numbers从上次中断的位置恢复执行,此时已是函数尾，numbers将直接return, 由于numbers已经执行完成，所以done为true, 由于numbers并没有显式地返回任何值，因此此次迭代value为undefined.

到此迭代结束，此后通过此迭代器的next方法，都将得到相同的结果`{ value: undefined, done: true }`

##### (4)通过迭代器可向生成器内部传值
```JS
function *hello() {
  let age =  yield 'want age';
  let name = yield 'want name';
  console.log(`Hello, my age: ${age}, name:${name}`);
}

let iterator = hello();
```
创建迭代器并开始如下迭代过程:

第1次迭代,生成器开始执行，到达第一个yield语句时，返回`value = want age, done = false`给迭代器, 并中断。
```JS
let result = iterator.next();
console.log(result);
//输出 => { value: 'want age', done: false }
```
第2次迭代,给next传参28, 生成器从上次中断的地方恢复执行，并将28作为苏醒后yield的内部返回值赋给age; 然后生成器继续执行，再次遇到yield，返回`value = want name, done = false`给迭代器, 并中断。
```JS
result = iterator.next(28);
console.log(result);
//输出 => { value: 'want name', done: false }
```
第3次迭代,给next传参'LiLei', 生成器从上次中断的地方恢复执行，并将'LiLei'作为苏醒后yield的内部返回值赋给name; 然后生成器继续执行，打印log:
```
Hello, my age: 28, name:LiLei
```
然后到达函数尾，彻底结束生成器，并返回`value = undefined, done = true`给迭代器。
```JS
result = iterator.next('LiLei');
console.log(result);
//输出 => { value: undefined, done: true }
```

##### 可见通过迭代器可与生成器“互相交换数据”，生成器通过yield返回数据A给迭代器并中断，而通过迭代器又可以把数据B传给生成器 并 让yied语句苏醒后以B作为右值. 这个特性是下一步"改进异步编程"的重要基础.

至此已基本了解了生成器与迭代器的语法与运用,总结起来:

 ###### 生成器是一个函数，直接调用得到其对应的迭代器，用以控制生成器的逐步执行;
 ###### 生成器内部通过yield语法向迭代器返回值，而且可以多次返回,并多次恢复执行，有别于传统函数"返回便消亡"的特点;
 ###### 可以通过迭代器向生成器内部传值，传入的值将作为本次生成器yield语句苏醒后的右值.


#### 2.通过生成器与迭代器改进异步编程
回想本文开头提到的读取文件例子,如果以callback模式编写:
```JS
function readFile(name, callback) {
  //异步读取文件
  fs.readFile(name, (err, data) => {
     callback(err, data);
  });
}

function read3Files() {
  //读取第1个文件
  readFile('file1.txt', (err, data) => {
    //读取第2个文件
    readFile('file2.txt', (err, data) => {
      //读取第3个文件
      readFile('file3.txt', (err, data) => {
        //3个文件读取完毕
      });
    });
  });
}
```
基于前面起到的"通过迭代器与生成器交换数据"的特性,拓展出新思路:

(1)把读取文件这个动作封装为一个异步操作，通过callback输出结果:err和data.

(2)把read3Files改变为生成器,内部通过yield返回异步操作给执行器(执行器第3步描述).

(3)执行器通过迭代器接收read3Files返回的异步操作,拿到异步操作后，发起该异步操作，得到结果后再其“交换”给生成器read3Files内的yield.

即:
```JS
function readFile(name) {
  //返回一个闭包作为异步操作
  return function(callback) {
    fs.readFile(name, (err, data) => {
       callback(err, data);
    });
  };
}

//执行器
function executor(generator) {
  //创建迭代器
  let iterator = generator();
  //开始第一次迭代
  let result = iterator.next();

  let nextStep = function() {
    //迭代还没结束
    if (!result.done) {
      //从生成器拿到的是一个异步操作
      if (typeof result.value === "function") {
        //发起异步操作
        result.value((err, data) => {
          if(err) {
            //在生成器内部引发异常
            iterator.throw(err);
          }
          else {
            //得到结果值,传给生成器
            result = iterator.next(data);
            //继续下一步迭代
            nextStep();
          }
        });
      }
      //从生成器拿到的是一个普通对象
      else {
        //什么都不做，直接传回给生成器
        result = iterator.next(result.value);
        //继续下一步迭代
        nextStep();
      }
    }
  };
  //开始后续迭代
  nextStep();
}

```
而read3Files改进为:

```js
executor(function *() {
  try {
    //读取第1个文件
    let data1 = yield readFile('file1.txt');
    //读取第2个文件
    let data2 = yield readFile('file2x.txt');
    //读取第3个文件
    let data3 = yield readFile('file3.txt');
  } catch (e) {
    //读取出错
  }
});
```

此时已经把callback模式改进为同步模式。

暂且把传给执行器的生成器函数叫做"异步函数"，执行过程总结起来就是:

异步函数但凡遇到异步操作，就通过yield交给执行器; 执行器但凡拿到异步操作，就发起该操作，拿到实际结果后再将其交换给异步函数。那么在异步函数内，就可以同步风格编写异步代码，因为有了执行器在背后运作，异步函数内的yield就具有了“你给我异步操作,我还你实际结果”的能力.

Promoise同样可作为异步操作:
```JS
function readFile(name) {
  //返回一个Promise作为异步操作
  return new Promise((resolve, reject) => {
    fs.readFile(name, (err, data) => {
        if (err) reject(err);
        else resolve(data);
    });
  });
}
```
在执行器中新增识别Promise的代码
```JS
function executor(generator) {
  //创建迭代器
  let iterator = generator();
  //开始第一次迭代
  let result = iterator.next();

  let nextStep = function() {
    //迭代还没结束
    if (!result.done) {
      if (typeof result.value === "function") {
        ....
      }
      //从生成器拿到的是一个Promise异步操作
      else if (result.value instanceof Promise) {
        //执行该Promise
        result.value.then(data => {
          //得到结果值,传给生成器
          result = iterator.next(data);
          //继续下一步迭代
          nextStep();
        }).catch(err => {
          //在生成器内部引发异常
          iterator.throw(err);
        });
      }
      else {
        ...
      }
    }
  };
  ...
}
```

到此已经成功把异步编程化为同步风格，但或许有个疑问:这个例子倒是化异步为同步风格了，但是那个执行器executor看起来好大一坨，并不优雅.实际上执行器当然是复用的，不用每次都实现执行器.

#### 3.async,await语法糖
到了ES7,async,await终于出来.async与await是上述执行器，生成器模式的语法糖,运用async,await,再也不需要每次都定义生成器作为异步函数,然后显式传给执行器,只要简单在函数定义前增加async,表示这是一个异步函数,内部将用await来等待异步结果:
```JS
async function foo() {
    let value = await 异步操作;
    let value = await 异步操作;
    let value = await 异步操作;
    let value = await 异步操作;
}
```
如读取文件例子:
```JS
async function read3Files() {
    //读取第1个文件
    let data1 = await readFile('file1.txt');
    //读取第2个文件
    let data2 = await readFile('file2.txt');
    //读取第3个文件
    let data3 = await readFile('file3x.txt');
    //3个文件读取完毕
}
```
然后直接调用即可:
```JS
read3Files();
```
async表示该函数内部包含异步操作，需要把它交给内置执行器;

await表示等待异步操作的实际结果。

至此，JS下async/await的来龙去脉已基本描述完毕.


### 回到iOS
光描述JS生成器，迭代器，async,await就花了大量篇幅，因为在iOS上将以它们的JS特性为目标，最终实现OC版的迭代器，生成器，async,await.

#### 1.类型定义
暂时无需在意怎么实现,既然是以前面描述的特性为目标，则可以根据其特性先做如下定义:

先定义yield如下:
```Objective-C
id yield(id value);
```
yield接受一个对象value作为返回给迭代器的值，同时返回一个迭代器设置的新值或者原本值value.

每次迭代的Result:
```Objective-C
@interface Result: NSObject
@property (nonatomic, strong, readonly) id value;
@property (nonatomic, readonly) BOOL done;
@end
```
value表示迭代的结果，为yield返回的对象，或者nil.   done指示是否迭代结束.

根据前面描述的生成器特性,那么在OC里，生成器首先应该是一个C函数/OC方法/block,且内部通过调用yield来返回结果给迭代器:
```Objective-C
void generator() {
    yield(value);
    yield(value);
}

- (void)generator {
    yield(value);
    yield(value);
}

^{
    yield(value);
    yield(value);
}
```
实际上不论是OC方法，还是block,底层调用时都与调用C函数无异.

```objective-c
只是调用block会默认以block结构体地址作为第一个隐含参数
调用方法会以对象自身self，和选择器_cmd作为前两个隐含参数
```

所以只要实现了C函数版生成器，其实现机制将也无缝适用于OC方法，block.



迭代器定义:
```Objective-C
@interface Iterator : NSObject
{
    void (*_func)(void);
}

- (id)initWithFunc:(void (*)(void))func;
- (Result *)next;
- (Result *)next:(id)value;
@end
```
迭代器的创建无法做到像JS一样直接调用生成器即可创建，需要显式创建:

```Objective-C
void generator() {
    yield(value);
    yield(value);
}

Iterator *iterator = [[Iterator alloc] initWithFunc: generator];
```
然后就可以像JS一样调用next来进行迭代:
```Objective-C
Result *result = iterator.next;
//迭代并传值
Result *result = [iterator next: value];
```

#### 2.实现生成器与迭代器

根据需求,yield调用会中断当前执行流,并期望将来能够从中断处继续恢复执行,那么必定要在触发中断时保存现场，包括:

(1)当前指令地址

(2)当前寄存器信息，包括当前栈帧栈顶

##### 而且 中断后到恢复的这段时间内，应当确保yield以及生成器generator的栈帧不会被销毁.

而恢复执行的过程是保存现场的逆过程，即恢复相关寄存器,并跳转到保存的指令地址处继续执行.

上述过程描述起来看似简单，但是如果要自己写汇编代码去保存与恢复现场，并适配各种平台，要保证稳定性还是很难的，好在有C标准库提供的现成利器:setjmp/longjmp。

setjmp/longjmp可以实现跨函数的远程跳转，对比goto只能实现函数内跳转，setjmp/longjmp实现远程跳转基于的就是保存现场与恢复现场的机制,非常符合此处的需求.

#### 实现思路

根据前面对生成器，迭代器的定义及需求推敲整理出如下的实现思路:

(1) 迭代器通过next方法与生成器进行交互时，在next方法内部会将控制流切换到生成器，生成器通过调用yield设置传给迭代器的返回值,并将执行流切换回到next方法。

(2) 切回next方法后,拿到这个值，正常返回给调用者。

(3) 为了确保next方法返回后，生成器的执行栈不被销毁，因此生成器方法的执行需要在一个不被释放的新栈上进行。

(4) 虽然next主要通过恢复现场方式切入生成器，但是首次还是需要通过函数调用方式来进入生成器，通过中介wrapper调用生成器的方式，可以检测到生成器执行结束的事件，然后wrapper再切回 next 方法，并设置done为YES，迭代结束.

整个流程图解如下:

![](http://oem96wx6v.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-04-26%20%E4%B8%8B%E5%8D%885.01.21.png)



乍一看好大一坨，但是只要跟着箭头流程走，思路将很快理清。




根据此思路,为迭代器新增属性如下:
```Objective-C
@interface Iterator : NSObject
{
    int *_ev_leave; //迭代器在next方法内保存的现场
    int *_ev_entry; //生成器通过yield保存的现场
    BOOL _ev_entry_valid; //指示生成器现场是否可用
    void *_stack; //为生成器新分配的栈
    int _stack_size; //为生成器新分配的栈大小
    void (*_func)(void);//迭代器函数指针
    BOOL _done; //是否迭代结束
    id _value; //生成器通过yield传回的值
}
- (id)initWithFunc:(void (*)(void))func;
- (Result *)next;
- (Result *)next:(id)value;
@end
```

为生成器分配新栈,正如前面所述，在迭代器和生成器的生命周期中，next方法的每次迭代是要正常返回的，如果直接在next自己的调用栈上调用wrapper，wrapper再调用生成器，那么next返回后，生成器就算保护了寄存器现场，它的栈帧也被破坏了，再次恢复执行将产生无法预料的结果.

```Objective-C
//默认为生成器分配256K的执行栈
#define DEFAULT_STACK_SIZE (256 * 1024)

- (id)init {
    if (self = [super init]) {
      //分配一块内存作为生成器的运行栈
        _stack = malloc(DEFAULT_STACK_SIZE);
        memset(_stack, 0x00, DEFAULT_STACK_SIZE);
        _stack_size = DEFAULT_STACK_SIZE;
        //jmp_buf类型来自C标准库<setjmp.h>
        _ev_leave = malloc(sizeof(jmp_buf));
        memset(_ev_leave, 0x00, sizeof(jmp_buf));
        _ev_entry = malloc(sizeof(jmp_buf));
        memset(_ev_entry, 0x00, sizeof(jmp_buf));
    }
    return self;
}
```

实现next方法:
```Objective-C
#define JMP_CONTINUE 1//生成器还可被继续迭代
#define JMP_DONE 2//生成器已经执行结束，迭代器应该结束

- (Result *)next:(id)value {
    if (_done) {
       //迭代器已结束,则每次调用next都返回最后一次结果
       return [Result resultWithValue:_value error:_error done:_done];
    }    
    //保存next当前环境
    int leave_value = setjmp(_ev_leave);
    //非恢复执行
    if (leave_value == 0) {
        //已经设置了生成器进入点
        if (_ev_entry_valid) {
            //设置传给生成器内yield的新值
            if (value) {
                self.value = value;
            }
            //直接从生成器进入点进入
            longjmp(_ev_entry, JMP_CONTINUE);
        }
        else {
            //生成器还没保存过现场,从wrapper进入生成器
            
            //next栈会销毁,所以为wrapper启用新栈
            intptr_t sp = (intptr_t)(_stack + _stack_size);
            //预留安全空间，防止直接move [sp] 传参 以及msgsend向上访问堆栈
            sp -= 256;
            //对齐sp
            sp &= ~0x07;
            //直接修改栈指针sp,指向新栈
#if defined(__arm__)
            asm volatile("mov sp, %0" : : "r"(sp));
#elif defined(__arm64__)
            asm volatile("mov sp, %0" : : "r"(sp));
#elif defined(__i386__)
            asm volatile("movl %0, %%esp" : : "r"(sp));
#elif defined(__x86_64__)
            asm volatile("movq %0, %%rsp" : : "r"(sp));
#endif
            //在新栈上调用wrapper,至此可以认为wrapper,以及生成器函数的运行栈和next无关
            [self wrapper];
        }
    }
    //从生成器内部恢复next
    else if (leave_value == JMP_CONTINUE) {
        //还可以继续迭代
    }
    //从生成器wrapper恢复next
    else if (leave_value == JMP_DONE) {
        //生成器结束，迭代完成
        _done = YES;
    }
        
    return [RJResult resultWithValue:_value error:_error done:_done];
}
```


如果没有中介wrapper,那么迭代器返回将会造成崩溃，因为迭代器的运行栈和生成器是分开的，如果生成器内部执行return语句，返回后的栈空间将是未定义的，很有可能造成非法内存访问而崩溃.中介wrapper很好地解决了这个问题:

```Objective-C
- (void)wrapper {
   //调用生成器函数
    if (_func) {
        _func();
    }
    //从生成器返回，说明生成器完全执行结束
    self.value = nil;
    //恢复next
    longjmp(_ev_leave, JMP_DONE);
    //不会到此
    assert(0);
}
```

通过中介wrapper调用方式进入生成器，生成器最终返回后将正确返回到wrapper末尾继续执行，而wrapper也就知道，此时生成器结束了，因此以longjmp方式恢复next的现场，并设置恢复值为JMP_DONE,next被恢复后拿到这个值就知道生成器执行结束，迭代该结束了.



yield的实现就更加简单，保存当前现场，将value值传递给迭代器对象，然后恢复迭代器next方法即可,而当后续从next恢复yield的现场后，yield再取迭代器设置的新值返回给生成器内部，如此达到生成器与迭代器的数据交换:

```objective-c
id yield(id value) {
    //获取当前线程正在获得的生成器
    Iterator *iterator = [IteratorStack top];
    return [iterator yield: value];
}

- (id)yield:(id)value {
    //设置生成器的现场已保护标志
    _ev_entry_valid = YES;
    //现场保护
    if (setjmp(_ev_entry) == 0) {
        //现场保护完成
        //给迭代器赋值
        self.value = value;
        //恢复迭代器next现场
        longjmp(_ev_leave, JMP_CONTINUE);
    }
    //从迭代器next恢复此现场
    //返回迭代器传进来的新值,或者默认值value
    return self.value;
}
```

这里的IteratorStack是一个线程本地存储的栈，栈顶永远是当前线程正在活动的迭代器，具体实现可以参考后边给出的结果项目。



至此已经实现了c函数版本的生成器，简单改变即可扩展到OC方法，block.首先是迭代器需要支持新的初始化方法:

```objective-c
@interface Iterator : NSObject
{
    int *_ev_leave; //迭代器在next方法内保存的现场
    int *_ev_entry; //生成器通过yield保存的现场
    BOOL _ev_entry_valid; //指示生成器现场是否可用
    void *_stack; //为生成器新分配的栈
    int _stack_size; //为生成器新分配的栈大小
    void (*_func)(void);//迭代器函数指针
    BOOL _done; //是否迭代结束
    id _value; //生成器通过yield传回的值
    id _target;//生成器方法所在对象
    SEL _selector;//生成器方法selector
    id _block;//生成器block
    NSMutableArray *_args;//传递给生成器的初始参数
}
- (id)initWithFunc:(void (*)(void))func;
- (id)initWithTarget:(id)target selector:(SEL)selector;
- (id)initWithBlock:(id)block;
- (Result *)next;
- (Result *)next:(id)value;
@end
```

wrapper支持新的生成器调用方式:

```objective-c
- (void)wrapper {
    if (_func) {
        _func();
    }
    else if (_target && _selector) {
       ((void (*)(id, SEL))objc_msgSend)(_target, _selector);
    }
    else if (_block) {
        ((void (^)(void))_block)();
    }
    //从生成器返回，说明生成器完全执行结束
    self.value = nil;
    //恢复next
    longjmp(_ev_leave, JMP_DONE);
    //不会到此
    assert(0);
}
```



#### 3.通过生成器与迭代器改进异步编程

正如前面描述的JS下的改进方法，现在可以用实现的生成器与迭代器来改进iOS的异步编程，且思路一模一样.

首先定义异步操作为如下闭包:

```objective-c
typedef void (^AsyncCallback)(id  value, id  error);
typedef void (^AsyncClosure)(AsyncCallback  callback);
```

跟JS下的定义一样，这种闭包内部可进行任何异步调用，最终以callback输出error和value即可.

同时PromiseKit提供的AnyPromise也可以作为异步操作.

iOS 版本readFile：

```objective-c
- (AsyncClosure)readFileWithPath:(NSString *)path {
    return  ^(void (^resultCallback)(id value, id error)) {
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            NSData *data =  [NSData dataWithContentsOfFile:path];
            resultCallback(data, [NSError new]);
        });
    };
}


- (AnyPromise *)readFileWithPath:(NSString *)path {
    return [AnyPromise promiseWithAdapterBlock:^(PMKAdapter  _Nonnull adapter) {
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            NSData *data =  [NSData dataWithContentsOfFile:path];
            adapter(data, [NSError new]);
        });
    }];
}

```

执行器executor：

```objective-c
@protocol LikePromise <NSObject>
- (id<LikePromise> __nonnull (^ __nonnull)(id __nonnull))then;
- (id<LikePromise>  __nonnull(^ __nonnull)(id __nonnull))catch;
@end

void executor(dispatch_block_t block) {
    Iterator *  iterator = [[Iterator alloc] initWithBlock:block];
    Result * __block result = nil;
    
    dispatch_block_t __block step;
    step = ^{
        if (!result.done) {
            id value = result.value;
            //oc闭包
            if ([value isKindOfClass:NSClassFromString(@"__NSGlobalBlock__")] ||
                [value isKindOfClass:NSClassFromString(@"__NSStackBlock__")] ||
                [value isKindOfClass:NSClassFromString(@"__NSMallocBlock__")]
                ) {
                ((AsyncClosure)value)(^(id value, id error) {
                    dispatch_async(dispatch_get_main_queue(), ^{
                        [result release];
                        //将此次异步操作的结果包装成Result，传给生成器
                        result = [iterator next: [Result resultWithValue:value error:error done:NO]].retain;
                        step();
                    });
                });
            }
            //AnyPromise
            else if (NSClassFromString(@"AnyPromise") &&
                     [value isKindOfClass:NSClassFromString(@"AnyPromise")] &&
                     [value respondsToSelector:@selector(then)] &&
                     [value respondsToSelector:@selector(catch)]
                     ) {
                id <LikePromise> promise = (id <LikePromise>)value;
                void (^__block then_block)(id) = NULL;
                void (^__block catch_block)(id) = NULL;
                
                then_block = Block_copy(^(id value){
                    if (then_block) { Block_release(then_block); then_block = NULL; }
                    if (catch_block) { Block_release(catch_block); catch_block = NULL; }
                    
                    [result release];
                    result = [iterator next: [Result resultWithValue:value error:nil done:NO]].retain;
                    step();
                });
                
                catch_block = Block_copy(^(id error){
                    if (then_block) { Block_release(then_block); then_block = NULL; }
                    if (catch_block) { Block_release(catch_block); catch_block = NULL; }
                    
                    [result release];
                    result = [iterator next: [Result resultWithValue:nil error:error done:NO]].retain;
                    step();
                });
                
                promise.then(then_block).catch(catch_block);
            }
            //普通对象
            else {
                Result *old_result = result;
                result = [iterator next: old_result].retain;
                [old_result release];
                
                step();
            }
        }
        else {
            //执行过程结束
            Block_release(step);
            [result release];
            [iterator release];
        }
    };
    
    step =  Block_copy(step);
    
    dispatch_async(dispatch_get_main_queue(), ^{
        result = iterator.next.retain;
        step();
    });
}
```



有了执行器executor,那么顺去读取文件的例子在iOS下可如下实现:

```objective-c
executor(^{
    Result *result1 = yield( [self readFileWithPath:@".../path1"] );
    if (result1.error) {/*第1步出错*/}
    NSData *data1 = result1.value;
    
    Result *result2 = yield( [self readFileWithPath:@".../path2"] );
    f (result1.error) {/*第2步出错*/}
    NSData *data2 = result2.value;
    
    Result *result3 = yield( [self readFileWithPath:@".../path3"] );
    f (result1.error) {/*第3步出错*/}
    NSData *data3 = result3.value;
});
```



#### 4.更好听的名字: async,await

将上一步实现的的执行器executor改名为async, 新增await函数:

```objective-c
RJResult * await(id value);

RJResult * await(id value) {
    return (Result *)yield(value);
}
```

其实await本质上就是yield.那么读取文件的例子就写成:

```objective-c
async(^{
    Result *result1 = await( [self readFileWithPath:@".../path1"] );
    if (result1.error) {/*第1步出错*/}
    NSData *data1 = result1.value;
    
    Result *result2 = await( [self readFileWithPath:@".../path2"] );
    f (result1.error) {/*第2步出错*/}
    NSData *data2 = result2.value;
    
    Result *result3 = await( [self readFileWithPath:@".../path3"] );
    f (result1.error) {/*第3步出错*/}
    NSData *data3 = result3.value;
});
```

至此便在iOS平台实现了async,await,且通过async,await可以化异步编程为同步风格.单靠短短字面描述无法面面俱到,比如setjmp,longjmp的原理及使用，函数调用过程与栈的联系,如有生疏要额外研究. 本文旨在描述在iOS平台上的一次对async,await的实现历程，可以通过下面的项目查看完整实现代码.

### 成果项目

[**RJIterator**](https://github.com/renjinkui2719/RJIterator)是我根据本文描述的思路实现的迭代器,生成器，yield,async,await的完整项目，欢迎交流与探讨.
