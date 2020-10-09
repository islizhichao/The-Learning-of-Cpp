# C++并发实战

## 第一章

### 并发（Concurrency）与并行（Parallelism）

并发与并行有重叠的含义，它们都表示在有限的资源上同时运行多个任务，但是它们的侧重点不同：

- 并发：并发主要关注分离，以及响应度
- 并行：并行主要关注于利用现有资源提高处理数据的性能

### C++中的并发与多线程

#### 一个简单的C++并发程序

```cpp
#include<thread>
#include<iostream>
void hello()
{
	std::cout << "Hello Concurrent World\n";
}
int main() {
	std::thread t(hello);
	t.join(); //等待线程t执行完毕
	return 0;
}
```

其中`#include<thread>`是C++标准库中对多线程的支持，用于管理线程的函数和类都定义在这个头文件中。

在C++多线程中，每个线程都有一个初始函数，线程开始执行，这个初始函数便开始执行。

对于上述程序，其初始函数即为`main`函数，但是对每个线程都构造了一个`std::thread`对象，上述以`hello`作为线程对象的初始函数。

于是就有了两个线程，初始线程`main`和新线程`t`。

## 第二章 线程管理

本章将介绍：

- 开启一个线程，如果用不同的代码运行一个新线程
- 等待一个线程执行完毕与让其运行
- 唯一标识的线程

### 2.1 基本的线程管理

#### 2.1.1 启动一个线程

与大多数C++标准库一样，`std::thread`可用于任何可调类型，所以我们能够传递一个拥有函数调用操作符的类实例给`std::thread`构造函数。

```c++
class background_task
{
public:
	void operator()() const
    {
        do_something();
        do_something_else();
    }
};
background_task f;
std::thread my_thread(f);
```

#### 2.1.2 等待一个线程完成

如果我们想要去等待一个线程完成，我们可以在一个相关的线程实例上调用`join()`函数，但是`join()`会消除相关线程的所有资源，所以线程对象不再于目前已完成的线程相关。

#### 2.1.3 在一个有异常的情况下等待线程完成

在等待线程完成代码中，我们必须仔细挑选调用`join()`代码的位置，因为如果发生异常情况，`join()`极有可能被跳过，为了避免在异常情况下出现线程生命周期的问题，可以使用如下代码：

```c++
struct func;
void f()
{
    int some_local_state = 0;
    func my_func(some_local_state);
    std::thread t(my_func);
    try
    {
        do_something_in_current_thread();
    }
    catch(...){
        t.join();
        throw;
    }
    t.join();
}
```

但是使用try/catch块是冗长且繁琐的，我们可以使用RAII（资源获取即初始化技术）

```c++
class thread_guard
{
    std::thread &t;
public:
    explicit thread_guard(std::thread& t_):t(t_){}
    ~thread_guard()
    {
        if(t.joinable()){ //因为一个线程只能被join一次，所以判断是否能被joinable非常重要
            t.join();
        }
    }
    thread_guard(thread_guard const &) =delete;
    thread_guard& operator=(thread_guard const &)=delete;
};

struct func;
void f()
{
    int some_local_state = 0;
   	func my_func(some_local_state);
    std::thread t(m_func);
    thread_guard g(t);
    do_something_in_current_thread();
}
```



#### 2.1.4 线程后台运行

在``std::thread`对象上调用`detach()`让线程在后台运行，执行命令后，没有直接与其交互的方式，并且不可能获得一个引用它的线程对象，所以无法去被join。

考虑我们的Word文字处理程序，我们可一个一次开多个窗口，并编辑不同文档，每个窗口相互独立，无须等待其他窗口编辑完成，所以用一个后台线程打开一个窗口是一个不错的选择。

```c++
void edit_document(std::string const & filename)
{
    open_document_and_display_gui(filename);
    while(!done_editing())
    {
        user_command cmd = get_user_input();
        if(cmd.type == open_new_document)
        {
            std::string const new_name = get_filename_from_user();
            std::thread t(edit_document,new_name);
            t.detach();
        }
        else
        {
            process_user_input(cmd);
        }
    }
}
```



### 2.2 给线程函数传递参数

我们要牢记于心，线程中函数的参数是被拷贝进入来的，即使这个函数需要被传入引用参数，一样被拷贝。

```c++
void f(int i, std::string const& s);
std::thread t(f,3,"hello");
```

如上所示，及时函数f的第二个参数需要`std::string`类型，这个段string文字也是逐字的被传递为`char const*`并在新线程的内部转换为`std::string`。特别注意的是，当传递的是一个自动变量指针。

```c++
void f(int i, std::string const& s);
void oops(int some_param)
{
    char buffer[1024];
    sprintf(buffer,"%i",some_param);
    std::thread t(f,3,buffer);
    t.detach();
}
```

在上述例子中，一个指向buffer的本地变量被转入新线程中，但是有可能，buffer在新线程中转换为`std::string`时，`oops`函数已经退出了，但`buffer`是一个局部变量，这个时候就会出现未定义的行为。

一种解决方式时，把`buffer`转为`std::string`类型，然后传入线程构造函数中。

```c++
void f(int i, std::string const& s);
void oops(int some_param)
{
    char buffer[1024];
    sprintf(buffer,"%i",some_param);
    std::thread t(f,3,std::string(buffer));
    t.detach();
}
```



### 2.3 传递一个线程的所有权

假设你想写一个函数，这个函数创建一个后台运行的线程，然后将这个线程的所有权返回给调用函数而不是等待他完成。

在C++标准库中，例如std::ifstream和std::unique_ptr是可移动的，而不是可复制的，std::thread也是其中之一，这就意味着一个执行期的线程所有权能够在两个线程实例中移动。

```c++
void some_function();
void some_other_function();
//t1线程被创建
std::thread t1(some_function);
//t2被创建时，t1的所有权被转移到t2
std::thread t2 = std::move(t1);
t1 = std::thread(some_other_function); //临时对象移动所有权是自动而且隐含的
std::thread t3;
t3 = std::move(t2);
//此时t1已经关联运行some_other_function的线程
//所以移动会出现错误
t1 = std::move(t3);
```

std::thread支持的移动操作意味着，线程的所有权能够从一个函数中传递出来

```c++
std::thread f()
{
    void some_function();
    return std::thread(some_function);
}

