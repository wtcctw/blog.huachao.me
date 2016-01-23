title:  "iBatis基础"
date: 2015-04-17
updated: 2015-12-26
categories: 
- db
tags:
- db
---

> iBatis只是一个模板，真正的内容全是数据库

## Datasource，SqlMapClient，Dao
对于任何一个与数据库打交道的程序来说，DataSource(DriverManager), Driver, Connection, Statement, FetchResult, 基本就理出了脉络。如果需要处理数据库事务，那么就再加上TransactionTemplate和TransactionManager，在TransactionManager中加上Datasource基本就成了。

## Spring配置
``` xml
<bean id="dataSource" name="dataSource"
      class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://ip:3306/test?useUnicode=true&characterEncoding=UTF-8"/>
    <property name="username" value="xxx"/>
    <property name="password" value="xxx"/>
    <property name="maxActive" value="20"/>
</bean>
<!--sql map -->
<bean id="sqlMapClient"
     class="org.springframework.orm.ibatis.SqlMapClientFactoryBean">
      <property name="configLocation" value="classpath:config/sqlmap/sqlMapConfig.xml"/>
      <property name="dataSource" ref="dataSource"/>
</bean>
<!--dao-->
<bean id="studentDAO" class="me.huachao.dao.StudentDao">
    <property name="sqlMapClient" ref="sqlMapClient"/>
    <!--<property name="sqlMapTransactionTemplate" ref="sqlMapTransactionTemplate"/>-->
</bean>
```

### mvn依赖
- commons-dbcp:commons-dbcp，当然也可以用c3p0，点评现在都已经用自己写的一个zebra来代替
- mysql:mysql-connector-java
- com.ibatis:ibatis


## SqlMap
``` xml
<sqlMap namespace="Student">
    <!-- Use type aliases to avoid typing the full classname every time. -->
    <typeAlias alias="Student" type="me.huachao.entity.StudentEntity"/>
    <!-- Result maps describe the mapping between the columns returned
         from a query, and the class properties.  A result map isn't
         necessary if the columns (or aliases) match to the properties
         exactly. -->
    <resultMap id="StudentMap" class="Student">
        <result property="id" column="Id"/>
        <result property="name" column="Name"/>
        <result property="sex" column="Sex"/>
        <result property="birthday" column="Birthday"/>
        <result property="gpa" column="Gpa"/>
    </resultMap>
    <sql id="allFields">
        Id,
        Name,
        Sex,
        Birthday,
        Gpa
    </sql>
    <!-- Select with no parameters using the result map for Account class. -->
    <select id="selectAllStudents" resultClass="Student">
        select
          <include refid="allFields"/>
        from
          Student
    </select>
    <!-- A simpler select example without the result map.  Note the
         aliases to match the properties of the target result class. -->
    <select id="selectStudentById" parameterClass="int" resultMap="StudentMap">
        select
          <include refid="allFields"/>
        from
          Student
        where
          Id = #id#
    </select>
     <!--Insert example, using the Account parameter class-->
    <insert id="insertStudent" parameterClass="Student">
        insert into Student
        (
            Name,
            Sex,
            Birthday,
            Gpa
        )
        values
        (
            #name#,
            #sex#,
            #birthday#,
            #gpa#
        )
        <selectKey resultClass="int" keyProperty="id">
            SELECT @@IDENTITY
            AS id
        </selectKey>
    </insert>
     <!--Update example, using the Account parameter class-->
    <update id="updateStudent" parameterClass="Student">
        update
            Student
        set
            Name = #name#,
            Birthday = #birthday#,
            Sex = #sex#,
            Gpa = #gpa#
        where
            Id = #id#
    </update>
    <!-- Delete example, using an integer as the parameter class -->
    <delete id="deleteStudentById" parameterClass="int">
        delete from Student where Id = #id#
    </delete>
</sqlMap>
```

## Dao实现
``` java
public class StudentDao extends SqlMapClientDaoSupport {

    public List selectAll() {
        return getSqlMapClientTemplate().queryForList("selectAllStudents");
    }

    public StudentEntity selectStudentById(int id) {
        return (StudentEntity) getSqlMapClientTemplate().queryForObject("selectStudentById", id);
    }

    public int insertStudent(StudentEntity studentEntity){
        Integer id = (Integer) getSqlMapClientTemplate().insert("insertStudent", studentEntity);
        return id;
    }

    public void updateStudent(StudentEntity studentEntity){
        getSqlMapClientTemplate().update("updateStudent", studentEntity);
    }

    public void deleteStudent(int id) {
        getSqlMapClientTemplate().delete("deleteStudentById", id);
    }

}
```

## 事务（编程方式）
``` xml
<!--service-->
<bean id="studentService" class="me.huachao.service.StudentService">
</bean>

<!--transaction-->
<bean id="sqlMapTransactionManager"
      class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
<bean id="sqlMapTransactionTemplate"
      class="org.springframework.transaction.support.TransactionTemplate">
    <property name="transactionManager" ref="sqlMapTransactionManager"/>
</bean>
```
*注意：一般transaction的逻辑都是写在Service里面，Dao不管transaction*

``` java
public class StudentService {
    @Resource
    private StudentDao studentDao;

    @Resource
    private TransactionTemplate sqlMapTransactionTemplate;

    public int insertWithTransaction(final StudentEntity studentEntity){
        Integer idObj = (Integer) sqlMapTransactionTemplate.execute(new TransactionCallback(){
            public Object doInTransaction(TransactionStatus status){
                try{
                    int id = studentDao.insertStudent(studentEntity);
                    throw new SQLException();
//                    return id;
                } catch (Exception e) {
                    status.setRollbackOnly();
                    return -1;
                }
            }
        });
        return (idObj!=null)?idObj.intValue():-1;
    }
}
```

## 事务（注解方式）
- 因为@Transactional需要用到AOP，所以cglib和asm都需要加上*
- Spring配置文件需要加上 `<tx:annotation-driven transaction-manager="sqlMapTransactionManager"/>`
- 具体写code的时候，需要在方法上注明rollbackfor=xxx.class, 默认是只回滚RuntimeException和unchecked Exception的
``` java
@Transactional(rollbackFor = SQLException.class)
public int insertWithTransactionAnnotation(StudentEntity studentEntity) throws SQLException {
    int id = studentDao.insertStudent(studentEntity);
    if (id > 0)
        throw new SQLException();
    return id;
}
```

## Code
[MySampleCode@Github](https://github.com/wtcctw/ibatis-basic)