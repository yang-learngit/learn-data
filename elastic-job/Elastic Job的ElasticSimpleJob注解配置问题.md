# Elastic Job的ElasticSimpleJob注解配置问题

## 问题

在elasticJob程序中，存在两个elasticJob作业，在使用ElasticSimpleJob注解的时候，如果配置的cron值一样的话，启动程序会报错，生成实体Bean冲突。

elasticJob作业的配置：

```java
@ElasticSimpleJob(cron = "0 5 2 * * ?", jobName = "CoinAnalyzeJob", shardingTotalCount = 1,
        listener = "coinAnalyzeJobListener")
public class CoinAnalyzeJob implements SimpleJob{
    ...
}


@ElasticSimpleJob(cron = "0 5 2 * * ?", jobName = "CoinMonthAnalyzeJob", shardingTotalCount = 1)
public class CoinMonthAnalyzeJob implements SimpleJob {
    ...
}
```

程序启动报错信息：

```java
org.springframework.beans.factory.BeanDefinitionStoreException: Failed to parse configuration class [cn.ecpark.finance.fans.AnalyticJobApplication]; nested exception is org.springframework.context.annotation.ConflictingBeanDefinitionException: Annotation-specified bean name '0 5 2 * * ?' for bean class [cn.ecpark.finance.fans.analytic.job.CoinMonthAnalyzeJob] conflicts with existing, non-compatible bean definition of same name and class [cn.ecpark.finance.fans.analytic.job.CoinAnalyzeJob]
```



## 问题追踪分析

查看ElasticSimpleJob注解内容：

```java
@Inherited
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface ElasticSimpleJob {
	@AliasFor("cron")
	String value() default "";
	@AliasFor("value")
	String cron() default "";
    
	...
}
```

ElasticSimpleJob注解里面的value属性和cron属性添加了AliasFor注解，这样使得value和cron就是互为别名，并且值一样；

（注：在同一个注解中成对使用即可，比如示例代码中，value和cron就是互为别名。但是要注意一点，@AliasFor标签有一些使用限制，但是这应该能想到的，比如要求互为别名的属性属性值类型，默认值，都是相同的，互为别名的注解必须成对出现，比如value属性添加了@AliasFor("cron")，那么cron属性就必须添加@AliasFor("value")，另外还有一点，互为别名的属性必须定义默认值。）



跟踪spring的Bean创建，找到根据生成bean名称的方法

```java
    @Override
    public String generateBeanName(BeanDefinition definition, BeanDefinitionRegistry registry) {
        // 判断是否是否是AnnotatedBeanDefinition的子类， AnnotatedBeanDefinition是BeanDefinition的一个子类
        // 如果是AnnotatedBeanDefinition ， 按照注解生成模式生成信息，否则生成默认的bean name
        if (definition instanceof AnnotatedBeanDefinition) {
            String beanName = determineBeanNameFromAnnotation((AnnotatedBeanDefinition) definition);
            // 保证生成的bean name 非空
            if (StringUtils.hasText(beanName)) {
                // Explicit bean name found.
                return beanName;
            }
        }
        // Fallback: generate a unique default bean name.
        return buildDefaultBeanName(definition, registry);
    }


    /**
     * Derive a bean name from one of the annotations on the class.
     * 从类的注解中包含value属性的注解生成一个bean name
     * @param annotatedDef the annotation-aware bean definition
     * @return the bean name, or {@code null} if none is found
     */
	@Nullable
	protected String determineBeanNameFromAnnotation(AnnotatedBeanDefinition annotatedDef) {
		// 获取注解类元信息
        AnnotationMetadata amd = annotatedDef.getMetadata();
		//一个类存在多个注解，故类型为集合
        Set<String> types = amd.getAnnotationTypes();
		String beanName = null;
		for (String type : types) {
			//获取该类型对应的属性
            AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(amd, type);
			//判断注解类型是否包含value属性
            if (attributes != null && isStereotypeWithNameValue(type, amd.getMetaAnnotationTypes(type), attributes)) {
				Object value = attributes.get("value");
				if (value instanceof String) {
					String strVal = (String) value;
					if (StringUtils.hasLength(strVal)) {
						//不多于1个注解配置了value属性且非空，比如无法在一个类上面同时使用Component和Sevice注解同时配置beanName值
                        if (beanName != null && !strVal.equals(beanName)) {
							throw new IllegalStateException("Stereotype annotations suggest inconsistent " +
									"component names: '" + beanName + "' versus '" + strVal + "'");
						}
						beanName = strVal;
					}
				}
			}
		}
		return beanName;
	}



	protected boolean isStereotypeWithNameValue(String annotationType,
			Set<String> metaAnnotationTypes, @Nullable Map<String, Object> attributes) {

		boolean isStereotype = annotationType.equals(COMPONENT_ANNOTATION_CLASSNAME) ||
				metaAnnotationTypes.contains(COMPONENT_ANNOTATION_CLASSNAME) ||
				annotationType.equals("javax.annotation.ManagedBean") ||
				annotationType.equals("javax.inject.Named");

		return (isStereotype && attributes != null && attributes.containsKey("value"));
	}


```

determineBeanNameFromAnnotation这个方法里面有两个关键的处理流程：

第一步，读取对应annotationType对应的所有属性。

第二步，判断annotationType是否具有value属性，但是metaAnnotationTypes.contains(COMPONENT_ANNOTATION_CLASSNAME)这部分并不是很懂？

生成bean name有两条处理线，使用AnnotationBeanDefinition注解和不使用的：

第一、不使用AnnotationBeanDefinition注解的，直接将类名（不含包名）改为驼峰形式作为bean name。

第二、使用AnnotationBeanDefinition注解的：

1，读取所有注解类型

2，便利所有注解类型，找到所有为Component、Service，Respository，Controller含有非空value属性的注解

3，不多于一个个有效配置时生效，大于一个会抛出异常。（spring无法明确具体哪个生效）





