---
title: "聊聊Pied Pier的压缩算法"
date: 2016-06-18
type: "posts"
draft: false
---

最近终于追完了HBO的自制喜剧《硅谷》的第三季。《硅谷》算是一部非常小众的美剧了，主要讲述湾区几个IT男创业的故事，剧情并没有过多围绕他们怎么写代码，而是把关注点聚焦在创业想法的诞生以及初期公司的成立以及与风投斡旋的过程中的戏剧冲突上，让"内行人"啼笑皆非。每季的后几集都有点儿燃烧，原本“改变世界”之类现实中会被嘲讽的话，却是最能触动内心的！

《硅谷》之所以与众不同，还因为剧中的很多理论都是很值得推敲的。我们今天就来聊一聊《硅谷》S2E08中提出的Pied Pier基础核心算法的“middle-out”数据压缩算法。为了便于理解，我们先来了解一下数据压缩算法的基本原理、“信息熵”以及霍夫曼编码。

## 数据压缩的原理

数据压缩原理很简单，概括起来就是找到那些重复出现的数据，然后用用更短的符号替代，从而达到缩短数据大小的目的。

例如，我有一段文本"ABCDABCDABCDABCDABCDABCD"，显然我们使用"6ABCD"也能替代原来的数据，因为可以根据"7ABCD"推算出原文本"ABCDABCDABCDABCDABCDABCD"，数据从原来的“28”byte变成了“5”byte，数据压缩比为“5/24”约等于“20.8”。事实上，只要保证对应关系，可以用任意字符代替那些重复出现的字符串。这让我想到了现在移动互联网时代广泛使用的[Emoji](https://en.wikipedia.org/wiki/Emoji)，我们可以使用一个简单的Emoji表情来表达原来需要多个字表达的意思。

本质上，所谓"压缩"就是找出文件数据内容的概率分布，将那些出现概率高的部分代替成更短的形式。所以，内容越是重复的文件，就可以压缩地越小。比如，"ABCDABCDABCDABCDABCDABCD"可以压缩成"6ABCD"。与之对应地，如果数据的内容毫无重复，就很难压缩。极端情况就是，遇到那些均匀分布的随机字符串，往往连一个字符都压缩不了。比如，任意排列的10个阿拉伯数字（5271839406），就是无法压缩的；再比如，无理数（比如`π`）也很难压缩。

总结一下，压缩就是消除冗余的过程，用更精简的形式表达相同的复杂内容。可以想象，压缩过一次以后，文件中的重复字符串将大幅减少。好的压缩算法，可以将冗余降到最低，以至于再也没有办法进一步压缩。所以，压缩已经压缩过的文件（递归压缩），通常是没有意义的。

## 数据压缩的极限

我们可以从数学上用反证法证明数据压缩是有极限的，也就是不可能无限压缩一份数据而保证内容不丢失。

假定任何文件都可以压缩到`N`个二进制位以内，那么最多有`2N`种不同的压缩结果。这就是说，如果有`2N+1`个文件，必然至少有两个文件会产生同样的压缩结果。这就意味着这两个文件不可能无损地还原。因此，得到证明，并非所有文件都可以压缩到`N`个二进制位以下。

N是一个基于压缩的数据确定的数字，我们很自然地想知道，这个N到底是多少？

按照我们前面的关于数据压缩的原理，我们知道数据压缩可以分解成两个步骤。

- 得到数据内容的概率分布，哪些部分出现的次数多，哪些部分出现的次数少
- 对数据内容进行编码，用较短的符号替代那些重复出现的部分

对于一封确定的数据文件来说，它的概率分布是确定的，不同的压缩算法主要是因为第二部编码方式的不同，最优的压缩算法，当然是最短的符号表示作为替代原数据内容。

我们使用数学归纳法来来演算一下`N`的值：

1. 最简单的情况，我们要压缩的数据只有一部分；这一部分只有两个值，那么一个二进制数就可以表示；这一部分只有三个值，那么就需要两个二进制数来表示；这一部分有n个不同的值，那么就需要"log<sub><sub>2</sub></sub>(n)"个二进制位来表示；
2. 假设在数据文件各个字符均匀出现的情况下，一个字符在某一部分中出现的概率是`p`，也就是说这一部分可能会出现`1/p`种不同的情况，那么，这一部分就需要至少"log<sub><sub>2</sub></sub>(1/p)"个二进制位来表示；
3. 推广开来，如果文件有`n`个部分组成，每个部分的内容在文件中的出现概率分别为`p1`、`p2`、...`pn`，那么替代符号占据的二进制最少为下面这个式子：

<blockquote>
<p>log<sub>2</sub>(1/p<sub>1</sub>) + log<sub>2</sub>(1/p<sub>2</sub>) + ... + log<sub>2</sub>(1/p<sub>n</sub>)</p>
<p>= ∑ log<sub>2</sub>(1/p<sub>n</sub>)</p>
</blockquote>

这就是数据压缩的极限。

### 信息熵

有了前面得到的数据压缩极限的公式，很容易知道，对于n相等的两个文件，概率p决定了这个式子的大小。p越大，表明文件内容越有规律，压缩后的体积就越小；p越小，表明文件内容越随机，压缩后的体积就越大。

我们将之前的数据压缩的极限公式除以数据的组成部分，既可以得到平均每个符号所占用的二进制位，这样我们就可以很方便的比较不同大小的文件的压缩极限：

<blockquote>
<p>∑ log<sub>2</sub>(1/p<sub>n</sub>)/n</p>
<p>= log<sub>2</sub>(1/p<sub>1</sub>)/n + log<sub>2</sub>(1/p<sub>2</sub>)/n + ... + log<sub>2</sub>(1/p<sub>n</sub>)/n</p>
</blockquote>

更进一步，我们可以得到每个字符所占用的二进制位的数学期望：

<blockquote>
<p>p<sub>1</sub>*log<sub>2</sub>(1/p<sub>1</sub>) + p<sub>2</sub>*log<sub>2</sub>(1/p<sub>2</sub>) + ... + p<sub>n</sub>*log<sub>2</sub>(1/p<sub>n</sub>)</p>
<p>= ∑ p<sub>n</sub>*log<sub>2</sub>(1/p<sub>n</sub>)</p>
<p>= E( log<sub>2</sub>(1/p) )</p>
</blockquote>

结果是每个字符所占用的二进制位的数学期望等于概率倒数的对数的数学期望。

为了便于理解，我们再举一个例子：

有一个文件包含`A`, `B`, `C`个三种不同的字符，内容`50%`是`A`，`30%`是`B`，`20%`是`C`，文件总共包含1024个字符，采用ASCII编码的情况，每个每个字符所占用的二进制位的数学期望为：

<blockquote>
<p>0.5*log<sub>2</sub>(1/0.5) + 0.3*log<sub>2</sub>(1/0.3) + 0.2*log<sub>2</sub>(1/0.2)</p>
<p>=1.49</p>
</blockquote>

这就是说该文件每个字符平均占用`1.49`个二进制位，那么压缩1024个符号，理论上最少需要`1526`个二进制位，约`0.186KB`，最终的压缩比接近于`18.6%`。

对于确定的一份数据文件，内容越是分散（随机），所需要的二进制位就越长。这个用来衡量文件内容的随机性（又称不确定性）的值叫做“信息熵”（information entropy）。它是1948年由美国数学家[Claude Shannon](https://en.wikipedia.org/wiki/Claude_Shannon)在经典论文《通信的数学理论》中提出的概念。

由上面介绍的内容我们知道；

- 信息熵只反映内容的随机性，与内容本身无关，熵是信息不确定性的度量
- 信息熵越大，表示占用的二进制位越长，因此就可以表达更多的符号

## 霍夫曼编码

信息熵理论编码定理早就把数据压缩的算法基本阐述地非常清晰了，所谓数据压缩，就是基于数据间的相关性，概率统计特性，尽量用短的码长表示源信息。最典型的变长不失真编码就是霍夫曼编码。

霍夫曼编码使用变长编码表对源符号进行编码，其中变长编码表是通过基于源符号出现概率而得到的，出现概率高的字符使用较短的编码，反之出现机率低的字符则使用较长的编码，这样便使编码之后的字符串的平均长度、期望值降低，从而达到无损压缩数据的目的。

霍夫曼编码基于霍夫曼树（又称最优二叉树），是一种带权路径长度最短的二叉树。编码的基本步骤是：

1. 将源数据中的字符按照出现的概率按从大到小的顺序排列
2. 反复地将当前排序中出现的最小的两个字符的概率相加，并且始终将较高的概率分支放在右边，直到最后变成概率1
3. 由概率1处(树根)到每个源数据中的字符的路径，顺序记下沿路径的0和1，所得就是该符号的霍夫曼编码
4. 将每对组合的左边一个指定为0，右边一个指定为1（或相反）

例如，现有一个由5个不同符号组成的30个符号的字符串"BABACACADADABBCBABEBEDDABEEEBB"进行霍夫曼编码：

1) 统一每个字符出现的次数（概率）

    | 字符 | 次数 |
    | ---- | --- |
    | B | 10 |
    | A | 8 |
    | C | 3 |
    | D | 4 |
    | E | 5 |

2) 把出现次数（概率）最小的两个字符相加，并作为左右子树，重复此过程，直到概率值为1

