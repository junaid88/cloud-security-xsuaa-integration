# Migration Guide for Applications that use Spring Security and java-container-security

This migration guide is a step-by-step guide explaining how to replace the following SAP-internal Java Container Security Client libraries
- com.sap.xs2.security:java-container-security
- com.sap.cloud.security.xsuaa:java-container-security  

with this open-source version.

**Please note, that this Migration Guide is NOT intended for applications that leverage token validation and authorization checks using SAP Java Buildpack.** This [documentation](https://github.com/SAP/cloud-security-xsuaa-integration#token-validation-for-java-web-applications-using-sap-java-buildpack) describes the setup when using SAP Java Buildpack.

## Prerequisite: Migrate to Spring 5

- spring boot 2
- spring framework 5
- link to official spring 5 migration guide
- sample --> bulletinboard commit


## Maven Dependencies
To use the new [java-security](/java-security) client library the dependencies declared in maven `pom.xml` need to be updated.

First make sure you have the following dependencies defined in your pom.xml:

```xml
<!-- take the latest spring-security dependencies -->
<!-- Spring deprecates it as it gets replaced with libs of groupId "org.springframework.security" -->
<dependency>
  <groupId>org.springframework.security.oauth</groupId>
  <artifactId>spring-security-oauth2</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-aop</artifactId>
</dependency>

<!-- new java-security dependencies -->
<dependency>
  <groupId>com.sap.cloud.security.xsuaa</groupId>
  <artifactId>api</artifactId>
  <version>2.5.2</version>
</dependency>
<dependency>
  <groupId>com.sap.cloud.security</groupId>
  <artifactId>java-security</artifactId>
  <version>2.5.2</version>
</dependency>

<!-- new java-security dependencies for unit tests -->
<dependency>
  <groupId>com.sap.cloud.security</groupId>
  <artifactId>java-security-test</artifactId>
  <version>2.5.2</version>
  <scope>test</scope>
</dependency>
```


Now you are ready to **remove** the **`java-container-security`** client library by deleting the following lines from the pom.xml:
```xml
<dependency>
  <groupId>com.sap.xs2.security</groupId>
  <artifactId>java-container-security</artifactId>
</dependency>
<dependency>
  <groupId>com.sap.xs2.security</groupId>
  <artifactId>java-container-security-api</artifactId>
</dependency>
```
Or
```xml
<dependency>
  <groupId>com.sap.cloud.security.xsuaa</groupId>
  <artifactId>java-container-security</artifactId>
</dependency>
<dependency>
  <groupId>com.sap.cloud.security.xsuaa</groupId>
  <artifactId>api</artifactId>
</dependency>
```

Make sure that you do not refer to any other sap security library with group-id `com.sap.security` or `com.sap.security.nw.sso.*`. 

## Configuration changes
After the dependencies have been changed, the spring security configuration needs some adjustments as well.

If your security configuration was using the `SAPOfflineTokenServicesCloud` class from the `java-container-security` library,
you need to change it slightly to use the `SAPOfflineTokenServicesCloud` adapter class from the new library.

> Note: There is no replacement for `SAPPropertyPlaceholderConfigurer` as you can always parameterize the `SAPOfflineTokenServicesCloud` bean with your `OAuth2ServiceConfiguration`.

### Code-based

For example see the following snippet on how to instantiate the `SAPOfflineTokenServicesCloud`. 

```java
@Bean
@Profile("cloud")
protected SAPOfflineTokenServicesCloud offlineTokenServices() {
	return new SAPOfflineTokenServicesCloud(
				Environments.getCurrent().getXsuaaConfiguration(), //optional
				new RestTemplate())                                //optional
			.setLocalScopeAsAuthorities(true);                         //optional
}
```
You might need to fix your java imports to get rid of the old import for the `SAPOfflineTokenServicesCloud` class.

### XML-based
As you may have updated the 

In case of XML-based Spring (Security) configuration you need to replace your current `SAPOfflineTokenServicesCloud` bean definition with that:

```xml
<bean id="offlineTokenServices"
         class="com.sap.cloud.security.adapter.spring.SAPOfflineTokenServicesCloud">
		<property name="localScopeAsAuthorities" value="true" />
</bean>
```

With `localScopeAsAuthorities` you can perform spring scope checks without referring to the XS application name (application id), e.g. `myapp!t1234`. For example:

Or
```java
...
.authorizeRequests()
	.antMatchers(POST, "/api/v1/ads/**").access(#oauth2.hasScopeMatching('Update')) //instead of '${xs.appname}.Update'
```


```xml
<sec:intercept-url pattern="/rest/addressbook/deletedata" access="#oauth2.hasScope('Delete')" method="GET" />
```

## Fetch basic infos from Token
You may have code parts that requests information from the access token, like the user's name, its tenant, and so on. So, look up your code to find its usage.


```java
import com.sap.xs2.security.container.SecurityContext;
import com.sap.xs2.security.container.UserInfo;
import com.sap.xs2.security.container.UserInfoException;


try {
	UserInfo userInfo = SecurityContext.getUserInfo();
	String logonName = userInfo.getLogonName();
} catch (UserInfoException e) {
	// handle exception
}

```

This can be easily replaced with the `Token` or `XsuaaToken` Api.

```java
import com.sap.cloud.security.token.*;

AccessToken token = SecurityContext.getAccessToken();
String logonName = token.getClaimAsString(TokenClaims.USER_NAME);
boolean hasDisplayScope = token.hasLocalScope("Display");	
GrantType grantType = token.getGrantType();
```

> Note, that no `XSUserInfoException` is raised, in case the token does not contain the requested claim.

## Fetch further `XSUserInfo` infos from Token

TODO - compare "Token" with "XSUserInfo" HDB Token z.b getHdbToken...

When you're done with the first part and need further information from the token you can use `XSUserInfoAdapter` in order to access the remaining methods exposed by [`XSUserInfo`](/api/src/main/java/com/sap/xsa/security/container/XSUserInfo.java) Interface.

```java
try {
	XSUserInfo userInfo = new XSUserInfoAdapter(token);
	String dbToken = userInfo.getHdbToken();
} catch (XSUserInfoException e) {
	// handle exception
}
```

## Further Spring Usages

https://github.com/SAP/cloud-security-xsuaa-integration/tree/master/spring-xsuaa#check-authorization-within-a-method

## Test Code Changes

### Security configuration for tests
If you want to overwrite the service configuration of the `SAPOfflineTokenServicesCloud` for your test, you can do so by
using some test constants provided by the test library. The following snippet shows how to do that:
```java 
import static com.sap.cloud.security.config.cf.CFConstants.*;

@Configuration
public class TestSecurityConfig {
	@Bean
	@Primary
	public SAPOfflineTokenServicesCloud sapOfflineTokenServices() {
		OAuth2ServiceConfiguration configuration = OAuth2ServiceConfigurationBuilder
				.forService(Service.XSUAA)
				.withClientId(SecurityTestRule.DEFAULT_CLIENT_ID)
				.withProperty(CFConstants.XSUAA.APP_ID, SecurityTestRule.DEFAULT_APP_ID)
				.withProperty(CFConstants.XSUAA.UAA_DOMAIN, SecurityTestRule.DEFAULT_DOMAIN)
				.build();

		return new SAPOfflineTokenServicesCloud(configuration).setLocalScopeAsAuthorities(true);
	}
}
```

### Unit testing 
In your unit test you might want to generate jwt tokens and have them validated. The new [java-security-test](/java-security-test) library provides it's own `JwtGenerator`.  This can be embedded using the new 
`SecurityTestRule`. See the following snippet as example: 

```java
@ClassRule
public static SecurityTestRule securityTestRule =
	SecurityTestRule.getInstance(Service.XSUAA)
		.setKeys("/publicKey.txt", "/privateKey.txt");
```

Using the `SecurityTestRule` you can use a preconfigured `JwtGenerator` to create JWT tokens with custom scopes for your tests. It configures the JwtGenerator in such a way that **it uses the public key from the [`publicKey.txt`](/java-security-test/src/main/resources) file to sign the token.**

```java
static final String XSAPPNAME = SecurityTestRule.DEFAULT_APP_ID;
static final String DISPLAY_SCOPE = XSAPPNAME + ".Display";
static final String UPDATE_SCOPE = XSAPPNAME + ".Update";

String jwt = securityTestRule.getPreconfiguredJwtGenerator()
    .withScopes(DISPLAY_SCOPE, UPDATE_SCOPE)
    .createToken()
    .getTokenValue();

```

Make sure, that your JUnit tests are running.

## Enable local testing
For local testing you might need to provide custom `VCAP_SERVICES` before you run the application. 
The new security library requires the following key value pairs in the `VCAP_SERVICES`
under `xsuaa/credentials` for jwt validation:
- `"uaadomain" : "localhost"`
- `"verificationkey" : "<public key your jwt token is signed with>"`

Before calling the service you need to provide a digitally signed JWT token to simulate that you are an authenticated user. 
- Therefore simply set a breakpoint in `JWTGenerator.createToken()` and run your `JUnit` tests to fetch the value of `jwt` from there. 

Now you can test the service manually in the browser using the `Postman` chrome plugin and check whether the secured functions can be accessed when providing a valid generated Jwt Token.


## Things to check after migration 
When your code compiles again you should first check that all your unit tests are running again. If you can test your
application locally make sure that it is still working and finally test the application in cloud foundry.

## Troubleshoot
- org.springframework.beans.factory.xml.XmlBeanDefinitionStoreException  
```
org.springframework.beans.factory.xml.XmlBeanDefinitionStoreException: Line XX in XML document from ServletContext resource [/WEB-INF/spring-security.xml] is invalid; 
nested exception is org.xml.sax.SAXParseException; lineNumber: 51; columnNumber: 118; cvc-complex-type.2.4.c: 
The matching wildcard is strict, but no declaration can be found for element 'oauth:resource-server'.
```  
You can fix this by changing the schema location to `https` for `oauth2` as below in the spring security xml. With this change, the local jar is available and solves the issue of server trying to connect to get the jar and fails due to some restrictions.

```
xsi:schemaLocation="http://www.springframework.org/schema/security/oauth2
https://www.springframework.org/schema/security/spring-security-oauth2.xsd
```

## Multiple Bindings to XSUAA Service instances
- TODO, either support or document

## Issues
In case you face issues to apply the migration steps feel free to open a Issue here on [Github.com](https://github.com/SAP/cloud-security-xsuaa-integration/issues/new).

## Samples
- refer to https://github.com/SAP-samples/cloud-bulletinboard-ads/blob/Documentation/Security/Exercise_24_MakeYourApplicationSecure.md
- https://github.com/SAP/cloud-security-xsuaa-integration/tree/master/samples/spring-security-xsuaa-usage


