# Dubbo打印日志

## Filter记录输入输出日志

### dubbo 之filter使用

1、继承接口com.alibaba.dubbo.rpc.Filter实现public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException 方法

2、配置文件修改

　a)、在resources目录下添加纯文本文件META-INF/dubbo/com.alibaba.dubbo.rpc.Filter

​     例如：xxxFilter=com.xxx.AuthorityFilter

​    b)、修改dubbo的provider配置文件，在dubbo:provider中添加配置的filter

​      例如：<dubbo:provider filter="xxxFilter" />  

3、扩展用法，trace 功能，ip黑白名单拦截，日志记录等等



实现效果
dubbo的provider和consumer接口调用的入参和出参都会打印日志。
dubbo配置

`<dubbo:consumer check="false" filter="dubboConsumerLogFilter"></dubbo:consumer>`
`<dubbo:provider filter="dubboProducerLogFilter"/>`

创建文件
/src/main/resources/META-INF/dubbo/com.alibaba.dubbo.rpc.Filter

文件内容
dubboConsumerLogFilter=com.ecpark.ops.utils.DubboLogFilter

dubboProducerLogFilter=com.ecpark.ops.utils.DubboLogFilter



Filter类

```java
package com.ecpark.ops.utils;

import com.alibaba.dubbo.rpc.Filter;
import com.alibaba.dubbo.rpc.Invocation;
import com.alibaba.dubbo.rpc.Invoker;
import com.alibaba.dubbo.rpc.Result;
import com.alibaba.dubbo.rpc.RpcException;
import com.alibaba.fastjson.JSONArray;
import com.alibaba.fastjson.JSONObject;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * @author yzjiang
 * @description
 * @date 2019/1/8 0008 16:44
 */
public class DubboLogFilter implements Filter {
    private static Logger logger = LoggerFactory.getLogger(DubboLogFilter.class);
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        String name = invoker.getInterface().getName();
        Object[] args = invocation.getArguments();
        String method = invocation.getMethodName();
        String prefix = "日志："+name+"."+method;
        logger.info(prefix+" 入参=>"+JSONArray.toJSONString(args));
        Result result = invoker.invoke(invocation);
        if(result.hasException()) {
            Throwable e = result.getException();
            if(e.getClass().getName().equals("java.lang.RuntimeException")){
                logger.error(prefix+" 运行时异常=>"+JSONObject.toJSONString(result));
            }else{
                logger.error(prefix+" 异常=>"+JSONObject.toJSONString(result));
            }
        }else{
            logger.info(prefix+" 结果=>"+JSONObject.toJSONString(result));
        }
        return result;
    }
}
```

