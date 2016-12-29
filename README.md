# OpenUnison Kubernetes Quickstart

This qucikstart will provide an identity provider and optionally provide self service user provisioning for RBAC roles.  This quickstart provides:

1. Authentication via OpenID Connect to an OpenID Connect identity provider supported by OpenUnison (https://www.tremolosecurity.com/wiki/#!UnisonMatrix108.md#OpenID_Connect_Systems) via a "portal"
2. SSO integration with Kubernetes via the OpenID Connect identity provider integrated into OpenUnison
3. Generation of session specific client_secret to avoid sharing client_secrets for use with kubectl
4. SSO into the Kubernetes Dashboard

For more details about how Kubernetes and OpenID Connect work to provide authentication see https://www.tremolosecurity.com/wiki/#!kubernetes.md.

## Parts List

Before starting you'll need to determine:

1.  How to deploy - OpenUnison is a J2EE application that will run on almost any Java Servlet container.  The OpenUnison documentation describes how to get started with Tomcat 8 (https://www.tremolosecurity.com/docs/tremolosecurity-docs/1.0.8/openunison/openunison-manual.html#_deploying_with_apache_maven) but you can also try using our source2image builder which will deploy OpenUnison directly into a hardened Tomcat 8 server on a docker container for you (https://hub.docker.com/r/tremolosecurity/openunisons2idocker/).  The examples here will assume you are using the Source2Image builder.
2.  Database - OpenUnison needs a relational database for audit and session data.  Any of the databases listed in the support matrix will work (https://www.tremolosecurity.com/wiki/#!UnisonMatrix108.md).  This quickstart assumes mariadb/mysql but can be easily changed.
3.  Email Server (Optional) - If using this quickstart for user provisioning, an SMTP server will be needed to send email
4.  Server / Service - Depending on your decision for #1 above, you'll need to deploy somewhere.  The output of the source2image process is a docker container that can be run on Kubernetes or any other server running Docker.

## Preparation Work

Before getting started we'll need to do a few things:

1.  Download the source2image binaries for your platform (https://github.com/openshift/source-to-image/releases) (note, you'll need access to a docker service for s2i to work)
2.  Create your database and get administrative credentials.  Don't worry about specific table creation, OpenUnison will take care of that on your first startup
3.  Create a keystore with keys and certificates
4.  Collect the information for each environment variable
5.  Fork this repository - You probably will want to make changes once you have gotten the process down (ie logos, links, etc)

### Environment Variables

This project is designed to be safe for managing in source control (ie git) and for resulting docker images to be safe for public repositories (ie dockerhub) so there is NO confidential or environment specific information stored in the configuration.  Everything that is environment specific or private is stored in environment variables that will be passed into docker:

| Variable | Description | Example |
| -------- | ----------- | ------- |
| OU_HOST | The host name users will use to access the site | oidcidp.tremolo.lan |
| OU_HIBERNATE_DIALECT | The hibernate dialect for your database (https://docs.jboss.org/hibernate/orm/4.2/javadocs/org/hibernate/dialect/package-summary.html) | org.hibernate.dialect.MySQL5Dialect |
| OU_JDBC_DRIVER | JDBC driver for your database, make sure that the driver is a dependency in your POM file | com.mysql.jdbc.Driver |
| OU_JDBC_URL | The connection URL for the OpenUnison audit database | jdbc:mysql://mariadb:3306/unison?useSSL=true |
| OU_JDBC_USER | User used to connect to the audit database | root |
| OU_JDBC_PASSWORD | Password used to connect to the audit database | ***** |
| OU_JDBC_VALIDATION | A query for validating connections on checkout | SELECT 1 |
| SMTP_HOST | Host for the SMTP server | smtp.gmail.com |
| SMTP_PORT | Port for the SMTP Server | 587 |
| SMTP_FROM | The "From" subject of emails to approvers | You have approvals waiting |
| SMTP_USER | User name for accessing the email server | user@domain.com |
| SMTP_PASSWORD | Password for the user for the email server | ***** |
| SMTP_TLS | true/false if the SMTP server uses TLS | true |
| JAVA_OPTS | List of Java system properties, MUST include unisonKeystorePassword | -Djava.awt.headless=true -Djava.security.egd=file:/dev/./urandom -DunisonKeystorePassword=start123 |
| K8S_DASHBOARD_URL | The URL used for accessing the dashboard, usually this is the API server's secure port | https://kubemaster.tremolo.lan:6443 |
| K8S_URL | The URL for the API server's secure port | https://kubemaster.tremolo.lan:8443 |
| K8S_CLIENT_SECRET | An OIDC client secret that can be used by consumers of the openid connect trust between Kubernetes and OpenUnison, this should be a long random string but is not used for any configuration options in Kubernetes | XXXXXXX-XXXXXXXX-XXXXXXXX-XXXXXX |
| OIDC_CLIENT_ID | The client ID used by OpenUnison to identify its self to the OIDC identity provider (ie google.com) | openunison |
| OIDC_CLIENT_SECRET | The shared secret provided by your identity provider to authenticate OpenUnison | XXXXXXXXXXXXXX |
| OIDC_IDP_URL | The authentication service for your provider | http://keycloak.tremolo.lan:8080/auth/realms/unison/protocol/openid-connect/auth |
| OIDC_SCOPE | The scope to be used with your OIDC identity provider (usually there's no need to use anything except the example) | openid email profile name |
| OIDC_LOAD_TOKEN_URL | Your OIDC identity provider's token service URL | http://keycloak.tremolo.lan:8080/auth/realms/unison/protocol/openid-connect/token |

### Building a Keystore

OpenUnison encrypts or signs everything that leaves it such as JWTs, workflow requests, session cookies, etc.  To do this, we need to create a Java keystore that can be used to store these keys as well as the certificates used for TLS by Tomcat.  When working with Kubernetes something to take note of is Go does NOT work with self signed certificates no matter how many ways you trust it.  In order to use a self signed certificate you have to create a self signed certificate authority and THEN create a certificate signed by that CA.  This can be done using Java's keytool but I like using OpenSSL better.  To make this easier, the makecerts.sh script in this repository (adapted from a similar script from CoreOS) will do this for you.  Just make sure to change the subject in the script first

#### Build the TLS certificate

```
$ sh makecerts.sh
$ cd ssl
$ openssl pkcs12 -export -chain -inkey key.pem -in cert.pem -CAfile ca.pem -out openunison.p12
Enter Export Password:
Verifying - Enter Export Password:
$ keytool -importkeystore -srckeystore ./openunison.p12 -srcstoretype PKCS12 -alias 1 -destKeystore ./unisonKeyStore.jks -deststoretype JCEKS -destalias unison-tls
Enter destination keystore password:  
Re-enter new password:
Enter source keystore password:
```

#### Create Static Keys

```
$ keytool -genseckey -alias session-unison -keyalg AES -keysize 256 -storetype JCEKS -keystore ./unisonKeyStore.jks
$ keytool -genseckey -alias lastmile-k8s -keyalg AES -keysize 256 -storetype JCEKS -keystore ./unisonKeyStore.jks
```

#### Trust Kubernetes' CA

Depending on how you deploy Kubernetes you will need to trust Kubernetes' CA.  For kubeadm deployments, the CA file is ```/etc/kubernetes/pki/ca.pem```.  You'll need to import that file into your keystore.  Assuming it's copied to the file k8s-ca.pem in the ssl directory:

```
$ keytool -import -alias trusted-k8sca -trustcacerts -rfc -file ./k8s-ca.pem -storetype JCEKS -keystore ./unisonKeyStore.jks
Enter keystore password:  
Owner: CN=kubernetes
Issuer: CN=kubernetes
Serial number: 0
Valid from: Wed Dec 28 21:05:49 EST 2016 until: Sat Dec 26 21:05:49 EST 2026
Certificate fingerprints:
	 MD5:  16:77:AF:F6:F5:6A:98:99:72:0C:6A:9A:BD:EF:AA:1D
	 SHA1: B7:FA:C8:0E:14:66:43:BE:86:C1:05:DA:E8:EC:EE:7E:6F:8C:10:09
	 SHA256: 2A:C9:AE:04:88:CD:9F:8C:21:B9:D8:15:C7:AD:0E:D0:FA:4B:20:90:E2:BE:D0:E2:D2:E4:87:89:C5:E0:C6:1F
	 Signature algorithm name: SHA256withRSA
	 Version: 3

Extensions:

#1: ObjectId: 2.5.29.19 Criticality=true
BasicConstraints:[
  CA:true
  PathLen:2147483647
]

#2: ObjectId: 2.5.29.15 Criticality=true
KeyUsage [
  DigitalSignature
  Key_Encipherment
  Key_CertSign
]

Trust this certificate? [no]:  yes
Certificate was added to keystore
```

## Deployment

This will be very specific to your own environment.  This part offers up how to deploy OpenUnison using the source2image builder on a standalone server running docker.

### Build The OpenUnison Image

The easiest way to build the image is against a git repository (either remote or local).  If you want to build OpenUnison's war locally first you can do that too by providing a directory with the OpenUnison war file in it.

```
$ ./s2i build https://github.com/TremoloSecurity/openunison-qs-kubernetes.git tremolosecurity/openunisons2idocker:1.0.8 myrepo/k8sopenunison
$ docker push myrepo/k8sopenunison
```

### Deploy The OpenUnison Image

Assuming you have docker running on your server:

1.  Create a directory called /etc/openunison
2.  Copy the ```unisonKeyStore.jks``` file to /etc/openunison
3.  Create a file for environment variables called ```docker.env``` (see below for an example)
4.  Run the image ```docker run -d --name openunison -p 8080:8080 -p 8443:8443 --env-file /etc/openunison/docker.env -v /etc/openunison:/etc/openunison:Z myrepo/k8sopenunison```

```
OU_HOST=oidcidp.tremolo.lan
K8S_DASHBOARD_URL=https://kubemaster.tremolo.lan:6443
OU_JDBC_DRIVER=com.mysql.jdbc.Driver
OU_JDBC_URL=jdbc:mysql://mariadb/unison
OU_JDBC_USER=unison
OU_JDBC_PASSWORD=XXXXX
OU_HIBERNATE_DIALECT=org.hibernate.dialect.MySQL5Dialect
OU_JDBC_VALIDATION=SELECT 1
K8S_URL=https://kubemaster.tremolo.lan:8443
K8S_CLIENT_SECRET=4d4af469-0f8f-4d91-9048-dae716721ce2
OIDC_CLIENT_ID=openunison
OIDC_CLIENT_SECRET=99355628-65ac-4159-a9e4-af299ce5bf31
OIDC_IDP_URL=http://keycloak.tremolo.lan:8080/auth/realms/unison/protocol/openid-connect/auth
OIDC_SCOPE=openid email profile name
OIDC_LOAD_TOKEN_URL=http://keycloak.tremolo.lan:8080/auth/realms/unison/protocol/openid-connect/token
JAVA_OPTS=-Djava.awt.headless=true -Djava.security.egd=file:/dev/./urandom -DunisonKeystorePassword=XXXXXXX
SMTP_HOST=smtp
SMTP_PORT=25
SMTP_USER=
SMTP_PASSWORD=
SMTP_FROM=donotreply@k8s-idm.com
SMTP_TLS=false
```

Once started, you can see the logs by running ```docker logs -f openunison```

### Integrating with Kubernetes

This will vary based on your Kubernetes deployment method.  For kubeadm you'll want to add your parameters to ```/etc/kubernetes/manifests/kube-apiserver.json```.

## Use and Testing

Once integrated, you'll want to make sure that the api-server isn't generating errors because it can't connect to OpenUnison.  Also, do a quick liveness test by going to https://OU_HOST:8443/auth/idp/OidcIdP/.well-known/openid-configuration where you'll see something like:

```
{
  "issuer": "https://oidcidp.tremolo.lan:8443/auth/idp/OidcIdP",
  "authorization_endpoint": "https://oidcidp.tremolo.lan:8443/auth/idp/OidcIdP/auth",
  "token_endpoint": "https://oidcidp.tremolo.lan:8443/auth/idp/OidcIdP/token",
  "userinfo_endpoint": "https://oidcidp.tremolo.lan:8443/auth/idp/OidcIdP/userinfo",
  "revocation_endpoint": "https://oidcidp.tremolo.lan:8443/auth/idp/OidcIdP/revoke",
  "jwks_uri": "https://oidcidp.tremolo.lan:8443/auth/idp/OidcIdP/certs",
  "response_types_supported": [
    "code",
    "token",
    "id_token",
    "code token",
    "code id_token",
    "token id_token",
    "code token id_token",
    "none"
  ],
  "subject_types_supported": [
    "public"
  ],
  "id_token_signing_alg_values_supported": [
    "RS256"
  ],
  "scopes_supported": [
    "openid",
    "email",
    "profile"
  ],
  "token_endpoint_auth_methods_supported": [
    "client_secret_post"
  ],
  "claims_supported": [
    "sub",
    "aud",
    "iss",
    "exp",
    "sub",
    "email_verified",
    "first_name",
    "family_name",
    "email"
  ],
  "code_challenge_methods_supported": [
    "plain",
    "S256"
  ]
}
```

Then you can go straight to https://OU_HOST:8443 where you should be redirected to your identity provider for authentication and brought to a screen that shows you links for the dashboard and your tokens.  If you click on the dashboard link and see the dashboard, you know the API server is configured correctly.  Click on the token and use the id_token, refresh_token and client_secret in your kubectl configuration and kubectl should work too.
