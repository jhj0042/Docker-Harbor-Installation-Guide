# 安装Docker Harbor管理Dockers镜像

[toc]

## 1，[软硬件要求]

+ 硬件要求

| 资源 | 最小 | 推荐 |
| :---- | :----: | :----: |
| CPU | 2核 | 4核 |
| 内存 | 4GB | 8GB |
| 硬盘 | 40GB | 80GB |

+ 软件要求

| 软件 | 版本 | 描述 |
| :---- | :----: | :---- |
| Docker Engine | 版本17.06.0-ce+或更高 | 详细安装说明，见[此处1]|
| Docker Compose |docker-compose (v1.18.0+) 或 docker compose v2 (docker-compose-plugin) | 详细安装说明，见[此处2] |
| Openssl | 推荐使用最新版本 | 用以生成ssl证书 |

+ 网络端口

| 端口 | 协议 | 描述 |
| :---- | :----: | :---- |
| 443 | HTTPS | Harbor的门户和核心API通过此端口接受HTTPS请求。你可以在配置文件中修改此端口号。 |
| 4443 | HTTPS | Harbor与Docker内容信任服务的连接端口。你可以在配置文件中修改此端口号。 |
| 80 | HTTP | Harbor的门户和核心API通过此端口接受HTTP请求。你可以在配置文件中修改此端口号。 |

[软硬件要求]:https://goharbor.io/docs/2.6.0/install-config/installation-prereqs/
[此处1]:https://docs.docker.com/engine/installation/
[此处2]:https://docs.docker.com/compose/install/

## 2，下载离线安装包

你可以在[官方下载]页面 -> Assets下载**Docker Harbor**的离线安装包。

下载完成后上传到服务器任意位置，进入该目录后输入

```
tar xzvf harbor-offline-installer-<version>.tgz
```

解压，可以看到如下6个文件：`common.sh  harbor.<version>.tar.gz  harbor.yml  install.sh  LICENSE  prepare`

[官方下载]:https://github.com/goharbor/harbor/releases

<p id='jump2'></p>

## 3，配置HTTPS


默认情况下，**Harbor**并不附带证书。你可以在没有安全保护的情况下部署**Harbor**，这样你就可以通过HTTP来连接它。但是使用HTTP协议连接，这种情况只有在隔绝外网连接测试环境或开发环境下才是可以被接受的。在环境中使用HTTP协议会将你暴露在中间人攻击的风险之下。在生产环境下，请务必一直使用HTTPS协议。若你通过Notary启用了内容信任来妥善的对所有镜像签名，你必须使用HTTPS协议。

<!-- By default, Harbor does not ship with certificates. It is possible to deploy Harbor without security, so that you can connect to it over HTTP. However, using HTTP is acceptable only in air-gapped test or development environments that do not have a connection to the external internet. Using HTTP in environments that are not air-gapped exposes you to man-in-the-middle attacks. In production environments, always use HTTPS. If you enable Content Trust with Notary to properly sign all images, you must use HTTPS. -->

要配置HTTPS，你必须创建SSL证书。你可以使用由受信任的第三方CA签名的证书，也可以使用自签名证书。这一节将介绍如何使用**OpenSSL**来创建CA证书，以及如何通过你的CA证书对服务器证书和客户端证书进行签名。

<!-- To configure HTTPS, you must create SSL certificates. You can use certificates that are signed by a trusted third-party CA, or you can use self-signed certificates. This section describes how to use OpenSSL to create a CA, and how to use your CA to sign a server certificate and a client certificate. You can use other CA providers, for example Let’s Encrypt. -->

以下的过程中假设你的**Harbor**的镜像仓库为`yourdomain.com`，并且其DNS记录指向运行Harbor的主机。

<!-- The procedures below assume that your Harbor registry’s hostname is yourdomain.com, and that its DNS record points to the host on which you are running Harbor. -->

### 3.1，生成CA证书

