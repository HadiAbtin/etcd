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

### check certificates
``` bash
openssl x509 -text -noout -in etcd-ca.crt
openssl x509 -text -noout -in etcd-admin.crt
openssl x509 -text -noout -in etcd-server.crt
```
### download latest etcd release
Download the version you need from [project repository](https://github.com/etcd-io/etcd/releases). Download and do below procedure on all server(etcd and admin servers)
``` bash
wget "https://github.com/etcd-io/etcd/releases/download/v3.5.10/etcd-v3.5.10-linux-amd64.tar.gz"
tar zxvf etcd-v3.5.10-linux-amd64.tar.gz
cd etcd-v3.5.10-linux-amd64/
mv etcdctl etcd /opt/    # for admin server just move etcdctl
```
### create systemd service 
``` bash
NAME=$(hostname -s)
IPADDR=$(ip -j -p a show ens19 | jq .[0].addr_info[0].local | sed 's/"//g')  # check your own interface name. my interface name was *ens19*
cat << EOF > /etc/systemd/system/etcd.service
[unit]
Description=etcd

[Service]
Type=notify
User=etcd
Restart=on-failure
RestartSec=5s
LimitNOFILE=50000
TimwoutStartSec=0
ExecStart=/usr/local/bin/etcd \\
  --name=$NAME \\
  --data-dir=/var/lib/etcd \\
  --client-cert-auth \\
  --peer-client-cert-auth \\
  --cert-file=/etc/etcd/certs/etcd-server.crt \\
  --key-file=/etc/etcd/certs/etcd-server.key \\
  --trusted-ca-file=/etc/etcd/certs/etcd-ca.crt \\
  --peer-cert-file=/etc/etcd/certs/etcd-server.crt \\
  --peer-key-file=/etc/etcd/certs/etcd-server.key \\
  --peer-trusted-ca-file=/etc/etcd/certs/etcd-ca.crt \\
  --advertise-client-urls=https://$IPADDR:2379 \\
  --listen-client-urls=https://localhost:2379,https://$IPADDR:2379 \\
  --listen-peer-urls=https://$IPADDR:2380 \\
  --initial-advertise-peer-urls=https://$IPADDR:2380 \\
  --initial-cluster etcd01=https://110.0.0.1:2380,etcd02=https://110.0.0.2:2380,etcd03=https://110.0.0.3:2380,etcd04=https://110.0.0.4:2380,etcd05=https://110.0.0.5:2380 \\
  --initial-cluster-token=etcd-cluster \\
  --initial-cluster-state=new \\


[Install]
WantedBy=multi-user.target
EOF
```

### reload systemd and start services on all servers 
``` bash
systemctl daemon-reload
systemctl start etcd
systemctl enable etcd   # start service after restart the server
```
### check status and performance of cluster
``` bash
 ETCDCTL_API=3 /opt/etcdctl --cacert etcd-ca.crt --cert etcd-admin.crt --key etcd-admin.key --endpoints https://110.0.0.1:2379,https://110.0.0.2:2379,https://110.0.0.3:2379,https://110.0.0.4:2379,https://110.0.0.5:2379 -w table member list
```
![image](https://github.com/HadiAbtin/etcd/assets/151436034/1588cee9-6091-479c-bb8b-15262c14dd33)

``` bash
ETCDCTL_API=3 /opt/etcdctl --cacert etcd-ca.crt --cert etcd-admin.crt --key etcd-admin.key --endpoints https://110.0.0.1:2379,https://110.0.0.2:2379,https://110.0.0.3:2379,https://110.0.0.4:2379,https://110.0.0.5:2379 -w table endpoint status
```
![image](https://github.com/HadiAbtin/etcd/assets/151436034/e5003b93-9797-45d5-9fa1-3dc83ebd60a4)

Check performance
``` bash
ETCDCTL_API=3 /opt/etcdctl --cacert etcd-ca.crt --cert etcd-admin.crt --key etcd-admin.key --endpoints https://110.0.0.1:2379,https://110.0.0.2:2379,https://110.0.0.3:2379,https://110.0.0.4:2379,https://110.0.0.5:2379 -w table check perf
```

Check the memory usage of holding data for different workloads on a given server endpoint.
``` bash
ETCDCTL_API=3 /opt/etcdctl --cacert etcd-ca.crt --cert etcd-admin.crt --key etcd-admin.key --endpoints https://110.0.0.1:2379,https://110.0.0.2:2379,https://110.0.0.3:2379,https://110.0.0.4:2379,https://110.0.0.5:2379 -w table check datascale --insecure-skip-tls-verify
```
