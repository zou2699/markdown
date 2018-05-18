## CA证书
### 在Docker守护进程的主机上生成CA私钥和公钥：
将$HOST以下示例中的所有实例替换为Docker守护进程主机的DNS名称。
```sh
$ openssl genrsa -aes256 -out ca-key.pem 4096
Generating RSA private key, 4096 bit long modulus
............................................................................................................................................................................................++
........++
e is 65537 (0x10001)
Enter pass phrase for ca-key.pem:
Verifying - Enter pass phrase for ca-key.pem:

$ openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
Enter pass phrase for ca-key.pem:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:Queensland
Locality Name (eg, city) []:Brisbane
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Docker Inc
Organizational Unit Name (eg, section) []:Sales
Common Name (e.g. server FQDN or YOUR name) []:$HOST
Email Address []:Sven@home.org.au
```
## server端证书
### 创建一个服务器密钥和证书签名请求（CSR）
```sh
$ openssl genrsa -out server-key.pem 4096
Generating RSA private key, 4096 bit long modulus
.....................................................................++
.................................................................................................++
e is 65537 (0x10001)

$ openssl req -subj "/CN=$HOST" -sha256 -new -key server-key.pem -out server.csr
```
### CA签署公钥
```sh
$ echo subjectAltName = DNS:$HOST,IP:10.10.10.20,IP:127.0.0.1 >> extfile.cnf
$ echo extendedKeyUsage = serverAuth >> extfile.cnf
$ openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -out server-cert.pem -extfile extfile.cnf
```
## client证书
```sh
$ openssl genrsa -out key.pem 4096
Generating RSA private key, 4096 bit long modulus
.........................................................++
................++
e is 65537 (0x10001)

$ openssl req -subj '/CN=client' -new -key key.pem -out client.csr
```
### CA签名证书
```sh
$ echo extendedKeyUsage = clientAuth >> extfile.cnf
$ openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -out cert.pem -extfile extfile.cnf
Signature ok
subject=/CN=client
Getting CA Private Key
Enter pass phrase for ca-key.pem:
```

## 防止证书被修改
```sh
$ chmod -v 0400 ca-key.pem key.pem server-key.pem
```
## 启动服务器端
```sh
$ dockerd --tlsverify --tlscacert=ca.pem --tlscert=server-cert.pem --tlskey=server-key.pem \
  -H=0.0.0.0:2376
```
### 使用systemd模式
修改/etc/docker/daemon.json
```json
{
    "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2376"],
    "tls": true,
    "tlsverify": true,
    "tlscacert":"/root/ca1/ca.pem",
    "tlscert": "/root/ca1/server-cert.pem",
    "tlskey": "/root/ca1/server-key.pem"
}
```
```sh
systemctl cat docker.service

first line: "# /lib/systemd/system/docker.service"

problem: ExecStart=/usr/bin/dockerd -H fd://

remove -H fd:// (comment out is not enough)

systemctl daemon-reload
systemctl restart docker.service
```
## 客户端连接
```sh
$ docker --tlsverify --tlscacert=ca.pem --tlscert=cert.pem --tlskey=key.pem \
  -H=$HOST:2376 version
```
### 默认启用安全连接
如果您想默认保护您的Docker客户端连接，您可以将文件移动到.docker您的主目录中的目录 - 并设置 DOCKER_HOST和DOCKER_TLS_VERIFY变量（而不是传递 -H=tcp://$HOST:2376和--tlsverify每次调用）。
```sh
$ mkdir -pv ~/.docker
$ cp -v {ca,cert,key}.pem ~/.docker

$ export DOCKER_HOST=tcp://$HOST:2376 DOCKER_TLS_VERIFY=1
```
如果找到，客户端发送它的客户端证书，所以你只需要放下你的密钥~/.docker/{ca,cert,key}.pem。或者，如果您想将密钥存储在其他位置，则可以使用环境变量指定该位置DOCKER_CERT_PATH。
```sh
$ export DOCKER_CERT_PATH=~/.docker/zone1/
$ docker --tlsverify ps
```