在生产环境中，你必须从证书颁发机构（CA，*Certification Authority*）获得一个CA证书。在测试或开发环境，你可以自己生成。运行如下指令以生成一个CA证书：

<!-- In a production environment, you should obtain a certificate from a CA. In a test or development environment, you can generate your own CA. To generate a CA certficate, run the following commands. -->

1. 生成一个CA证书私钥

```
openssl genrsa -out ca.key 4096

```

2. 生成CA证书

修改指令中的`-subj`选项为你所在组织的信息，若你使用的是一个全限定域名（FQDN，*Fully Qualified Domain Name*）来连接你的**Harbor**主机，你需要将其指定为公用名(`CN`)属性。

<!-- Adapt the values in the -subj option to reflect your organization. If you use an FQDN to connect your Harbor host, you must specify it as the common name (CN) attribute. -->

```
openssl req -x509 -new -nodes -sha512 -days 3650 \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=yourdomain.com" \
    -key ca.key \
    -out ca.crt
```

### 3.2，生成服务器证书

证书通常包含一个`.crt`文件和一个`.key`文件，例如`yourdomain.com.crt`和`yourdomain.com.key`

1. 生成一个私钥

```
openssl genrsa -out yourdomain.com.key 4096
```

2. 生成一个证书签名请求（CSR,*certificate signing request*）

修改指令中的`-subj`选项为你所在组织的信息，若你使用的是一个全限定域名（FQDN，*Fully Qualified Domain Name*）来连接你的**Harbor**主机，你需要将其指定为公用名(`CN`)属性，并将其使用为key文件和CSR文件的文件名。

```
openssl req -sha512 -new \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=yourdomain.com" \
    -key yourdomain.com.key \
    -out yourdomain.com.csr
```

3. 生成x509 v3拓展文件

无论你是使用一个全限定域名或一个IP地址来访问你的**Harbor**主机，你必须创建这个文件，这样你才能为你的**Harbor**主机生成一个符合x509 v3拓展要求的包含证书主体别名（SAN，*Subject Alternative Name*）的证书。替换DNS为你的域名。

（*P.S.需要在别名中加入IP地址，否则远程登录的时候会报：`x509: cannot validate certificate for <your IP> because it doesn't contain any IP SANs`*）

<!-- Regardless of whether you’re using either an FQDN or an IP address to connect to your Harbor host, you must create this file so that you can generate a certificate for your Harbor host that complies with the Subject Alternative Name (SAN) and x509 v3 extension requirements. Replace the DNS entries to reflect your domain. -->

```
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
IP=<your IP>
DNS.1=yourdomain.com
DNS.2=yourdomain
DNS.3=hostname
EOF
```

4. 使用`v3.ext`文件来为你的**Harbor**主机生成证书

将CRS和CRT文件名中的`yourdomain.com`替换为你的**Harbor**主机。

```
openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in yourdomain.com.csr \
    -out yourdomain.com.crt
```

### 3.3，向Docker和Harbor提交证书

生成`ca.crt`，`yourdomain.com.crt`和`yourdomain.com.key`后，你需要向**Harbor**和**Docker**提交，并重配置**Harbor**以使用它们。

1. 将服务器证书和私钥导入**Harbor**主机的证书文件夹中

```
cp yourdomain.com.crt /data/cert/
cp yourdomain.com.key /data/cert/
```

2. 将`yourdomain.com.crt`转换为`yourdomain.com.cert`供**Docker**使用

Docker守护进程将`.crt`文件解释为CA证书，将`.cert`文件解释为客户端证书。

```
openssl x509 -inform PEM -in yourdomain.com.crt -out yourdomain.com.cert
```

3. 将服务器证书，私钥和CA文件放入**Harbor**主机的**Docker**证书文件夹中。你必须先将相应的文件夹建立好。

