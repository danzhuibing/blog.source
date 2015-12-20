title: Python工具箱：日志与配置文件 
date: 2015-12-20 16:17:49
tags:
- 工具箱
- Python

categories: 程序员工具箱

---

上班以来，负责了两个独立的Python工程，一个是为清华合作单位做的GPS点道路匹配的Hadoop Streaming程序，一个是为交通预测项目做的进行在线预测的Spark程序，开发语言都是Python。其中用了同事提供的两个脚本实现了日志和配置文件，感觉非常方便，因此写出来，方便自己以后使用，也分享给其他感兴趣的朋友。用到了一些别人的代码，表示感谢。如果觉得侵权，请联系我，立刻删除。

# 代码准备
首先，拷贝以下几个代码文件。

## 日志**logger.py**

作者是百度的裴欣，感谢他的代码，尽管我和他并不相识

``` python
#-*-coding:gbk-*-
"""
实现log相关功能,分级和输出形式模仿了ullog样式


******************使用方法*********华丽的分割线***********
import logger

# 初始化，输出DEUBG及以上级别的日志
# DEBUG, TRACE和NOTICE结果打在/log/a.log文件中
# WARNING和FATAL结果打在/log/a.log.wf文件中
logger.init('/log/a', 'DEBUG')

# 如果不做初始化，则会直接打在标准错误

# 打DEBUG日志
logger.debug_log('sdlksdlks')

# 打FATAL日志
logger.fatal_log('ksldsll')

# 其他级别log类似

"""
__authors__ = ['裴欣<peixin@baidu.com>', ]

import os
import inspect
import logging
import threading
from logging import handlers

logging.TRACE = 15
logging.addLevelName(logging.TRACE, 'TRACE')
logging.NOTICE = logging.INFO
logging.addLevelName(logging.NOTICE, "NOTICE")
logging.FATAL = logging.ERROR
logging.addLevelName(logging.FATAL, 'FATAL')

log_level_dict = {'DEBUG':10, 'TRACE':15, 'NOTICE':20, 'WARNING':30, 'FATAL':40}
formatter = logging.Formatter('%(levelname)s: %(asctime)s: %(message)s', datefmt='%m-%d %H:%M:%S')

"""
class LoggerState(object):
    def __init__():
        "disable the __init__ method"

    __instance = None
    __lock = threading.Lock()

    @staticmethod
    def has_initialized():
        LoggerState.__lock.acquire()
        result = True
        if None == LoggerState.__instance:
            LoggerState.__instance = object.__new__(LoggerState)
            result = False
        LoggerState.__lock.release()
        return result
"""
__initialized = False

__normal_logger = logging.getLogger('normal')
__wf_logger = logging.getLogger('wf')
#if not LoggerState.has_initialized():
if not __initialized:
    __normal_handler = logging.StreamHandler()
    __normal_handler.setFormatter(formatter)
    __normal_logger = logging.getLogger('normal')
    __normal_logger.addHandler(__normal_handler)
    __normal_logger.setLevel(0)
    __wf_handler = logging.StreamHandler()
    __wf_handler.setFormatter(formatter)
    __wf_logger = logging.getLogger('wf')
    __wf_logger.addHandler(__wf_handler)
    __wf_logger.setLevel(logging.WARNING)
    __initialized = True

def init(log_file, log_level='NOTICE'):
    global __normal_handler
    global __wf_handler
    global __normal_logger
    global __wf_logger

    log_dir = os.path.dirname(log_file)
    if not os.path.exists(log_dir):
        os.makedirs(log_dir)
    if os.path.isfile(log_dir):
        raise IOException('the path [%s] is regular file but not a dir'%(log_dir))

    real_log_level = log_level_dict.get(log_level, 20)
    __normal_logger.removeHandler(__normal_handler)
    __normal_handler = logging.handlers.WatchedFileHandler('%s.log'%(log_file))
    __normal_handler.setFormatter(formatter)
    __normal_logger = logging.getLogger('normal')
    __normal_logger.addHandler(__normal_handler)
    __normal_logger.setLevel(real_log_level)

    __wf_logger.removeHandler(__wf_handler)
    __wf_handler = logging.handlers.WatchedFileHandler('%s.log.wf'%(log_file))
    __wf_handler.setFormatter(formatter)
    __wf_logger = logging.getLogger('wf')
    __wf_logger.addHandler(__wf_handler)
    __wf_logger.setLevel(logging.WARNING)

def close():
    __normal_handler.flush()
    __normal_handler.close()

    __wf_handler.flush()
    __wf_handler.close()

    logging.shutdown()
    

def __get_call_func_frame_info():
    frame = inspect.getouterframes(inspect.currentframe())[2]
    frame_info = inspect.getframeinfo(frame[0])
    info = '[%s][%d][%s]'%(frame_info.filename, frame_info.lineno, frame_info.function)
    return info

def warning_log(info):
    stack_info = __get_call_func_frame_info()
    __wf_logger.warning('%s %s'%(stack_info, info))

def fatal_log(info):
    stack_info = __get_call_func_frame_info()
    __wf_logger.log(logging.FATAL, '%s %s'%(stack_info, info))

def notice_log(info):
    stack_info = __get_call_func_frame_info()
    __normal_logger.log(logging.NOTICE, '%s %s'%(stack_info, info))

def trace_log(info):
    stack_info = __get_call_func_frame_info()
    __normal_logger.log(logging.TRACE, '%s %s'%(stack_info, info))

def debug_log(info):
    stack_info = __get_call_func_frame_info()
    __normal_logger.debug('%s %s'%(stack_info, info))

```

