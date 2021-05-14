# Elasticsearch 文本分析

> 官方文档的阅读理解。Elasticsearch 

## Overview

文本分析分为标记化（Toeknization）和规范化（Normalization）。

标记化是指将全文打碎，分散成为 Token，大多数时候 Tokne 就表示单个单词。

标记化可以使搜索可以按照单个条件（Terms）匹配，但是此时的匹配仍然为字面意思的。

例如 jump 和 leap 是相同意思的，但此时却无法相互搜索，甚至 QUICK 和 quick 之间都无法互相搜索，规范化就是将同义词或者去除大小写的影响。



在 ES 中整个过程都被表示为分析器（ Analyzer），一个 Analyzer 就标识一个规则的集合，一起影响着整个的分析流程。

