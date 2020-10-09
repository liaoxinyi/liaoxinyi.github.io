---
layout:     post
title:      "让我的IDEA好用到飞起"
subtitle:   "My configuration of IDEA."
date:       2020-10-09 23:59:00
author:     "ThreeJin"
header-img: "https://i.loli.net/2020/09/23/cpdwJsZi93e8LNB.png"
tags:
    - 工具
---
> 古有云，欲善其工必先利其器。

### 文件编码格式
- File Encoding设置为UTF-8
- C盘的idea64.exe.vmoptions，在文件末尾添加-Dfile.encoding=UTF-8
### 自动提示时忽略大小写
- Editor--general--CodeCompletion--Case sensitive修改为none
### SVN&Git
- SVN忽略iml文件以及idea文件
Editor--File Types--Ignore files and folders中：增加*.iml
注：每个忽略项之间是通过；连接的，最后一项也要以；结尾
- 自定义SVN添加版本控制(Add to VCS)、更新(update)、提交(Commit File)、历史记录(Show History)、版本对比(Compare with the same Repository Version和Compare with...)、回退（Revert）、Pull和Push等快捷图标到工具栏
### 自定义工具栏
鼠标放在工具栏上，右键，选择Customize Menus and Toolbars--Main Toolbar
- Main ToolbarSettings--VCS Actions
- 添加Show in Explorer的快捷图标
### 字体调整
- 代码：字体：Courier New     size：18     line spacing：1.1
- 控制台：
### 窗口相关
- 关闭tab页：editor-general-editor tabs-Placement设置为none
- 关闭导航栏：view--Navigation bar
### 代码快捷模板
#### Editor-Live Template（建议新建自己的group）
- 自动生成方法说明：
（1）Abbreviation:  mcom（可以自取缩写）
（2）Descriptions:  generate method level comment
（3）Template text中的文本如下：
```java
/**
 * <>说明</>
$params$
 * @return  $returns$
 * @author  <>liaoxinyi</>
 * @date  <>$date$</>
 */
```
（4）然后点击 Edit variable，其中：date变量对应date()，time变量对应time()，params变量对应的脚本内容如下：
```txt
groovyScript("def result=''; def params=\"${_1}\".replaceAll('[\\\\[|\\\\]|\\\\s]', '').split(',').toList(); for(i = 0; i < params.size(); i++) {if(params[i] == '') return result;result+=' * @param  <>' + params[i] + '</>  <>说明</>' + ((i < params.size() - 1) ? '\\n' : '')}; return result", methodParameters())
```
最后，retrurns变量对应methodReturnType()
（5）点击Define设置信息
（6）点击Apply，OK

- 自动生成类说明：
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
- 自动生成成员变量注入
```java
/**
 * $TYPE$
 */
@Autowired
private $TYPE$ $NAME$;
```
其中的变量：TYPE  capitalize(clipboard())     NAME   decapitalize(clipboard())
- 自动生成log的final成员变量
```java
/**
 * <>logger</>
 */
private static final Log LOG = LoggerFactory.getLogger($CLASS$.class);
```
其中的变量：CLASS     className()
#### Editor-File and Code Template
- Class
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
- Interface
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
#### JPA 

内容如下： 
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

