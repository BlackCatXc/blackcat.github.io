
# std::sort 对包含 nan 的 vector<float> 排序引起的 core dump 原因分析

在开发过程中碰到一个因为std::sort产生的 core dump，在这里记录一下并分析产生的原因。

    总结：使用 sort 进行排序时，传入的比较函数一定要满足“严格弱序”(Strict Weak Ordering)，否则就可能会出core。严格弱序可以简单的理解为：对两个数 a 和 b 进行比较时，如果有 comp(a, b) 为 true，那么一定要有 comp(b, a) 为 false 。绝对不能出现 comp(a, b) 和 comp(b, a) 同时为 true 或者同时为 false 的情况。

## 概念：

1)  NaN: Not A Number，表示算数上无效的数，可能由于除 0 或者对负数进行开平方等计算产生。NaN有如下性质：
             
    |expression|result|
    |:---|---:|
    |nan < nan|false|
    |nan > nan|false|
    |nan == nan|false|
    |nan != nan|true|
    |nan < 1.0|false|
    |nan > 1.0|false|
    |nan != 1.0|true|
            
2)  std::sort: C++ STL标准库中常用的排序函数

调用方式：
```
 std::vector<float> vec;
 std::sort(vec.begin(), vec.end(), comp);
```


## 问题描述：

除去代码中的业务，出 core 的代码简化如下：

```
#include "iostream"
#include <vector>
#include <map>
#include <cmath>
#include <algorithm>
using namespace std;

struct sort_cmp {
    bool operator() (const float& i1, const float& i2) const {
        return i1 > i2;
    };
};

void func(vector<float>& v) {
    float coredata[] = {
        1.0,2.0,3.0,40.0,5.0,
        6.0,7.0,8.0,9.0,10.0,
        11.0,12.0,13.0,14.0,15.0,
        16.0,NAN,18.0,19.0,20.0,
        21.0,22.0,23.0,24.0,25.0,
        26.0,27.0,28.0,29.0,30.0,
        31.0,32.0,33.0
    };
    
    int corelen = sizeof(coredata) / sizeof(float);
    for (int i = 0; i < corelen; i++) {
        v.push_back(coredata[i]);
    }
    std::sort(v.begin(), v.end(), sort_cmp());
}

int main() {
    vector<float> v1;
    func(v1);
}
```

程序报错 *** glibc detected *** double free or corruption (out): 0x0000000000****** *** 并出 core。

core文件的backtrace如下：

```
(gdb) bt
#0  0x0000003f0b02e2ed in raise () from /lib64/tls/libc.so.6
#1  0x0000003f0b02fa3e in abort () from /lib64/tls/libc.so.6
#2  0x0000003f0b062d41 in __libc_message () from /lib64/tls/libc.so.6
#3  0x0000003f0b06881e in _int_free () from /lib64/tls/libc.so.6
#4  0x0000003f0b068b66 in free () from /lib64/tls/libc.so.6
#5  0x0000003f0bfae19e in operator delete(void*) () from /usr/lib64/libstdc++.so.6
#6  0x0000000000401d97 in __gnu_cxx::new_allocator<float>::deallocate (this=0x7fff285abbf0, __p=0x504160) at /usr/lib/gcc/x86_64-redhat-linux/3.4.5/../../../../include/c++/3.4.5/ext/new_allocator.h:86
#7  0x0000000000401830 in std::_Vector_base<float, std::allocator<float> >::_M_deallocate (this=0x7fff285abbf0, __p=0x504160, __n=64) at /usr/lib/gcc/x86_64-redhat-linux/3.4.5/../../../../include/c++/3.4.5/bits/stl_vector.h:117
#8  0x000000000040169a in std::_Vector_base<float, std::allocator<float> >::~_Vector_base (this=0x7fff285abbf0, __in_chrg=<optimized out>) at /usr/lib/gcc/x86_64-redhat-linux/3.4.5/../../../../include/c++/3.4.5/bits/stl_vector.h:106
#9  0x000000000040102a in std::vector<float, std::allocator<float> >::~vector (this=0x7fff285abbf0, __in_chrg=<optimized out>) at /usr/lib/gcc/x86_64-redhat-linux/3.4.5/../../../../include/c++/3.4.5/bits/stl_vector.h:256
#10 0x0000000000400da6 in ***** () at *.cpp:**
```
 
