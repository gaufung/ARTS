---
date: 2017-07-07 22:00
status: public
title: 'Pda project using java'
---

# 1 Overview
As for a fresh man learning java language, the best practice of learning a specific programming language is to develop an application using it. The Pda is a statictical model about energy comsumption. industry revenue and carbon dioxide emission. This model was developed by [Python language](https://github.com/gaufung/LMDI). There are some pros and cons of Python completaton.
**Pros**

+ Python can build prototype immediatly.
+ Python is armed with countless packages which are sovle every possible situations.  

**Cons**

+ Python is dynamic and weak type language, that brings out difficulties of refactoring.
+ As its relaying on packages, it takes time to share this application with others, especially when he or she hasn't installed Python environments.

Java is the best substitution of python. The open source code can be refered [here](https://github.com/gaufung/PDA).

# 2 Maven
Maven is a widely used java build tool with features like compile, test, package, denpendencies management and so on. 
## 2.1 Maven Jdk setting
Maven chooses jdk 1.6 as default, and it doesn't meet our requirements. So we add some configurations in `pom.xml` file.
```XML
   <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
```
## 2.2 Denpendency
Maven set free us from manual management of application libraray jar file. If some jars also have other denpendencies, it sounds extradinary terriable. We just add dependencies in `pom.xml`.
```XML
<dependencies>
    <dependcy>
       <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
    </dependcy>
</dependencies>
```
Maven can download the dependencies and those dependencies from local repostitory, central repository and remote repository.
## 2.3 Add jar to repository
If your need a specific dependency which does not upload to central repository, you can add it to local repository as follows:
```Shell
mvn install:install-file
   -Dfile=<path-to-file>
   -DgroupId=<group-id>
   -DartifactId=<artifact-id>
   -Dversion=<version>
   -Dpackaging=<packaging>
   -DgeneratePom=true

Where: <path-to-file>  the path to the file to load
   <group-id>      the group that the file should be registered under
   <artifact-id>   the artifact name for the file
   <version>       the version of the file
   <packaging>     the packaging of the file e.g. jar
```
# 3 Hibernate
Hibernate is very popular ORM framework.  To fetch data effectively, we take approach of getting data from database but not excel file.
## 3.1 Mysql database
```
./mysql -u root -p
#password: 
create database industry;
use industry;
create table production(
    id int(4) not null primary key auto_increment,
    province varchar(10) not null,
    year char(4) not null,
    prod double(20,8) default 0.00
)default charset=utf8;
```
## 3.2 Hibernate class
```Java
class ProductionDb implements Serializable {
    private int id;
    private String province;
    private String year;
    private double prod;
    public ProductionDb(){}
    public ProductionDb(int id,String province, String year, double prod){
        this.id = id;this.province = province;
        this.year = year;this.prod = prod;}
    public int getId(){return id;}
    public String getProvince(){return province;}
    public String getYear(){return year;}
    public double getProd(){return prod;}
    public void setId(int id){this.id=id;}
    public void setProvince(String province){this.province=province;}
    public void setYear(String year){this.year = year;}
    public void setProd(double prod){this.prod=prod;}
}
```
## 3.3 Hibernate Configuration
Mapping configuration
```xml
<?xml version="1.0"?>
<!DOCTYPE hibernate-mapping PUBLIC "-//Hibernate/Hibernate Mapping DTD 3.0//EN"
        "http://hibernate.sourceforge.net/hibernate-mapping-3.0.dtd">
<hibernate-mapping>
    <class name="org.iecas.pda.io.db.ProductionDb" table="production">
        <id name="id" type="int" column="id">
            <generator class="native"/>
        </id>
        <property name="year" type="string">
            <column name="year"/>
        </property>
        <property name="province" type="string">
            <column name="province"/>
        </property>
        <property name="prod" type="double">
            <column name="prod"/>
        </property>
    </class>
</hibernate-mapping>
```
Hibernate Configruation
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE hibernate-configuration PUBLIC "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
        "http://hibernate.sourceforge.net/hibernate-configuration-3.0.dtd">
<hibernate-configuration>
    <session-factory name="">
        <property name="hibernate.connection.driver_class">com.mysql.jdbc.Driver</property>
        <property name="hibernate.connection.password">admin</property>
        <property name="hibernate.connection.url">jdbc:mysql://localhost/industry?</property>
        <property name="hibernate.connection.username">root</property>
        <property name="hibernate.dialect"> org.hibernate.dialect.MySQLDialect</property>
        <property name="show_sql">true</property>
        <mapping resource="IndustryDb.hhm.xml" />
    </session-factory>
</hibernate-configuration>
```
Java Usage
```java
public class HibernateUtil {
    private static final SessionFactory sessionFactory = buildSessionFactory();
    private static SessionFactory buildSessionFactory() {
        try {
            // Create the SessionFactory from hibernate.cfg.xml
            return new Configuration().configure().buildSessionFactory();
        } catch (Throwable ex) {
            // Make sure you log the exception, as it might be swallowed
            System.err.println("Initial SessionFactory creation failed." + ex);
            throw new ExceptionInInitializerError(ex);
        }
    }
    public static SessionFactory getSessionFactory() {
        return sessionFactory;
    }
    public static void shutdown() {
        getSessionFactory().close();
    }
}
//
public class ProductionDbReader implements ProductionReader {
    @Override
    public List<Production> read(String year) throws Exception{
        Session session = HibernateUtil.getSessionFactory().openSession();
        session.beginTransaction();
        List<ProductionDb> produtions = session.createQuery("From ProductionDb where year=:year ORDER BY id ASC")
                .setParameter("year",year).list();
        session.getTransaction().commit();
        return produtions.stream().map(db->new Production(db.getProvince(),db.getProd())).collect(Collectors.toList());
    }
}
```
# 4 Linear Programming 
Joptimizer is choice of linear programming.
$ min \quad C^Tx $
subject to $Gx \le h,Ax=b$
```java
protected double optimize(double[] energies, double[] co2s, double[] productions, double energy,double co2, double production){
        LPOptimizationRequest or = new LPOptimizationRequest();
        double[] c = energies;
        double[][] G = new double[1][];
        G[0]=productions;
        double[] h =new double[]{production};
        double[][] A = new double[1][];
        A[0]=co2s;
        double[] b = new double[]{co2};
        double[] lb = new double[energies.length];
        Arrays.fill(lb,0.0);
        or.setC(c);or.setG(G);
        or.setH(h);or.setA(A);
        or.setB(b);or.setLb(lb);
        or.setDumpProblem(true);
        LPPrimalDualMethod opt = new LPPrimalDualMethod();
        opt.setLPOptimizationRequest(or);
        try{
            opt.optimize();
            double[] sol = opt.getOptimizationResponse().getSolution();
            return IntStream.range(0, energies.length).mapToDouble(i->sol[i]*energies[i]).sum()/energy;
        }catch (Exception e){
            return -1.0;
        }
    }
```
# 5 Log4j
Log4j is a unseparate component of a java program. There two basic steps of using log4j. 
**Configruation**
In maven projcet, create a `log4j.properties` file in `src/main/resourecs` folder.
```c
# Root logger option
log4j.rootLogger=ERROR, stdout, file

# Redirect log messages to console
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.Target=System.out
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} %-5p %c{1}:%L - %m%n

# Redirect log messages to a log file, support file rolling.
log4j.appender.file=org.apache.log4j.RollingFileAppender
log4j.appender.file.File=log4j-application.log
log4j.appender.file.MaxFileSize=5MB
log4j.appender.file.MaxBackupIndex=10
log4j.appender.file.layout=org.apache.log4j.PatternLayout
log4j.appender.file.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} %-5p %c{1}:%L - %m%n
```
**Get logger**
```java
Logger log = Logger.getLogger(MyClass.name);
PropertiesConfigurator.configure(MyClass.class.getClassLoader().getResourceAsStream("log4j.properties");
```