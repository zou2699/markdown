

使用TLS和管理CA是一个高级主题。在使用它之前，请熟悉OpenSSL，x509和TLS。

使用OpenSSL创建CA，服务器和客户端密钥
注意：将$HOST以下示例中的所有实例替换为Docker守护进程主机的DNS名称。

## 首先，在Docker守护进程的主机上生成CA私钥和公钥：
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
现在您拥有一个CA，您可以创建一个服务器密钥和证书签名请求（CSR）。确保“通用名称”与用于连接到Docker的主机名相匹配：

注意：将$HOST以下示例中的所有实例替换为Docker守护进程主机的DNS名称。
```sh
$ openssl genrsa -out server-key.pem 4096
Generating RSA private key, 4096 bit long modulus
.....................................................................++
.................................................................................................++
e is 65537 (0x10001)

$ openssl req -subj "/CN=$HOST" -sha256 -new -key server-key.pem -out server.csr
```
接下来，我们将用我们的CA签署公钥：

由于TLS连接可以通过IP地址和DNS名称进行，因此创建证书时需要指定IP地址。例如，要允许使用10.10.10.20和连接127.0.0.1：
(listen ip)
```sh
$ echo subjectAltName = DNS:$HOST,IP:10.10.10.20,IP:127.0.0.1 >> extfile.cnf
```
将Docker守护进程密钥的扩展使用属性设置为仅用于服务器身份验证：
```sh
$ echo extendedKeyUsage = serverAuth >> extfile.cnf
```
现在，生成签名证书：
```sh
$ openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -out server-cert.pem -extfile extfile.cnf
Signature ok
subject=/CN=your.host.com
Getting CA Private Key
Enter pass phrase for ca-key.pem:
```
授权插件提供更细致的控制，以补充相互TLS的认证。除了上述文档中介绍的其他信息之外，在Docker守护程序上运行的授权插件会接收用于连接Docker客户端的证书信息。

对于客户端身份验证，请创建客户端密钥和证书签名请求：

注意：为了简化接下来的几个步骤，您也可以在Docker守护进程的主机上执行此步骤。
```sh
$ openssl genrsa -out key.pem 4096
Generating RSA private key, 4096 bit long modulus
.........................................................++
................++
e is 65537 (0x10001)

$ openssl req -subj '/CN=client' -new -key key.pem -out client.csr
```
要使密钥适合客户端认证，请创建一个扩展配置文件：
```sh
$ echo extendedKeyUsage = clientAuth >> extfile.cnf
```
现在，生成签名证书：
```sh
$ openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -out cert.pem -extfile extfile.cnf
Signature ok
subject=/CN=client
Getting CA Private Key
Enter pass phrase for ca-key.pem:
```
生成后cert.pem，server-cert.pem您可以安全地删除两个证书签名请求：
```sh
$ rm -v client.csr server.csr
```
默认umask值为022时，您和您的团队的秘密密钥是世界可读和可写的。

为防止意外损坏您的密钥，请删除其写入权限。要让它们只能被你读取，请按如下方式更改文件模式：
```sh
$ chmod -v 0400 ca-key.pem key.pem server-key.pem
```
证书可以是世界可读的，但您可能想要删除写入访问以防止意外损坏：
```sh
$ chmod -v 0444 ca.pem server-cert.pem cert.pem
```
现在你可以让Docker守护进程只接受来自提供你的CA信任的证书的客户端的连接：
```sh
$ dockerd --tlsverify --tlscacert=ca.pem --tlscert=server-cert.pem --tlskey=server-key.pem \
  -H=0.0.0.0:2376
  ```
要连接到Docker并验证其证书，请提供您的客户端密钥，证书和可信CA：

在客户机上运行它

这一步应该在你的Docker客户端机器上运行。因此，您需要将CA证书，服务器证书和客户端证书复制到该机器。

注意：将$HOST以下示例中的所有实例替换为Docker守护进程主机的DNS名称。
```sh
$ docker --tlsverify --tlscacert=ca.pem --tlscert=cert.pem --tlskey=key.pem \
  -H=$HOST:2376 version
  ```
注意：通过TLS的Docker应该在TCP端口2376上运行。

警告：如上例所示，当您使用证书身份验证时，无需docker使用sudo或docker组运行客户端。这意味着任何拥有密钥的人都可以给你的Docker守护进程提供任何指令，让他们可以访问托管守护进程的机器。保护这些密钥，就像你使用root密码一样！

安全默认
如果您想默认保护您的Docker客户端连接，您可以将文件移动到.docker您的主目录中的目录 - 并设置 DOCKER_HOST和DOCKER_TLS_VERIFY变量（而不是传递 -H=tcp://$HOST:2376和--tlsverify每次调用）。
```sh
$ mkdir -pv ~/.docker
$ cp -v {ca,cert,key}.pem ~/.docker

$ export DOCKER_HOST=tcp://$HOST:2376 DOCKER_TLS_VERIFY=1
```
Docker默认安全连接：
```sh
$ docker ps
```
其他模式
如果你不想有完整的双向认证，你可以通过混合标志来以各种其他模式运行Docker。

守护进程模式
tlsverify，tlscacert，tlscert，tlskey集：验证客户端
tls，tlscert，tlskey：不要验证客户端
客户端模式
tls：基于公共/默认CA池对服务器进行身份验证
tlsverify，tlscacert：根据给定的CA验证服务器
tls，tlscert，tlskey：以验证客户端证书，不验证服务器基于给定CA
tlsverify，tlscacert，tlscert，tlskey：以验证客户端证书并认证服务器基于给定CA
如果找到，客户端发送它的客户端证书，所以你只需要放下你的密钥~/.docker/{ca,cert,key}.pem。或者，如果您想将密钥存储在其他位置，则可以使用环境变量指定该位置DOCKER_CERT_PATH。
```sh
$ export DOCKER_CERT_PATH=~/.docker/zone1/
$ docker --tlsverify ps
```
使用连接到安全的Docker端口 curl
要使用curl测试API请求，您需要使用三个额外的命令行标志：
```sh
$ curl https://$HOST:2376/images/json \
  --cert ~/.docker/cert.pem \
  --key ~/.docker/key.pem \
  --cacert ~/.docker/ca.pem
```