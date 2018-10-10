依赖注入（Dependency Injection）是Angular的特点之一，也是它主要的卖点。对于Angular应用开发者来说，它就像能将各个模块粘合在一起的胶水。Angular其他的特性都是（半）独立的，而依赖注入能够整合所有的特性，真正把Angular变成一个框架，供开发者使用。

正因为依赖注入对Angular的重要性，我们更应该了解它是如何运作。不幸的是，它也是Angular中较难掌握的部分，这从网络社区问答中就可见一斑。导致理解依赖注入困难的原因有很多，可能是文档（解释不清晰），也可能是实际运用过程中的专业术语（让人困惑），还可能是因为依赖注入在事实上解决了连接应用各个模块的这个难点。

依赖注入也是Angular中较有争议的特性之一。质疑Angular的人，常常会以依赖注入作为攻击点。有些人批评Angular文档中对依赖注入的阐述不够清晰，而有些人对在JavaScript中应用依赖注入的做法表示怀疑。这些质疑虽然有一定的道理，但都没有对Angular注射器的原理有一个透彻的分析。希望这部分内容不仅能让你更好地开发应用，也能让质疑者在讨论之前对注入机制有更深入的了解。
