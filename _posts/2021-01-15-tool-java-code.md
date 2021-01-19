---
layout:     post
title:      "积跬步，终以至千里"
subtitle:   "我的Java代码烂笔头"
color:      "white"
date:       2021-01-15
author:     "ThreeJin"
header-mask: 0.6
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-tool-code.jpg"
tags:
    - 工具
    - Java
---
> 主要是自己日常使用的一些工具代码，不定期更新

### 各种拦截器
##### 统一结果&异常
- ResultResponseAdvice  
```java
import org.apache.commons.lang3.StringUtils;
import org.springframework.amqp.AmqpRejectAndDontRequeueException;
import org.springframework.context.MessageSource;
import org.springframework.context.i18n.LocaleContextHolder;
import org.springframework.core.MethodParameter;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.MissingPathVariableException;
import org.springframework.web.bind.MissingServletRequestParameterException;
import org.springframework.web.bind.ServletRequestBindingException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.servlet.mvc.method.annotation.ResponseBodyAdvice;

import javax.annotation.Resource;
import javax.validation.ConstraintViolationException;
import javax.validation.ValidationException;
import java.net.BindException;

@RestControllerAdvice(basePackages = {"com.xxx.xxx.xxx"})
public class ResultResponseAdvice implements ResponseBodyAdvice<Object> {


    /**
     * 是否对返回值进行封装，返回true则执行beforeBodyWrite，否则不执行
     */

    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return !returnType.hasMethodAnnotation(ResultVO_IGNORE.class);
    }
    /**
     * 统一封装返回数据
     */
    @Override
    public Object beforeBodyWrite(Object o, MethodParameter methodParameter, MediaType mediaType, Class aClass,
                                  ServerHttpRequest serverHttpRequest, ServerHttpResponse serverHttpResponse) {
        //BaseResult作为统一的返回结果
        if (o instanceof BaseResult||o instanceof String) {
            return o;
        }
        o = o==null?"":o;
        return BaseResult.success(o);
    }

    /**
     * 未知异常处理
     * @param ex
     * @return
     */
    @ExceptionHandler(value = {Exception.class})
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public BaseResult handleRunTimeException(Exception ex) {
        return logAndReturnBaseResult(errorCode,ex);
    }

    /**
     * 记录日志并返回错误信息
     * @param ex
     * @return
     */
    private BaseResult logAndReturnBaseResult(String errorCode,Exception ex){
        LOG.errorWithErrorCode(errorCode,ex.getMessage(),ex);
        return BaseResult.fail(errorCode, getTranslatedErrorMsg(errorCode));
    }

    /**
     * 获取错误码对应的错误信息（结合国际化一起使用）
     * @param errorCode
     * @return
     */
    private String getTranslatedErrorMsg(String errorCode){
        //对应classpath目录下的国际化文件中的对应的key值
        String message = "aaa." + errorCode + ".bbb";
        return messageSource.getMessage(message,null, LocaleContextHolder.getLocale());
    }

}
```

- ResultVO_IGNORE  
```java
import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ResultVO_IGNORE {
}

```