#### Mapper 
内容如下：

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
### 插件-Junit-Junit4
内容如下： 
```java
        --------------------------------
        ----------------------------------
        --------------Junit-------------------
        OutputPath：  ${SOURCEPATH}/../../test/java/${PACKAGE}/${FILENAME}

        -----Junit4-----
        ######################################################################################## 
        ## 
        ## Available variables: 
        ##         $entryList.methodList - List of method composites 
        ##         $entryList.privateMethodList - List of private method composites 
        ##         $entryList.fieldList - ArrayList of class scope field names 
        ##         $entryList.className - class name 
        ##         $entryList.packageName - package name 
        ##         $today - Todays date in MM/dd/yyyy format 
        ## 
        ##            MethodComposite variables: 
        ##                $method.name - Method Name 
        ##                $method.signature - Full method signature in String form 
        ##                $method.reflectionCode - list of strings representing commented out reflection code to access method (Private Methods) 
        ##                $method.paramNames - List of Strings representing the method's parameters' names 
        ##                $method.paramClasses - List of Strings representing the method's parameters' classes 
        ## 
        ## You can configure the output class name using "testClass" variable below. 
        ## Here are some examples: 
        ## Test${entry.ClassName} - will produce TestSomeClass 
        ## ${entry.className}Test - will produce SomeClassTest 
        ## 
        ######################################################################################## 
        ## 
        #macro (cap $strIn)$strIn.valueOf($strIn.charAt(0)).toUpperCase()$strIn.substring(1)#end
        #macro (low $strIn)$strIn.valueOf($strIn.charAt(0)).toLowerCase()$strIn.substring(1)#end 
        ## Iterate through the list and generate testcase for every entry. 
        #foreach ($entry in $entryList) 
        #set( $testClass="${entry.className}Test") 
        ## 
        package java.$entry.packageName; 

        import com.alibaba.fastjson.JSONObject;
        import com.alibaba.fastjson.TypeReference;
        import mockit.Expectations;
        import mockit.Injectable;
        import mockit.Tested;
        import org.junit.Assert;
        import org.junit.Test;

        /** 
         * </>${entry.className} Tester</>
         *
         * @author  <>liaoxinyi</>
         * @date <>$today</>
         * @since  <>V1.0</>
         */
        public class $testClass { 

        @Tested
        private $entry.className #low(${entry.className});


        #foreach($method in $entry.methodList) 
        /** 
        * 
        * Method: $method.signature 
        * 
        */ 
        @Test
        public void successTestOf#cap(${method.name})(
        #foreach($field in $entry.fieldList) 
        @Injectable #cap(${field}) #low(${field}),
        #end
        )
        { 
                // Record
                new Expectations() {
                    {
                String jsonResult="XXXXX";
                BaseResult modelResult=JSONObject.parseObject(jsonResult, new TypeReference<BaseResult>(){});
                //feign method:  xxxx.yyy((XXX)any);

                result=modelResult;
                   }
                 };
                //private method
                new MockUp<XXXImpl>(){
                    @Mock
                    private String getXXX(String xxx){
                        return "XXXX";
                    }
                };
                 //Replay
                BaseResult result= #low(${entry.className}).${method.name}();
                //Verification
                Assert.assertTrue(result.getType() == 0);
        }
        #end 

        #foreach($method in $entry.methodList) 
        /** 
        * 
        * Method: $method.signature 
        * 
        */ 
        @Test(expected = Exception.class)
        public void failureTestOf#cap(${method.name})(
        #foreach($field in $entry.fieldList) 
        @Injectable #cap(${field}) #low(${field}),
        #end
        )
        { 
                // Record
                new Expectations() {
                    {
                String jsonResult="XXXXX";
                BaseResult modelResult=JSONObject.parseObject(jsonResult, new TypeReference<BaseResult>(){});
                //feign method:  xxxx.yyy((XXX)any);
                
                result=modelResult;
                   }
                 };
                 //Replay
                BaseResult result= #low(${entry.className}).${method.name}();
                //Verification
                Assert.assertTrue(result.getType() == 0);
        }
        #end 


        } 
        }
        #end
        -----------------------------------------------
```
### 常用插件中groovy配置文件位置
目录：C:\Users\用户名\.IntelliJIdea2018.2\config\extensions\com.intellij.database\schema
### 常用插件
CodeGlance、CodeMaker、JUnitGenerator V2.0、Lombok、Maven Helper、POJO to JSON、RestfulToolkit、Statistics、String Manipulation、VisualVM Launcher
### 几个Groovy脚本
#### DTO.groovy
内容如下：
```groovy
import com.intellij.database.model.DasTable
import com.intellij.database.model.ObjectKind
import com.intellij.database.util.Case
import com.intellij.database.util.DasUtil
import java.io.*
import java.text.SimpleDateFormat

/*
 * Available context bindings:
 *   SELECTION   Iterable<DasObject>
 *   PROJECT     project
 *   FILES       files helper
 */
packageName = ""
typeMapping = [
        (~/(?i)tinyint|smallint|mediumint/)      : "Integer",
        (~/(?i)int/)                             : "Long",
        (~/(?i)bool|bit/)                        : "Boolean",
        (~/(?i)float|double|decimal|real/)       : "Double",
        (~/(?i)datetime|timestamp|date|time/)    : "Timestamp",
        (~/(?i)blob|binary|bfile|clob|raw|image/): "InputStream",
        (~/(?i)/)                                : "String"
]

FILES.chooseDirectoryAndSave("Choose directory", "Choose where to store generated files") { dir ->
    SELECTION.filter { it instanceof DasTable && it.getKind() == ObjectKind.TABLE }.each { generate(it, dir) }
}

def generate(table, dir) {
    //根据表名生成类名
    def className = javaClassName(table.getName(), true)+"DTO"
    //def className = javaName(table.getName(), true)
    def fields = calcFields(table)
    packageName = getPackageName(dir)
    PrintWriter printWriter = new PrintWriter(new OutputStreamWriter(new FileOutputStream(new File(dir, className + ".java")), "UTF-8"))
    printWriter.withPrintWriter {out -> generate(out, className, fields,table)}
//    new File(dir, className + ".java").withPrintWriter { out -> generate(out, className, fields,table) }
}

// 获取包所在文件夹路径
def getPackageName(dir) {
    return dir.toString().replaceAll("\\\\", ".").replaceAll("/", ".").replaceAll("^.*src(\\.main\\.java\\.)?", "") + ";"
}

def generate(out, className, fields,table) {
    def tableName = table.getName()
    out.println "package $packageName"
    out.println ""
    out.println "import io.swagger.annotations.ApiModel;"
    out.println "import io.swagger.annotations.ApiModelProperty;"
    out.println "import javax.validation.constraints.NotNull;"  
    out.println "import javax.validation.constraints.Max;"
    out.println "import javax.validation.constraints.Min;"
    out.println "import javax.validation.constraints.NotBlank;"
    out.println "import org.hibernate.validator.constraints.Length;"
    out.println "import java.io.Serializable;"
	out.println "import lombok.Data;"

    Set types = new HashSet()

    fields.each() {
        types.add(it.type)
    }

    if (types.contains("Date")) {
        out.println "import java.util.Date;"
    }

    if (types.contains("InputStream")) {
        out.println "import java.io.InputStream;"
    }
    out.println ""
    out.println "/**\n" +
            " * <>"+tableName+"表-DTO类</> \n" +
            " * @author <>liaoxinyi</> \n" +
            " * @date ["+ new SimpleDateFormat("yyyy-MM-dd").format(new Date()) + "] \n" +
            " * @since <>V1.0</> \n" +
            " */"
    out.println ""
    out.println "@Data"
	out.println "@ApiModel()"
    out.println "public class $className  implements Serializable {"
    out.println ""


    fields.each() {
        out.println ""
        // 输出注释
        if (isNotEmpty(it.commoent)) {
            commoentStr=it.commoent.toString()
        }else{
            commoentStr="XXX"
        }
        out.println "\t/**"
        out.println "\t * ${commoentStr}"
        out.println "\t */"
        //输出swagger内容和校验内容
            if ("Boolean".equals(it.type)){
                out.println "\t@ApiModelProperty(value = \"${commoentStr}\", required = true, example = \"XXX\")"
                out.println "\t@NotNull(message = \"${it.name}不能为空\")"
            }
            if ("String".equals(it.type)){
                out.println "\t@ApiModelProperty(value = \"${commoentStr}\", required = true,dataType = "string",allowMultiple = false, example = \"XXX\")"
                out.println "\t@NotBlank(message = \"${it.name}不能为空\")"
                out.println "\t@Size(min = 1,max = 255)"
            }            
            if ("Long".equals(it.type) || "Double".equals(it.type)){
                out.println "\t@ApiModelProperty(value = \"${commoentStr}\", required = true,dataType = "int",allowMultiple = false, example = \"1\")"
                out.println "\t@Min(0)"
                out.println "\t@Max(Integer.MAX_VALUE)"
            }
            if ("Timestamp".equals(it.type)){
                out.println "\t@NotNull(message = \"${it.name}不能为空\")"
            }            
            out.println "\t@ApiModelProperty(value = \"${commoentStr}\",required = true,allowMultiple = false, example = \"XXX\")"
        
        //if (it.annos != "") out.println "${it.annos}"

        // 输出成员变量
        out.println "\tprivate ${it.type} ${it.name};"
    }
    
        //tostring方法
    out.println ""
    out.println "\t/**"
    out.println "\t * 重写tostring方法为alibaba格式的json"
    out.println "\t */"
     out.println "\tpublic String toString(){"
     out.println "\treturn JSON.toJSONString(this);"
     out.println "}"
    out.println ""
    out.println "}"
} 

def calcFields(table) {
    DasUtil.getColumns(table).reduce([]) { fields, col ->
        def spec = Case.LOWER.apply(col.getDataType().getSpecification())

        def typeStr = typeMapping.find { p, t -> p.matcher(spec).find() }.value
        def comm =[
                colName : col.getName(),
                name :  javaName(col.getName(), false),
                type : typeStr,
                commoent: col.getComment(),
                annos: "\t@Column(name = \""+col.getName()+"\" )"]
        if("pk".equals(Case.LOWER.apply(col.getName().substring(0,2)))){
            comm.annos ="\t@Id"
            comm.name ="id"
        }
        fields += [comm]
    }
}

// 处理类名（这里是因为我的表都是以tb_命名的，所以需要处理去掉生成类名时的开头的T，
// 如果你不需要那么请查找用到了 javaClassName 这个方法的地方修改为 javaName 即可）
def javaClassName(str, capitalize) {
    def s = com.intellij.psi.codeStyle.NameUtil.splitNameIntoWords(str)
            .collect { Case.LOWER.apply(it).capitalize() }
            .join("")
            .replaceAll(/[^\p{javaJavaIdentifierPart}[_]]/, "_")
    if("tb".equals(s.substring(0,2))){
        // 去除开头的tb
        s = s.substring(2)
    }
    if("t".equals(s.substring(0,1))){
        // 去除开头的t
        s = s.substring(1)
    }
    capitalize || s.length() == 1? s : Case.LOWER.apply(s[0]) + s[1..-1]
}

def javaName(str, capitalize) {
//    def s = str.split(/(?<=[^\p{IsLetter}])/).collect { Case.LOWER.apply(it).capitalize() }
//            .join("").replaceAll(/[^\p{javaJavaIdentifierPart}]/, "_")
//    capitalize || s.length() == 1? s : Case.LOWER.apply(s[0]) + s[1..-1]
    def s = com.intellij.psi.codeStyle.NameUtil.splitNameIntoWords(str)
            .collect { Case.LOWER.apply(it).capitalize() }
            .join("")
            .replaceAll(/[^\p{javaJavaIdentifierPart}[_]]/, "_")
    capitalize || s.length() == 1? s : Case.LOWER.apply(s[0]) + s[1..-1]
}

def isNotEmpty(content) {
    return content != null && content.toString().trim().length() > 0
}

static String changeStyle(String str, boolean toCamel){
    if(!str || str.size() <= 1)
        return str

    if(toCamel){
        String r = str.toLowerCase().split('_').collect{cc -> Case.LOWER.apply(cc).capitalize()}.join('')
        return r[0].toLowerCase() + r[1..-1]
    }else{
        str = str[0].toLowerCase() + str[1..-1]
        return str.collect{cc -> ((char)cc).isUpperCase() ? '_' + cc.toLowerCase() : cc}.join('')
    }
}

static String genSerialID()
{
    return "\tprivate static final long serialVersionUID =  "+Math.abs(new Random().nextLong())+"L;"
}
```

