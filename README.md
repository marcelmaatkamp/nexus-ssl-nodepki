# Sonatype Nexus with Self Signed Certificates

## Start proxy and nodepki
```
docker-compose up -d proxy nodepki
```
(nodepki takes some time to start!)

## setup proxy
Set proxy in browser: type: "socks5", hostname: "docker_host", port: 1080

## nodepki
Goto http://nodepki:5000 and generate root certificate and a certificate for your nexus host, eg. "nexus.example.local"

### add root certificate in Mac OSX
Double-click the root_certificate.pem, add it to System in Keystore, double click it and "trust always" and close(!) the Keystore to save.

### add root certificate in Ubuntu
Convert cerificate to crt and place it in /usr/share/ca-certificates/extra
```
$ openssl x509 -in root_ca.cert.pem -inform PEM -out root_ca.cert.crt &&\
   sudo cp root_ca.cert.crt /usr/local/share/ca-certificates &&\
   sudo update-ca-certificates
```

## Generate keystore.jks for nexus
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

Goto https://nexus.example.local which will show a valid certificate!
