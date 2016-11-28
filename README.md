# Tag-Suggestion
这部分实验模型代码是在[hed-dlg-truncated](https://github.com/julianser/hed-dlg-truncated)基础上进行修改的。不过他们的代码非常复杂，实现的功能也很多，我最终所用的部分其实很少。由于刚开始学习Theano时就接触到这份代码，当时也是一边参考theano官方文档一边学习这份代码的，最后索性自己的模型也就在这份代码的基础上进行搭建了。

========================================================

模型
--------------------------------------------------------
我所做的任务是社会媒体上的文本标签注释（简单的说就是给文本推荐标签），比如[知乎](https://www.zhihu.com/)，对于每个问题用户都会给出1~5个话题标签，而我希望通过构建一个模型能够自动根据用户的问题描述给出话题标签。知乎上大部分问题都会有问题的详细描述，我们把它视为是一个文本，由于是对文本进行建模，所以采用了层次化的深度神经网络（即先利用CNN模型从词学习得到句子的向量表示，再通过GRU模型从句子学习得到文本表示）。具体做法是，输入词向量利用CNN模型学习得到一个句子向量，然后再将句子向量输入到GRU中，将GRU的最后一个状态向量作为文本向量表示，最后接入一个sigmoid层（相当于构建多个sigmoid分类器），预测每个标签词的置信度。模型结构如下图所示，

![model](model.jpg?raw=true "model")

事实上，除了上述用CNN学习句子向量表示以外我们还尝试了用RNN（GRU）、char-rnn（可以避免分词带来的错误以及解决未登录词的问题）、Skip-Thought学习句子表示，以及NAACL2016年的一篇论文中的模型[Hierarchical Attention](http://www.aclweb.org/anthology/N16-1174)。

实验
--------------------------------------------------------
我们总共爬取了50W知乎数据，随机抽取了10万数据作为训练语料，5千作为测试语料，采用了两种评价指标PRF和R-Precision。目前我们评价的是模型预测生成的top-3的结果，以下是实验结果，

|   模型               | P      | R      | F      | R-Precision  |
| -------------------- |:------:|:------:|:------:|:------------:|
| CNN+GRU(this code)   | **0.2828** | **0.3247** | **0.2828** | **0.2998**  |
| GRU+GRU              | 0.2762 | 0.3173 | 0.2766 | 0.2957       |
| char-rnn+GRU         | 0.2763 | 0.3190 | 0.2772 | 0.2976       |
|Hierarchical Attention| 0.2732 | 0.3142 | 0.2732 | 0.2880       |


**实验结论：**
- 1. 实验结果中F并不高，这是该任务普标存在的，主要有以下两个原因：（数据方面）由于数据来源于网络，所有数据难免会有噪声；（评价方法方面）对于标签，可能存在多种合理的情况，既模型给出的推荐是合理但和原始数据的标签不一致，这也会PRF平价值很低
- 2. CNN+GRU模型的效果最好，这说明CNN在文本标签注释任务上能更好的学到句子表示，分析原因可能是CNN能够更好的捕获局部信息，而这些信息对于模型给出正确的语义标签是有帮助的
- 3. Char-rnn+GRU模型和GRU+GRU模型差不多，没有显著提高的原因可能是任务是标签注释，即对文本的语义理解粒度并不要求非常细，所以即使存在一些未登录词以及分词错误，对这个层次（或者说这个粒度）的语义理解影响不大。但Char-rnn的优势还是有的，比如在一个无法提供分词工具的环境下，基于词的RNN就没法做了。不过Char-rnn也有劣势，就是训练时间更长，因为循环次数更多了


**豆瓣数据集上的实验**

为了更好的比较模型的性能，我们与之前做标签推荐任务的模型进行比较，主要选了TAM(X Si, 2010)和WTM(Z Liu, 2011)这两个模型。[TAM和WTM](https://github.com/YeDeming/THUTag)两个模型的作者都是豆瓣语料数据上做的实验，所以为了和这两个模型进行比较，我也同样在豆瓣数据上进行了相应的实验。我们用49050个豆瓣语料作为训练数据，用12132规模的语料作为测试数据。Top-3实验结果如下

![result](result.jpg?raw=true "result")

由于我们的模型没有对标题进行建模，所以只在不包含标题的豆瓣数据进行了实验。从实验结果来看，基于深度神经网络的三个模型均优于WTM，并且远远优于TAM，证明了基于深度神经网络的模型能够更好的学习到文本语义信息。