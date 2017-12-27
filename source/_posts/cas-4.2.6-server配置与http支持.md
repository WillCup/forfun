
title: cas-4.2.6-server配置与http支持
date: 2016-12-05 17:11:31
tags: [youdaonote]
---

下载
---
下载cas-server-webapp-4.2.6.war，放置到tomcat9的webapps下。

开始使用的tomcat7，结果报错：Unable to process Jar entry。查了一下是tomcat版本过低的原因。

启动tomcat，确认war正常解压，cas server正常启动。
关闭tomcat。

编辑
---
cas的主要配置文件在于deployerConfigContext.xml。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:c="http://www.springframework.org/schema/c"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:util="http://www.springframework.org/schema/util"
       xmlns:sec="http://www.springframework.org/schema/security"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd
       http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd
       http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/security http://www.springframework.org/schema/security/spring-security.xsd
       http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util.xsd">


    <util:map id="authenticationHandlersResolvers">
        <entry key-ref="proxyAuthenticationHandler" value-ref="proxyPrincipalResolver" />
        <entry key-ref="primaryAuthenticationHandler" value-ref="primaryPrincipalResolver" />
    </util:map>

    <util:list id="authenticationMetadataPopulators">
        <ref bean="successfulHandlerMetaDataPopulator" />
        <ref bean="rememberMeAuthenticationMetaDataPopulator" />
    </util:list>

    <bean id="attributeRepository" class="org.jasig.services.persondir.support.NamedStubPersonAttributeDao"
          p:backingMap-ref="attrRepoBackingMap" />


    <!-- 设置数据源 -->
   <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
      <property name="driverClassName" value="com.mysql.jdbc.Driver"></property>
      <property name="url" value="jdbc:mysql://127.0.0.1:3306/metamap1?useUnicode=true&amp;characterEncoding=utf8"></property>
      <property name="username" value="root"></property>
      <property name="password" value=""></property>
    </bean>


    <bean id="primaryAuthenticationHandler"
      class="com.will.cas.auth.WillAuthenticationHandler"
      p:dataSource-ref="dataSource"
      />


    <!-- <alias name="acceptUsersAuthenticationHandler" alias="primaryAuthenticationHandler" /> -->
    <alias name="personDirectoryPrincipalResolver" alias="primaryPrincipalResolver" />

    <util:map id="attrRepoBackingMap">
        <entry key="uid" value="uid" />
        <entry key="eduPersonAffiliation" value="eduPersonAffiliation" />
        <entry key="groupMembership" value="groupMembership" />
        <entry>
            <key><value>memberOf</value></key>
            <list>
                <value>faculty</value>
                <value>staff</value>
                <value>org</value>
            </list>
        </entry>
    </util:map>

    <alias name="serviceThemeResolver" alias="themeResolver" />

    <!-- <alias name="jsonServiceRegistryDao" alias="serviceRegistryDao" /> -->

    <!-- 注册服务 -->
     <bean id="serviceRegistryDao" class="org.jasig.cas.services.InMemoryServiceRegistryDaoImpl"
             p:registeredServices-ref="registeredServicesList" />
 
     <util:list id="registeredServicesList">
         <bean class="org.jasig.cas.services.RegexRegisteredService"
               p:id="0" p:name="HTTP and IMAP" p:description="Allows HTTP(S) and IMAP(S) protocols"
               p:serviceId="^(https?|http?|imaps?)://.*" p:evaluationOrder="10000001" />
     </util:list>


    <alias name="defaultTicketRegistry" alias="ticketRegistry" />
    
    <alias name="ticketGrantingTicketExpirationPolicy" alias="grantingTicketExpirationPolicy" />
    <alias name="multiTimeUseOrTimeoutExpirationPolicy" alias="serviceTicketExpirationPolicy" />

    <alias name="anyAuthenticationPolicy" alias="authenticationPolicy" />
    <alias name="acceptAnyAuthenticationPolicyFactory" alias="authenticationPolicyFactory" />

    <bean id="auditTrailManager"
          class="org.jasig.inspektr.audit.support.Slf4jLoggingAuditTrailManager"
          p:entrySeparator="${cas.audit.singleline.separator:|}"
          p:useSingleLine="${cas.audit.singleline:false}"/>

    <alias name="neverThrottle" alias="authenticationThrottle" />

    <util:list id="monitorsList">
        <ref bean="memoryMonitor" />
        <ref bean="sessionMonitor" />
    </util:list>

    <alias name="defaultPrincipalFactory" alias="principalFactory" />
    <alias name="defaultAuthenticationTransactionManager" alias="authenticationTransactionManager" />
    <alias name="defaultPrincipalElectionStrategy" alias="principalElectionStrategy" />
    <alias name="tgcCipherExecutor" alias="defaultCookieCipherExecutor" />
