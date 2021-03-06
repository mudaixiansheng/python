<div class="BlogAnchor">
   <p>
   <b id="AnchorContentToggle" title="收起" style="cursor:pointer;">目录[+]</b>
   </p>
  <div class="AnchorContent" id="AnchorContent"> </div>
</div>

# python之理解垃圾回收机制

# 1、Garbage Collection(GC)

高级语言中如java、c#，都采用了垃圾回收机制。python采用的是计数为主，标记-清除和分代回收两种机制为辅的机制。

# 2、引用计数

python里每个变量都是对象，它们核心有一个结构体`PyObject`如下：

	 typedef struct_object {
	 	int ob_refcnt;
	 	struct_typeobject *ob_type;
	} PyObject;

`PyObject`是每个对象必有的内容，其中`ob_refcnt`是作为引用计数。当一个对象有一个新的引用时，它的`ob_refcnt`就会增加，当引用它的对象被删除时，它的`ob_refcnt`就会减少。

	#define Py_INCREF(op)   ((op)->ob_refcnt++) //增加计数
	#define Py_DECREF(op) \ //减少计数
	    if (--(op)->ob_refcnt != 0) \
	        ; \
	    else \
	        __Py_Dealloc((PyObject *)(op))

当引用计数为0时，该对象声明就结束了。

引用计数机制的缺点：

1、维护引用技术消耗资源：虽然引用计数必须在每次分配和释放内存的时候加入管理计数的动作，然而与其它主流的垃圾回收技术相比，引用计数有这一个最大的特点，即实用性。任何内存，一旦没有指向它的引用，就会被立即被回收。而其它的垃圾回收计数必须在某种条件下（如内存分配失败）才能进行无效内存回收。

引用计数机制的执行效率问题：引用计数机制所带来的维护引用计数的额外操作与python运行所进行的内存分配和释放，引用赋值的次数是成正比的。而这点与其它主流的垃圾回收机制如标记-清除、停止-复制，是缺点，因为这些技术所带来的额外才做基本只与待回收的内存数量有关。

2、循环引用

	list1 = []
	list2 = []
	list1.append(list2)
	list2.append(list1)

list1与list2相互引用，如果不存在其它对象对它们的引用，list1与list2的引用计数也仍然为1，所占用的内存永远无法被回收，这将是致命的。要解决这个问题，python引入的其它的垃圾回收机制来弥补引用计数的缺陷：标记-清除和分代回收两种技术。

# 3、标记-清除

标记-清除是为了解决循环引用产生的问题。可以包含其它对象引用的容器对象（比如list、set、dict、class、instance）都可能产生循环引用。

我们先认清一个事实，如果两个对象的引用计数都为1，都是仅仅存在他们之间的循环引用，那么这两个对象都是需要被回收的。也就是说，他们的引用计数虽然表现为非0，但实际有用的引用计数为0。我们就必须先将循环引用去掉，那么这两个对象的有效计数就出来了。

>假设有两个对象A、B，我们从A出发，因为他有一个B的引用，则将B的引用计数减1，然后顺着引用达到B，因为B有一个对A的引用，同样将A的引用减1，这样，就完成了循环引用对象间环摘除。

标记清除采用了更好的做法，它并不改动真实的引用计数，而是将集合中对象的引用计数复制一份副本，改动该对象的引用副本。对于副本做任何的改动都不会影响到对象生命周期的维护。

这个计数副本唯一的作用是寻找root object集合（该集合的对象是不能被回收的）。当成功寻找的root object集合后，首先将现在的内存链表一分为二，一条链表维护root object集合，称为root链表，而另一条链表维护剩下的对象，称为unreachable链表。之所以要剖成两个链表，基于以下考虑：现在的unreachable可能存在被root链表中的对象，直接或间接引用的对象，这些对象是不能被回收的，一旦标记的过程中，发现这样的对象吗，就将其从unreachable链表黄总移到了root链表中；当完成标记后，unreachable链表中剩下的所有对象就是名副其实的立即对象了，接下来的垃圾回收只需限制在unreachable链表中即可。