```
cp yourdomain.com.cert /etc/docker/certs.d/yourdomain.com/
cp yourdomain.com.key /etc/docker/certs.d/yourdomain.com/
cp ca.crt /etc/docker/certs.d/yourdomain.com/
```

如果你将默认的**Nginx**端口443转发到可另一个端口，建立`/etc/docker/certs.d/yourdomain.com:port`，或是`/etc/docker/certs.d/harbor_IP:port.`以替代。

4. 重启Docker Engine

```
systemctl restart docker
```

你可能同样需要在系统层面信任该证书，[详情请见此处]。

以下示例展示了一则使用自定义证书的配置：

```
/etc/docker/certs.d/
    └── yourdomain.com:port
       ├── yourdomain.com.cert  <-- Server certificate signed by CA
       ├── yourdomain.com.key   <-- Server key signed by CA
       └── ca.crt               <-- Certificate authority that signed the registry certificate
```

[详情请见此处]:https://goharbor.io/docs/2.6.0/install-config/troubleshoot-installation/#https

<p id='jump3'></p>

### 3.4，部署及重新配置Harbor

若你还没有部署**Harbor**，请见[如何配置]**Harbor**的yml文件来获得关于如何在`harbor.yml`文件中指定`hostname`和`https`属性来配置**Harbor**。

如已经部署好HTTP协议下的**Harbor**并期望重新配置以使用HTTPS协议，请按以下操作进行：

1. 运行`prepare`脚本来启用HTTPS

**Harbor**通过一个`Nginx`实例作为全部服务的反向代理。你可以使用`prepare`脚本来配置`Nginx`来启用HTTPS协议。`prepare`脚本与**Harbor**安装脚本在同一层目录之下。

```
./prepare
```

2. 若**Harbor**正在运行，将现有的实例暂停并移除

你的镜像数据仍旧保存在文件系统之中，所以不会有数据丢失。

```
docker-compose down -v
```

3. 重启**Harbor**

```
docker-compose up -d
```

[如何配置]:#jump1

### 3.5，验证HTTPS连接

在**Harbor**的HTTPS设置完成后，你可以通过以下步骤验证HTTPS连接：

+ 打开浏览器输入https://yourdomain.com，**Harbor**界面应当被展示出来。

一些浏览器会显示警告界面——未知的证书颁发机构，这是因为使用了自签名的CA证书，而不是从一个受信任的第三方证书颁发机构处。你可以在浏览器中导入证书以消除该警告。

+ 在运行**Docker守护程序**的机器上，检查`/etc/docker/daemon.json`确保https://yourdomain.com没有被设置为`-insecure-registry`。

+ 使用**Docker**客户端登录**Harbor**

```
docker login yourdomain.com
```

若你将`Nginx`443端口转发为其他端口，在`login`命令中加入该端口号

```
docker login yourdomain.com
```

## 4，配置Harbor

<p id='jump1'>在安装包里的`harbor.yml`文件内包含了<b>Harbor</b>系统层面的参数。这些参数将会在运行`install.sh`脚本或是重配置<b>Harbor</b>时生效。</p>

<!-- You set system level parameters for Harbor in the harbor.yml file that is contained in the installer package. These parameters take effect when you run the install.sh script to install or reconfigure Harbor. -->

在初次部署**Harbor**并启动后，你可以在**Harbor**门户网页上进行额外的配置。

下表列出了部署**Harbor**时必须设置的参数。默认情况下，`harbor.yml`文件中的所有的必需参数都未注释。可选参数用<kbd>#</kbd>注释了。你不一定需要更改默认的必需参数的值，但这些参数必须保持未注释状态。至少，你需要更新`hostname`参数。

<!-- The table below lists the parameters that must be set when you deploy Harbor. By default, all of the required parameters are uncommented in the harbor.yml file. The optional parameters are commented with #. You do not necessarily need to change the values of the required parameters from the defaults that are provided, but these parameters must remain uncommented. At the very least, you must update the hostname parameter. -->

