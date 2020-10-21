---
title: mybatis动态SQL整理
date: 2020-10-21 21:59:41
tags: mybatis
categories: java
---

{%note info%}

本文近记录常用的动态sql

{%endnote%}
<!--more-->

- 表达式规则基于OGNL(（Object Graph Navigation Language 对象图导航语言)
- if
    ``` 
    <if test="id!=null">
     	id=#{id}
     </if>
    ```
- choose
    - 类似switch case，只会进入一个逻辑分支
    ``` 
     <choose>
     	<when test="id!=null">
     		id=#{id}
     	</when>
     	<otherwise>
     		gender = 0
     	</otherwise>
     </choose>
    ```
- trim
    -  prefix="":
        - 前缀：trim标签体中是整个字符串拼串后的结果。prefix给拼串后的整个字符串加一个前缀
    - prefixOverrides=""
        - 前缀覆盖：去掉整个字符串前面多余的字符
    - suffix=""
        - 后缀suffix给拼串后的整个字符串加一个后缀 
    - suffixOverrides=""
        - 后缀覆盖：去掉整个字符串后面多余的字符
``` 
 <trim prefix="where" suffixOverrides="and">
 	<if test="id!=null">
 		id=#{id} and
 	</if>
 	<if test="lastName!=null &amp;&amp; lastName!=&quot;&quot;">
 		last_name like #{lastName} and
 	</if>
 	<!-- ognl会进行字符串与数字的转换判断  "0"==0 -->
 	<if test="gender==0 or gender==1">
 	 	gender=#{gender}
 	</if>
 </trim>
```

- foreach
    - collection
        - 指定要遍历的集合：list类型的参数会特殊处理封装在map中，map的key就叫list
    - item
        - 将当前遍历出的元素赋值给指定的变量
    - separator
        - 每个元素之间的分隔符
    - open
        - 遍历出所有结果拼接一个开始的字符
    - close
        - 遍历出所有结果拼接一个结束的字符
    - index
        - 索引。遍历list的时候是index就是索引，item就是当前值。      遍历map的时候index表示的就是map的key，item就是map的值
    ``` 
    <foreach collection="emps" item="emp" separator=",">
    	(#{emp.lastName},#{emp.email},#{emp.gender},#{emp.dept.id})
    </foreach>
    ```
- where
    - 可以自动抹除多余的and
    ``` 
     <where>
     	<if test="lastName!=null &amp;&amp; lastName!=&quot;&quot;">
     		and last_name like #{lastName}
     	</if>
     	<if test="gender==0 or gender==1">
     	 	and gender=#{gender}
     	</if>
     </where>
    ```
- set
     - 可以自动抹除多余的“,”
     - 也可通过trim实现
    ``` 
    <set>
    	<if test="lastName!=null">
    		last_name=#{lastName},
    	</if>
    	<if test="email!=null">
    		email=#{email},
    	</if>
    	<if test="gender!=null">
    		gender=#{gender}
    	</if>
    </set>
    ```
- include
    - 抽取可重用的sql片段。方便后面引用 
    1. sql抽取：经常将要查询的列名，或者插入用的列名抽取出来方便引用
    2. include来引用已经抽取的sql：
    3. include还可以自定义一些property，sql标签内部就能使用自定义的属性
        - include-property：取值的正确方式${prop},
        - #{不能使用这种方式}
    ``` 
    <sql id="Base_Column_List" >
        id, data_name, table_name
    </sql>
    ```
- bind
    - 可以将OGNL表达式的值绑定到一个变量中，方便后来引用这个变量的值
    ``` 
    bind name="_lastName" value="'%'+lastName+'%'"/>
    ```


---

### 批量保存
- MySQL
    - foreach遍历，mysql支持values(),(),()语法
        ``` 
        <insert id="addEmps">
         	insert into tbl_employee(
         		<include refid="insertColumn"></include>
         	) 
        	values
        	<foreach collection="emps" item="emp" separator=",">
        		(#{emp.lastName},#{emp.email},#{emp.gender},#{emp.dept.id})
        	</foreach>
         </insert>
        ```
    - 多insert拼接
        - ==这种方式需要数据库连接属性allowMultiQueries=true，这种分号分隔多个sql可以用于其他的批量操作（删除，修改）==
        ``` 
         <foreach collection="emps" item="emp" separator=";">
         	insert into tbl_employee(last_name,email,gender,d_id)
         	values(#{emp.lastName},#{emp.email},#{emp.gender},#{emp.dept.id})
         </foreach>
        ```
- Oracle
    - Oracle不支持values(),(),()
    1. 多个insert放在begin - end里面
        ``` 
         begin
            insert into employees(employee_id,last_name,email) 
            values(employees_seq.nextval,'test_001','test_001@atguigu.com');
            insert into employees(employee_id,last_name,email) 
            values(employees_seq.nextval,'test_002','test_002@atguigu.com');
        end;
        ```
    2. 利用中间表
        ``` 
        insert into employees(employee_id,last_name,email)
           select employees_seq.nextval,lastName,email from(
                  select 'test_a_01' lastName,'test_a_e01' email from dual
                  union
                  select 'test_a_02' lastName,'test_a_e02' email from dual
                  union
                  select 'test_a_03' lastName,'test_a_e03' email from dual
           )	
        ```

---

### 内置参数
- 不只是方法传递过来的参数
- 可以被用来判断，取值
- _parameter（代表整个参数）
    - 单个参数：_parameter就是这个参数
    - 多个参数：参数会被封装为一个map；_parameter就是代表这个map

- _databaseId（如果配置了databaseIdProvider标签）
    - _databaseId就是代表当前数据库的别名oracle