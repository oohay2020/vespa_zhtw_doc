# 嘗試寫入Vespa吧

在寫入之前，你知道嗎？
* Vespa 有分container 和 content cluster，知道寫入Vespa是寫入哪個cluster嗎?

>基本上是寫入content cluster，如果想複習一下請參考 [vespa_overview](../welcome/vespa_overview.md)

* 寫入Vespa的document需要被明確定義，知道在哪邊定義嗎？    

>通常是定義在search-definitions(sd)，忘記的話可以看這邊 [search-definitions](https://docs.vespa.ai/documentation/search-definitions.html)


**以下進入正題** -----------------------


寫入Vespa的document需要符合JSON格式，可參考[document-json-format](https://docs.vespa.ai/documentation/reference/document-json-format.html)說明，實際做法有以下4種選擇：

+ **使用REST API**

  包含get, put, remove, update, visit. 詳細用法可參考[document-api](https://docs.vespa.ai/documentation/document-api.html)。

+ **使用Vespa HTTP client**

  使用Vespa提供的Java jar檔將document寫入Vespa，可以選擇使用呼叫Java method或是command line。
   jar檔採用多執行緒的平行非同步連線，以達到高效能表現，當需要從Vespa cluster外部node寫入document時，建議採用此方法，
   詳細用法請參考[vespa-http-client](https://docs.vespa.ai/documentation/vespa-http-client.html)。

+ **使用Document API**

  當需要在不同Java components(例如`searcher`,`document processors`.)間進行document讀寫時，
   可以採用vespa 提供的[Java package](https://javadoc.io/doc/com.yahoo.vespa/documentapi) 來完成，利用此package，能夠在Vespa不同的溝通層進行document交換，
   詳細說明可參考[document-api-guide](https://docs.vespa.ai/documentation/document-api-guide.html)。

+ **vespa-feeder**

  Vesapa提供了不少 command-line tools，vespa-feeder便是其中一種，透過此tool，可使document寫入Vespa更有效率，
   更多tools 以及詳細用法可參考[vespa-cmdline-tools](https://docs.vespa.ai/documentation/reference/vespa-cmdline-tools.html#vespa-feeder)。

除了以上4種方式寫入Vespa外，寫入的document數量以及Vespa本身capacity也需要考量，更多指南請參閱[sizing-feeding](https://docs.vespa.ai/documentation/performance/sizing-feeding.html)


## Vespa 的 CRUD
+ [Put](https://docs.vespa.ai/documentation/reference/document-json-format.html#put)

  用於寫入Vespa的操作，document由name-value成對組成，在vespa中也可稱為fields。通常document type中會定義所屬的fields有哪些(所有的`field types`在[這邊](https://docs.vespa.ai/documentation/reference/search-definitions-reference.html#field))，而這些定義會寫在application的[search definition](https://docs.vespa.ai/documentation/search-definitions.html)。
> **使用Put操作時，相同document id會被覆寫，需要特別注意!!!**

+ [Remove](https://docs.vespa.ai/documentation/reference/document-json-format.html#remove)

  此操作可將不需要的document移除，移除後的document其實在系統內會保留一段時間，詳情可[參考這邊](https://docs.vespa.ai/documentation/elastic-vespa.html#data-retention-vs-size)

+ [Update](https://docs.vespa.ai/documentation/reference/document-json-format.html#update)

  更新操作不僅可更新完整的document，也可進行部分fields更新，稱為`partial update`。假如欲更新的document不存在，系統會回覆`no document was found`。

+ [Get](https://docs.vespa.ai/documentation/document-api.html#get)

  會根據最新的update [timestamps](./writing_to_vespa.md#timestamps)回傳document內容

## Feed block
當cluster的disk，memory達到上限時，就會開始寫入vespa失敗，此時可透過調整resource設定來優化vespa整體效能，
更多resource limits請參考[這邊](https://docs.vespa.ai/documentation/reference/services-content.html#resource-limits)

需要注意的是，document不只定義了fields，還有與之對應的attribute，attribute影響vespa的`sorting`,`grouping`與`ranking`效能，
其中attribute.multivaluelimit與attribute.enumstorelimit設定也會造成block feeding，可參考[設定方式](https://docs.vespa.ai/documentation/performance/attribute-memory-usage.html#multivalue-mapping-enum-store)

為了修正以上問題，在content cluster增加nodes或調整capacity，調整後，data將重新分配不會受到影響。

以下兩種指標可用於辨別是否發生feeding block

| content.proton.resource_usage.feeding_blocked      | disk or memory |
| :------------ |:---------------|
| **content.proton.documentdb.attribute.resource_usage.feeding_blocked**| **attribute enum store or multivalue**|

> 什麼是[multivalue](https://docs.vespa.ai/documentation/search-definitions.html#multivalue-fields)?

發生feeding block的訊息長得會像
```
Put operation rejected for document 'id:test:test::0': 'diskLimitReached: {
  action: \"add more content nodes\",
  reason: \"disk used (0.85) > disk limit (0.8)\",
  capacity: 100000000000,
  free: 85000000000,
  available: 85000000000,
  diskLimit: 0.8
}'
```

## Batch delete
以下提供3種方式刪除document
+ 重複使用`search`, `delete`操作 [參考這邊](https://docs.vespa.ai/documentation/document-api.html#delete)

   Pseudocode 長得像
   ```
    while True; do
       query and read document ids, if empty exit
       delete document ids using /document/v1
       wait a sec
   ```

+ 使用Vespa提供的Java jar檔 [參考這邊](https://docs.vespa.ai/documentation/vespa-http-client.html)

   利用jar檔，批次刪除documents，command line長得像
   ```sh
	$ java -jar $VESPA_HOME/lib/jars/vespa-http-client-jar-with-dependencies.jar --host document-api-host < deletes.json
   ```

   > json file長什麼樣？[找這邊](https://docs.vespa.ai/documentation/reference/document-json-format.html)

+ 使用documents selection("services-content.html#documents")

   使用時要特別小心，他會刪除不符合selection條件的其他documents，另外也可以將selection條件加在services.xml，像這樣
   ```xml
    <documents garbage-collection="true">
        <document type="mytype" selection="mytype.version &gt; 4" >
    </documents>
   ```
   
## Ordering

  這邊要注意的是，是使用 [document api](https://docs.vespa.ai/documentation/document-api-guide.html)時，
  每份document會分配到一個serialize id，當同樣的document操做多次時，也能夠按照操做指令的先後順序進行。
  
  > 假如put操作同一份document連續兩次，且第一次put操作失敗，那麼第二次put開始進入等待中;
  若是第一次put操作失敗又被送出一次，那麼操作順序就會改變

 
## Timestamps

  + 當使用`put, update and remove`這類操作時，[distributor](https://docs.vespa.ai/documentation/content/content-nodes.html#distributor)
會分配timestamp給該document
  + 每個[buckets](https://docs.vespa.ai/documentation/content/buckets.html)不會出現重複timestamp的document
  + 利用timestamp可以知道哪個操作是最新的
  + 使用[visit](https://docs.vespa.ai/documentation/content/visiting.html)操作時，可指定timeframe取回特定documents
  + timestamp單位使用microseconds，好處是可以降低與其他document衝突的機率
  + 假如documents從 cluster A搬移到cluster B, documents將有新的timestamp，即便是documents沒有修改過 
  
