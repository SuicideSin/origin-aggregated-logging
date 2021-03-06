# Generating JKS When Container Starts

## Overview
This proposal is the design to support [creating Java Keystores](https://trello.com/c/6a9bxQdX/) on container start from secrets that are mounted into the container.  This will:

* Allow the logging stack to consume certificates provided by an OpenShift certificate generation service
* Remove the dependency on Java for the OpenShift installer.
* Remove the need for the deployer pod and Ansible role to generate certificates and Java Keystores
* Allow the cluster to regenerate certificates when they expire

## Assumptions
* Something (e.g. intially OpenShift installer) will generate [client certificates](https://trello.com/c/QddRRfrS/#comment-58912b58b3005f71db0f81e6) and
populate secrets.

## Design
### Elasticsearch Configuration
Elasticsearch is configured such that it will utilize the keystore and truststore generated by the ```run.sh``` script at startup.

### Container Resource Definition:
Generated client certificates will be mounted as:
```
/etc/elasticsearch/secrets/client-cert/<ALIAS>/tls.{cert,key}
```
For example:
```
/etc/elasticsearch/secrets/client-cert
  |- elasticsearch
  |    |- tls.cert
  |    |- tls.key
  |
  |- fluentd
  |    |- tls.cert
  |    |- tls.key
  |
  |- kibana
  |    |- tls.cert
  |    |- tls.key
  |
  |- curator
  |    |- tls.cert
  |    |- tls.key
  |
```

Generated client CAs will be mounted as:
```
/etc/elasticsearch/secrets/client-ca/<ALIAS>/ca.crt
```
For example:
```
/etc/elasticsearch/secrets/client-ca
  |- elasticsearch
  |    |- ca.crt
  |
  |- fluentd
  |    |- ca.crt
  |
  |- kibana
  |    |- ca.crt
  |
  |- curator
  |    |- ca.crt
  |
```

### Container Starts
In the ```run.sh``` before starting Elasticsearch:

**Psuedo Code:**
```
  if $CLIENT_CERT_DIR has directories {
    for each $DIR in $CLIENT_CERT_DIR {
      basename = basename($DIR)
      if $DIR has tls.cert & tls.key & exists?($CLIENT_CA_DIR/basename/ca.cert)) {

        prep_certs_for_import_to_keystore
        keytool import --alias=basename tls.cert tls.key --out_file=keystore

        prep_certs_for_import_to_truststore
        keytool import --alias=basename tls.cert ca.cert --out_file=truststore
      }
    }
  }
```
### References
[1] [Convert cert to PKCS12](http://stackoverflow.com/a/17710626/262768)

[2] [PEM to Java Keystore Util](https://github.com/jimmidyson/pemtokeystore)