**<font color = 'red'>重要</font>**：**Harbor**不附带任何证书。在1.9.x及以下版本中，**Harbor** 默认使用HTTP来处理镜像仓库请求。这仅在测试或开发环境中是可接受的。在生产环境中，始终使用 HTTPS。如果您通过Notary启用内容信任来妥善的为所有镜像签名，则必须使用 HTTPS。

<!-- IMPORTANT: Harbor does not ship with any certificates. In versions up to and including 1.9.x, by default Harbor uses HTTP to serve registry requests. This is acceptable only in air-gapped test or development environments. In production environments, always use HTTPS. If you enable Content Trust with Notary to properly sign all images, you must use HTTPS. -->

你可以使用来自受信任的第三方证书颁发机构颁发的证书，或可以使用自签名的证书。关于如何创建一个证书颁发机构来为服务器证书和客户端证书签名，请参照[配置HTTPS]。

<!-- You can use certificates that are signed by a trusted third-party CA, or you can use self-signed certificates. For information about how to create a CA, and how to use a CA to sign a server certificate and a client certificate, see Configuring Harbor with HTTPS Access. -->

### 必要参数

| 参数 | 子参数 | 描述以及额外的参数 |
| :---- | :---- | ---- |
| hostname | 无 | 指定要部署**Harbor**的目标主机的IP地址或全限定域名。这是访问**Harbor**门户和镜像仓库服务的地址。例如：`192.168.1.10`或`reg.yourdomain.com`。镜像仓库服务必须是能够外部访问的，因此不指定localhost，127.0.0.1或0.0.0.0作为主机名。 |
| HTTP |  | 不要在生产环境中使用 HTTP。仅在没有连接到外部网络的测试或开发环境中才可接受使用HTTP。在非隔绝环境中使用HTTP会使您暴露在中间人攻击的危险中。 |
|  | port | HTTP的端口号，用于**Harbor**门户和**Docker**命令。默认值为80。 |
| HTTPS |  | 使用HTTPS访问**Harbor**门户和令牌/通知服务。在生产环境和非隔绝环境中始终使用 HTTPS。 |
|  | port | HTTPS的端口号，用于**Harbor**门户和**Docker**命令。默认值为443。 |
|  | certicate | SSL证书的目录。 |
|  | private_key | 私钥的目录。 |
| harbor_admin _password	 | 无 | 为**Harbor**系统管理员设置密码。该密码仅在**Harbor**启动后的第一次生效，之后的登录中，该设置将被忽视，请在**Harbor**门户页面设置管理员密码。 |
| database |  | 使用本地**PostgreSQL**数据库。您可以选择配置外部数据库，在这种情况下，您可以禁用此选项。 |
|  | password | 为本地数据库设置root密码。在生产环境中你必须更改此密码。 |
|  | max_idle_conns | 空闲连接池中的最大连接数。如果设置为<=0，则不保留空闲连接。默认值为50。如果未设置，则为2。 |
|  | max_open_conns | 连接到数据库的最大打开连接数。如果<=0，则打开连接的数量没有限制。连接到**Harbor**数据库的最大连接数默认值为100。如果未配置，则值为 0。 |
| data_volume |  | 目标主机上存储**Harbor**数据的位置。即使删除和/或重新创建**Harbor**的容器，该数据也保持不变。您可以选择配置外部存储，在这种情况下禁用此选项并启用storage_service. 默认值为`/data`。 |
| trivy |  |配置Trivy扫描功能。 |
|  | ignore_unfixed | 将标识设置为`true`以仅展示修复的漏洞。默认值为`false`。 |
|  | skip_update | 您可能希望在测试或CI/CD环境中启用此标志以避免GitHub速率限制问题。如果启用该标志，您必须手动下载`trivy-offline.tar.gz`存档，解压缩`trivy.db`和`metadata.json`文件，并将它们安装在容器中的路径`/home/scanner/.cache/trivy/db/trivy.db`中。默认值为`false`。|
|  | insecure | 将标志设置`true`为跳过验证镜像证书。默认值为`false`。 |
|  | github_token | 设置GitHub访问令牌来下载Trivy DB。Trivy DB从GitHub上的Trivy发布页面下载。GitHub的匿名下载受到每小时60个请求的限制。通常这样的速率限制对于生产操作来说已经足够了。如果出于任何其他原因，这样的速率不足够，可以通过指定GitHub令牌将速率限制增加到每小时5000个请求。有关GitHub速率限制的详细信息，请[参阅]。您可以[按照此处的说明]创建GitHub令牌。 |
| jobservice | max_job_workers | 作业服务中复制工作的最大数量。对于每个镜像复制作业，工作将存储库的所有标签同步到远程目标。增加这个数字允许系统中更多的并发复制作业。但是由于每个工作都会消耗一定的网络/CPU/IO资源，所以根据主机的硬件资源来设置这个属性的值。默认值为10。 |
| notification | webhook_job_max_retry | 设置web hook作业的最大重试次数。默认值为10。 |
| chart | absolute_url | 设置`enabled`以使用Chart的绝对URL。设置`deactivated`以使用相对URL。 |
| log |  | 配置日志记录。Harbor使用 `rsyslog` 来收集每个容器的日志。 |
|  | level | 设置日志级别`debug`，`info`，`warning`，`error`，或`fatal`。默认值为`info`。 |
|  | local | 默认值为/var/log/harbor。 |
|  | external_endpoint | 启用此选项可将日志转发到系统日志服务器。 |
| cache |  | 为**Harbor**实例配置缓存层。当启用时，**Harbor**会通过Redis缓存一些**Harbor**资源（例如项目、项目元数据等），以减少对相同的**Harbor**资源提交的重复的请求所消耗的时间和资源，强烈推荐在同一时间有大量的拉取请求的情况下开启该功能，以提高**Harbor**实例的总体性能。更多关于**Harbor**缓存层的性能提升请参阅[cache layer wiki page] |
|  | enabled | 默认值为`false`，设置为`true`来启用**Harbor**缓存层。 |
|  | expire_hours | 设置缓存过期时间，小时计。默认为24小时。 |

