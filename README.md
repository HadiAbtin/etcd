# etcd
etcd cluster deployment and manual

### prepare CA certificates
``` bash
openssl req -newkey rsa:2048 -nodes -keyout etcd-ca.key -subj "/CN=etcd-ca" -days 3650 -out etcd-ca.crt
```

### create CSR for etcd servers
First create openssl config for certificate specifications
``` bash
# vi openssl-etcd-server.conf
[req]
req_extentions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = localhost
DNS.2 = etcd01
DNS.3 = etcd02
DNS.4 = etcd03
DNS.5 = etcd04
DNS.6 = etcd05
DNS.7 = etcd01.abtinfar.com
DNS.8 = etcd02.abtinfar.com
DNS.9 = etcd03.abtinfar.com
DNS.10 = etcd04.abtinfar.com
DNS.11 = etcd05.abtinfar.com
IP.1 = 127.0.0.1
IP.2 = 110.0.0.1
IP.3 = 110.0.0.2
IP.4 = 110.0.0.3
IP.5 = 110.0.0.4 
IP.6 = 110.0.0.5 
```

``` bash
openssl req -newkey rsa:2048 -nodes -keyout etcd-server.key -subj "/CN=etcd-server" -config openssl-etcd-server.conf -out etcd-server.csr
```

### sign the request by CA certificate
``` bash
openssl x509 -req -in etcd-server.csr -CA etcd-ca.crt -CAkey etcd-ca.key -CAcreateserial -days 3650 -extensions v3_req -extfile openssl-etcd-server.conf -out etcd-server.crt
```
Transfer `etcd-ca.crt`, `etcd-server.crt` and `etcd-server.key` to all etcd server

### create admin certificate (for remote management)
``` bash
openssl req -newkey rsa:2048 -nodes -keyout etcd-admin.key -subj "/CN=admin" -out etcd-admin.csr   # create csr
openssl x509 -req -in etcd-admin.csr  -CA etcd-ca.crt -CAkey etcd-ca.key -CAcreateserial -days 3650 -out etcd-admin.crt  # sign the csr
```

## check certificates
``` bash
openssl x509 -text -noout -in etcd-ca.crt
openssl x509 -text -noout -in etcd-admin.crt
openssl x509 -text -noout -in etcd-server.crt
```

 
