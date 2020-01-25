# Vespa詢問語言

Vespa允許人類口語式無序的輸入或者是有序的請求。
總之Vespa可以把它們這群東西結合起來執行。
最後進入應用的公式就叫YQL。
> 這段真的很神，講完也沒說為啥不叫VQL卻是YQL。
還不如直接告訴我因為Vespa是Yahoo開源的，所以要用Yahoo Query Language，簡稱YQL。

## 寫法整理
會SQL或是HiveQL的人就用感覺寫。
剩下的或不會的就依照下面的規則來思考吧：
1. `string`或numeric可以做order by
2. `string`可以contains
3. `numeric`可以用三一律
4. `boolean`可以用=
5. 多個字串比對可以用括號()
6. 關鍵字`limit`/`offset`/`timeout`直接加尾巴
7. regex請搭配`matches`

剩餘的進階，未來補齊。

## 參考
1. [原文](https://docs.vespa.ai/documentation/query-language.html)
