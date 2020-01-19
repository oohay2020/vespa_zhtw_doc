# 瞄一眼

## 用Docker來執行看看Vespa
1. 確認環境
```sh
$ docker info | grep "Total Memory"
```
> Vespa以整份教學出現的資訊來說，記憶體使用量不少於6~10G。實際使用VirtualBox執行，留下10G是一個比較保守的數量。

2. 使用`git clone`取得範例`application`
```sh
$ git clone https://github.com/vespa-engine/sample-apps.git
$ export VESPA_SAMPLE_APPS=`pwd`/sample-apps
```
> 注意到目前為止暗示了我們只有建立application，也就是處理stateless container的部分。

3. 啟動**Vespa容器**
```sh
$ docker run --detach --name vespa --hostname vespa-container --privileged \
  --volume $VESPA_SAMPLE_APPS:/vespa-sample-apps --publish 8080:8080 vespaengine/vespa
```
> 使用的Vespa運作架構都被包在vespaengine/vespa這個容器裡面囉。

4. 確認Vespa容器真的有被啟動(HTTP回應要有200)
```sh
$ docker exec vespa bash -c 'curl -s --head http://localhost:19071/ApplicationStatus'
```

5. 部署並啟動Vespa應用
```sh
$ docker exec vespa bash -c '/opt/vespa/bin/vespa-deploy prepare \
  /vespa-sample-apps/album-recommendation-selfhosted/src/main/application/ && \
  /opt/vespa/bin/vespa-deploy activate'
```

6. 確定應用被啟動了
```sh
$ curl -s --head http://localhost:8080/ApplicationStatus
```

7. 餵資料
```sh
$ curl -H "Content-Type:application/json" --data-binary @${VESPA_SAMPLE_APPS}/album-recommendation-selfhosted/src/test/resources/A-Head-Full-of-Dreams.json \
  http://localhost:8080/document/v1/mynamespace/music/docid/1
$ curl -H "Content-Type:application/json" --data-binary @${VESPA_SAMPLE_APPS}/album-recommendation-selfhosted/src/test/resources/Love-Is-Here-To-Stay.json \
  http://localhost:8080/document/v1/mynamespace/music/docid/2
$ curl -H "Content-Type:application/json" --data-binary @${VESPA_SAMPLE_APPS}/album-recommendation-selfhosted/src/test/resources/Hardwired...To-Self-Destruct.json \
  http://localhost:8080/document/v1/mynamespace/music/docid/3
```

8. 練習來個服務請求吧
```sh
$ curl "http://localhost:8080/search/?ranking=rank_albums&yql=select%20%2A%20from%20sources%20%2A%20where%20sddocname%20contains%20%22music%22%3B&ranking.features.query(user_profile)=%7B%7Bcat%3Apop%7D%3A0.8%2C%7Bcat%3Arock%7D%3A0.2%2C%7Bcat%3Ajazz%7D%3A0.1%7D"
```
> 這字串正常人是看不懂的，請善用[URL Decoder](https://www.urldecoder.org/)。

9. 嘗試來GET一個被儲存在Content Cluster的文件吧
```sh
$ curl -s http://localhost:8080/document/v1/mynamespace/music/docid/2
```

10. 清除容器
```sh
$ docker rm -f vespa
```
