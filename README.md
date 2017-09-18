# Sonatype Nexus with Self Signed Certificates in docker

Setting up SSL certificates in [Sonatype Nexus](https://www.sonatype.com/nexus-repository-sonatype) can be a somewhat [daunting task (like modifing jetty-https.xml)](https://github.com/TerrenceMiao/nexus/wiki/Setup-HTTPS-access-in-Nexus-Repository-Manager-OSS-3.0.0) if one isn't a developer. 

I've made a docker-compose.yml which will modify the jetty-https.xml with the right settings, installs a little PKI infrastructue whith the help of [NodePKI](https://github.com/aditosoftware/nodepki) to generate root certificates which can be installed on all systems which need to connect to Nexus and which will generate a server certificate for Nexus itself in Nexus' own keystore.

TLDR; 
 * Generate root and server certificates with NodePKI
 * import root certificate on your host 
 * import server certificate in Nexus and 
 * start Nexus 

# Start NodePKI

```
docker-compose up -d nodepki
```
(nodepki takes some time to start!)

## nodepki

Goto http://localhost:5001 and generate root certificate and a certificate for your nexus host, eg. "nexus.example.local"
Note: NodePKI takes some time to start!

## Generate and Import root certificate

Goto [http://docker_host:5001/](http://docker_host:5001/) and generate root certificate, download the certificate.

### Add root certificate in Mac OSX

Double-click the root_certificate.pem, add it to System in Keystore, double click it and "trust always" and close(!) the Keystore to save.

### Add root certificate in Ubuntu

Convert cerificate to crt and place it in /usr/share/ca-certificates/extra
```
$ openssl x509 -in root_ca.cert.pem -inform PEM -out root_ca.cert.crt &&\
   sudo cp root_ca.cert.crt /usr/local/share/ca-certificates &&\
   sudo update-ca-certificates
```

## Generate server certificate
Generate server certificate for your nexus host, eg. "nexus.example.local". The files will be written into /certs which is linked into the nexus container and will picked up in the next step. 

## Import certificate in Nexus' keystore
```
$ docker-compose run nodepki ash -c 'cd /certs/nexus.example.local && openssl pkcs12 -export -in signed.crt   -inkey domain.key  -chain -CAfile chained.pem   -name "my-domain.com" -out my.p12'
```
And use password 'password'
```
$ docker-compose run nexus ash -c 'cd /certs/nexus.example.local && keytool -importkeystore -deststorepass password -destkeystore /nexus-data/keystore.jks -srckeystore my.p12 -srcstoretype PKCS12'
```

## Start Nexus
```
$ docker-compose up -d nexus
$ docker-compose logs -f nexus
```

wait until these logs are shown:
```
nexus_1    | 2017-08-29 07:42:28,302+0000 INFO  [jetty-main-1] *SYSTEM org.eclipse.jetty.server.ServerConnector - Started ServerConnector@74fa680b{SSL,[ssl, http/1.1]}{0.0.0.0:443}
nexus_1    | 2017-08-29 07:42:28,303+0000 INFO  [jetty-main-1] *SYSTEM org.eclipse.jetty.server.Server - Started @82804ms
nexus_1    | 2017-08-29 07:42:28,304+0000 INFO  [jetty-main-1] *SYSTEM org.sonatype.nexus.bootstrap.jetty.JettyServer -
nexus_1    | -------------------------------------------------
nexus_1    |
nexus_1    | Started Sonatype Nexus OSS 3.4.0-02
nexus_1    |
nexus_1    | -------------------------------------------------
```

Goto https://nexus.example.local which will now show a valid certificate!

# Troubleshooting

## Import certs in java

```
keytool -printcert -rfc -sslserver hostname > hostname.pem
keytool -importcert -file hostname.pem -keystore $(/usr/libexec/java_home)/jre/lib/security/cacerts -storepass changeit
```

```
openssl x509 -outform der -in ~/root_ca.cert.pem -out certificate.der
sudo keytool -import -alias your-alias -keystore $(/usr/libexec/java_home)/jre/lib/security/cacerts -storepass changeit -file certificate.der
```

## Fix OrientDB 

$ docker-compose exec nexus java -jar ./lib/support/nexus-orient-console.jar
```
CONNECT PLOCAL:/opt/sonatype/sonatype-work/nexus3/db/component admin admin
REBUILD INDEX *
REPAIR DATABASE --fix-graph
REPAIR DATABASE --fix-links
REPAIR DATABASE --fix-ridbags
REPAIR DATABASE --fix-bonsai
DISCONNECT
```

