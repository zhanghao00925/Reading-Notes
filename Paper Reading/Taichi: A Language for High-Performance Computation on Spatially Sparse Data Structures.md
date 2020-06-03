# Taichi: A Language for High-Performance Computation on Spatially Sparse Data Structures

# INTRODUCTION

三维可视化计算数据通常是空间稀疏的。为了利用这种稀疏性，人们开发了分层稀疏数据结构，如多层稀疏体素网格、粒子和三维哈希表。然而，由于其固有的复杂性和开销，开发和使用这些高性能的稀疏数据结构非常具有挑战性。我们提出了一种新的面向数据的编程语言Taichi，用于高效地创作、访问和维护这样的数据结构。该语言为编写计算代码提供了一个高级的、与数据结构无关的接口。用户独立地指定数据结构。我们提供了几个具有不同稀疏性属性的基本组件，这些组件可以任意组合以创建广泛的多层稀疏数据结构。这种数据结构与计算的解耦使得在不改变计算代码的情况下，可以很容易地试验不同的数据结构，并允许用户像处理密集数组一样编写计算。然后，我们的编译器使用数据结构和索引分析的语义自动优化局部性、删除冗余操作的一致性访问、保持稀疏性和内存分配，并为cpu和gpu生成高效的并行和向量化指令。

我们的计算-数据结构解耦允许我们快速试验不同的数据安排，并为特定的计算任务开发高性能的数据结构。与手工优化的参考实现相比，我们的代码行数是原来的1 / 10，平均性能提高了4.55倍。

为了实现高性能，库通常必须向用户公开底层接口，这导致了抽象的泄漏，使得计算代码与数据结构高度耦合。我们提出了一种新的编程模型，它将数据结构从计算中分离出来，以实现高性能和简单的编程(图3)。用户使用高级和数据结构无关的接口编写计算代码，就像在一个密集的多维数组上操作一样。内部数据排列和相关的稀疏性是通过组合基本组件(如密集数组和哈希表)来独立于计算代码指定的，从而形成一个层次结构。

我们的编译器针对指定的数据结构组件进行优化，并生成有效的稀疏性和内存维护代码。我们利用从数据布局和访问模式的高级信息中获得的索引分析，开发了几种特定于领域的策略来优化空间相关访问。我们的编译器分析访问，以有效地计算内存地址，使用缓存策略以获得更好的局部性，并并行化/向量化来自程序员的高级指令的循环。这是由我们紧凑的中间表示支持的，专门为优化分层稀疏数据结构而设计。我们的编译器从中间表示生成c++代码或CUDA代码，使得切换后端变得毫不费力。

我们的贡献总结如下:

一种将数据结构与计算*解耦*的编程语言(第3.1节)。我们提供了一个统一的抽象来将多维索引映射到内存地址。这样的抽象允许程序员独立于所涉及的数据结构的内部安排来定义计算。
一种数据结构描述的微型语言，它提供了几个基本的数据结构组件，这些组件可以组合成具有静态层次结构的广泛的稀疏数组(第3.2节)。
一个优化编译器,使用指数分析和信息数据结构自动优化的地方,减少冗余的操作一致的访问,管理稀疏,生成并行和矢量化后端代码x86_64和CUDA(秒。4秒。5)。
全面评价的体系,和最先进的实现的几个图形和视觉算法作为副产品。


#  GOALS AND DESIGN DECISIONS

The four high-level goals are as follows:

Expressiveness

Performance

Productivity

Portability

## Design Decisions

将数据结构与计算解耦。使用笛卡尔索引抽象数据结构访问来实现这一点，而实际的数据结构定义了从索引到实际内存地址的映射(第3.1节)。

通过层次结构描述数据结构。开发了一种数据结构微型语言来组成数据结构层次(第3.2节)。

Regular grids as building blocks.

固定的数据结构层次。我们假设层次结构在编译时是固定的，从而简化了编译器优化和内存分配。我们不支持具有动态深度的八叉树或包围卷层次结构。

带有稀疏迭代器的单程序多数据(SPMD)。

自动生成优化后端代码。我们的编译器应该自动生成高性能的后端代码，同时优化局部性(第4.1节)、使用访问一致性(第4.2节)最小化冗余访问、自动并行(第4.3节)和分配内存(第5.2节)。

# THE TAICHI PROGRAMMING LANGUAGE

We demonstrate our language using a 2D Laplace operator.

## Defining Computation

We adopt the Single-Program-Multiple-Data paradigm. Our language is similar to other SPMD languages such as ispc and CUDA, with three additional components: 1) parallel sparse For loops, 2) multi-dimensional sparse array accessors, and 3) compiler hints for optimizing program scheduling.

## Describing Internal Structures Hierarchically

# DOMAIN-SPECIFIC OPTIMIZATIONS

我们的编译器减少了三个典型来源的访问开销:

Out-of-cache access. 

Data structure hierarchy traversal.

Instance activation.

## Scratchpad Optimization through Boundary Inference

## Removing Redundant Accesses

## Automatic Parallelization and Task Management

# COMPILER AND RUNTIME IMPLEMENTATION

The Taichi programming language is embedded in C++14, providing easy interoperability with the host language. We plan to release a Python 3 embedding to further lower the language learning barrier and development cost. The compiler is implemented in C++17, borrowing infrastructure from the Taichi library [Hu 2018]

## Simplification

简化阶段还应用了最常见的通用编译器优化，比如公共子表达式消除、本地变量存储转发、死指令消除，以及将“if”语句降低到条件移动中。

## Memory Management

## Loop Vectorization on CPUs

## Interaction with the Host Language