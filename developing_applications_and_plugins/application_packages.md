# 應用整合包
一個Vespa的應用整合包顧名思義就是**一群可以被部署且有結構的檔案群**。
內容有
- 設定檔
- 元件檔案
- 附屬檔案

總之所謂的整合包就是可以整包帶走的東西就對了。
設定檔要和程式碼一致，有差就會在編譯階段直接出問題，比較好debug，真的是超貼心。

應用整合包要啟動一個應用，至少要包含以下兩個設定檔：
```
application/services.xml
application/hosts.xml
```
想要有更進階的設定，請參考[應用整合包引用](https://docs.vespa.ai/documentation/reference/application-packages-reference.html)。

## 部署
因為Vespa非常強調分散式運算，代價就是部署必須被驗證。
搜尋定義的變更通常不需要重新啟動服務或重新建立索引，如果有，部署就會失敗。
部署會經過兩階段，簡稱兩階段部署：
1. 準備期
2. 啟動期

## 應用服務文檔範例
```
<?xml version="1.0" encoding="utf-8" ?>
<services version="1.0">

  <container id="default" version="1.0">
    <processing/>      <!-- Request-response processors go here. -->
    <search/>          <!-- Use to run the search middleware. Searchers go here. -->
    <docproc/>         <!-- Use to run the document processing middleware. DocumentProcessors go here. -->
    <nodes>            <!-- Nodes in the container cluster -->
      <node hostalias="node1"/>
      <node hostalias="node2"/>
      <node hostalias="node3"/>
    </nodes/>
  </container>

  <content id="my-content" version="1.0">
    <redundancy>1</redundancy>
    <documents>         <!-- Add document schemas here -->
      <document type="my-searchable-type" mode="index"/>
      <document type="my-other-type" mode="streaming"/>
    </documents>
    <nodes>             <!-- # nodes in the content cluster -->
      <node hostalias="node4"/>
      <node hostalias="node5"/>
      <node hostalias="node6"/>
    </nodes/>
  </content>

</services>
```

## 引用
1. [Application Packages](https://docs.vespa.ai/documentation/cloudconfig/application-packages.html)