##### 统一事务
```java
import org.aspectj.lang.annotation.Aspect;
import org.springframework.aop.Advisor;
import org.springframework.aop.aspectj.AspectJExpressionPointcut;
import org.springframework.aop.support.DefaultPointcutAdvisor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.TransactionDefinition;
import org.springframework.transaction.interceptor.DefaultTransactionAttribute;
import org.springframework.transaction.interceptor.NameMatchTransactionAttributeSource;
import org.springframework.transaction.interceptor.TransactionInterceptor;

@Aspect
@Configuration
public class TransactionAdvice {

    private static final String AOP_POINTCUT_EXPRESSION = "execution (* com.xxxx.*.service.*.*(..))";

    @Autowired
    private PlatformTransactionManager transactionManager;

    @Bean
    public TransactionInterceptor txAdvice() {

        DefaultTransactionAttribute txattrRequired = new DefaultTransactionAttribute();
        txattrRequired.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);

        DefaultTransactionAttribute txattrRequiredReadonly = new DefaultTransactionAttribute();
        txattrRequiredReadonly.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
        txattrRequiredReadonly.setReadOnly(true);

        NameMatchTransactionAttributeSource source = new NameMatchTransactionAttributeSource();
        source.addTransactionalMethod("add*", txattrRequired);
        source.addTransactionalMethod("save*", txattrRequired);
        source.addTransactionalMethod("delete*", txattrRequired);
        source.addTransactionalMethod("update*", txattrRequired);
        source.addTransactionalMethod("sync*", txattrRequired);
        source.addTransactionalMethod("exec*", txattrRequired);
        source.addTransactionalMethod("set*", txattrRequired);
        source.addTransactionalMethod("is*", txattrRequiredReadonly);
        source.addTransactionalMethod("back*", txattrRequiredReadonly);
        source.addTransactionalMethod("leave*", txattrRequiredReadonly);
        source.addTransactionalMethod("unPost*", txattrRequiredReadonly);
        return new TransactionInterceptor(transactionManager, source);
    }

    @Bean
    public Advisor txAdviceAdvisor() {
        AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
        pointcut.setExpression(AOP_POINTCUT_EXPRESSION);
        return new DefaultPointcutAdvisor(pointcut, txAdvice());
    }
}

```

### JPA
#### 分页查询
```java
public void jpaPageQueryWithCondition() {
        int pageNo = 1;
        int pageSize = 500;
        boolean valueLeft = Boolean.TRUE;
        while (valueLeft) {
            //按照id进行排序
            Sort sort = new Sort(Direction.ASC, "id");
            Pageable pageRequest = PageRequest.of(pageNo - 1, pageSize, sort);
            //组合条件查询
            Specification<XX> querySpecification = new Specification<XX>() {
                @Override
                public Predicate toPredicate(Root<XX> root, CriteriaQuery<?> criteriaQuery,
                                             CriteriaBuilder criteriaBuilder) {
                    List<Predicate> predicates = new ArrayList<Predicate>();
                    //or条件
                    predicates.add(criteriaBuilder.or(criteriaBuilder.equal(root.get("XX"), 123),
                            criteriaBuilder.equal(root.get("YY"), 123)));
                    //in条件
                    CriteriaBuilder.In<Object> xx = criteriaBuilder.in(root.get("XX"));
                    in集合.forEach(item -> {
                        xx.value(item.zzzz());
                    });
                    predicates.add(xx);
                    return criteriaQuery.where(predicates.toArray(new Predicate[predicates.size()])).getRestriction();
                }
            };
            Page<XX> all = xXRepository.findAll(querySpecification, pageRequest);
            if (null == all || CollectionUtils.isEmpty(all.getContent())) {
                valueLeft = Boolean.FALSE;
            } else {
                if (all.getContent().size() < pageSize) {
                    valueLeft = Boolean.FALSE;
                }
                分页结果集 = all.getContent();
            }
            pageNo++;
        }
    }
```


### 其他
#### UUID
```java
//唯一标示，6代表6位，长度可以自行决定
String userIndex = UUID.randomUUID().toString().replace("-", "").toUpperCase().substring(0, 6);
```

#### 单元测试
```java
String jsonResult = "XXX";
List<XX> modelResult = JSONObject.parseObject(jsonResult, new TypeReference<List<XX>>() {});
//db method
Pageable pageable = new PageRequest(1, 1, Sort.Direction.ASC, "id");
Page<XX> page=new PageImpl<XX>(modelResult,pageable,1);
XxRepository.findAll((Pageable)any);
result = page;
```
#### 日志+事务回滚
```java
//事务回滚
@Transactional
TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
//或者
@Transactional(rollbackFor=Exception.class)
```

