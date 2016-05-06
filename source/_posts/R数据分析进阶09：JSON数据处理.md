title: R数据分析进阶09：JSON数据处理.md
date: 2016-04-15 09:00:00
tags:
- R语言
- 统计学
categories: R数据分析进阶
---

数据分析的主流格式是DataFrame，起源于R，后被python pandas和spark DataFrame纷纷效仿。但是在网络数据传输领域，DataFrame不是主流，而是JSON（Javascript Object Notation）。本文讲解如何在R里发送网络请求，解析返回的JSON数据。

# httr：发送网络请求获取数据
httr是Hadley Wickham的又一力作，可以发送POST和GET等HTTP请求，举个简单的POST请求示例。假设我们有一个POST服务，接受json的输入，返回json的输出。

``` R
url <- "http://XXXXX"
req <- '[1,"info",1,0,{"1":{"lst":["i64",3,5941105583405931848,5941105583405931847,5941105583405931838]}}]'

p <- POST(url,  content_type_json(),  accept_json(), body = req)
content(p, "text") # 调用content获得返回的JSON内容
```

# jsonlite
fromJSON()方法，将JSON数据转化为list或data.frame结构。
```
json <-
'[
  {"Name" : "Mario", "Age" : 32, "Occupation" : "Plumber"}, 
  {"Name" : "Peach", "Age" : 21, "Occupation" : "Princess"},
  {},
  {"Name" : "Bowser", "Occupation" : "Koopa"}
]'
mydf <- fromJSON(json)
mydf
```
# jsonview：JSON可视化
采用github方式安装，然后调用json_tree_view即可。
```
#devtools::install_github("hrbrmstr/jsonview")
library(jsonview)
json_tree_view(fromJSON(res_json))
```

