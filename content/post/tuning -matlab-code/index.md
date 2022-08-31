---
title: 'Tuning MATLAB Code'
subtitle: ''
summary: Tuning MATLAB Code
authors:
- admin
tags:
- Academic
categories:
- Teaching
date: "2019-05-31T00:00:00Z"
lastmod: "2019-05-31T00:00:00Z"
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Placement options: 1 = Full column width, 2 = Out-set, 3 = Screen-width
# Focal point options: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight
image:
placement: 2
caption: 'Image credit: [**Unsplash**](https://unsplash.com/photos/CpkOjOcXdUY)'
focal_point: ""
preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---
## Slides
[Slides](http://cloud.pbeta.me/s/6DygHBjdyFrb7LB)

写高效率的MATLAb code和写好英文文章的最核心，最重要的思路是完全一样的： 不要重复 (avoid redundancy). 做到了这一点，代码的效率会得到保障，也能让读者更好的理解自己英文文章的目的。

这一个简单思路之所以如此重要是因为大部分MATLAB code的bottleneck都是重复次数非常多的for loop， 而for loop中即使一个很小的计算，在几万次的重复下也会对整体程序的效率起到很大的影响。因此，写好MATLAB code的关键就是如何避免重复计算。基于这点思路，下面具体介绍三类如何避免重复计算的方法

## Move loop-invariant code out of the loop
如果一个计算在每次loop时不发生变化，那么这个计算就应该被调整到for loop之外。这个思路虽然简单，但是有的时候并不是特别容易发现重复的计算，特别是这个计算是在另外一个函数中进行时。 下面举例说明：

'''

    for loopIDX = 1:1000
        array =1:100;
        process(array,loopIDX);
    end
    function process(array, loopIDX)
        arrayLen = length(array);
        for arrayIDX =1:arrayLen
            doSomething(arrary(arrayIDX),loopIDX);
        end
    end

'''
这里的redundancy不是特别容易发现，arrayLen是在 process 函数里面计算的，但是这个变量不会因为loopIDX的改变而变化， 因此不需要重复计算。 优化后的代码如下

'''

    array =1:100;      
    arrayLen = length(array);
    for loopIDX = 1:1000
        process(array,arrayLen,loopIDX);
    end
    function process(array,arrayLen,loopIDX)
        for arrayIDX =1:arrayLen
            doSomething(arrary(arrayIDX),loopIDX);
        end
    end
'''

## Minimize Dereferencing

不仅是不必要的计算需要尽量减少，其他一切会消耗计算时间的代码都需要优化。这里面就包括了：overhead of calling function and referencing. 每次call function和指代矩阵元素，不仅是函数计算和矩阵运算会耗费时间，而且会有额外的overhead的成本。 下面举例说明

```

    a = [0,0];
    for idx =1:1e8
        a(2)=a(2)+idx;
    end


    a = [0,0];
    b = 0;
    for idx =1:1e8
        b = b +idx;
    end
    a(2) = b;
```

这几行代码只是对a(2)进行一个累计的加总，但是第一种方法由于每个loop里面都需要进行通过index找到a(2)具体值的计算，而第二种方法通过使用一个temporary variable b，避免了每次的索引运算，可以将运算时间从0.9秒缩短到0.5秒。


## Vectorization
作为一个矩阵运算语言，MATLAB进行矩阵计算通常比通过for loop进行计算要更有效率，因此使用矩阵语言代替for loop通常能够节约计算时间，这种方法被统称为vectorization。下面，我们举一个例子，来介绍vectorization的思路。

假设，我们有一个班级的学生成绩，以vector form储存。我们需要在这些学生中抽出4人组成一个小组,得到一个储存了每种可能的矩阵，每行对应抽中学生的成绩。 下面代码可以通过for loop来实现这一目的。

```

tic
data = rand(5,1);
for i =1:1e5
    c_mod = nchoosek(1:5,3);
    exp_cbnt = zeros(size(c_mod, 1), 3);
    exp_cbnt(:,1) = data(c_mod(:,1),1);
    for k = 2:3
        exp_cbnt(:,k) = data(c_mod(:,k),1);
    end
end
toc 

```

使用vectorization的方法如下。这里的思路是先计算最终需要得到的矩阵的row vector form (one dimension)，再通过reshape得到最终的目标矩阵。 使用vectorization的方法会比第一种for loop的方法节约大概百分之十的计算时间，loop次数越多节约的时间会越明显。
```

tic
for i =1:1e5
    c_mod = nchoosek(1:5,3);
    c_mod_length = size(c_mod, 1);
    c_mod_vec = reshape(c_mod,c_mod_length*3,1);
    exp_cbnt_vec  = data(c_mod_vec,1);
    exp_cbnt_b = reshape(exp_cbnt_vec,c_mod_length,3);
end
toc 

isequal(exp_cbnt,exp_cbnt_b)

```

Vectorization的方法还有很多，这里无法一一列举，但是重要的是当我们看到for loop时，应该立刻去想是不是有可能用矩阵的方法来表达,然后再测试对比两种方法所花的时间.Matlab已经能够对for loop进行非常高效的优化了，所以并不是vectorization一定会节约时间。