## 配置**read_conf.py**
作者不详，送出不明方向的祝福。

``` python
#-*-coding:gbk-*-
"""
提供了从python文件中读取配置的简单检查函数
"""
import imp
import sys
import types
import os.path
import logger

def get_float(value, min=sys.float_info.min, max=sys.float_info.max):
    """
    检查一个值的类型是否是浮点数
    并检查值是否正常
    """
    if type(value) != types.FloatType and type(value) != types.IntType:
        raise TypeError("type of [%s] is not float"%(value))
    if value < min or value > max:
        raise ValueError("invalid value [%d]"%(value))
    return value

def get_int(value, min = -sys.maxint - 1, max=sys.maxint ):
    """
    检查一个值的类型是否是整数
    并检查值是否正常
    """
    if type(value) != types.IntType:
        raise TypeError("type of [%s] is not int"%(value))
    if value < min or value > max:
        raise ValueError("invalid value [%d]"%(value))
    return value

def get_str(value):
    """
    检查一个值的类型是否是字符串
    """
    if type(value) != types.StringType:
        raise TypeError("type of [%s] is not string"%(value))
    return value

def get_list(value, min_size=0, max_size=sys.maxint, item_type=None):
    """
    检查一个值的类型是否是list
    同时对list的size进行检查
    是否小于最小size或者大于最大size
    如果item_type不是None，则对list中每个元素的类型进行检查
    要求list中每个元素的类型都要符合要求
    """
    if type(value) != types.ListType:
        raise TypeError("type of [%s] is not list"%(value))
    if len(value) < min_size or len(value) > max_size:
        raise ValueError("the length of input list is invalid")
    if item_type != None:
        if any([type(item)!=item_type for item in value]):
            raise TypeError("the type of all the element must be [%s]"%(item_type))
    return value

def get_conf_module(conf_path):
    if not os.path.exists(conf_path):
        logger.fatal_log('conf file [%s] does not exist'%(conf_path))
        raise Exception('conf file [%s] does not exist'%(conf_path))
    path_split = os.path.split(conf_path)
    try:
        conf_module = imp.load_source(path_split[1], conf_path)
    except Exception, e:
        logger.fatal_log('can not load the config file [%s] because [%s]'%(conf_path, e))
        raise Exception('can not load the config file [%s]'%(conf_path))
    return conf_module
```

# 构建自己的应用
首先，设计自己程序需要配置的参数， 文件名假设为**conf.py**

``` python
ODPS_PATH="XXXXXX"
INPUT_PROJECT="XXXXX"
INPUT_TABLES=["XXXXXX"]
OUTPUT_PROJECT="XXXXXX"
OUTPUT_TABLES=["XXXXX"]
BIN_NAME="XXXXX"
MAP_HOME="XXXXXX"
MAP_VERSION="XXXXX"
ADCODE="XXXXX"
MONTH="XXXXX"
```
然后写主程序，文件名假设为**test.py**
``` python
import read_conf
import logger
from optparse import OptionParser
import types

def get_conf(conf_name): 
#conf_name为配置文件的地址，返回字典，保存配置文件的信息
    try:
        conf_module = read_conf.get_conf_module(conf_name)
    except Exception, e:
        logger.fatal_log('cannot load the config file [%s] because [%s]' % (conf_name, e))
        raise Exception('cannot load the config file [%s]' % (conf_name))
    result = {}
    try:
        result['odps_path'] = read_conf.get_str(conf_module.ODPS_PATH)
        result['input_project'] = read_conf.get_str(conf_module.INPUT_PROJECT)
        result['input_tables'] = read_conf.get_list(conf_module.INPUT_TABLES, 1, types.StringType)
        result['output_project'] = read_conf.get_str(conf_module.OUTPUT_PROJECT)
        result['output_tables'] = read_conf.get_list(conf_module.OUTPUT_TABLES, 1, types.StringType)
        result['bin_name'] = read_conf.get_str(conf_module.BIN_NAME)
        result['map_home'] = read_conf.get_str(conf_module.MAP_HOME)
        result['map_version'] = read_conf.get_str(conf_module.MAP_VERSION)
        result['adcode'] = read_conf.get_str(conf_module.ADCODE)
        result['month'] = read_conf.get_str(conf_module.MONTH)
    except Exception, e:
        logger.fatal_log('failed to read conf [%s]'%(e))
        sys.exit(1)
    return result

if __name__ == '__main__':
    usage_string = "usage: python %prog [options] arg"
    parser = OptionParser(usage=usage_string)
    parser.add_option('-c', dest='conf', default=None, help='read conf here')
    (options, args) = parser.parse_args()
    if options.conf == None:
        logger.fatal_log(" -c parameter must be specified. See help for more information.")
        sys.exit(-1)
    home = os.path.dirname(sys.path[0])
    try:
        logger.init('%s/log/log_%s'%(home,options.posix), 'NOTICE')
    except Exception, e:
        logger.fatal_log(' failed to create log')
        sys.exit(-1)
    conf = get_conf(options.conf)

```


