title: Python工具箱：与外部程序管道交互 
date: 2015-12-19 08:56:02
tags:
- 工具箱
- Python

categories: 程序员工具箱

---

假设外部有一个streaming风格的程序decode，即输入从stdin读取，输出写到stdout。现在想用python调用它，可以采用os模块的popen2（已不推荐）或者subprocess模块的Popen。

``` python
import sys,os
import subprocess
#首先，用python构造出decode需要的输入content
content = "XXXXXX"
try:
    #pipe_in, pipe_out = os.popen2("./decode", "wr")
    #pipe_in.write(content)
    #pipe_in.close()
    #ret = pipe_out.readline()
    #pipe_out.close()
    cur_dir = sys.path[0]
    p = subprocess.Popen("%s/KeyPoints2ConLinks"%cur_dir, stdin = subprocess.PIPE, stdout = subprocess.PIPE, shell = True)  
    p.stdin.write(content)
    (ret,erroutput) = p.communicate()
    if ret:
        print ret,
except Exception, e:
    print >> sys.stderr, "error with Exception [%s]" % e
```


