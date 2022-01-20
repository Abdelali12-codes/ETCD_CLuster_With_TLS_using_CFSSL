# ETCD_CLuster_With_TLS_using_CFSSL

## install the prerequisites

- install the etcd and the etcdctl

```
{
 wget https://github.com/etcd-io/etcd/releases/download/v3.4.16/etcd-v3.4.16-linux-amd64.tar.gz
 tar xzf etcd*
 sudo mv etcd-v3.4.16-linux-amd64/etcd* /usr/local/bin
 rm -rf etcd*
}
```

- install the cfssl and cfssljson

```
{
  wget -q --show-progress \
    https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssl \
    https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssljson

  chmod +x cfssl cfssljson
  sudo mv cfssl cfssljson /usr/local/bin/
}
```

## create certificate authority

```
{
cat ca-config <<EOF
   {
    "signing": {
        "default": {
            "expiry": "43800h"
        },
        "profiles": {
            "server": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "peer": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            }
        }
    }
 }
EOF

cat > ca-csr.json <<EOF

   {
    "CN": "abdelali.com",
    "hosts": [
        "abdelali.com",
        "www.abdelali.com"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "US",
            "ST": "CA",
            "L": "San Francisco"
        }
    ]
   }

EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

}

```


## create server, peer and client certificates

```
{
cat server.json <<EOF
   {
    "CN": "server",
    "hosts": [
        "abdelali.com"
        "127.0.0.1",
        "192.168.108.163"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "MR",
            "ST": "CA",
            "L": "Casablanca"
        }
    ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server server.json | cfssljson -bare server


cat peer.json <<EOF
   {
    "CN": "peer",
    "hosts": [
        "abdelali.com"
        "127.0.0.1",
        "192.168.108.163"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "MR",
            "ST": "CA",
            "L": "Casablanca"
        }
    ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=mpeer peer.json | cfssljson -bare peer


cat client.json <<EOF
   {
    "CN": "client",
    "hosts": [
        "abdelali.com"
        "127.0.0.1",
        "192.168.108.163"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "MR",
            "ST": "CA",
            "L": "Casablanca"
        }
    ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client client.json | cfssljson -bare client

}

```
[Unit]
Description=etcd service
Documentation=https://github.com/coreos/etcd
[Service]
User=root
Type=notify
ExecStart=/usr/local/bin/etcd \
 --name peer1 \
 --data-dir /var/lib/etcd \
 --initial-advertise-peer-urls https://192.168.108.163:2380 \
 --listen-peer-urls https://192.168.108.163:2380 \
 --listen-client-urls https://192.168.108.163:2379,https://127.0.0.1:2379 \
 --advertise-client-urls https://192.168.108.163:2379 \
 --initial-cluster-token etcd-cluster-1 \
 --initial-cluster peer1=https://192.168.108.163:2380 \
 --client-cert-auth \
 --trusted-ca-file=/var/lib/etcd/cfssl/ca.pem \
 --cert-file=/var/lib/etcd/cfssl/server.pem \
 --key-file=/var/lib/etcd/cfssl/server-key.pem \
 --peer-client-cert-auth \
 --peer-trusted-ca-file=/var/lib/etcd/cfssl/ca.pem \
 --peer-cert-file=/var/lib/etcd/cfssl/peer.pem \
 --peer-key-file=/var/lib/etcd/cfssl/peer.pem \
 --initial-cluster-state new
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target

```

