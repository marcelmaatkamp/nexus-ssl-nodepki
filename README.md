# Sonatype Nexus with Self Signed Certificates

Setting up SSL certificates in [Sonatype Nexus](https://www.sonatype.com/nexus-repository-sonatype) can be a somewhat daunting task (like modifing server.xml) if one isn't a developer. Therefore I've made a docker-compose.yml which will modify the server.xml, installs a little PKI infrastructue (whith the help of [NodePKI](https://github.com/aditosoftware/nodepki) which will generate root certificates which can be installed on all systems which need to connect to Nexus and generates a certificate for Nexus itself in Nexus' own keystore.

TLDR; 
 * Generate root and server certificates
 * import root certificate on your host 
 * import server certificate in nexus and 
 * start the server. 

## Start NodePKI
```
docker-compose up -d nodepki
```
(nodepki takes some time to start!)

## Start NodePKI
Goto [http://docker_host:5001/](http://docker_host:5001/) and generate root certificate and a certificate for your nexus host, eg. "nexus.example.local"

### Add root certificate in Mac OSX
Double-click the root_certificate.pem, add it to System in Keystore, double click it and "trust always" and close(!) the Keystore to save.

### Add root certificate in Ubuntu
Convert cerificate to crt and place it in /usr/share/ca-certificates/extra
```
$ openssl x509 -in root_ca.cert.pem -inform PEM -out root_ca.cert.crt &&\
   sudo cp root_ca.cert.crt /usr/local/share/ca-certificates &&\
   sudo update-ca-certificates
```

## Import certificate in Nexus' keystore
```
$ docker-compose run nodepki ash -c 'cd /certs/nexus.example.local && openssl pkcs12 -export -in signed.crt   -inkey domain.key  -chain -CAfile chained.pem   -name "my-domain.com" -out my.p12'
```
And use password 'changeit'
```
$ docker-compose run nexus ash -c 'cd /certs/nexus.example.local && keytool -importkeystore -deststorepass changeit -destkeystore /nexus-data/keystore.jks -srckeystore my.p12 -srcstoretype PKCS12'
```

## Start nexus
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
