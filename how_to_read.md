对于react这样的巨型项目来说，不了解其设计思路，直接钻到海量代码里，最终的结果就是消磨兴趣，从入门到放弃。我个人觉得阅读大型项目源码的思路应该是首选用google搜索一下react源码解析。大体了解react的设计思路，然后再把源代码克隆下来，通过readme了解源码的组织形式，最后亲自debug。

本文就具体解释一下如何debug react的源码。
1. 首先把react源码克隆到本地。这里有一个小建议，那就是去阅读最新版的代码，因为react最新版本的代码组织结构更清晰，更易读。之前版本用的是gulp和grunt打包工具。仅项目的各种引用关系都理的让人头疼。源码结构可以先读一下官方文档的说明：https://reactjs.org/docs/codebase-overview.html
2. 利用create-react-app创建一个自己的项目
3. 把react源码和自己刚刚创建的项目关联起来。到react源码的目录下运行yarn build。这个命令会build源码到build文件夹下面，然后cd到react文件夹下面的build文件夹下。里面有node_modules文件夹，进入此文件夹。发现有react文件夹和react-dom文件夹。分别进入到这两个文件夹。分别运行yarn link。此时创建了两个快捷方式。react和react-dom。
4. cd到自己项目的目录下，运行yarn link react react-dom 。此时在你项目里就使用了react源码下的build的相关文件。如果你对react源码有修改，经过build之后，就能里面体现在你的项目里。你可以在react源码里打断点，甚至修改react源码。然后在项目里验证你的修改。

### 调试技巧
1. 利用浏览器的开发者工具，在适当的地方打断点，进行追踪
2. 全局搜索大法
3. 充分利用console.trace，打印出函数的调用栈，分析函数的调用关系