std::thread g()
{
    void some_other_function)(int);
    std::thread t(some_other_function,42);
    return t;
}
```

同样，也支持所有权被传入

```c++
void f(std::thread t);
void g()
{
    void some_function();
    f(std::thread(some_function));
    std::thread t(some_function);
    f(std::move(t));
}
```

得益于std::thread的移动操作，我们可以重写thread_guard类。

```c++
class scoped_thread
{
    std::thread t;
public:
    explicit scoped_thread(std::thread t_): t(std::move(t_))
    {
        if(!t.joinable()
           throw std::logic_error("No thread");
    }
    ~scoped_thread()
    {
		t.join();
    }
    scoped_thread(scoped_thread const &) = delete;
    scoped_thread& operator=(scoped_thread const &)=delete;
};
struct func;
void f()
{
      int some_local_state;
      scoped_thread t{std::thread(func(some_local_state))}
      do_something_in_current_thread();
}
```

### 2.4 选择运行时线程数目

C++标准库有个函数std::thread::hardware_concurrency()，这个函数可以返回真实运行在程序中的并发线程数目。

```c++
template<typename Iterator,typename T>
struct accumulate_block
{
	void operator()(Iterator first, Iterator last, T& result) {
		result = std::accumulate(first, last, result);
	}
};

template<typename Iterator,typename T>
T parallel_accumulate(Iterator first, Iterator last, T init) {
	unsigned long const length = std::distance(first, last);
	if (!length)
		return init;
	unsigned long const min_per_thread = 25;
	unsigned long const max_threads = (length + min_per_thread - 1) / min_per_thread;
	unsigned long const hardware_threads = std::thread::hardware_concurrency();
	unsigned long const num_threads = std::min(hardware_threads != 0 ? hardware_threads : 2, max_threads);
	unsigned long const block_size = length / num_threads;
	std::vector<T> results(num_threads);
	std::vector<std::thread> threads(num_threads - 1);
	Iterator block_start = first;
	for (unsigned long i = 0; i < (num_threads - 1); ++i)
	{
		Iterator block_end = block_start;
		std::advance(block_end, block_size);
		threads[i] = std::thread(accumulate_block<Iterator, T>(), block_start, block_end, std::ref(results[i]));
		block_start = block_end;
	}
	accumulate_block<Iterator, T>()(block_start, last, results[num_threads - 1]);
	for (auto& entry : threads)
		entry.join();
	return std::accumulate(results.begin(), results.end(), init);
}
int main() {
	std::vector<int> vec;
	for (int i = 0; i < 10000; ++i) vec.push_back(i);
	int init = 0;
	std::cout << parallel_accumulate(vec.begin(), vec.end(), init);
	return 0;
}
```

### 2.5 标识线程

在C++标准库中，用`std::thread::id`作为线程标识，并且能通过两种方式得到：

- 线程对象通过调用get_id()成员函数，如果线程对象没有相关执行的线程，调用`get_id()`将会得到一个默认构造的`std::thread::id`对象。
- 当前线程可以通过`std::this_thread::get_id()`来获取相应线程id；

## 第三章 数据共享

本章将包含：

- 线程之间共享数据的问题
- 利用锁机制保护数据
- 其他保护共享数据的机制

### 3.1 线程之间共享数据的问题

所有共享数据所造成的问题都是由于并发对数据的修改造成的，如果只对共享的数据进行修改，就不会出现问题。

我们接下来以一个双端队列中删除节点为例，其中删除节点的步骤如下：

- 找到要被删除的节点N
- 更新链表中节点N之前的节点，使其指向N之后的节点
- 更新链表中节点N之后的节点，使其指向N之前的节点

在步骤2和3中，容易出现不一致情况，试想一个线程在读链表，而另一个线程在更新链表。对于这样会造成不一致问题的代码，我们将它称为**竞争条件。

#### 3.1.1 竞争条件

在并发中，竞争条件（竞争状态）就是由于两个以上线程执行操作顺序而造成问题。

#### 3.1.2 避免不确定性的竞争条件

