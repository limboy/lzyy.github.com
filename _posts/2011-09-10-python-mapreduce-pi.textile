---
layout: post
title: python的简单MapReduce实现：计算π
category: tech
---

MapReduce是Google提出的一个软件架构，一般用于大规模数据集的并行运算。核心概念就是"Map（映射）"和"Reduce（化简）"。

简单说来就是把一个任务分割成多个独立的子任务，子任务的分发由map实现，子任务计算结果的合并由reduce实现。

mapreduce的应用场景多是那种互不依赖，上下文无关的任务。所以类似Fibonacci数列这种对输入有依赖的就不适合使用mapreduce。

回到正题，要计算圆周率，我们先构建这么个模型

<img src="http://code.google.com/edu/parallel/img/inscribe.png" />

{% highlight python %}
# 外面的正方形面积
As = (2r)(2r) or 4r*r
# 里面的圆的面积
Ac = pi*r*r

pi = Ac / (r*r)
As = 4r*r
r*r = As / 4
pi = 4 * Ac / As
{% endhighlight %}

也就是说只要算出圆的面积与正方形面积的比，就可以求出圆周率。

可以通过以下步骤计算Ac / As：

1) 随机在正方形里生成许多点
2) 计算点在圆内与在正方形内的比例

测试的随机点越多，结果越精确

{% highlight python %}
#coding=utf-8
import random
import multiprocessing
from multiprocessing import Process

class MapReduce(object):
    
    def __init__(self, map_func, reduce_func, workers_num=None):
        self.map_func = map_func
        self.reduce_func = reduce_func
        self.workers_num = workers_num
        if not workers_num:
            workers_num = multiprocessing.cpu_count()*2
        self.pool = multiprocessing.Pool(workers_num)

    def __call__(self, inputs):
        map_result = self.pool.map(self.map_func, inputs)
        reduce_result = self.reduce_func(map_result)
        return reduce_result

def calculator(*args):
    print multiprocessing.current_process().name,' processing'
    points, circle_round = args[0]
    points_in_circle = 0
    for i in range(points):
	    # 这里其实只取了1/4圆
        x = random.random()*circle_round
        y = random.random()*circle_round
        if (x**2 + y**2) < circle_round**2:
            points_in_circle += 1
    return points_in_circle

def count_circle_points(points_list):
    return sum(points_list)

if __name__ == '__main__':
    # 半径
    CIRCLE_ROUND = 10
    # 总点数
    POINTS = 10000000
    # 总进程数
    WORKERS_NUM = 10

    map_reduce = MapReduce(calculator, count_circle_points, WORKERS_NUM)
    inputs = [(POINTS/WORKERS_NUM, CIRCLE_ROUND)] * WORKERS_NUM
    all_points_in_circle = map_reduce(inputs)
    ac_as = float(all_points_in_circle)/POINTS
    print 'pi approach to:%7f'%(4*ac_as)
{% endhighlight %}

这是比较简单的单机mapreduce，用多进程就可以实现。如果是多机运算的话，就麻烦多了，类似这张图：

<img src="http://code.google.com/edu/parallel/img/mrfigure.png" width='700px'/>

参考链接[2]有对这张图的解释

参考：
* [1] <a href="http://blog.doughellmann.com/2009/04/implementing-mapreduce-with.html">http://blog.doughellmann.com/2009/04/implementing-mapreduce-with.html</a>
* [2] <a href="http://code.google.com/edu/parallel/mapreduce-tutorial.html">http://code.google.com/edu/parallel/mapreduce-tutorial.html</a>