![](https://i.loli.net/2019/04/05/5ca71abe510e6.jpg)

3) 将霍夫曼树的左边分叉路径指定为0，右边路径指定为1

![](https://i.loli.net/2019/04/05/5ca71bb8d5537.jpg)

4) 沿霍夫曼树从树根到树叶路径组合，获得每个字符的编码

    | 字符 | 次数 | 编码 |
    | ---- | --- | --- |
    | B | 10 | 11 |
    | A | 8 | 10 |
    | C | 3 | 010 |
    | D | 4 | 011 |
    | E | 5 | 00 |

从上面的例子我们看到出现次数（概率）越多的字符路径越短，编码也越短，出现次数越少路径越长，编码也较长。在实际编码的时候，是按二进制为来编码和解码的，如果有这样的bitset: `10111101100`，那么根据最后的编码/解码的字典表解码后就是“ABBDE”。所以，通过霍夫曼树我们建立了编码和解码的字典表。

> Note：霍夫曼编码使得每一个字符的编码都与另一个字符编码的前一部分不同，不会出现像"A": `00` "B": `001`的情况，解码也不会出现冲突。

## Pied Pier压缩算法

现在，我们回到《硅谷》第一季中，年轻的计算机天才Richard发明了一种"middle-out"的超越理论极限的压缩算法，由此组建了Pied Piper公司。那么，真的又能超越理论极限的压缩算法吗？

