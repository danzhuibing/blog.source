title: R数据分析进阶11：Shiny打造定制化数据分析平台.md
date: 2016-05-06 13:21:12
tags:
- R语言
- 统计学
categories: R数据分析进阶
---

# Shiny基础
Shiny的作用是把你在R的分析代码包装为一个交互式网站，分享给不会或不愿直接接触R代码的朋友，让他也享受到用R进行数据分析的快感。一个最基本的shiny程序如下：

``` R
library(shiny)
ui <- fluidPage(
    numericInput(inputId = "n", "Smaple size", value=25),
    plotOutput(outputId = "hist")
)
server <- function(input, output) {
    output$hist <- renderPlot({
        hist(rnorm(input$n))
    })
}
shinyApp(ui=ui, server=server)
```

本文不对Shiny的基础知识进行过多的介绍，只记录在使用Shiny时几个重要的笔记。基础知识请参考官网的tutorial和article。

# 关于reactive programming
## 理解reactive expression **reactive({})**
reative expression用于避免重复计算，适用于计算较慢的操作。第一次计算reactive expression后值会保留。下一次调用reactive expression时，它会检查它所依赖的输入是否变化，若变化了重新计算，若无变化直接返回保存的结果。

reactive expression的input是UI input的值，或其他reactive expression，只能在其他reactive expression或render*函数调用reactive expression。

## 用**isolate({})**删除依赖
isolate的作用是让observer/endpoint能够访问一个reactive value或reactive expression，但不要依赖它。即她的值改变以后，并不会立刻触发observer/endpoint进行重算。一般地，我们会使用actionButton来作为触发isolate后的重算信号。actionButton每点击一次，会发送给server一个递增1的数字，初始化界面时值为0. 一个常用的模式是，在observer/endpoint中加入actionButton的依赖，同时将想要访问的reative value和reactive expression放到isolate里

# 输入组件不满足需求怎么办？
可以使用HTML函数，然后用HTML来写。举个例子，我想有一个文本框，可以采用下面的代码：

```R
default_text <- "Hallo Shiny"
HTML(paste('<textarea id="sql_cmd" rows="10", cols="180">', default_text,'</textarea>'))
```

# 想要根据用户的操作改变输入组件的状态
R提供了update*函数来更新输入组件的状态。举个例子：
``` R
# 原始UI
sliderInput(inputId="eta_slider", label="路线ETA范围", min=0, max=10, value=c(0,5)),

# 更新UI
updateSliderInput(session, "eta_slider", value = c(5, 10),
                      min = 5, max = 10)
```

# 输出自定义的html组件
一开始我使用了textOutput，但是没法实现换行。那么怎么办呢？有两种方法，一种是使用verbatimTextOutput，一种是使用htmlOutput。示例代码如下：
```  R
require(shiny)
runApp(
  list(
    ui = pageWithSidebar(
      headerPanel("multi-line test"),
      sidebarPanel(
        p("Demo Page.")
      ),
      mainPanel(
        verbatimTextOutput("text"),
        htmlOutput("text2")
      )
    ),
    server = function(input, output){

      output$text <- renderText({
        paste("hello", "world", sep="\n")
      })

      output$text2 <- renderUI({
        HTML(paste("hello", "world", sep="<br/>"))
      })

    }
  )
  )
```

# DT表格响应行点击
希望点击表格的一列后，获得这列的值进行相应的处理。
``` R
# UI
DT::dataTableOutput('tab')
otherOutput('example')

# Server
  output$tab <- DT::renderDataTable({
    final_df <- ...
    DT::datatable(final_df,
                  filter='top',
                  selection='single')
  })
output$example <- renderOther({
    # 单击事件
    s = input$tab_rows_selected
     if(is.null(s)) {
      s = c(1)
    }
    # 多选事件
    s = input$tab_rows_selected
})
```