</beans>

```

cas提供了几种连接数据库验证的实现，但是对于我们验证django用户的需求不适合，因为django用户的密码的加密salt是动态的。如果是简单的数据库用户验证，可以参考：https://apereo.github.io/cas/4.2.x/installation/Database-Authentication.html

这里我们实现了一个自己的AuthenticationHandler，主要代码如下：
```java
package com.will.cas.auth;/* Example implementation of password hasher similar on Django's PasswordHasher
 * Requires Java8 (but should be easy to port to older JREs)
 * Currently it would work only for pbkdf2_sha256 algorithm
 *
 * Django code: https://github.com/django/django/blob/1.6.5/django/contrib/auth/hashers.py#L221
 */

import org.jasig.cas.authentication.HandlerResult;
import org.jasig.cas.authentication.PreventedException;
import org.jasig.cas.authentication.UsernamePasswordCredential;
import org.jasig.cas.authentication.handler.support.AbstractUsernamePasswordAuthenticationHandler;
import org.springframework.jdbc.core.JdbcTemplate;

import javax.crypto.SecretKey;
import javax.crypto.SecretKeyFactory;
import javax.crypto.spec.PBEKeySpec;
import javax.security.auth.login.FailedLoginException;
import javax.sql.DataSource;
import javax.validation.constraints.NotNull;
import java.nio.charset.Charset;
import java.security.GeneralSecurityException;
import java.security.NoSuchAlgorithmException;
import java.security.spec.InvalidKeySpecException;
import java.security.spec.KeySpec;
import java.util.Base64;


class WillAuthenticationHandler extends AbstractUsernamePasswordAuthenticationHandler {

    private JdbcTemplate jdbcTemplate;

    private DataSource dataSource;

    /**
     * Method to set the datasource and generate a JdbcTemplate.
     *
     * @param dataSource the datasource to use.
     */
    public void setDataSource(@NotNull final DataSource dataSource) {
        this.jdbcTemplate = new JdbcTemplate(dataSource);
        this.dataSource = dataSource;
    }

    /**
     * Method to return the jdbcTemplate.
     *
     * @return a fully created JdbcTemplate.
     */
    protected final JdbcTemplate getJdbcTemplate() {
        return this.jdbcTemplate;
    }

    protected final DataSource getDataSource() {
        return this.dataSource;
    }

    public final Integer DEFAULT_ITERATIONS = 10000;
    public final String algorithm = "pbkdf2_sha256";

    public WillAuthenticationHandler() {
    }

    protected HandlerResult authenticateUsernamePasswordInternal(UsernamePasswordCredential credential) throws GeneralSecurityException, PreventedException {
        String username = credential.getUsername();
        String pwd = credential.getPassword();
        System.out.println("Got username : " + username + " and password : " + pwd);
        int count = this.jdbcTemplate.queryForObject("select count(1) from auth_user where username=?", Integer.class, username);
        if (count == 0) {
            System.out.println(username + " not found with SQL query.");
            throw new FailedLoginException(username + " not found with SQL query.");
        }
        String encyptedPassword = this.jdbcTemplate.queryForObject("select password from auth_user where username=?", String.class, username);
        System.out.println("Got encyptedPassword : " + encyptedPassword);
        if (checkPassword(pwd, encyptedPassword)) {
            System.out.println(" auth success for user : " + username);
            return createHandlerResult(credential, this.principalFactory.createPrincipal(username), null);
        }
        throw new FailedLoginException(username + "'s password is wrong for : " + pwd);
    }

    public String getEncodedHash(String password, String salt, int iterations) {
        // Returns only the last part of whole encoded password
        SecretKeyFactory keyFactory = null;
        try {
            keyFactory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256");
        } catch (NoSuchAlgorithmException e) {
            System.err.println("Could NOT retrieve PBKDF2WithHmacSHA256 algorithm");
            System.exit(1);
        }
        KeySpec keySpec = new PBEKeySpec(password.toCharArray(), salt.getBytes(Charset.forName("UTF-8")), iterations, 256);
        SecretKey secret = null;
        try {
            secret = keyFactory.generateSecret(keySpec);
        } catch (InvalidKeySpecException e) {
            System.out.println("Could NOT generate secret key");
            e.printStackTrace();
        }

        byte[] rawHash = secret.getEncoded();
        byte[] hashBase64 = Base64.getEncoder().encode(rawHash);

        return new String(hashBase64);
    }