从 core 中可以看到是在 vector 的析构函数中执行 delete 释放 vector 堆上空间的时候出的 core。

## 问题原因及解决方法
 
由于待排序的 vector<float> 中包含了 Nan，而由Nan的性质可知，对 Nan 进行比较时不能满足 sort 函数对于比较函数“严格弱序”的要求，导致 sort 出现了未定义行为。

简单的在比较函数 comp() 中添加对 Nan 的处理不能完全解决问题，需要在排序之前先对 vector 中的 Nan 进行过滤，例如：

```
for (int i = 0; i < corelen; i++) {
   if (! isnan(coredata[i])) {    
       v.push_back(coredata[i]);
   }
}
```

## sort出core的原因分析
 
那么究竟为什么在这个 case 下 sort 会出 core 呢？本节对这个问题进行进一步的分析。**（下面的分析以降序排列为例，即比较函数的排序依据是 `return a>b;` ）**
 
#### (1) std::sort的执行过程
 
std::sort基于快速排序进行优化，对快速排序进行了几点改进。

* 如果调用std::sort时定义了比较函数comp()，所有的大小比较都将基于comp，comp 返回 false 表示前面的元素不应该在后面的元素前面，应该对这两个元素做调换。

* sort 设定了一个 \_S\_threshold = 16, 首先会对数据集合进行跟快速排序同样的递归 partition 过程，当递归到一定深度，数据集合中数据的个数小于 \_S\_threshold 时，sort 会对数据集合做插入排序，而不是做快速排序。（当递归超过一定层数之后，sort 还会做堆排序，不过与本次的 case 无关，暂不讨论）

* 为了提高效率，sort 对划分过程实现了未检查边界的版本 unguarded\_partition()；对插入排序过程实现了两个版本：带边界检查的版本insertion\_sort() 和不带边界检查的版本 unguarded\_linear_insert。在确定要执行插入排序的元素前面有哨兵的前提下，会调用不检查边界的版本。<span style="color:red">**这两个不带边界检查的函数是导致core的原因。**</span>
    
    sort的代码简化如下：

    ```
    //std::sort
    void sort(first, last, comp) {
        if (first != last) {
            //先做快排
            introsort_loop(first, last, comp);
            //再对_S_threshold范围内的元素做插入排序
            final_insertion_sort(first, last, comp);
        }
    }
   
    //快排
    void introsort_loop(first, last, comp) {
        //根据数据规模决定是否快排
        while (last - first > _S_threshold) {
            //省略了确定pivot的部分
            cut = unguarded_partition(first, last, pivot, comp);
            //递归划分
            introsort_loop(cut, last, comp);
            last = cut;
        }
    }
    
    //插入排序
    void final_insertion_sort(first, last, comp) {
        if (last - first > _S_threshold) {
            insertion_sort(first, first + _S_threshold, comp);
            //pivot作为哨兵
            unguarded_insertion_sort(first + _S_threshold, last, comp);
        }
        else
            insertion_sort(first, last, comp);
    }
    
    //带边界检查的插入排序
    void insertion_sort(first, last, comp) {
        if (first == last)
            return;
        for (i = first + 1; i != last; ++i) {
            val = *i;
            if (comp(val, *first)) {
                //带边界检查，直接跟第一个元素比，如果比第一个元素还大，
                //直接插入到第一个元素的位置，其余元素往后移动
                copy_backward(first, i, i + 1);
                *first = val;
            }
            else
                //无边界检查，first作为哨兵
                unguarded_linear_insert(i, val, comp);
        }
    }
    ```
    
#### (2) unguarded\_partition()分析

unguarded\_partition() 函数的简化代码如下：
    
```
unguarded_partition(first, last, pivot, comp) {
    while (true) {
        while (comp(*first, pivot))
            ++first
        --last;
        while (comp(pivot, *last))
            --last;
        if (!(first < last))
            return first;
        iter_swap(first, last);
        ++first;
    }
}
```
    
划分函数将 first 指针指向的元素、last指针指向的元素分别与 pivot 进行比较，如果比较函数返回 false，说明这个元素不应该 pivot 元素的这一边，需要与另一边同样不满足条件的一个元素对换，实现划分完成之后左边的元素都比 pivot 大，右边的元素都比 pivot 元素小。

但是unguarded\_partition() 函数存在两个问题：

