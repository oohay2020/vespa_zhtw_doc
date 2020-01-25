# 詢問源文件(輪廓)

> 由於Query Profile目前尚無統一中文化稱呼，下面還是以Query Profile來指代Vespa中特定檔案形式。

一個Query Profile是一群用來搜尋所下參數的集合體。
換句話說，原本可以使用Search API以參數方法後綴的參數，都可以用Query Profile來表示。
這樣做有什麼好處呢？
- 減輕客戶端要帶入一堆繁雜參數的負擔
- 有特殊狀況想要進行單一調控
  > Bucket-Test可以包含在這個優點中

## 使用一個Query Profile
Vespa中的Query Profile以XML格式來撰寫，如下：
```xml
<query-profile id="MyProfile">
  <field name="hits">10</field>
  <field name="unique">merchantid</field>
</query-profile>
```

## 裡面可以放什麼
參數
> 就肉身測試的結果，YQL設定似乎不行放在QueryProfile中。Query Profile可以幫忙暫存的參數屬於非必要之參數。

## 放在哪
Query Profile請務必放在`application/search/query-profile/`

## 使用流程
當想使用自行創建的Query Profile，請務必三步驟完成：
1. 建立`<query_profile_name>.xml`
2. 複製到目標`application/search/query-profile/`中
3. 重新佈署`application`