    public String encode(String password, String salt, int iterations) {
        // returns hashed password, along with algorithm, number of iterations and salt
        String hash = getEncodedHash(password, salt, iterations);
        return String.format("%s$%d$%s$%s", algorithm, iterations, salt, hash);
    }

    public String encode(String password, String salt) {
        return this.encode(password, salt, this.DEFAULT_ITERATIONS);
    }

    public boolean checkPassword(String password, String hashedPassword) {
        // hashedPassword consist of: ALGORITHM, ITERATIONS_NUMBER, SALT and
        // HASH; parts are joined with dollar character ("$")
        String[] parts = hashedPassword.split("\\$");
        if (parts.length != 4) {
            // wrong hash format
            return false;
        }
        Integer iterations = Integer.parseInt(parts[1]);
        String salt = parts[2];
        String hash = encode(password, salt, iterations);

        return hash.equals(hashedPassword);
    }

    // Following examples can be generated at any Django project:
    //
    //  >>> from django.contrib.auth.hashers import make_password
    //  >>> make_password('mystery', hasher='pbkdf2_sha256')  # salt would be randomly generated
    //  'pbkdf2_sha256$10000$HqxvKtloKLwx$HdmdWrgv5NEuaM4S6uMvj8/s+5Yj+I/d1ay6zQyHxdg='
    //  >>> make_password('mystery', salt='mysalt', hasher='pbkdf2_sha256')
    //  'pbkdf2_sha256$10000$mysalt$KjUU5KrwyUbKTGYkHqBo1IwUbFBzKXrGQgwA1p2AuY0='
    //
    //
    // mystery
    // pbkdf2_sha256$10000$qx1ec0f4lu4l$3G81rAm/4ng0tCCPTrx2aWohq7ztDBfFYczGNoUtiKQ=
    //
    // s3cr3t
    // pbkdf2_sha256$10000$BjDHOELBk7fR$xkh1Xf6ooTqwkflS3rAiz5Z4qOV1Jd5Lwd8P+xGtW+I=
    //
    // puzzle
    // pbkdf2_sha256$10000$IFYFG7hiiKYP$rf8vHYFD7K4q2N3DQYfgvkiqpFPGCTYn6ZoenLE3jLc=
    //
    // riddle
    // pbkdf2_sha256$10000$A0S5o3pNIEq4$Rk2sxXr8bonIDOGj6SU4H/xpjKHhHAKpFXfmNZ0dnEY=

    public static void main(String[] args) {
        runTests();
    }

    private static void runTests() {
        System.out.println("===========================");
        System.out.println("= Testing password hasher =");
        System.out.println("===========================");
        System.out.println();

        System.out.println();
        passwordShouldMatch("admin", "pbkdf2_sha256$12000$g3UeiuUEnwfp$Qr/Qb6cCntnCflkBcJ+OenxWubIu6O84xLWGgWqNftw=");
        passwordShouldMatch("will@2016", "pbkdf2_sha256$30000$tTJpdOsd3eoN$GcBF0YoHg8eV7s6NBqjX1i3VllEIeYuMUsyH5stzuw4=");

        System.out.println();
        passwordShouldNotMatch("foo", "");
        passwordShouldNotMatch("mystery", "pbkdf2_md5$10000$qx1ec0f4lu4l$3G81rAm/4ng0tCCPTrx2aWohq7ztDBfFYczGNoUtiKQ=");
        passwordShouldNotMatch("mystery", "pbkdf2_sha1$10000$qx1ec0f4lu4l$3G81rAm/4ng0tCCPTrx2aWohq7ztDBfFYczGNoUtiKQ=");
        passwordShouldNotMatch("mystery", "pbkdf2_sha256$10001$Qx1ec0f4lu4l$3G81rAm/4ng0tCCPTrx2aWohq7ztDBfFYczGNoUtiKQ=");
        passwordShouldNotMatch("mystery", "pbkdf2_sha256$10001$qx1ec0f4lu4l$3G81rAm/4ng0tCCPTrx2aWohq7ztDBfFYczGNoUtiKQ=");
        passwordShouldNotMatch("mystery", "pbkdf2_sha256$10000$qx7ztDBfFYczGNoUtiKQ=");
        passwordShouldNotMatch("s3cr3t", "pbkdf2_sha256$10000$BjDHOELBk7fR$foobar");
        passwordShouldNotMatch("puzzle", "pbkdf2_sha256$10000$IFYFG7hiiKYP$rf8vHYFD7K4q2N3DQYfgvkiqpFPGCTYn6ZoenLE3jLcX");
    }