#### Entity_JPA.groovy
内容如下： 

```groovy
import com.intellij.database.model.DasTable
import com.intellij.database.model.ObjectKind
import com.intellij.database.util.Case
import com.intellij.database.util.DasUtil
import java.io.*
import java.text.SimpleDateFormat

/*
 * Available context bindings:
 *   SELECTION   Iterable<DasObject>
 *   PROJECT     project
 *   FILES       files helper
 */
packageName = ""
typeMapping = [
        (~/(?i)tinyint|smallint|mediumint/)      : "Integer",
        (~/(?i)int/)                             : "Long",
        (~/(?i)bool|bit/)                        : "Boolean",
        (~/(?i)float|double|decimal|real/)       : "Double",
        (~/(?i)datetime|timestamp|date|time/)    : "Timestamp",
        (~/(?i)blob|binary|bfile|clob|raw|image/): "InputStream",
        (~/(?i)/)                                : "String"
]

FILES.chooseDirectoryAndSave("Choose directory", "Choose where to store generated files") { dir ->
    SELECTION.filter { it instanceof DasTable && it.getKind() == ObjectKind.TABLE }.each { generate(it, dir) }
}

def generate(table, dir) {
    //根据表名生成类名
    def className = javaClassName(table.getName(), true)
    //def className = javaName(table.getName(), true)
    def fields = calcFields(table)
    packageName = getPackageName(dir)
    PrintWriter printWriter = new PrintWriter(new OutputStreamWriter(new FileOutputStream(new File(dir, className + ".java")), "UTF-8"))
    printWriter.withPrintWriter {out -> generate(out, className, fields,table)}
//    new File(dir, className + ".java").withPrintWriter { out -> generate(out, className, fields,table) }
}

// 获取包所在文件夹路径
def getPackageName(dir) {
    return dir.toString().replaceAll("\\\\", ".").replaceAll("/", ".").replaceAll("^.*src(\\.main\\.java\\.)?", "") + ";"
}

def generate(out, className, fields,table) {
    def tableName = table.getName()
    out.println "package $packageName"
    out.println ""
    out.println "import com.alibaba.fastjson.JSON;"
    out.println "import java.io.Serializable;"
    out.println "import javax.persistence.Id;"
    out.println "import javax.persistence.SequenceGenerator;"
    out.println "import javax.persistence.GeneratedValue;"
    out.println "import javax.persistence.GenerationType;"
    out.println "import java.sql.Timestamp;"
	out.println "import lombok.Data;"
				
    Set types = new HashSet()

    fields.each() {
        types.add(it.type)
    }

    if (types.contains("Date")) {
        out.println "import java.util.Date;"
    }

    if (types.contains("InputStream")) {
        out.println "import java.io.InputStream;"
    }
    out.println ""
    out.println "/**\n" +
            " * <>"+tableName+"表-mybatis-entity类</> \n" +
            " * @author <>liaoxinyi</> \n" +
            " * @date ["+ new SimpleDateFormat("yyyy-MM-dd").format(new Date()) + "] \n" +
            " * @since <>V1.0</> \n" +
            " */"
    out.println ""
    out.println "@Data"
    out.println "public class $className  implements Serializable {"
    out.println ""
    out.println "\t/**"
    out.println "\t * 序列化ID"
    out.println "\t */"
    out.println genSerialID()


    fields.each() {
        out.println ""
        // 输出注释
        if (isNotEmpty(it.commoent)) {
            out.println "\t/**"
            out.println "\t * ${it.commoent.toString()}"
            out.println "\t */"
        }

        //if (it.annos != "") out.println "${it.annos}"

        // 输出成员变量
        out.println "\tprivate ${it.type} ${it.name};"
    }

    //tostring方法
    out.println ""
    out.println "\t/**"
    out.println "\t * 重写toString方法为alibaba的Json字符串格式"
    out.println "\t */"
     out.println "\tpublic String toString(){"
     out.println "\treturn JSON.toJSONString(this);"
     out.println "}"

    out.println ""
    out.println "}"
}

def calcFields(table) {
    DasUtil.getColumns(table).reduce([]) { fields, col ->
        def spec = Case.LOWER.apply(col.getDataType().getSpecification())

        def typeStr = typeMapping.find { p, t -> p.matcher(spec).find() }.value
        def comm =[
                colName : col.getName(),
                name :  javaName(col.getName(), false),
                type : typeStr,
                commoent: col.getComment(),
                annos: "\t@Column(name = \""+col.getName()+"\" )"]
        if(Case.LOWER.apply(col.getName()).contains("pk")){
        comm.annos ="\t@Id\n\t@SequenceGenerator(name = \""+table.getName()+"\", sequenceName = \""+table.getName()+"_seq\", allocationSize = 1)\n\t@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = \""+table.getName()+"\")\n\t@Column(name = \""+col.getName()+"\" )"
            comm.name ="id"
        }  
        fields += [comm]
    }
}

// 处理类名（这里是因为我的表都是以tb_命名的，所以需要处理去掉生成类名时的开头的T，
// 如果你不需要那么请查找用到了 javaClassName 这个方法的地方修改为 javaName 即可）
def javaClassName(str, capitalize) {
    def s = com.intellij.psi.codeStyle.NameUtil.splitNameIntoWords(str)
            .collect { Case.LOWER.apply(it).capitalize() }
            .join("")
            .replaceAll(/[^\p{javaJavaIdentifierPart}[_]]/, "_")
    if("tb".equals(s.substring(0,2))){
        // 去除开头的tb
        s = s.substring(2)
    }
    if("t".equals(s.substring(0,1))){
        // 去除开头的t
        s = s.substring(1)
    }
    capitalize || s.length() == 1? s : Case.LOWER.apply(s[0]) + s[1..-1]
}

def javaName(str, capitalize) {
//    def s = str.split(/(?<=[^\p{IsLetter}])/).collect { Case.LOWER.apply(it).capitalize() }
//            .join("").replaceAll(/[^\p{javaJavaIdentifierPart}]/, "_")
//    capitalize || s.length() == 1? s : Case.LOWER.apply(s[0]) + s[1..-1]
    def s = com.intellij.psi.codeStyle.NameUtil.splitNameIntoWords(str)
            .collect { Case.LOWER.apply(it).capitalize() }
            .join("")
            .replaceAll(/[^\p{javaJavaIdentifierPart}[_]]/, "_")
    capitalize || s.length() == 1? s : Case.LOWER.apply(s[0]) + s[1..-1]
}

def isNotEmpty(content) {
    return content != null && content.toString().trim().length() > 0
}

static String changeStyle(String str, boolean toCamel){
    if(!str || str.size() <= 1)
        return str

    if(toCamel){
        String r = str.toLowerCase().split('_').collect{cc -> Case.LOWER.apply(cc).capitalize()}.join('')
        return r[0].toLowerCase() + r[1..-1]
    }else{
        str = str[0].toLowerCase() + str[1..-1]
        return str.collect{cc -> ((char)cc).isUpperCase() ? '_' + cc.toLowerCase() : cc}.join('')
    }
}

static String genSerialID()
{
    return "\tprivate static final long serialVersionUID =  "+Math.abs(new Random().nextLong())+"L;"
}
```

