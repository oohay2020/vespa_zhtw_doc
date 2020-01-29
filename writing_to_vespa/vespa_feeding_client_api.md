# 客戶端資料輸入Vespa的API介紹
## 為什麼要用 vespa feeding client api ??
vespa提供`vespa-http-client-jar-with-dependencies.jar`，可透過[java command line](##java command line模式)或是[java code](##java code模式)實作，優點包含：
+ 使用HTTP 協定
+ 平行化寫入一個或多個Vespa clusters，且寫入順序不變
+ 可編程特性，且與vespa無相依性
+ 可以從不同的java處理平台寫入vespa，如Hadoop，Storm
+ 效能優於原生的HTTP  
+ 自動化重新連線
+ 無限制寫入文件數量
+ 控制寫入vespa的文件數量，避免vespa壞掉

> 如果想將文件寫入`container cluster`，記得加上 `document-api` 
```xml
    <?xml version="1.0" encoding="utf-8" ?>
    <services version="1.0">
    
         <container version="1.0" id="default">
            <document-api/>
         </container>

    </services>
``` 

## java command line模式
+ 安裝[vespa-http-client](https://jar-download.com/artifacts/com.yahoo.vespa/vespa-http-client)，安裝完成後可以在以下位置找到 
```bash
$VESPA_HOME/lib/jars/vespa-http-client-jar-with-dependencies.jar
```
+ 使用範例
```bash
$ java -jar $VESPA_HOME/lib/jars/vespa-http-client-jar-with-dependencies.jar --file file.json --endpoint http://document-api-host:8080
```
+ 也可以使用 stdin
```bash
$ java -jar $VESPA_HOME/lib/jars/vespa-http-client-jar-with-dependencies.jar --host document-api-host < file.json
```
> --help 可以查到更多command
+ 使用TLS/HTTPS 憑證
```bash
$ java -jar $VESPA_HOME/lib/jars/vespa-http-client-jar-with-dependencies.jar --useTls --certificate /path/to/cert.pem --privateKey /path/to/private-key.pem --caCertificates /path/to/ca-certs.pem ...
```

## java code模式
以下分為四個步驟說明如何在java專案中套用`vespa-http-client-jar-with-dependencies.jar`
+ 加上dependency到`pom.xml`
```xml
<dependency>
  <groupId>com.yahoo.vespa</groupId>
  <artifactId>vespa-http-client</artifactId>
  <version>7.25.26</version> <!-- Find latest version at search.maven.org/search?q=g:com.yahoo.vespa%20a:vespa-http-client -->
</dependency>
```
+ 建立一個client實體
  + 利用FeedClientFactory建立client實體，並實作一個result callback [sample code: SampleFileFeeder](##sample code)
  + 盡可能在不同執行緒間，重複利用client實體 [sample code: SampleFileFeeder with thread](##sample code)
  + 使用結束後，記得關閉client實體
  
+ 準備寫入Vespa
  + 在client實體中使用`stream` 或 `feed`存取文件，並開始寫入Vespa

+ 實作callbacks
  + 覆寫`onCompletion`，當文件寫入Vespa後，callback會收到成功與否的訊息
  + 可考慮覆寫`onEndpointException`，當連線中斷時可借此監控   

## 優化技巧
  `vespa-http-client-jar-with-dependencies.jar`中預設的參數基本上適用於大部分的情境，但如果想更加優化寫入速度或傳輸量，可透過以下幾個準則
+ 釐清寫入速度的瓶頸
  + 假如增加client數量，傳輸量也跟著提升，瓶頸就是client數量不足，可以考慮增加機器或是執行緒
  > 但只有在執行緒分享相同client實體時，才保證寫入順序相同
  + 假如增加client數量，傳輸量沒有跟著提升，問題可能出在vespa cluster，可以考慮增加node數量
+ 調整`SessionParams`
  + 實作client實體時，調用SessionParams並優化其中`ConnectionParams`與`FeedParams`參數
    + ConnectionParams
      + 假如問題來自網路流量過小，可以啟用`compression`
      + `numPersistentConnectionsPerEndpoint`也是常用參數，可根據真實使用情境調整
    + FeedParams
      + 假如cluster過載，等待cluster回應時間便會超過`serverTimeout`
      + 假如client文件送出排隊過長，超過`clientTimeout`的文件**可能有被丟掉**的風險
      + `maxChunkSizeBytes`可以視網路狀況調整每次送出的package大小，此參數也跟`numPersistentConnectionsPerEndpoint`有關，通常建議**200kbytes**
      + 對每座cluster優化`maxInFlightRequests`，也可預防cluster過載
+ 調整`Endpoints`
  + 一個cluster通常有一到多個endpoints，當endpoints發生過載時，常見的的方法是啟用`DocumentProcessor`
+ 找出最慢的cluster
  + 利用client實體的`FeedClient.getStatsAsJson()`輸出的json字串，可觀察各cluster的狀況
  ```json
  {
      "clusters": [
          {
              "clusterid": 0,
              "stats": {
                  "session": [
                      {
                          "endpoint": {
                              "host": "localhost",
                              "port": 8080
                          },
                          "stats": {
                              "wrongSessionDetectedCounter": 0,
                              "wrongVersionDetectedCounter": 0,
                              "problemStatusCodeFromServerCounter": 0,
                              "executeProblemsCounter": 0,
                              "docsReceivedCounter": 4,
                              "statusReceivedCounter": 4,
                              "pendingDocumentStatusCount": 0
                          }
                      }
                  ]
              }
          }
      ],
      "sessionParams": {
        // .. The configuration parameters used.
      }
  }
  ```
## sample code
+ SampleFileFeeder
```java
package com.yahoo.example.feed;

import com.yahoo.vespa.http.client.FeedClient;
import com.yahoo.vespa.http.client.FeedClientFactory;
import com.yahoo.vespa.http.client.Result;
import com.yahoo.vespa.http.client.config.*;

import java.io.InputStream;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.logging.Logger;

/**
 * Sample feeder demonstrating how to programmatically feed to a Vespa cluster.
 */
public class SampleFileFeeder {

    private final Endpoint endpoint;
    private final SessionParams sessionParams;
    private final static Logger log = Logger.getLogger(SampleFileFeeder.class.getCanonicalName());

    /**
     * Whenever a result is received, this class is invoked. It keeps track of basic statistics.
     */
    static class ResultCallBack implements FeedClient.ResultCallback {

        final AtomicInteger resultsReceived = new AtomicInteger(0);
        final AtomicInteger errorsReceived = new AtomicInteger(0);
        final long startTimeMillis = System.currentTimeMillis();;

        @Override
        public void onCompletion(String docId, Result documentResult) {
            resultsReceived.incrementAndGet();
            if (!documentResult.isSuccess()) {
                log.warning("Problems with docID " + docId + ":" + documentResult.toString());
                errorsReceived.incrementAndGet();
            }
        }

        void dumpStatsToLog() {
            log.info("Received in total " + resultsReceived.get() + ", " + errorsReceived.get() + " errors.");
            log.info("Time spent receiving is " + (System.currentTimeMillis() - startTimeMillis) + " ms.");
        }
    }

    /**
     * Sample constructor without compression
     * @param endpoint The endpoint to feed to
     * @param dryRun if true, data is not sent to cluster.
     */
    public SampleFileFeeder(Endpoint endpoint, boolean dryRun) {
        this(endpoint, false, dryRun);
    }

    /**
     * More advanced constructor, that supports compression.
     *
     * @param endpoint The endpoint to feed to
     * @param useCompression  Whether to use compression or not
     * @param dryRun if true, will not send data to real cluster
     */
    public SampleFileFeeder(Endpoint endpoint, boolean useCompression, boolean dryRun) {
        this.endpoint = endpoint;
        this.sessionParams = new SessionParams.Builder()
                .addCluster(new Cluster.Builder().addEndpoint(this.endpoint).build())
                .setConnectionParams(new ConnectionParams.Builder()
                        .setDryRun(dryRun)
                        .setUseCompression(useCompression).build())
                .setFeedParams(new FeedParams.Builder()
                        .setDataFormat(FeedParams.DataFormat.JSON_UTF8)
                        .build())
                .build();
    }

    /**
     * Feed all operations from a stream.
     *
     * @param stream The input stream to read operations from (JSON formatted).
     */
    public ResultCallBack batchFeed(InputStream stream, String batchId) {
        ResultCallBack results = new ResultCallBack();
        FeedClient feedClient = FeedClientFactory.create(this.sessionParams, results);
        AtomicInteger numSent = new AtomicInteger(0);
        log.info("Starting feed to " + endpoint + " for batch '" + batchId + "'");
        FeedClient.feedJson(stream, feedClient, numSent);
        feedClient.close();  // Close will wait for all operations
        log.info("Feed " + numSent.get() + " operations to single cluster " + endpoint + ", batch: '" + batchId +".");
        results.dumpStatsToLog();
        return results;
    }
}
```
+ SampleFileFeeder with thread
```java
package com.yahoo.example.feed;

import com.yahoo.log.LogLevel;
import com.yahoo.vespa.http.client.FeedClient;
import com.yahoo.vespa.http.client.FeedClientFactory;
import com.yahoo.vespa.http.client.SimpleLoggerResultCallback;
import com.yahoo.vespa.http.client.config.Cluster;
import com.yahoo.vespa.http.client.config.ConnectionParams;
import com.yahoo.vespa.http.client.config.Endpoint;
import com.yahoo.vespa.http.client.config.FeedParams;
import com.yahoo.vespa.http.client.config.SessionParams;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.logging.Logger;

/**
 * Simple Streaming feeder implementation which will send operations to a Vespa endpoint.
 * Other threads communicate with the feeder by adding new operations on the BlockingQueue
 */

public class SampleStreamFeeder extends Thread {

    public static class Operation {
        final public String documentId;
        final public CharSequence data;

        public Operation(String id, CharSequence data) {
            this.documentId = id;
            this.data = data;
        }
    }

    private BlockingQueue<Operation> operations;
    private FeedClient feedClient;
    private AtomicInteger pending = new AtomicInteger(0);
    private final static Logger log = Logger.getLogger(SampleStreamFeeder.class.getCanonicalName());
    private AtomicBoolean  drain = new AtomicBoolean(false);
    private final CountDownLatch finishedDraining = new CountDownLatch(1);

    /**
     * Constructor
     * @param operations The shared blocking queue where other threads can put document operations to.
     * @param endPoint The endpoint to feed to
     */
    public SampleStreamFeeder(BlockingQueue<SampleStreamFeeder.Operation> operations, Endpoint endPoint, boolean dryRun) {
        this.operations = operations;
        SessionParams sessionParams = new SessionParams.Builder()
                .addCluster(new Cluster.Builder().addEndpoint(endPoint).build())
                .setConnectionParams(new ConnectionParams.Builder().setDryRun(dryRun).build())
                .setFeedParams(new FeedParams.Builder()
                        .setDataFormat(FeedParams.DataFormat.JSON_UTF8)
                        .build())
                .build();
        // Simple bundled logger result callback
        this.feedClient = FeedClientFactory.create(sessionParams, new SimpleLoggerResultCallback(this.pending, 10));
    }

    /**
     * Shutdown this feeder, waits until operations on queue is drained
     */
    public void close() throws InterruptedException {
        log.info("Shutdown initiated, awaiting operations queue to be drained. Queue size is " + operations.size());
        drain.set(true);
        finishedDraining.await();
    }

    @Override
    public void run() {
        while (!drain.get() || !operations.isEmpty()) {
            try {
                SampleStreamFeeder.Operation op = operations.poll(1, TimeUnit.SECONDS);
                if(op == null) // no operations available
                    continue;
                pending.incrementAndGet();
                log.info("Put document " + op.documentId);
                feedClient.stream(op.documentId, op.data);
            } catch (InterruptedException e) {
                log.log(LogLevel.ERROR, "Got interrupt exception.", e);
                break;
            }
        }
        log.info("Shutting down feeding thread");
        this.feedClient.close();
        finishedDraining.countDown();
    }
}
```