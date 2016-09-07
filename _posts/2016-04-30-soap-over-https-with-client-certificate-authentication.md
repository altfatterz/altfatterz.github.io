---
layout: post
title: SOAP over HTTPS with client certificate authentication
tags: [wsdl, springboot, WebService, soapui]
---

Recently I had to consume a SOAP web service over HTTPS using client certificate authentication. I thought I will write a blog post about it describing my findings.
For the example I will build a simple service which exposes team information about the [UEFA EURO 2016](http://www.uefa.com/uefaeuro/season=2016/teams/index.html) football championship. The service will be secured with client certificate authentication and accessible only over HTTPS.

### Producer

First we define the web service domain with XML Schema, which [Spring-WS](http://projects.spring.io/spring-ws/) will expose automatically as a WSDL. The schema defines that for a given country code we return information about the team like nick name, coach, which country they represent.

```xml
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:tns="http://www.uefa.com/uefaeuro/season=2016/teams"
           targetNamespace="http://www.uefa.com/uefaeuro/season=2016/teams" elementFormDefault="qualified">

    <xs:element name="getTeamRequest">
        <xs:complexType>
            <xs:sequence>
                <xs:element name="countryCode" type="xs:string"/>
            </xs:sequence>
        </xs:complexType>
    </xs:element>

    <xs:element name="getTeamResponse">
        <xs:complexType>
            <xs:sequence>
                <xs:element name="team" type="tns:team"/>
            </xs:sequence>
        </xs:complexType>
    </xs:element>

    <xs:complexType name="team">
        <xs:sequence>
            <xs:element name="countryCode" type="xs:string"/>
            <xs:element name="country" type="xs:string"/>
            <xs:element name="nickName" type="xs:string"/>
            <xs:element name="coach" type="xs:string"/>
        </xs:sequence>
    </xs:complexType>

</xs:schema>
```

We generate domain classes from XSD file during build time using the `maven-jaxb2-plugin` plugin.

```xml
<plugin>
    <groupId>org.jvnet.jaxb2.maven2</groupId>
    <artifactId>maven-jaxb2-plugin</artifactId>
    <version>0.12.3</version>
    <executions>
        <execution>
            <goals>
                <goal>generate</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <generatePackage>com.uefa.euro.season</generatePackage>
        <schemas>
            <schema>
                <url>${project.basedir}/src/main/resources/teams.xsd</url>
            </schema>
        </schemas>
    </configuration>
</plugin>

```

With Spring-WS we create endpoints (using the `@Endpoint` annotation) which handle incoming XML requests. The `@PayloadRoot` tells Spring-WS that the `getTeam` method knows how to handle XML messages that have `getTeamRequest` as local part with the `http://www.uefa.com/uefaeuro/season=2016/teams` namespace.
The `@ResponsePayload` indicates that the return value of the method will be the payload of the response message.

```java
@Endpoint
public class TeamEndpoint {

    private static final String NAMESPACE_URI = "http://www.uefa.com/uefaeuro/season=2016/teams";

    private TeamRepository teamRepository;

    public TeamEndpoint(TeamRepository teamRepository) {
        this.teamRepository = teamRepository;
    }

    @PayloadRoot(namespace = NAMESPACE_URI, localPart = "getTeamRequest")
    @ResponsePayload
    public GetTeamResponse getTeam(@RequestPayload GetTeamRequest request) throws TeamNotFoundException, EmptyCountryCodeException {
        if (StringUtils.isEmpty(request.getCountryCode())) {
            throw new EmptyCountryCodeException("country code cannot be empty");
        }

        GetTeamResponse response = new GetTeamResponse();
        Team team = teamRepository.findByCountryCode(request.getCountryCode());

        if (team == null) {
            throw new TeamNotFoundException("invalid country code or country did not qualify");
        }

        response.setTeam(team);
        return response;
    }
}
```

To enable support for the Spring-WS annotations we use Java config, by declaring the `@EnableWs` annotation. The WSDL file is exposed with the help of `DefaultWsdl11Definition` bean definition, where the bean name determines under which name the generated WSDL file is available.

```java
@EnableWs
@Configuration
public class WebServiceConfig extends WsConfigurerAdapter {

    @Bean
    public ServletRegistrationBean messageDispatcherServlet(ApplicationContext applicationContext) {
        MessageDispatcherServlet servlet = new MessageDispatcherServlet();
        servlet.setApplicationContext(applicationContext);
        servlet.setTransformWsdlLocations(true);
        return new ServletRegistrationBean(servlet, "/uefaeuro2016/*");
    }

    @Bean(name = "teams")
    public DefaultWsdl11Definition defaultWsdl11Definition(XsdSchema teamSchema) {
        DefaultWsdl11Definition wsdl11Definition = new DefaultWsdl11Definition();
        wsdl11Definition.setPortTypeName("TeamsPort");
        wsdl11Definition.setLocationUri("/uefaeuro2016");
        wsdl11Definition.setTargetNamespace("http://www.uefa.com/uefaeuro/season=2016/teams");
        wsdl11Definition.setSchema(teamSchema);
        return wsdl11Definition;
    }

    @Bean
    public XsdSchema countriesSchema() {
        return new SimpleXsdSchema(new ClassPathResource("teams.xsd"));
    }
}
```

After starting the uefa service the WSDL will be available at `http://localhost:8080/uefaeuro2016/teams.wsdl`. It can be easily tested with [SoapUI](https://www.soapui.org/) after importing the WSDL file

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:team="http://www.uefa.com/uefaeuro/season=2016/teams">
   <soapenv:Header/>
   <soapenv:Body>
      <team:getTeamRequest>
      	<team:countryCode>HU</team:countryCode>
      </team:getTeamRequest>
   </soapenv:Body>
</soapenv:Envelope>
```

the response:

```xml
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/">
   <SOAP-ENV:Header/>
   <SOAP-ENV:Body>
      <ns2:getTeamResponse xmlns:ns2="http://www.uefa.com/uefaeuro/season=2016/teams">
         <ns2:team>
            <ns2:countryCode>HU</ns2:countryCode>
            <ns2:country>Hungary</ns2:country>
            <ns2:nickName>Mighty Magyars</ns2:nickName>
            <ns2:coach>Bernd Storck</ns2:coach>
         </ns2:team>
      </ns2:getTeamResponse>
   </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```

### Enable HTTPS

So far so good, but we would like to secure the service with client certificate and making it only available over HTTPS.

First we need to get an SSL certificate (self-signed or get one from a certificate authority). Let's generate a self-signed certificate with the `keytool` utility which comes bundled in JRE.

```
keytool -genkey -keyalg RSA -alias selfsigned -keystore keystore.jks -storepass password -validity 360
```

This will generate a keystore called `keystore.jks` with a newly generated certificate in it with certificate alias `selfsigned`, which you can check with the following command:

```
keytool -list -keystore keystore.jks -storepass password

Keystore type: JKS
Keystore provider: SUN

Your keystore contains 1 entry

selfsigned, Apr 27, 2016, PrivateKeyEntry,
Certificate fingerprint (SHA1): 01:83:FE:F0:44:7A:87:63:66:4B:F0:2F:20:B7:D0:14:87:6D:A8:16
```

Then we use this certificate in our uefa service by declaring the followings in the default `application.properties`:

```
server.port=8443
server.ssl.key-store=classpath:keystore.jks
server.ssl.key-store-password=password
server.ssl.key-alias=selfsigned
```

Here we included the `keystore.jks` into the project which is of course not recommended but for this simple example is ok.

After restarting the uefa service our WSDL file will be available at `https://localhost:8443/uefaeuro2016/teams.wsdl`. If you access it in Chrome browser for example, the browser will complain that it is using a self-signed certificate.
In SoapUI we are no longer able to send SOAP messages to `http://localhost:8080/uefaeuro2016` instead we need to use `https://localhost:8443/uefaeuro2016` target url.

### Authentication with client certificate

However any client is able to call the service. Let's create separate certificates for two clients one for SoapUI and one for a java client.

```

keytool -genkey -keyalg RSA -alias soapui -keystore soapui.jks -storepass password -validity 360
keytool -genkey -keyalg RSA -alias javaclient -keystore javaclient.jks -storepass password -validity 360

// extract the certificate from the keystores
keytool -export -alias soapui -file soapui.crt -keystore soapui.jks -storepass password
keytool -export -alias javaclient -file javaclient.crt -keystore javaclient.jks -storepass password

```

Then for the uefa service we need to configure the `truststore`, which determines the remote authentication credentials which should be trusted, in contrast with `keystore` which determines the authentication credentials to send to the remote host.
First we import the `soapui` and `javaclient` certificates into the `truststore` keystore. After the first import the `truststore` keystore will be automatically created.

```
keytool -import -alias soapui -file soapui.crt -keystore truststore.jks -storepass password
keytool -import -alias javaclient -file javaclient.crt -keystore truststore.jks -storepass password
```

The `truststore` should have both `soapui` and `javaclient` certificates:

```
keytool -list -keystore truststore.jks -storepass password

Keystore type: JKS
Keystore provider: SUN

Your keystore contains 2 entries

javaclient, Apr 27, 2016, trustedCertEntry,
Certificate fingerprint (SHA1): 7A:44:89:59:21:C7:8D:24:C8:78:30:8C:B7:C1:C9:ED:B7:DD:19:ED
soapui, Apr 27, 2016, trustedCertEntry,
Certificate fingerprint (SHA1): 37:0F:1A:AF:03:BA:44:DC:BC:0A:9B:77:4B:60:5F:D5:B7:FC:63:6E
```

After this we configure the `truststore` of the uefa service.

```
server.port=8443
server.ssl.key-store=classpath:keystore.jks
server.ssl.key-store-password=password
server.ssl.key-alias=selfsigned

server.ssl.trust-store=classpath:truststore.jks
server.ssl.trust-store-password=password
server.ssl.client-auth=need
```

Is important to set the `server.ssl.client-auth` to `need` in order to make the client authentication mandatory.
Now `SoapUI` is not able to call our uefa service only just with a trusted certificate, otherwise it returns `javax.net.ssl.SSLHandshakeException`
After configuring the client `soapui` certificate in the `SoapUI Preferences -> SSL Settings` form with `KeyStore` and `KeyStore Password` fields we can successfully send SOAP requests.

As an exercise you can create a `dummy` certificate (not included in the truststore of the service) and use it in `SoapUI` and verify that the connection is not established.

### Java client

Now let's connect to the uefa service from a java client.
We are using the `maven-jaxb2-plugin` again to generate java classes from the exposed WSDL by our uefa service.

```xml
<plugin>
    <groupId>org.jvnet.jaxb2.maven2</groupId>
    <artifactId>maven-jaxb2-plugin</artifactId>
    <version>0.13.0</version>
    <executions>
        <execution>
            <goals>
                <goal>generate</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <schemaLanguage>WSDL</schemaLanguage>
        <generatePackage>com.eufa.euro</generatePackage>
        <schemas>
            <schema>
                <url>${project.basedir}/src/main/resources/uefaeuro.wsdl</url>
            </schema>
        </schemas>
    </configuration>
</plugin>
```

We leverage the `WebServiceGatewaySupport` class from Spring-WS which provides a reference to an already configured `WebServiceTemplate` instance.

```java
public class TeamClient extends WebServiceGatewaySupport {

    public GetTeamResponse getTeamByCountryCode(String countryCode) {
        GetTeamRequest request = new GetTeamRequest();
        request.setCountryCode(countryCode);

        GetTeamResponse response = (GetTeamResponse) getWebServiceTemplate().marshalSendAndReceive(request);

        return response;
    }
}
```

Next we need to configure the web service components. The `Jaxb2Marshaller` is responsible to serialize and deserialize XML requests. By default the `WebServiceTemplate` is using `HttpUrlConnectionMessageSender` to send messages which is not good for us since it does not come with HTTPS certificate support. Instead we use `HttpsUrlConnectionMessageSender`.
We use the previously created `javaclient` (trusted by the uefa service) when setting the `keystore` of the java client. But we need to set also the `truststore` of the java client with the `selfsigned` certificate of uefa service in order have a successful SSL handshake.

```java
@Configuration
public class WebServiceClientConfig {

    private static final Logger LOGGER = LoggerFactory.getLogger(WebServiceClientConfig.class);

    @Value("${uefa.ws.endpoint-url}")
    private String url;

    @Value("${uefa.ws.key-store}")
    private Resource keyStore;

    @Value("${uefa.ws.key-store-password}")
    private String keyStorePassword;

    @Value("${uefa.ws.trust-store}")
    private Resource trustStore;

    @Value("${uefa.ws.trust-store-password}")
    private String trustStorePassword;

    @Bean
    public Jaxb2Marshaller marshaller() {
        Jaxb2Marshaller marshaller = new Jaxb2Marshaller();
        marshaller.setContextPath("com.eufa.euro");
        return marshaller;
    }

    @Bean
    public TeamClient teamClient(Jaxb2Marshaller marshaller) throws Exception {
        TeamClient client = new TeamClient();
        client.setDefaultUri(this.url);
        client.setMarshaller(marshaller);
        client.setUnmarshaller(marshaller);

        KeyStore ks = KeyStore.getInstance("JKS");
        ks.load(keyStore.getInputStream(), keyStorePassword.toCharArray());

        LOGGER.info("Loaded keystore: " + keyStore.getURI().toString());
        try {
            keyStore.getInputStream().close();
        } catch (IOException e) {
        }
        KeyManagerFactory keyManagerFactory = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
        keyManagerFactory.init(ks, keyStorePassword.toCharArray());

        KeyStore ts = KeyStore.getInstance("JKS");
        ts.load(trustStore.getInputStream(), trustStorePassword.toCharArray());
        LOGGER.info("Loaded trustStore: " + trustStore.getURI().toString());
        try {
            trustStore.getInputStream().close();
        } catch(IOException e) {
        }
        TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
        trustManagerFactory.init(ts);

        HttpsUrlConnectionMessageSender messageSender = new HttpsUrlConnectionMessageSender();
        messageSender.setKeyManagers(keyManagerFactory.getKeyManagers());
        messageSender.setTrustManagers(trustManagerFactory.getTrustManagers());

        // otherwise: java.security.cert.CertificateException: No name matching localhost found
        messageSender.setHostnameVerifier((hostname, sslSession) -> {
            if (hostname.equals("localhost")) {
                return true;
            }
            return false;
        });

        client.setMessageSender(messageSender);
        return client;
    }
}
```

### Summary

In this post we saw how can we expose a simple SOAP web service over HTTPS and have two clients (soapUI, java client) connect to it using client certificates. If you would like to try out this have a look at this [https://github.com/altfatterz/spring-ws-with-keystore](https://github.com/altfatterz/spring-ws-with-keystore) repository.