### Swagger
##### Maven
```xml
<dependency>
	<groupId>io.springfox</groupId>
	<artifactId>springfox-swagger2</artifactId>
	<version>2.6.1</version>
</dependency>
<dependency>
	<groupId>io.springfox</groupId>
	<artifactId>springfox-swagger-ui</artifactId>
	<version>2.6.1</version>
</dependency>
```
##### SwaggerConfig
```java
import com.google.common.base.Function;
import com.google.common.base.Optional;
import com.google.common.base.Predicate;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import springfox.documentation.RequestHandler;
import springfox.documentation.builders.ApiInfoBuilder;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.service.ApiInfo;
import springfox.documentation.service.Contact;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;

import java.util.HashSet;

@Configuration
public class SwaggerConfig {
    // 定义分隔符,配置Swagger多包
    private static final String STRING = ";";
    @Bean
    public Docket createRestApi() {
        HashSet<String> protocols = new HashSet<>();
        protocols.add("http");
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .protocols(protocols)
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.xxx.xxx.xxx"))
                .paths(PathSelectors.any())
                .build();
    }

    private ApiInfo apiInfo() {
        Contact openApi = new Contact("threejin", null, "threejin@xxx.com");
        return new ApiInfoBuilder()
                .title("xxx服务接口")
                .description("xxx服务接口文档")
                .version("1.0.0")
                .contact(openApi)
                .build();
    }
    public static Predicate<RequestHandler> basePackage(final String basePackage) {
        return input -> declaringClass(input).transform(handlerPackage(basePackage)).or(true);
    }

    private static Function<Class<?>, Boolean> handlerPackage(final String basePackage)     {
        return input -> {
            // 循环判断匹配
            for (String strPackage : basePackage.split(STRING)) {
                boolean isMatch = input.getPackage().getName().startsWith(strPackage);
                if (isMatch) {
                    return true;
                }
            }
            return false;
        };
    }

    private static Optional<? extends Class<?>> declaringClass(RequestHandler input) {
        return Optional.fromNullable(input.declaringClass());
    }
}
```

##### Controller、DTO、VO
```java
//Controller
@Api(description = "Xxx服务")
@RestController
@RequestMapping("/test")
public class XxxController {
    @ApiOperation(value = "xxx接口", notes = "该接口用于xxxx")
    @PostMapping("/aaa/bbb")
    public void test(@RequestBody @Valid XxxDTO xxxDTO) {
        //do something
    }
    
    @ApiOperation(value = "xxx接口", notes = "该接口用于xxxx")
    @GetMapping("/ccc/eee")
    @ApiImplicitParam(paramType = "query", name = "eee", value = "释义", required = true, dataType = "int")
    public XxxVO test(@RequestParam(value = "eee") @Min(value = 0,message = "eee最小为0") @Max(value = Integer.MAX_VALUE, message = "eee已超过规定范围") int eee) {
        //do something
    }
    
    @ApiOperation(value = "xxx接口", notes = "该接口用于xxxx")
    @GetMapping("/ccc/ddd")
    @ApiImplicitParams({
    @ApiImplicitParam(paramType = "query", name = "ccc", value = "释义", required =true, dataType = "String"), 
    @ApiImplicitParam(paramType = "query", name = "ddd", value = "释义", required = true, dataType = "int")
    })
    public XxxVO test(@RequestParam(value ="ccc") @NotBlank String ccc, 
                      @RequestParam(value = "ddd") @Min(value = 0,message = "ddd最小为0") @Max(value = Integer.MAX_VALUE, message = "ddd已超过规定范围") int ddd) {
        //do something
    }
}

//DTO
@Data
@ApiModel()
public class XxxDTO {
    @ApiModelProperty(value = "属性", required = true, example = "true")
    private boolean aaaa;

    @ApiModelProperty(value = "属性",dataType = "int",required = true, example = "67")
    @Min(value = 0,message = "属性最低为0")
    @Max(value = 100,message = "属性最高为100")
    private int bbbb;
    
    @ApiModelProperty(value = "属性", required = true, example = "2020-04-22")
    @NotBlank(message = "日期不能为空")
    @Pattern(regexp ="^((((1[6-9]|[2-9]\\d)\\d{2})-(0?[13578]|1[02])-(0?[1-9]|[12]\\d|3[01]))|(((1[6-9]|[2-9]\\d)" + "\\d{2})" + "-" + "(0?[13456789]|1[012])-(0?[1-9]|[12]\\d|30))|(((1[6-9]|[2-9]\\d)\\d{2})-0?2-" + "(0?[1-9]|1\\d|2[0-8])" + ")|(((1[6-9]|[2-9]\\d)(0[48]|[2468][048]|[13579][26])|(" + "(16|[2468][048]|[3579][26])00))-0?2-29-)" + ")$", message = "请确认日期格式：yyyy-MM-dd，例如：2020-04-22")
    private String cccc;
}

//VO
//参考DTO，只是去除了required的属性和参数校验的注解
```

