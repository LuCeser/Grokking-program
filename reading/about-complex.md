什么是复杂性？是让你的系统变的难以理解、难以修改的东西

复杂性表现在：

1. **Change amplification**，也就是修改散落在各处
2. **Cognitive load**，可以理解为学习成本或者认知负担
3. **Unknown unknowns**，未知的未知，开发人员也不知道项目中有哪些是不了解的知识

如果一种方法没有降低复杂性并且你没有用错的话，果断放弃它

引起复杂度提升的原因往往是
1. 重复，意味着不必要
2. 耦合

以下做法可能会引起复杂度提升

1. Pass-through methods
2. Pass-through variables

一个耦合程度高的系统就是复杂性高的，因为每次修改都非常困难，所以**领域驱动设计**这种方法就是为了设计出高度内聚的模型。
良好的**软件抽象**能够帮助我们降低复杂度

链接：

- [A Philosophy of Software Design](https://book.douban.com/subject/30218046/)
