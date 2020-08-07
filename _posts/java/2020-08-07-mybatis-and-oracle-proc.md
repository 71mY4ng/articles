
# mybatis 和 ORACLE PROCEDURE

更多详细用法查看此repo：

[demo-oracle-mybatis](https://github.com/71mY4ng/demo-oracle-mybatis.git)

## oracle 存储过程的声明：

```sql
-- 存储过程签名， 没有返回值， 两个参数
create or replace procedure exec_proc_1 (today in varchar2, nextday in varchar2)

-- 两个参数， 1个返回值
create or replace procedure get_proc_1 (today in varchar2, nextday in varchar2, summary out number)

-- 两个参数， 1个返回值是 sys_cursor 的游标结果集
create or replace procedure get_proc_2 (today in varchar2, nextday in varchar2, list out sys_cursor)
```

__mybatis 对第一种的定义：__

## mapper 接口：

```java
public void executeProc1(@Param("today") String today,
                        @Param("nextday") String nextDay);
```

## mapper xml 定义：

```xml
<update id="executeProc1" statementType="CALLABLE">
    CALL executeProc1(#{today}, #{nextday})
</update>
```

__mybatis 对第二种的定义：__

mapper 接口方法定义，可见即时有存储过程有返回值也是通过参数来拿，java方法还是定义成返回void

```java
void getProcSummaryOut(@Param("today") String today,
                        @Param("nextday") String lastName,
                        @Param("summary") Map<String, Double> summary);
```

mapper xml 定义， 指定各参数的类型，有如下几种：
* IN
* OUT
* INOUT
具体使用选择看需求

```xml
<update id="getProcSummaryOut" statementType="CALLABLE">
    {CALL exec_proc_out(#{today, jdbcType=VARCHAR, mode=IN},
                        #{nextday, jdbcType=VARCHAR, mode=IN},
                        #{summary.sumpoints, jdbcType=NUMBER, mode=OUT})}
</update>
```


__mybatis 对第三种的定义：__

```xml
<resultMap id="procMapping">
...
</resultMap>

<select id="getProcCursor" resultMap="procMapping">
    {CALL get_proc_2(#{getClientsQuery.departmentId, jdbcType=NUMERIC, mode=IN},
                     #{getClientsQuery.employees, jdbcType=CURSOR, mode=OUT, javaType=java.sql.ResultSet, resultMap=getEmployeesResultSet})}
</select>
```