* ++first 和 --last 没有做边界检查，<span style="color:red">如果用户实现的 comp() 函数对于相同的元素返回true，极端的情况下，数组里所有元素都相同，while 循环条件一直满足，first 指针和 last 指针将会越过边界，写坏数组之外的数据，造成溢出。</span>

* <span style="color:red">如果 pivot 选择的恰好是 nan ，那么根据上面测试出来的真值表，两个 while 循环中的 comp() 返回值永远都是 false ，执行的结果是整个数组中的元素镜像调换了一下顺序。***此时并不能满足划分完成之后左边的元素都比 pivot 大，右边的元素都比 pivot 元素小***</span>
    
#### (3) unguarded\_linear\_insert() 分析
unguarded\_linear\_insert() 函数的简化代码如下：

```
//last 为要插入的元素指针，val 为要插入的元素的值
//要插入的元素前面为已经从大到小排好序的序列
void unguarded_linear_insert(last, val, comp) {
    next = last;
    --next;
    //找到第一个比 val 大的数（comp返回false），由于假设有哨兵的存在，循环总是能够停止
    //如果 *next 指向的值是 nan，comp() 返回 false， 循环也会终止，所以 nan 也可以起到哨兵的作用
    while (comp(val, *next)) {
        //如果比val小，依次往后移动
        *last = *next;
        last = next;
        --next;
    }
    //把last插入到找到的最大元素后面的位置
    *last = val;
}
```
 
从 (1) 中sort的代码可知，unguarded\_linear\_insert()会在两种情况下被调用：
    
* 第一种是确定要插入的元素比序列中第一个元素小，由第一个元素起到哨兵的作用

* 第二种是对 pivot 右边的序列进行插入排序，由于正常情况下 pivot 总是比右边的元素大，pivot 也可以作为哨兵终止 while 循环
<span style="color:red">但是由 (2) 可知，如果 pivot 被选中为 nan ，那么做完 unguarded\_partition() 划分之后，序列不满足 pivot 元素比右边的元素大。如果序列的最大值出现在了 pivot 右侧，而本可以作为哨兵的 pivot nan 又被移动到了这个值的右边（在(4)中的例子就是这种情况），那么在插入这个值的时候，while 循环将不能正确停止，而是会一直往前找，超出数组的范围，直到找到一个比他大的值，此时就发生了溢出。</span>
 
 
####(4) 举个例子
 

为了简化过程，将 \_S\_threshold 设置为4，也就是长度对于大于等于4的序列将执行快速排序，对于长度小于4的序列，将执行插入排序。
原始序列为：
 
|5.0|8.0|7.0|nan|3.0|2.0|4.0| 
|:---|:---:|:---:|:---:|:---:|:---:|---:|
 
第一轮排序，对序列 [0 - 6] 进行排序，长度 7 > 4，进行快速排序，选择中间值 nan 作为 pivot，由(2)中的结论，序列将会进行左右镜像调换，partition 完成之后的结果为

|4.0|2.0|3.0|nan|7.0|8.0|5.0| 
|:---|:---:|:---:|:---:|:---:|:---:|---:|

第二轮排序，对序列 [0 - 2] 递归进行排序，长度 3 < 4, 进行插入排序，三个值都是正常数值，排序正常进行，完成之后的结果为

|4.0|3.0|2.0|nan|7.0|8.0|5.0| 
|:---|:---:|:---:|:---:|:---:|:---:|---:|

第三轮排序，对序列 [3 - 6] 递归进行排序， 长度 4 == 4， 进行快速排序，选择 7.0 作为 pivot， partition完成之后的结果为（注意此时 nan 被移动走了）

|4.0|3.0|2.0|8.0|7.0|nan|5.0| 
|:---|:---:|:---:|:---:|:---:|:---:|---:|

第四轮排序，对序列 [3 - 3] 进行插入排序，此时进行的是不带边界检查的 unguarded\_linear\_insert()， 待插入的元素为 8.0， 从 8.0 往前寻找第一个比 8.0 大的数，此时前面已经没有哨兵了，while 循环会超出数组的边界一直找下去，覆盖了数组前面内存空间的值。
至此分析完成在 float 数组存在 nan 元素时，sort 是如何发生溢出的。
 

## 为什么core中是delete的时候程序崩溃