### 各种Utils
##### SpringUtils
- 代码  
```java
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.NoSuchBeanDefinitionException;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.stereotype.Component;

/**
 * SpringUtils
 * 
 */
@Component
public final class SpringUtils implements BeanFactoryPostProcessor {

    private static ConfigurableListableBeanFactory beanFactory;

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        SpringUtils.beanFactory = beanFactory;
    }

    public static ConfigurableListableBeanFactory getBeanFactory() {
        return beanFactory;
    }

    /**
     * 获取对象
     *
     * @param name
     * @return Object 一个以所给名字注册的bean的实例
     * @throws org.springframework.beans.BeansException
     *
     */
    @SuppressWarnings("unchecked")
    public static <T> T getBean(String name) throws BeansException {
        if(null==getBeanFactory()){
            return null;
        }else{
            T result = (T) getBeanFactory().getBean(name);
            return result;
        }
    }

    /**
     * 获取类型为requiredType的对象
     *
     * @param name
     * @return
     * @throws org.springframework.beans.BeansException
     *
     */
    public static <T> T getBean(Class<T> name) throws BeansException {
        if(null==getBeanFactory()){
            return null;
        }else{
            T result = (T) getBeanFactory().getBean(name);
            return result;
        }
    }

    /**
     * 如果BeanFactory包含一个与所给名称匹配的bean定义，则返回true
     *
     * @param name
     * @return boolean
     */
    public static boolean containsBean(String name) {
        return getBeanFactory().containsBean(name);
    }

    /**
     * 判断以给定名字注册的bean定义是一个singleton还是一个prototype。 如果与给定名字相应的bean定义没有被找到，将会抛出一个异常（NoSuchBeanDefinitionException）
     *
     * @param name
     * @return boolean
     * @throws org.springframework.beans.factory.NoSuchBeanDefinitionException
     *
     */
    public static boolean isSingleton(String name) throws NoSuchBeanDefinitionException {
        return getBeanFactory().isSingleton(name);
    }

    /**
     * @param name
     * @return Class 注册对象的类型
     * @throws org.springframework.beans.factory.NoSuchBeanDefinitionException
     *
     */
    public static Class<?> getType(String name) throws NoSuchBeanDefinitionException {
        return getBeanFactory().getType(name);
    }

    /**
     * 如果给定的bean名字在bean定义中有别名，则返回这些别名
     *
     * @param name
     * @return
     * @throws org.springframework.beans.factory.NoSuchBeanDefinitionException
     *
     */
    public static String[] getAliases(String name) throws NoSuchBeanDefinitionException {
        return getBeanFactory().getAliases(name);
    }

}
```

- 使用   
`private XXX xxx = SpringUtils.getBean(XXX.class);`  

##### RedisUtil
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.redis.core.RedisCallback;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;
import org.springframework.util.CollectionUtils;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.TimeUnit;

@Component
public class RedisUtil {

  @Autowired
  private RedisTemplate<String, String> redisTemplate;

  //设置redis的key过期时间，单位为分钟
  @Value("${spring.cache.redis.time-to-live}")
  private long redisTimeToLive;

  /**
   * 读取
   * @param key
   * @return
   */
  public String stringGet(final String key) {
    return redisTemplate.opsForValue().get(key);
  }

