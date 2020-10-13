---
layout:     post
title:      "让我的IDEA好用到飞起"
subtitle:   "My configuration of IDEA."
date:       2020-10-09 23:59:00
author:     "ThreeJin"
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/article-head.png"
tags:
    - 工具
---
> 古有云，欲善其工必先利其器。

### 输入相关
##### 文件编码格式
- File Encoding设置为UTF-8
- C盘的idea64.exe.vmoptions，在文件末尾添加-Dfile.encoding=UTF-8 

##### 自动提示时忽略大小写
- Editor--general--CodeCompletion--Case sensitive修改为none 

### 快捷操作
##### SVN&Git
- SVN忽略iml文件以及idea文件
Editor--File Types--Ignore files and folders中：增加
`*.iml;`
注：每个忽略项之间是通过；连接的，最后一项也要以；结尾
- 自定义SVN操作到工具栏
添加版本控制(Add to VCS)、更新(update)、提交(Commit File)、历史记录(Show History)、版本对比(Compare with the same Repository Version和Compare with...)、回退（Revert）、Pull和Push等快捷图标到工具栏 

##### 自定义工具栏
鼠标放在工具栏上，右键，选择Customize Menus and Toolbars--Main Toolbar
- Main ToolbarSettings--VCS Actions
- 添加Show in Explorer的快捷图标 

### 视觉相关
##### 字体调整
- 代码：字体：Courier New     size：18     line spacing：1.1
- 控制台：

##### 窗口相关
- 关闭tab页：editor-general-editor tabs-Placement设置为none
- 关闭导航栏：view--Navigation bar 

### 代码快捷模板 
###### Editor-Live Template（建议新建自己的group）
- **自动生成方法说明**  
Abbreviation:  mcom（可以自取缩写）  
Descriptions:  generate method level comment  
Template text中的文本如下： 

```java
/**
 * <>说明</>
$params$
 * @return  $returns$
 * @author  <>liaoxinyi</>
 * @date  <>$date$</>
 */
```

然后点击 Edit variable，其中：date变量对应date()，time变量对应time()，params变量对应的脚本内容如下：  
```txt
groovyScript("def result=''; def params=\"${_1}\".replaceAll('[\\\\[|\\\\]|\\\\s]', '').split(',').toList(); for(i = 0; i < params.size(); i++) {if(params[i] == '') return result;result+=' * @param  <>' + params[i] + '</>  <>说明</>' + ((i < params.size() - 1) ? '\\n' : '')}; return result", methodParameters())
```
retrurns变量对应methodReturnType()，最后点击Define设置信息，点击Apply，OK   
- **自动生成类说明** 

```java
/**
 * <>功能</>
 *
 * @author  <>liaoxinyi</>
 * @date <>$DATE$</>
 * @since  <>V1.0.0</>
 */
```
其中的变量：DATE     date()  
- **自动生成成员变量注入**

```java
/**
 * $TYPE$
 */
@Autowired
private $TYPE$ $NAME$;
```
其中的变量：TYPE为capitalize(clipboard())，NAME为decapitalize(clipboard())  
- **自动生成log的final成员变量**

```java
/**
 * <>logger</>
 */
private static final Log LOG = LoggerFactory.getLogger($CLASS$.class);
```
其中的变量：CLASS为className()   
###### Editor-File and Code Template
- **Class**
```java
        #if (${PACKAGE_NAME} && ${PACKAGE_NAME} != "")package ${PACKAGE_NAME};#end
        #parse("File Header.java")
        /** 
        <>${description}</>
        @author  <>liaoxinyi</>
        @date <>${YEAR}/${MONTH}/${DAY}</>
        @since  <>V1.0</>
        **/
        public class ${NAME} {
        }
```
- **Interface**  
```java
        #if (${PACKAGE_NAME} && ${PACKAGE_NAME} != "")package ${PACKAGE_NAME};#end
        #parse("File Header.java")
        /** 
        <>${description}</>
        @author  <>liaoxinyi</>
        @date <>${YEAR}/${MONTH}/${DAY}</>
        @since  <>V1.0</>
        **/
        public interface ${NAME} {
        }
```

### 插件-Codemaker
###### JPA  
```java
        ---------------------------------------------
        ---------------Codemaker配置--------------------------
        -------------------------------------------------------
        Template Name：RepositoryJPA
        Class Number：1
        Class Name：${class0.className}Repository
        -----body
        #macro (low $strIn)$strIn.valueOf($strIn.charAt(0)).toLowerCase()$strIn.substring(1)#end 
        package ${class0.PackageName};

        import ${class0.PackageName}.$class0.className;
        import org.springframework.data.domain.Page;
        import org.springframework.data.domain.Pageable;
        import org.springframework.data.jpa.repository.JpaRepository;
        import org.springframework.data.jpa.repository.JpaSpecificationExecutor;
        import org.springframework.data.jpa.repository.Modifying;
        import org.springframework.data.jpa.repository.Query;
        import java.util.List;

        /** 
         * <>功能</>
         *
         * @author  <>liaoxinyi</>
         * @date <>$TIME</>
         * @since  <>V1.0</>
         */
        public interface $ClassName extends JpaRepository<$class0.className, Long>,
            JpaSpecificationExecutor<$class0.className> {

          /**
           * @return 
           * @description 
           * @param  
           */
          @Modifying
          @Query(value = "UPDATE tb_user AS u "
              + "SET u.user_phone = ?1 , u.user_mail = ?2 , u.user_address = ?3 "
              + "WHERE u.user_name = ?4", nativeQuery = true)
          int update(String userPhone, String userMail, String userAddress, String userName);
            
          /**
           * @return 
           * @description 
           * @param  
           */
          @Query(value = "SELECT pk_user_id FROM tb_user AS u WHERE u.user_name = ?1", nativeQuery = true)
          int find(String userName);
          
          /**
           * @return 
           * @description 
           * @param  
           */
          Page<$class0.className> findByContaining(String userName, Pageable pageable);
          
          /**
           * @return 
           * @description 
           * @param  
           */
          List<$class0.className> findBy(String userName);
          
          /**
           * @return 
           * @description 
           * @param  
           */
          @Modifying
          @Query(value = "DELETE FROM tb_user AS u WHERE u.user_name = ?1", nativeQuery = true)
          void delete(String userName);

        }
```