#### VO.groovy
内容如下： 
```groovy
import com.intellij.database.model.DasTable
import com.intellij.database.model.ObjectKind
import com.intellij.database.util.Case
import com.intellij.database.util.DasUtil
import java.io.*
import java.text.SimpleDateFormat

/*
 * Available context bindings:
 *   SELECTION   Iterable<DasObject>
 *   PROJECT     project
 *   FILES       files helper
 */
packageName = ""
typeMapping = [
        (~/(?i)tinyint|smallint|mediumint/)      : "Integer",
        (~/(?i)int/)                             : "Long",
        (~/(?i)bool|bit/)                        : "Boolean",
        (~/(?i)float|double|decimal|real/)       : "Double",
        (~/(?i)datetime|timestamp|date|time/)    : "Timestamp",
        (~/(?i)blob|binary|bfile|clob|raw|image/): "InputStream",
        (~/(?i)/)                                : "String"
]

FILES.chooseDirectoryAndSave("Choose directory", "Choose where to store generated files") { dir ->
    SELECTION.filter { it instanceof DasTable && it.getKind() == ObjectKind.TABLE }.each { generate(it, dir) }
}

def generate(table, dir) {
    //根据表名生成类名
    def className = javaClassName(table.getName(), true)+"VO"
    //def className = javaName(table.getName(), true)
    def fields = calcFields(table)
    packageName = getPackageName(dir)
    PrintWriter printWriter = new PrintWriter(new OutputStreamWriter(new FileOutputStream(new File(dir, className + ".java")), "UTF-8"))
    printWriter.withPrintWriter {out -> generate(out, className, fields,table)}
//    new File(dir, className + ".java").withPrintWriter { out -> generate(out, className, fields,table) }
}

// 获取包所在文件夹路径
def getPackageName(dir) {
    return dir.toString().replaceAll("\\\\", ".").replaceAll("/", ".").replaceAll("^.*src(\\.main\\.java\\.)?", "") + ";"
}

def generate(out, className, fields,table) {
    def tableName = table.getName()
    out.println "package $packageName"
    out.println ""
    out.println "import io.swagger.annotations.ApiModel;"
    out.println "import io.swagger.annotations.ApiModelProperty;"
    out.println "import javax.validation.constraints.NotNull;"  
    out.println "import javax.validation.constraints.Max;"
    out.println "import javax.validation.constraints.Min;"
    out.println "import javax.validation.constraints.NotBlank;"
    out.println "import org.hibernate.validator.constraints.Length;"
    out.println "import java.io.Serializable;"
	out.println "import lombok.Data;"
				
    Set types = new HashSet()

    fields.each() {
        types.add(it.type)
    }

    if (types.contains("Date")) {
        out.println "import java.util.Date;"
    }

    if (types.contains("InputStream")) {
        out.println "import java.io.InputStream;"
    }
    out.println ""
    out.println "/**\n" +
            " * <>"+tableName+"表-VO类</> \n" +
            " * @author <>liaoxinyi</> \n" +
            " * @date ["+ new SimpleDateFormat("yyyy-MM-dd").format(new Date()) + "] \n" +
            " * @since <>V1.0</> \n" +
            " */"
    out.println ""
    out.println "@Data"
	out.println "@ApiModel()"
    out.println "public class $className  implements Serializable {"
    out.println ""


    fields.each() {
        out.println ""
        // 输出注释
        if (isNotEmpty(it.commoent)) {
            commoentStr=it.commoent.toString()
        }else{
            commoentStr="XXX"
        }
            out.println "\t/**"
            out.println "\t * ${commoentStr}"
            out.println "\t */"
            out.println "\t@ApiModelProperty(value = \"${commoentStr}\",example = \"XXX\")"
        if ("Timestamp".equals(it.type)){
                it.type="String"
        }

        if ("Long".equals(it.type)){
                    it.type="Integer"
        }    

        // 输出成员变量
        out.println "\tprivate ${it.type} ${it.name};"
    }
    
        //tostring方法
    out.println ""
    out.println "\t/**"
    out.println "\t * 重写tostring方法为alibaba格式的json"
    out.println "\t */"
     out.println "\tpublic String toString(){"
     out.println "\treturn JSON.toJSONString(this);"
     out.println "}"

    out.println ""
    out.println "}"
}

def calcFields(table) {
    DasUtil.getColumns(table).reduce([]) { fields, col ->
        def spec = Case.LOWER.apply(col.getDataType().getSpecification())

        def typeStr = typeMapping.find { p, t -> p.matcher(spec).find() }.value
        def comm =[
                colName : col.getName(),
                name :  javaName(col.getName(), false),
                type : typeStr,
                commoent: col.getComment(),
                annos: "\t@Column(name = \""+col.getName()+"\" )"]
        if("pk".equals(Case.LOWER.apply(col.getName().substring(0,2)))){
            comm.annos ="\t@Id"
            comm.name ="id"
        }
        fields += [comm]
    }
}

// 处理类名（这里是因为我的表都是以tb_命名的，所以需要处理去掉生成类名时的开头的T，
// 如果你不需要那么请查找用到了 javaClassName 这个方法的地方修改为 javaName 即可）
def javaClassName(str, capitalize) {
    def s = com.intellij.psi.codeStyle.NameUtil.splitNameIntoWords(str)
            .collect { Case.LOWER.apply(it).capitalize() }
            .join("")
            .replaceAll(/[^\p{javaJavaIdentifierPart}[_]]/, "_")
    if("tb".equals(s.substring(0,2))){
        // 去除开头的tb
        s = s.substring(2)
    }
    if("t".equals(s.substring(0,1))){
        // 去除开头的t
        s = s.substring(1)
    }
    capitalize || s.length() == 1? s : Case.LOWER.apply(s[0]) + s[1..-1]
}

def javaName(str, capitalize) {
//    def s = str.split(/(?<=[^\p{IsLetter}])/).collect { Case.LOWER.apply(it).capitalize() }
//            .join("").replaceAll(/[^\p{javaJavaIdentifierPart}]/, "_")
//    capitalize || s.length() == 1? s : Case.LOWER.apply(s[0]) + s[1..-1]
    def s = com.intellij.psi.codeStyle.NameUtil.splitNameIntoWords(str)
            .collect { Case.LOWER.apply(it).capitalize() }
            .join("")
            .replaceAll(/[^\p{javaJavaIdentifierPart}[_]]/, "_")
    capitalize || s.length() == 1? s : Case.LOWER.apply(s[0]) + s[1..-1]
}

def isNotEmpty(content) {
    return content != null && content.toString().trim().length() > 0
}

static String changeStyle(String str, boolean toCamel){
    if(!str || str.size() <= 1)
        return str

    if(toCamel){
        String r = str.toLowerCase().split('_').collect{cc -> Case.LOWER.apply(cc).capitalize()}.join('')
        return r[0].toLowerCase() + r[1..-1]
    }else{
        str = str[0].toLowerCase() + str[1..-1]
        return str.collect{cc -> ((char)cc).isUpperCase() ? '_' + cc.toLowerCase() : cc}.join('')
    }
}

static String genSerialID()
{
    return "\tprivate static final long serialVersionUID =  "+Math.abs(new Random().nextLong())+"L;"
}
```