  /**
   * 写入
   * @param key
   * @param value
   * @throws Exception
   */
  public void stringSet(final String key, String value) throws Exception {
      redisTemplate.opsForValue().set(key, value,redisTimeToLive, TimeUnit.MINUTES);
  }

  /**
   * 更新
   * @param key
   * @param value
   * @throws Exception
   */
  public void stringGetAndSet(final String key, String value) throws Exception {
    redisTemplate.opsForValue().getAndSet(key, value);
    redisTemplate.expire(key,redisTimeToLive, TimeUnit.MINUTES);
  }

  /**
   * 删除
   * @param key
   * @throws Exception
   */
  public void stringDelete(final String key) {
    redisTemplate.delete(key);
  }

  /**
   * 获取hash中field对应的值
   *
   * @param key
   * @param field
   * @return
   */
  public String hashGet(String key, String field) {
    Object val = redisTemplate.opsForHash().get(key, field);
    return val == null ? null : val.toString();
  }

  /**
   * 添加or更新hash的值
   *
   * @param key
   * @param field
   * @param value
   */
  public void hashSet(String key, String field, String value) {
    redisTemplate.opsForHash().put(key, field, value);
    redisTemplate.expire(key,redisTimeToLive, TimeUnit.MINUTES);
  }

  /**
   * 删除hash中field这一对kv
   *
   * @param key
   * @param field
   */
  public void hashDel(String key, String field) {
    redisTemplate.opsForHash().delete(key, field);
  }


  /**
   * 全部查询
   * @param key
   * @return
   */
  public Map<String, String> hashGetAll(String key) {
    return redisTemplate.execute((RedisCallback<Map<String, String>>) con -> {
      Map<byte[], byte[]> result = con.hGetAll(key.getBytes());
      if (CollectionUtils.isEmpty(result)) {
        return new HashMap<>(0);
      }

      Map<String, String> ans = new HashMap<>(result.size());
      for (Map.Entry<byte[], byte[]> entry : result.entrySet()) {
        ans.put(new String(entry.getKey()), new String(entry.getValue()));
      }
      return ans;
    });
  }

  /**
   * 批量查询
   * @param key
   * @param fields
   * @return
   */
  public Map<String, String> hashMultiGet(String key, List<String> fields) {
    List<String> result = redisTemplate.<String, String>opsForHash().multiGet(key, fields);
    Map<String, String> ans = new HashMap<>(fields.size());
    int index = 0;
    for (String field : fields) {
      if (result.get(index) == null) {
        continue;
      }
      ans.put(field, result.get(index));
    }
    return ans;
  }
}
```

##### ParameterAssertUtil
```java

import org.apache.commons.lang3.StringUtils;
import org.springframework.util.CollectionUtils;

import java.util.Collection;

/**
 * <>参数校验断言工具类</>
 *
 * @since <>V1.0</>
 **/
public class ParameterAssertUtil {

    public static void expressionWithException(boolean expression, ErrorCode errorCode) {
        if (expression) {
            throw new Exception(errorCode.getCode(), errorCode.getMessage());
        }
    }

    public static void isEmptyStringWithException(String param, ErrorCode errorCode) {
        if (StringUtils.isBlank(param)) {
            throw new Exception(errorCode.getCode(), errorCode.getMessage());
        }
    }

    public static void isNullObjectWithException(Object object, ErrorCode errorCode) {
        if (null ==object) {
            throw new Exception(errorCode.getCode(), errorCode.getMessage());
        }
    }

    public static void isZeroNumWithException(Number number, ErrorCode errorCode) {
        if (0 == number.intValue()) {
            throw new Exception(errorCode.getCode(), errorCode.getMessage());
        }
    }

    public static void isZeroOrNullNumWithException(Long number, ErrorCode errorCode){
        if (null==number || 0 == number.intValue()) {
            throw new Exception(errorCode.getCode(), errorCode.getMessage());
        }
    }

