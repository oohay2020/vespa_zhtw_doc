# 組織聯邦化

Vespa的容器層允許多個資料來源組成共同一個搜尋服務體。
這些來源可以都來自同個應用或外部的服務。

> 這裡請不要問我search cluster代表的是container cluster還是content cluster...

好處有什麼呢：
- 可以強化內容(輔助文章的圖片?)
- 提供更符合邏輯的結果(以合理的替代來源取代無結果的網頁)
- 建立一個可以提供不同資料來源的前端應用(被拿來當跳板?)

如果我們想建立這種多元成"邦"方案的最主要任務包括：
1. 與不同的來源建立連結
2. 選擇會接收特定請求的資料來源
3. 重新導向請求以回傳需要資料
4. 建立粽會統合的結果

這個邦容會以搜尋鏈(search chain)的方式提供整合執行的指令。
多看幾遍容器簡介[2]和鏈元件說明[3]。
喔對了，如何在多種文件型態間搜尋[4]也會是你的知音。

## 設定提供者 (Provider)
一個提供者就是一個用來從資料來源產生資料的搜尋鏈。
所以，提供者必須包含一個連接資料來源搜尋器並且產生結果。
設定提供者的方法如下：
```xml
<search>
  <provider id="my-provider" >
    <searcher id="MyDataFetchingSearcher" bundle="my-bundle" />
  </provider>
</search>
```

## 設定來源
一個簡單的提供者可以用來產生很多種不同的結果。
而怎麼實作及呈現，要用關鍵字`sources`。
一個`source`是一個用來提供一個或多個提供者特定種類結果的搜尋鏈。
範例如下：
```xml
<search>
  <provider id="my-provider">
    <searcher id="MyDataFetchingSearcher" bundle="my-bundle" />
    <source id="web" />
    <source id="java-api">
      <searcher id="com.yahoo.example.JavaApiSearcher" />
    </source>
  </provider>
```

### 多個提供者卻只有一個來源的狀況

## 邦化
範例：
```xml
<search>
  <provider id="my-provider">
    <searcher id="MyDataFetchingSearcher" bundle="my-bundle" />
    <source id="web" />
    <source id="java-api">
      <searcher id="com.yahoo.example.JavaApiSearcher" />
    </source>
  </provider>
  <chain id="combined">
    <federation id="combinator">
      <source idref="web" />
      <source idref="java-api" />
    </federation>
  </chain>
```

## 參考
1. [com.yahoo.search.federation JavaDoc](https://javadoc.io/static/com.yahoo.vespa/container-search/7.162.26/com/yahoo/search/federation/vespa/package-summary.html)
> 不是我要吐槽，官方文件出來一個Error也是不簡單...
2. [容器簡介](https://docs.vespa.ai/documentation/jdisc/)
3. [鏈元件說明](https://docs.vespa.ai/documentation/chained-components.html)
4. [在多種文件型態間搜尋](https://docs.vespa.ai/documentation/search-definitions.html#searching-multiple-document-types)
