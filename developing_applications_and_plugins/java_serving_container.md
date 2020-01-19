# Java 資料密集服務容器 (JDisc)
Vespa使用JDisc來提供全部的應用元件，而他的設定就在`application/service.xml`中。
JDisc主要功能包括：
- HTTP服務(內層包Jetty)
- 即時的設定系統?
- Felix?
- 符合OSGi架構的元件模型
- Vespa要重新設定的時候不用重啟喔?
- 標準元件類型
  - 一般型
  - 鎖鏈型
  - 文件寫入型
  - 截取型
  - 重整型
- 把支援類型整合成鍊的拼裝機制

## 應用範例
- [Vespa Searcher版本的Hello World!](./application_development_basics.md)
- [用調節器與處理器來建立API吧](https://docs.vespa.ai/documentation/jdisc/http-api-tutorial.html)
- [用搜尋器來建立API吧](https://docs.vespa.ai/documentation/handler-tutorial.html)

## 開發Vespa元件

### 型態來區分
元件可以分成三大類
1. 鏈型結構(Chain)：以有序方法連結一群轉換方法
2. 處理器(Processor)：用來有效改變輸入或輸出結構
3. 渲染器(Renderer)：將處理器完成結果結構化給使用者

一般來說，順序大致上為
```
client -> processor -> chain -> processor -> renderer -> client
```

### 應用開發週期(包含單元測試)
TODO