###### Mapper  
```java
        -------body
        Template Name：Mapper
        Class Number：1
        Class Name：${class0.className}Mapper
        -------------------
        #macro (low $strIn)$strIn.valueOf($strIn.charAt(0)).toLowerCase()$strIn.substring(1)#end 
        package ${class0.PackageName};

        import ${class0.PackageName}.$class0.className;
        import java.util.List;
        import org.apache.ibatis.annotations.Delete;
        import org.apache.ibatis.annotations.Insert;
        import org.apache.ibatis.annotations.Result;
        import org.apache.ibatis.annotations.ResultType;
        import org.apache.ibatis.annotations.Results;
        import org.apache.ibatis.annotations.Select;
        import org.apache.ibatis.annotations.Update;
        import org.springframework.stereotype.Component;

        /** 
         * <>功能</>
         *
         * @author  <>liaoxinyi</>
         * @date <>$TIME</>
         * @since  <>V1.0</>
         */
        @Component
        public interface $ClassName {

          /**
           * @return 
           * @description 
           * @param  
           */
          @Select("SELECT pk_user_id userId , user_phone userPhone FROM tb_user AS u WHERE u.user_name = #{xxx}")
          @ResultType(${class0.className}.class)
          List<${class0.className}> get${class0.className}ByXxx(String xxx);
          
            
          /**
           * @return 
           * @description 
           * @param  
           */
            @Insert("INSERT INTO users(userName,passWord,user_sex) VALUES(#{userName}, #{passWord}, #{userSex})")
            void insert(${class0.className} #low(${class0.className}));
          
          /**
           * @return 
           * @description 
           * @param  
           */
          @Update("UPDATE users SET userName=#{userName},nick_name=#{nickName} WHERE id =#{id}")
          void update(${class0.className} #low(${class0.className}));
          
          /**
           * @return 
           * @description:
           * @param  
           */
          @Delete("DELETE FROM users WHERE id =#{id}")
          void delete(Long id);
        }
```

#### 插件-Junit-Junit4  
内容参考IDEA仓库中的内容 
#### 常用插件中groovy配置文件位置
目录：C:\Users\用户名\.IntelliJIdea2018.2\config\extensions\com.intellij.database\schema
#### 常用插件
CodeGlance、CodeMaker、JUnitGenerator V2.0、Lombok、Maven Helper、POJO to JSON、RestfulToolkit、Statistics、String Manipulation、VisualVM Launcher 
#### 几个Groovy脚本
- DTO.groovy、Entity_JPA.groovy、VO.groovy、Entity_JPA.groovy，可在Idea仓库下载

### 代码风格
###### StyleByLXY.xml

```xml
<code_scheme name="StyleByLXY" version="173">
  <option name="WRAP_WHEN_TYPING_REACHES_RIGHT_MARGIN" value="true" />
  <JavaCodeStyleSettings>
    <option name="CLASS_COUNT_TO_USE_IMPORT_ON_DEMAND" value="200" />
    <option name="NAMES_COUNT_TO_USE_IMPORT_ON_DEMAND" value="200" />
    <option name="JD_ADD_BLANK_AFTER_PARM_COMMENTS" value="true" />
    <option name="JD_ADD_BLANK_AFTER_RETURN" value="true" />
  </JavaCodeStyleSettings>
  <codeStyleSettings language="JAVA">
    <option name="KEEP_LINE_BREAKS" value="false" />
    <option name="KEEP_FIRST_COLUMN_COMMENT" value="false" />
    <option name="KEEP_CONTROL_STATEMENT_IN_ONE_LINE" value="false" />
    <option name="KEEP_SIMPLE_METHODS_IN_ONE_LINE" value="true" />
    <option name="KEEP_SIMPLE_LAMBDAS_IN_ONE_LINE" value="true" />
    <option name="KEEP_SIMPLE_CLASSES_IN_ONE_LINE" value="true" />
    <option name="DOWHILE_BRACE_FORCE" value="3" />
    <option name="WHILE_BRACE_FORCE" value="3" />
    <option name="WRAP_LONG_LINES" value="true" />
    <option name="WRAP_ON_TYPING" value="1" />
  </codeStyleSettings>
</code_scheme>
```

##### GoogleStyle.xml
在Idea仓库下载 
#### 总结
- 工具只是工具，最重要的还是能力和习惯。
- 最为便捷的方式是，将设置好的IDEA配置，直接导出，方便以后直接配置。
