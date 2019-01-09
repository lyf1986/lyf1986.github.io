---
layout: post
title:  "通过扩展点解决spring和dubbo版本不兼容下的属性不解析问题"
date:   2018-06-01 22:14:54
categories: springboot
excerpt: 通过扩展点解决spring和dubbo版本不兼容下的属性不解析问题
---
# 通过扩展点解决spring和dubbo版本不兼容下的属性不解析问题

[TOC]

#### 问题描述
spring+dubbo项目加入配置中心
```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-config</artifactId>
	<version>2.0.1.RELEASE</version>
</dependency>
```

异常信息如下：
>Caused by: org.I0Itec.zkclient.exception.ZkException: Unable to connect to ${zookeeper.address}:9090


如下配置，使用了占位符${zookeeper.address}：

<dubbo:registry id="clientZookeeper" protocol="zookeeper" address="${zookeeper.address}" />


通过propertyPlaceholderConfigurer加载zookeeper.ip

同样的配置，在服务提供者端没有问题，在消费者端${zookeeper.address}没有被解析，导致连不上zookeeper。

于是尝试在原本运行正常的服务提供者端，加入dubbo:reference标签引用自己的服务，也就是同时也作为消费者端，
结果是${zookeeper.address}没有被解析。

也就是确定了问题出现的场景是：应用作为消费者时就会出现占位符未解析问题。


#### 解决方法

通过自定义BeanFactoryPostProcessor，修改bean定义

spring.xml

```xml
<bean class="com.*.processor.ZookeeperConfigProcessor">
		<property name="dubboZookeeperIds" value="preZookeeper" />
	</bean>
```

Java

```JAVA
import com.alibaba.dubbo.config.RegistryConfig;
import org.apache.commons.lang.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.BeansException;
import org.springframework.beans.MutablePropertyValues;
import org.springframework.beans.PropertyValue;
import org.springframework.beans.factory.config.BeanDefinition;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;

import java.util.List;

public class ZookeeperConfigProcessor implements BeanFactoryPostProcessor {
    private static Logger logger= LoggerFactory.getLogger(ZookeeperConfigProcessor.class);
    private String dubboZookeeperIds;

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory arg0) throws BeansException {
        if (StringUtils.isNotBlank(this.dubboZookeeperIds)){
            String[] zkIds = this.dubboZookeeperIds.split(",");
            for(String zkId:zkIds) {
                BeanDefinition obj = arg0.getBeanDefinition(zkId);
                Object singObj = arg0.getSingleton(zkId);

                MutablePropertyValues pvs = obj.getPropertyValues();
                List<PropertyValue> propertyValueList = pvs.getPropertyValueList();
                for (PropertyValue pv : propertyValueList) {
                    logger.info("key:{} - value:{}" , pv.getName() , pv.getValue());

                    if ("address".equalsIgnoreCase(pv.getName())) {
                        if (singObj instanceof RegistryConfig) {
                            RegistryConfig registryConfig = (RegistryConfig) singObj;
                            registryConfig.setAddress(pv.getValue().toString());
                        }
                    }
                }
            }
        }
    }

    public String getDubboZookeeperIds() {
        return dubboZookeeperIds;
    }

    public void setDubboZookeeperIds(String dubboZookeeperIds) {
        this.dubboZookeeperIds = dubboZookeeperIds;
    }
}
```

#### 参考文献
[【Spring中BeanFactoryPostProcessor和BeanPostProcessor区别】](./Spring中BeanFactoryPostProcessor和BeanPostProcessor区别.md)