    private static void passwordShouldMatch(String password, String expectedHash) {
        WillAuthenticationHandler willAuthenticationHandler = new WillAuthenticationHandler();

        if (willAuthenticationHandler.checkPassword(password, expectedHash)) {
            System.out.println(" => OK");
        } else {
            String[] parts = expectedHash.split("\\$");
            if (parts.length != 4) {
                System.out.printf(" => Wrong hash provided: '%s'\n", expectedHash);
                return;
            }
            String salt = parts[2];
            String resultHash = willAuthenticationHandler.encode(password, salt);
            String msg = " => Wrong! Password '%s' hash expected to be '%s' but is '%s'\n";
            System.out.printf(msg, password, expectedHash, resultHash);
        }
    }

    private static void passwordShouldNotMatch(String password, String expectedHash) {
        WillAuthenticationHandler willAuthenticationHandler = new WillAuthenticationHandler();

        if (willAuthenticationHandler.checkPassword(password, expectedHash)) {
            System.out.printf(" => Wrong (password '%s' did '%s' match but were not supposed to)\n", password, expectedHash);
        } else {
            System.out.println(" => OK (password didn't match)");
        }
    }

}

```

基本是抄袭了AbstractJdbcUsernamePasswordAuthenticationHandler的内容，然后执行了自己的算法实现。本来是可以直接继承AbstractJdbcUsernamePasswordAuthenticationHandler，重写方法实现的，但是考虑到我们要兼容很多个系统的数据库用户，所以需要多个dataSource和jdbcTemplate，就放弃了。

对于cas配置的切入点在于authenticationHandlersResolvers，以前版本的都是有个authenticationManager，然后编辑他的构造参数来定义认证处理器的，4.2.6貌似把它内置了。

```xml
  <util:map id="authenticationHandlersResolvers">
        <entry key-ref="proxyAuthenticationHandler" value-ref="proxyPrincipalResolver" />
        <entry key-ref="primaryAuthenticationHandler" value-ref="primaryPrincipalResolver" />
    </util:map>

 <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
      <property name="driverClassName" value="com.mysql.jdbc.Driver"></property>
      <property name="url" value="jdbc:mysql://127.0.0.1:3306/metamap1?useUnicode=true&amp;characterEncoding=utf8"></property>
      <property name="username" value="root"></property>
      <property name="password" value=""></property>
    </bean>


    <bean id="primaryAuthenticationHandler"
      class="com.will.cas.auth.WillAuthenticationHandler"
      p:dataSource-ref="dataSource"
      />

```

如此，便指定了自定义的认证处理器。


问题
---
在app1配置cas server为当前cas server后，跳转到登录页面，结果提示：
> 未认证的授权服务 不允许使用CAS来认证您访问的目标应用

查了一下解决方案，同样是在deployerConfigContext.xml中配置服务注册的DAO serviceRegistryDao为内存实现类，然后使用正则表达式表示这个cas server接收的服务：
```xml
<alias name="jsonServiceRegistryDao" alias="serviceRegistryDao" />
```

改为：
```xml
<bean id="serviceRegistryDao" class="org.jasig.cas.services.InMemoryServiceRegistryDaoImpl"
             p:registeredServices-ref="registeredServicesList" />
 
     <util:list id="registeredServicesList">
         <bean class="org.jasig.cas.services.RegexRegisteredService"
               p:id="0" p:name="HTTP and IMAP" p:description="Allows HTTP(S) and IMAP(S) protocols"
               p:serviceId="^(https?|http?|imaps?)://.*" p:evaluationOrder="10000001" />
     </util:list>

```

其实理论上应该也可以在默认的jsonServiceRegistryDao配置的，但是找了一会儿没有找到json配置文件的位置。注意了一下貌似它是属于cas-addons的，有些插件的意思，后面再探查一下。

参考：
- https://github.com/Unicon/cas-addons/wiki/Configuring-JSON-Service-Registry
- http://www.cnphp6.com/archives/133139