    public static Boolean isZeroNum(Number number){
        if (null==number ||0 == number.intValue()) {
            return Boolean.TRUE;
        }
        return Boolean.FALSE;
    }
    public static void isEmptyNormalCollectionWithException(Collection collection, ErrorCode errorCode) {
        if (CollectionUtils.isEmpty(collection)) {
            throw new Exception(errorCode.getCode(), errorCode.getMessage());
        }
    }

    public static void isEmptyStrCollectionWithException(Collection<String> collection, ErrorCode errorCode) {
        if (isEmptyStrCollection(collection)) {
            throw new Exception(errorCode.getCode(), errorCode.getMessage());
        }
    }

    public static boolean isEmptyStrCollection(Collection<String> collection) {
        if (CollectionUtils.isEmpty(collection)) {
            return Boolean.TRUE;
        }
        for (String s : collection) {
            if (StringUtils.isBlank(s)) {
                return Boolean.TRUE;
            }
        }
        return Boolean.FALSE;
    }

    public static void isZeroFloatCollectionWithException(Collection<Float> collection, ErrorCode errorCode) {
        if (isZeroFloatCollection(collection)) {
            throw new Exception(errorCode.getCode(), errorCode.getMessage());
        }
    }

    public static boolean isZeroFloatCollection(Collection<Float> collection) {
        if (CollectionUtils.isEmpty(collection)) {
            return Boolean.TRUE;
        }
        for (Float aFloat : collection) {
            if (aFloat == 0) {
                return Boolean.TRUE;
            }
        }
        return Boolean.FALSE;
    }
}
```
##### DateUtil
```java

import org.apache.commons.lang3.StringUtils;
import java.sql.Timestamp;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.time.temporal.ChronoUnit;
import java.util.Date;

/**
 * <>时间工具类</>
 **/
public class DateUtil {

    //时间格式：2017-04-22T15:39:01+08:00
    private static final String DATA_FORMAT_YYYY_MM_DD_T_HH_MM_SS_800 = "yyyy-MM-dd'T'HH:mm:ssXXX";
    //时间格式：2020-06-13 08:08:49
    public static final String DATA_FORMAT_YYYY_MM_DD_SPACE_HH_MM_SS = "yyyy-MM-dd HH:mm:ss";
    
    private static final String DATE_FORMAT_YYYY_MM_DD_T_HH_MM_SS = "yyyy-MM-dd'T'HH:mm:ss";
    public static final String DATA_FORMAT_YYYY_MM_DD = "yyyy-MM-dd";
    public static final String DATA_FORMAT_HH_MM_SS = "HH:mm:ss";
    public static final String DATA_FORMAT_YYYY_MM_DD_HH_MM_SS_EXP = "yyyy-MM-dd:HH:mm:ss";
    public static final String DATA_FORMAT_YYYY_MM_DD_HH_MM_SS_T = "yyyy-MM-dd'T'HH:mm:ss";
    public static final String DATA_FORMAT_YYYY_MM_DD_HH_MM_SS_SSS_XXX = "yyyy-MM-dd'T'HH:mm:sssXXX";
    public static final String DATA_FORMAT_YYYY_MM_DD_HH_MM_SS_SSS = "yyyy-MM-dd'T'HH:mm:ss.SSS";
    public static final String DATA_FORMAT_YYYY_MM_DD_HH_MM_SS_SSZ = "yyyy-MM-dd'T'HH:mm:ss.SSSZ";


    /**
     * 字符串时间转为LocalDateTime，根据时间格式解析
     * @param timeStr
     * @param dateTimeFormat
     * @return
     */
    public static LocalDateTime parseTimeWithDateFormat(String timeStr,String dateTimeFormat) {
        DateTimeFormatter dateTimeFormatter = DateTimeFormatter.ofPattern(dateTimeFormat);
        return LocalDateTime.parse(timeStr, dateTimeFormatter);
    }