在这个 case 中，被排序的是 vector 中的元素，传入 sort 的参数是 vector 的 begin() 和 end() 迭代器，分别对应 vector 内部的 \_p\_start 和 \_p\_finish 指针，也就是 vector 在堆上申请的内存空间。

本节简要介绍堆内存的结构，以解释在这个 case 中，sort 写坏了什么东西，导致了 core

Linux 目前使用 glibc 中的 prmalloc2 对堆内存进行管理，glibc 将堆上空闲的内存分为很多大小不同的块，这些块被称为 chunk ，每一个 chunk 包含两个部分，第一个部分是用户内存空间，也就是 new 或者 malloc 返回给用户操作的内存空间；第二个部分是在 chunk 头上保存的控制信息。glibc 使用链表将这些空闲 chunk 连接在一起，chunk 头部的控制信息里包括 fd 指针（上一个 chunk 的地址），bk 指针（下一个 chunk 的地址），prev_size（上一个 chunk 的大小），size（当前 chunk 的大小）。

未被分配的 chunk 结构如下图所示：

```
    // 空闲 chunk 被存放在双向环链表

    chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |             Size of previous chunk                            |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    `head:' |             Size of chunk, in bytes                         |P|
      mem-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |             Forward pointer to next chunk in list             |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |             Back pointer to previous chunk in list            |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |             Unused space (may be 0 bytes long)                .
      .                                                               .
      .                                                               |
nextchunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+  
    `foot:' |             Size of chunk, in bytes                           |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

术语解释：'chunk' 就是整个 chunk 开头, 'mem' 就是用户数据的开始, 'Nextchunk' 就是下一个 chunk 的开头

glibc根据 chunk 的大小将所有 chunk 分为几类，分别链接成几个独立的链表，这些链表成为 bin。当用户使用 new 或者 malloc 申请内存的时候，glibc 根据用户申请空间的大小从合适的 bin 里的链表上找到一块空闲的 chunk 返回给用户。当 chunk 被分配给用户之后，chunk 的结构变成了如下图所示：

```
    // 正在使用的 chunk 布局

    chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |             Size of previous chunk, if allocated            | |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |             Size of chunk, in bytes                       |M|P|
      mem-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |             User data starts here...                          .
      .                                                               .
      .             (malloc_usable_size() bytes)                      .
      .                                                               |
nextchunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+  
      |             Size of chunk                                     |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
```

注意，被分配出去的 chunk 没有指向前后 chunk 的指针了。

当用户使用完这块内存，进行 delete 或者 free 操作时，glibc 需要干一件额外的事情， 它会检查与当前 chunk 相邻的上一个 chunk 和下一个 chunk 是否是未被分配的状态，如果未被分配，glibc 会将这两个空闲的 chunk 连接起来形成一个更大的 chunk，并重新添加到合适的 bin 中，以此防止堆内存的碎片化。这个过程需要用到 chunk 头上的 prev\_size 和 size 两个数据，从当前 chunk 的起始地址减去 prev\_size 就是上一个 chunk 所处的位置，从当前的 chunk 的起始位置加上 size 就是下一个 chunk 的位置。如果 prev_size 和 size 的值是错的，glibc 找到的上一个 chunk 就是错的，无法通过 glibc 的安全检查，glibc无法继续进行堆内存管理，报错 "glibc double free detected", 并且 dump core。

所以在这个 case 中，begin 迭代器指向的是上图中的 mem 地址，sort 函数是覆盖了 begin 迭代器前面的数据，写坏了 prev_size 和 size 两个值，是导致 core 和 该错误提示的原因。

    P.S. 除了 case 中的意外溢出写坏 chunk 信息的情况，glibc 的安全检查主要是针对堆溢出攻击做的防御，报错信息中的 double free 也是堆溢出的一种攻击方法，而不一定真是程序里进行了两次 free 操作。想要了解更多 glibc malloc 和堆管理的信息，可以参考[http://www.csyssec.org/20170104/glibcmalloc/]() 和 [http://paper.seebug.org/255/]()


## 调试技巧

在本次 core 排查中帮助最大的工具是 cgdb 和 gdb 的 watch 添加硬件断点功能。

调试时在 vector begin 前面的内存地址和 end 后面的内存地址上分别添加硬件断点，发生写入时程序就会中断，就明确了溢出是发生在什么时间点。
