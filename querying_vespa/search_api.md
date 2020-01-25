# 查詢API

以下簡要敘述Vespa當中使用的搜尋API，詳細的部分請參考[這裡](https://docs.vespa.ai/documentation/reference/search-api-reference.html)。

## HTTP
本篇章中說明的主體，是對Vespa的Request。
而通常進行所謂的查詢/搜尋，會使用GET-request。
服務的提供層，牽涉對象是容器叢集(Container Cluster)中的Search Container。
> 請不要混淆，一個合法的request只能向容器提出需求；而只有Search Container才能夠執行Search Node當中的對於Content Cluster實體文件的操作。

使用的語法如下：
```html
http://<host:port>/search/?param1=value1&param2=value2&...
```

有關Vespa的說明，所謂的Endpoint指涉的對象就是`<host:port>`，它是一個位址，也是一個容器。

## 指令請求(Queries)
Vespa可以使用兩種請求表達法：
1. YQL
2. Legacy simple query language

對應來說，參數表達唯一必要的參數就是YQL。
剩下有些常用的：
- ranking
- searchChain
- sources
- pos.II
- queryProfile
- tracelevel
