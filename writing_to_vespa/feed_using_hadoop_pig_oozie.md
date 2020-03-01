# 開始從hadoop寫入vespa吧
## Hive or HBase table
假設有data在Hive或HBase，可將vespa-hadoop package加入專案，用法如下，且vespa-hadoop package是基於vespa-http-client package的輕量版
```xml
<dependency>
  <groupId>com.yahoo.vespa</groupId>
  <artifactId>vespa-hadoop</artifactId>
  <version>7.25.26</version> <!-- Find latest version at search.maven.org/search?q=g:com.yahoo.vespa%20a:vespa-hadoop -->
</dependency>
```
> vespa-hadoop是基於Apache HTTP client library實作而成，因Hadoop中也有引用，如果出現衝突訊息`NoSuchFieldErrors`，可嘗試加入`-Dmapreduce.job.user.classpath.first=true`

## Pig
[Apache Pig](https://pig.apache.org/)是大數據分析常用的高階語言，同時也是yahoo內部ETL(Extract-Transform-Load)的標準語言，以下會說明如何在pig中寫入vespa的基本用法，想了解更多的應用範例可[參考](https://docs.vespa.ai/documentation/tutorials/blog-recommendation.html)
### VespaStorage輕鬆寫入vespa

+ vespa-hadoop提供VespaStorage類，只要在pig中註冊即可馬上使用
```pig
REGISTER path/to/vespa-hadoop.jar;

DEFINE VespaStorage com.yahoo.vespa.hadoop.pig.VespaStorage(<options>);

A = LOAD '<path>' [USING <storage>] [AS <schema>];

-- apply any transformations

STORE A INTO '$ENDPOINT' USING VespaStorage();
```
+ 執行pig指令如下
```bash
pig -x local -f feed.pig
    -p ENDPOINT=endpoint-1,endpoint-2
    -Dvespa.feed.proxy.host=proxy-host
    -Dvespa.feed.proxy.port=proxy-port
    -Dvespa.feed.defaultport=8080
```
+ vespa-hadoop提供同時寫入多個endpoint的服務，只要使用逗號隔開`-p ENDPOINT=endpoint-1,endpoint-2`，endpoint指的是vespa位置，通常長的像`http://vespa-document-api-host:8080`

#### 寫入JSON

+ 如果寫入vespa的檔案已經整理成json檔了，寫入方式非常簡單
```pig
REGISTER vespa-hadoop.jar;

DEFINE VespaStorage com.yahoo.vespa.hadoop.pig.VespaStorage();

-- Load data - one column for json data
data = LOAD '<source>' AS (json:chararray);

-- Store into Vespa
STORE data INTO '$ENDPOINT' USING VespaStorage();
```

#### 寫入pig結構檔

> 假如pig 處理的data還沒轉成JSON，透過以下幾個步驟也可輕鬆完成寫入vespa。
+ 假設已知vespa文檔定義如下
```text
search music {
    document music {
        field album type string  { indexing: index }
        field artist type string { indexing: summary | index }
        field duration type int  { indexing: summary }
        field title type string  { indexing: summary | index }
        field year type int      { indexing: summary | attribute }
    }
}
```

+ pig 處理的原始檔如下
```text
Bad         Michael Jackson Bad     1987    247
Recovery    Eminem          So Bad  2010    240
```

+ 註冊VespaDocumentOperation輕鬆轉成JSON
```pig
REGISTER vespa-hadoop.jar;

-- Create valid Vespa put operations
DEFINE VespaPutOperation
       com.yahoo.vespa.hadoop.pig.VespaDocumentOperation(
            'operation=put',
            'docid=id:namespace:music::<artist>-<year>'
       );

DEFINE VespaStorage
       com.yahoo.vespa.hadoop.pig.VespaStorage();

-- Load data from any source - here we load using PigStorage
data = LOAD '<hdfs-path>' AS (album:chararray, artist:chararray, title:chararray, year:int, duration:int);

-- Transform tabular data to a Vespa document operation JSON format
data = FOREACH data GENERATE VespaPutOperation(*);

-- Store into Vespa
STORE data INTO '$ENDPOINT' USING VespaStorage();
```

> 透過`operation=put`定義寫入模式，`PUT`是預設值，其他還有`REMOVE、UPDATE`。 JSON Vespa types與Pig data types的對應表如下

>| Pig type     | JSON-Vespa type |
>| :------------ |:---------------|
|int	|number
|long	|number
|float	|number
|double	|number
|chararray	|string
|bytearray	|base64 encoded string
|boolean	|boolean
|datetime	|long - milliseconds since epoch
|biginteger	|number
|bigdecimal	|number
|tuple	|array
|bag	|array of arrays
|map	|JSON object

+ 大功告成！最後寫入vespa檔案長得像
```json
{
    "put": "id:namespace:music::Michael Jackson-1987"
    "fields": {
        "album": "Bad",
        "artist": "Michael Jackson",
        "duration": 247,
        "title": "Bad",
        "year": 1987
    }
}
```

+ 然而，使用者情境百百種，當VespaDocumentOperation不敷使用時，我們還有兩種選擇，一種是寫UDF(user defined function)，怎麼寫pig的UDF可以[看這邊](http://pig.apache.org/docs/r0.14.0/udf.html)，另一種是使用**VespaStorage**
```pig
REGISTER vespa-hadoop.jar;

-- Transform tabular data to a Vespa document operation JSON format
DEFINE VespaStorage
       com.yahoo.vespa.hadoop.pig.VespaStorage(
            'create-document-operation=true',
            'operation=put',
            'docid=id:namespace:music::<artist>-<year>'
       );

-- Load data from any source - here we load using PigStorage
data = LOAD '<source>' AS (<schema>);

-- transform and select fields

-- Store into Vespa
STORE data INTO '$ENDPOINT' USING VespaStorage();
```
> 補充一下VespaStorage與VespaDocumentOperation可定義的參數有哪些

>|parameter|default|description|
>|:--------|:------|:----------|
|create-document-operation	|false|只有VespaStorage|
|operation	|put|寫入vespa模式,還有`REMOVE、UPDATE`|
|docid		| |定義document id 模板|
|exclude-fields| |定義要排除寫入的fields|

### 評估寫入狀態
+ `VespaStorage`寫入vespa之前，並不會檢查schema，因此schema不吻合文檔設定時，就會寫入失敗
+ pig在執行結束後，會顯示寫入vespa的統計數量，長得像
```text
...
	Vespa Feed Counters
		Documents failed=0
		Documents ok=100000
		Documents sent=100000
		Documents skipped=1

...
Output(s):
Successfully stored 100001 records in: "<endpoint>"
...
```
> 表示總共有100001 文檔，100000成功，1筆被略過

### 評估寫入效能
> VespaStorage是基於[Vespa HTTP Client](https://docs.vespa.ai/documentation/vespa-http-client.html)的高效率package，但因為pig的MapReduce特性，以下有幾的小技巧能讓vespa寫入更加快速

+ 設定MapReduce split size
```text
SET mapreduce.input.fileinputformat.split.minsize 128*1024*1024
```
> 針對Yarn and Hadoop v2優化，檔案建議分割為128Mb

+ 設定pig平行化處理
```pig
data = LOAD '/projects/comms_psearch/psearch_suggest/hiveDB/ext_us_users_content_score/*' USING PigStorage('\u0001') AS (guid:chararray, query:chararray, to:double);
grouped_data = GROUP data by guid PARALLEL 100;
STORE grouped_data INTO '$ENDPOINT' USING VespaStorage();
```

+ 設定work job數量
```text
SET mapreduce.jobtracker.maxtasks.perjob 100
```

## 自己寫個jar檔吧
+ VespaStorage是個基於VespaOutputFormat的MapReduce job，我們也可以完全自己寫一個jar檔，將文檔從hadoop寫入vespa

```java
import java.io.IOException;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;

public class VespaFeed {

    public static class FeedMapper extends Mapper<LongWritable, Text, LongWritable, Text>& {
        public void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
            context.write(key, value);
        }
    }

    public static void main(String... args) throws Exception {

        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf, "vespafeed");
        job.setJarByClass(VespaFeed.class);
        job.setMapperClass(FeedMapper.class);
        job.setMapOutputValueClass(Text.class);

        // Set output format class
        job.setOutputFormatClass(VespaOutputFormat.class);

        FileInputFormat.setInputPaths(job, new Path(args[0]));
        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
}
```

+ hadoop 上執行jar檔
```text
hadoop jar my-jar.jar VespaFeed <hdfs-path>
    -Dvespa.feed.endpoint=endpoint
    -Dvespa.feed.defaultport=8080
    -Dvespa.feed.proxy.host=proxyhost
    -Dvespa.feed.proxy.port=proxyport
```

## Hadoop to Vespa相關設定
> Pig與MapReduce中，使用`-Dparam=value` 可以微調一些vespa寫入參數

| parameter     | default | description
| :------------ |:--------|:-------|
|vespa.feed.endpoint| |vespa端點|
|vespa.feed.defaultport| |vespa端點埠(e.g 8080)|
|vespa.feed.proxy.host| |vespa端點的proxy位置|
|vespa.feed.proxy.port| |vespa端點的proxy埠|
|vespa.feed.dryrun	|false|true表示不寫入vespa|
|vespa.feed.usecompression	|true|寫入時要壓縮文檔|
|vespa.feed.data.format	|json|只可寫入json|
|vespa.feed.progress.interval	|1000|每n個record產生log檔(包含sent/ok/failed/skipped)|
|vespa.feed.connections	|1|連線數量|
|vespa.feed.route	|default|使用Route於vespa端點|