答案是没有。既保证任何一个源数据中每个字符都有唯一对应的编码，还要尽量减小平均编码长度，此外还有考虑定长编码，限失真编码(忽略小概率事件，减小码长，减小码字空间)，但不管怎么样，最优的编码码长一定是受源数据统计特性影响的，也就是都有编码效率极限。一串毫无规律绝对均匀分布的字符（如`π`）几乎没有压缩的空间；实际上，一般的英语文章，图片，视频，音频，数据间有很明显的相关性，也有比较稳定的概率统计特性，对压缩质量的不同要求(文本和视频（音频）对压缩质量的要求就不一样，视频（音频）压缩后即使不能完全还原源信息也基本能明白，但文本却不能)，这才为压缩提供了可能。

实际上，为了更真实地反映硅谷创业场景，《硅谷》的编剧雇佣了一名有硅谷创业公司经验的技术顾问Jonathan Dotan，但Jonathan本人并不是压缩领域的专家，于是Dotan请教了斯坦福大学教授Tsachy Weissman。Weissman和一名斯坦福博士生Vinith Misra很快就想出了许多关于基因数据压缩算法、有损数据压缩、压缩去噪的点子，但是最后还是回到了压缩界中最神秘的概念——一种比现有压缩算法效率更高，压缩比更高的神奇无损压缩算法，这种压缩算法可以应用在任何类型的数据上，这种压缩算法就和圣杯一样神秘，大家都知道它，但就是没人见过真货。而且还创建了衡量压缩算法指标[Weissman Score](https://en.wikipedia.org/wiki/Weissman_score)。

我们来具体说说什么是"middle-out"压缩算法。

在之前的介绍中，我们知道霍夫曼编码使用树形结构编译模型数据来建立编码字典，是一种无损压缩编码方式，编码顺序是从上到下（树根到树叶）。实际上，还可以从树叶到树根来建立建立编码字典。很自然的，有人就提出了从中间开始想向树根和树叶遍历来建立编码字典，这就是"middle-out"压缩算法。事实上，"middle-out"压缩算法可以应用到特定的数据压缩领域，如视频，音频的压缩。它打破数据的相关性，之前的压缩算法，需要一个先前的值来计算当前的值，那么就需要知道所有以前的值。但是"middle-out"压缩算法从块的多个位置开始处理，它只依赖于这个片段中的先行值。压缩一直是关于速度和压缩比之间的一种折衷，当然也要在压缩和解压之间找到平衡。

事实上，却是有人已经实现了"middle-out"压缩算法并且和目前已有的算法做了对比，效果还不错，详情请移步：https://medium.com/@vaclav.loffelmann/the-worlds-first-middle-out-compression-for-time-series-data-part-1-1e5ad5312757