### 可选参数

不重要，有空再说。

[配置HTTPS]:#jump2
[参阅]:https://developer.github.com/v3/#rate-limiting
[按照此处的说明]:https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line
[cache layer wiki page]:https://github.com/goharbor/perf/wiki/Cache-layer

## 5，安装Docker Harbor

安装**Harbor**的不同配置

+ 纯净的**Harbor**
+ **Harbor**+**Notart**
+ **Harbor**+**Trivy**
+ **Harbor**+**Chart**库服务
+ **Harbor**+**Notart**、**Trivy**、**Chart**库服务中的两个，或三个全部

纯净安装时，确认`harbor.yml`文件中的如下选项被正确设置了：

+ `hostname`
+ `https`及其下的
  + `port`
  + `certicate`
  + `private_key`

运行如下命令：

```
./install.sh
```

若安装成功，打开浏览器输入`<hostname>:<port>`即可访问**Harbor**门户

若安装失败，请检查命令行报错信息，修改后，[参照此处]重配置**Harbor**

[参照此处]:jump3

## 6，使用Harbor

在使用**Harbor**镜像仓库功能时，需要先修改本机的

```
/etc/docker/daemon.json
```

在`insecure-registries`中加入`<your IP>`

然后按以下步骤：

1. 按照`<your IP>/<library>/<image name>:<tag>`格式为镜像打标签

```
docker tag <image name>:<tag> <your IP>/<library>/<image name>:<tag>
```

2. 登录**Harbor**镜像仓库

```
docker login <your IP>
```

输入相应的用户名和密码

3. 上传镜像

```
docker push <your IP>/<library>/<image name>:<tag>
```

上传完成后可以在**Harbor**门户上看到对应的镜像了。