    /**
     * 字符串时间转为另外格式的字符串时间
     * @param timeStr
     * @param preDateTimeFormat
     * @param afterDateTimeFormat
     * @return
     */
    public static String transferTimeWithDateFormats(String timeStr,String preDateTimeFormat,
                                                        String afterDateTimeFormat) {
        DateTimeFormatter preDateTimeFormatter = DateTimeFormatter.ofPattern(preDateTimeFormat);
        return parseLocalDateTimeToStrWithDateFormat(LocalDateTime.parse(timeStr, preDateTimeFormatter),afterDateTimeFormat);
    }
    
    /**
     * LocalDateTime转为字符串，根据时间格式解析
     * @param localDateTime
     * @param dateTimeFormat
     * @return
     */
    public static String parseLocalDateTimeToStrWithDateFormat(LocalDateTime localDateTime,String dateTimeFormat){
        DateTimeFormatter dateTimeFormatter = DateTimeFormatter.ofPattern(dateTimeFormat);
        return localDateTime.format(dateTimeFormatter);
    }

    /**
     * 根据传入时间字符串+格式+最大允许时间间隔（毫秒），判断时间是否超期
     * @param timeStr
     * @param dateTimeFormat
     * @param maxTimeInterval
     * @return
     */
    public static boolean expireTime(String timeStr,String dateTimeFormat,long maxTimeInterval){
        LocalDateTime past = parseTimeWithDateFormat(timeStr, dateTimeFormat);
        long between = ChronoUnit.MILLIS.between(past, LocalDateTime.now());
        return between>(maxTimeInterval*1000);
    }

    //检验日期是否合法
    public static Boolean checkTimeIsYYYYMMDD(String time){
        String rexp = "((\\d{2}(([02468][048])|([13579][26]))[\\-]((((0?[13578])|(1[02]))[\\-]((0?[1-9])|" +
                "([1-2][0-9])|(3[01])))|(((0?[469])|(11))[\\-]((0?[1-9])|([1-2][0-9])|(30)))|(0?2[\\-]((0?[1-9])|" +
                "([1-2][0-9])))))|(\\d{2}(([02468][1235679])|([13579][01345789]))[\\-]((((0?[13578])|(1[02]))[\\-](" +
                "(0?[1-9])|([1-2][0-9])|(3[01])))|(((0?[469])|(11))[\\-]((0?[1-9])|([1-2][0-9])|(30)))|(0?2[\\-](" +
                "(0?[1-9])|(1[0-9])|(2[0-8]))))))";
        if (StringUtils.isBlank(time) || !time.matches(rexp)) {
            return false;
        }
        return true;
    }
}
```  

##### Bean拷贝

- BeanCopyUtil  
```java

import org.springframework.beans.BeanUtils;
import java.util.ArrayList;
import java.util.List;
import java.util.function.Supplier;

/**
 * <>bean拷贝</>
 */
public class BeanCopyUtil extends BeanUtils {
    /**
     * 集合数据的拷贝
     * @param sources: 数据源类
     * @param target: 目标类::new(eg: UserVO::new)
     * @return
     */
    public static <S, T> List<T> copyListProperties(List<S> sources, Supplier<T> target) {
        return copyListProperties(sources, target, null);
    }

    /**
     * 带回调函数的集合数据的拷贝（可自定义字段拷贝规则）
     * @param sources: 数据源类
     * @param target: 目标类::new(eg: UserVO::new)
     * @param callBack: 回调函数
     * @return
     */
    public static <S, T> List<T> copyListProperties(List<S> sources, Supplier<T> target, BeanCopyUtilCallBack<S, T> callBack) {
        List<T> list = new ArrayList<>(sources.size());
        for (S source : sources) {
            T t = target.get();
            copyProperties(source, t);
            list.add(t);
            if (callBack != null) {
                // 回调
                callBack.callBack(source, t);
            }
        }
        return list;
    }
}

```

- BeanCopyUtilCallBack  
```java
@FunctionalInterface
public interface BeanCopyUtilCallBack <S, T> {
    /**
     * 定义默认回调方法
     * @param t
     * @param s
     */
    void callBack(S t, T s);
}
```

- 使用  
`List<NewXxx> newXxxList =BeanCopyUtil.copyListProperties(oldXxxList, NewXxx::new);`  