#### Entity_JPA.groovy
内容如下： 

```groovy
import com.intellij.database.model.DasTable
import com.intellij.database.model.ObjectKind
import com.intellij.database.util.Case
import com.intellij.database.util.DasUtil
import java.io.*
import java.text.SimpleDateFormat

/*
 * Available context bindings:
 *   SELECTION   Iterable<DasObject>
 *   PROJECT     project
 *   FILES       files helper
 */
packageName = ""
typeMapping = [
        (~/(?i)tinyint|smallint|mediumint/)      : "Integer",
        (~/(?i)int/)                             : "Long",
        (~/(?i)bool|bit/)                        : "Boolean",
        (~/(?i)float|double|decimal|real/)       : "Double",
        (~/(?i)datetime|timestamp|date|time/)    : "Timestamp",
        (~/(?i)blob|binary|bfile|clob|raw|image/): "InputStream",
        (~/(?i)/)                                : "String"
]

FILES.chooseDirectoryAndSave("Choose directory", "Choose where to store generated files") { dir ->
    SELECTION.filter { it instanceof DasTable && it.getKind() == ObjectKind.TABLE }.each { generate(it, dir) }
}

def generate(table, dir) {
    //根据表名生成类名
    def className = javaClassName(table.getName(), true)
    //def className = javaName(table.getName(), true)
    def fields = calcFields(table)
    packageName = getPackageName(dir)
    PrintWriter printWriter = new PrintWriter(new OutputStreamWriter(new FileOutputStream(new File(dir, className + ".java")), "UTF-8"))
    printWriter.withPrintWriter {out -> generate(out, className, fields,table)}
//    new File(dir, className + ".java").withPrintWriter { out -> generate(out, className, fields,table) }
}

// 获取包所在文件夹路径
def getPackageName(dir) {
    return dir.toString().replaceAll("\\\\", ".").replaceAll("/", ".").replaceAll("^.*src(\\.main\\.java\\.)?", "") + ";"
}

def generate(out, className, fields,table) {
    def tableName = table.getName()
    out.println "package $packageName"
    out.println ""
    out.println "import javax.persistence.Column;"
    out.println "import javax.persistence.Entity;"
    out.println "import javax.persistence.Table;"
    out.println "import javax.persistence.GenerationType;"
    out.println "import java.sql.Timestamp;"
    out.println "import java.io.Serializable;"
    out.println "import javax.persistence.Id;"
    out.println "import javax.persistence.SequenceGenerator;"
    out.println "import javax.persistence.GeneratedValue;"
	out.println "import lombok.Data;"
				
    Set types = new HashSet()

    fields.each() {
        types.add(it.type)
    }

    if (types.contains("Date")) {
        out.println "import java.util.Date;"
    }

    if (types.contains("InputStream")) {
        out.println "import java.io.InputStream;"
    }
    out.println ""
    out.println "/**\n" +
            " * <>"+tableName+"表-JPA-entity类</> \n" +
            " * @author <>liaoxinyi</> \n" +
            " * @date ["+ new SimpleDateFormat("yyyy-MM-dd").format(new Date()) + "] \n" +
            " * @since <>V1.0</> \n" +
            " */"
    out.println ""
    out.println "@Data"
	out.println "@Entity"
    out.println "@Table ( name =\""+table.getName() +"\" )"
    out.println "public class $className  implements Serializable {"
    out.println ""
    out.println "\t/**"
    out.println "\t * 序列化ID"
    out.println "\t */"
    out.println genSerialID()

    fields.each() {
        out.println ""
        // 输出注释
        if (isNotEmpty(it.commoent)) {
            out.println "\t/**"
            out.println "\t * ${it.commoent.toString()}"
            out.println "\t */"
        }

        if (it.annos != "") out.println "${it.annos}"

        // 输出成员变量
        out.println "\tprivate ${it.type} ${it.name};"
    }

    out.println ""
    out.println "}"
}

def calcFields(table) {
    DasUtil.getColumns(table).reduce([]) { fields, col ->
        def spec = Case.LOWER.apply(col.getDataType().getSpecification())

        def typeStr = typeMapping.find { p, t -> p.matcher(spec).find() }.value
        def comm =[
                colName : col.getName(),
                name :  javaName(col.getName(), false),
                type : typeStr,
                commoent: col.getComment(),
                annos: "\t@Column(name = \""+col.getName()+"\" )"]
        if(Case.LOWER.apply(col.getName()).contains("pk")){
        comm.annos ="\t@Id\n\t@SequenceGenerator(name = \""+table.getName()+"\", sequenceName = \""+table.getName()+"_seq\", allocationSize = 1)\n\t@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = \""+table.getName()+"\")\n\t@Column(name = \""+col.getName()+"\" )"
            comm.name ="id"
        }  
        fields += [comm]
    }
}

// 处理类名（这里是因为我的表都是以tb_命名的，所以需要处理去掉生成类名时的开头的T，
// 如果你不需要那么请查找用到了 javaClassName 这个方法的地方修改为 javaName 即可）
def javaClassName(str, capitalize) {
    def s = com.intellij.psi.codeStyle.NameUtil.splitNameIntoWords(str)
            .collect { Case.LOWER.apply(it).capitalize() }
            .join("")
            .replaceAll(/[^\p{javaJavaIdentifierPart}[_]]/, "_")
    if("tb".equals(s.substring(0,2))){
        // 去除开头的tb
        s = s.substring(2)
    }
    if("t".equals(s.substring(0,1))){
        // 去除开头的t
        s = s.substring(1)
    }
    capitalize || s.length() == 1? s : Case.LOWER.apply(s[0]) + s[1..-1]
}

def javaName(str, capitalize) {
//    def s = str.split(/(?<=[^\p{IsLetter}])/).collect { Case.LOWER.apply(it).capitalize() }
//            .join("").replaceAll(/[^\p{javaJavaIdentifierPart}]/, "_")
//    capitalize || s.length() == 1? s : Case.LOWER.apply(s[0]) + s[1..-1]
    def s = com.intellij.psi.codeStyle.NameUtil.splitNameIntoWords(str)
            .collect { Case.LOWER.apply(it).capitalize() }
            .join("")
            .replaceAll(/[^\p{javaJavaIdentifierPart}[_]]/, "_")
    capitalize || s.length() == 1? s : Case.LOWER.apply(s[0]) + s[1..-1]
}

def isNotEmpty(content) {
    return content != null && content.toString().trim().length() > 0
}

static String changeStyle(String str, boolean toCamel){
    if(!str || str.size() <= 1)
        return str

    if(toCamel){
        String r = str.toLowerCase().split('_').collect{cc -> Case.LOWER.apply(cc).capitalize()}.join('')
        return r[0].toLowerCase() + r[1..-1]
    }else{
        str = str[0].toLowerCase() + str[1..-1]
        return str.collect{cc -> ((char)cc).isUpperCase() ? '_' + cc.toLowerCase() : cc}.join('')
    }
}

static String genSerialID()
{
    return "\tprivate static final long serialVersionUID =  "+Math.abs(new Random().nextLong())+"L;"
}
```

### 代码风格
#### StyleByLXY.xml

内容如下： 
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

#### GoogleStyle.xml
在Idea仓库下载
### 总结
- 工具只是工具，最重要的还是能力和习惯。
- 最为便捷的方式是，将设置好的IDEA配置，直接导出，方便以后直接配置。