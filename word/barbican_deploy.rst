========================
OpenStack Barbican部署
========================

Barbican项目用来管理证书，使用场景包括LBaasV2等。

软件安装
========

::

   yum install openstack-barbican

CentOS源中的barbican RPM 包有 bug:

::

   Could not load 'simple_certificate_event': cannot import name certificate_manager

需要拉取M版本的barbican，然后使用pip安装，安装的时候出错

::

   Collecting backports.ssl_match_host_name (from ldap3>=0.9.8.2->barbican==2.0.1.dev9)
     Could not find a version that satisfies the requirement backports.ssl_match_host_name (from ldap3>=0.9.8.2->barbican==2.0.1.dev9) (from versions: )
   No matching distribution found for backports.ssl_match_host_name (from ldap3>=0.9.8.2->barbican==2.0.1.dev9)

解决办法

::

   pip install ldap3==1.0.2

数据库
======

需要手动创建数据库、用户名和密码

::

   mysql> create database barbican;
   mysql> grant all privileges on barbican.* to 'barbican'@'%' identified by 'barbican';

然后修改配置文件 `/usr/lib/python2.7/site-packages/barbican/model/migration/alembic.ini`

::
   # 根据实际情况，调整数据库的安装节点的IP地址
   sqlalchemy.url = mysql+pymysql://barbican:barbican@192.168.0.3/barbican

再使用配置文件在数据库中创建各种表

::

   alembic -c /usr/lib/python2.7/site-packages/barbican/model/migration/alembic.ini upgrade head

创建用户和服务
==============

::

   openstack user create --project service --password barbican barbican
   openstack role add --project service --user barbican admin

   openstack service create --name barbican --description 'Barbican Service' key-manager

   # 将下面的 `192.168.0.3` 改为控制节点的地址，如果控制节点采用HA模式的话，则为VIP地址
   openstack endpoint create --region RegionOne barbican public http://192.168.0.3:9311
   openstack endpoint create --region RegionOne barbican internal http://192.168.0.3:9311

配置文件
========

这里列出的是需要手动修改的选项，其他的采用 RPM 自动生成的默认配置即可，也可以根据具体情况酌情修改。

- /etc/barbican/barbican.conf

  ::

     [DEFAULT]
     # 将下面的 localhost 改为数据库所在主机的地址
     sql_connection = mysql+pymysql://barbican:barbican@localhost/barbican

- /etc/barbican/gunicorn-config.py

  ::

     import multiprocessing

     bind = '0.0.0.0:9311'
     user = 'barbican'
     group = 'barbican'

     timeout = 30
     backlog = 2048
     keepalive = 2

     workers = multiprocessing.cpu_count() * 2

     loglevel = 'debug'
     capture_output = True
     errorlog = '/var/log/barbican/api.log'

- /etc/barbican/barbican-api-paste.ini

  ::

     [pipeline:barbican_api]
     pipeline = keystone_authtoken context apiapp

     [filter:keystone_authtoken]
     paste.filter_factory = keystonemiddleware.auth_token:filter_factory
     identity_uri = http://localhost:35357 # localhost改成控制节点的IP地址，如果HA部署，则为VIP地址
     admin_tenant_name = service
     admin_user = barbican
     admin_password = barbican
     auth_version = v3.0

注意其中的 `identity_uri` 改为 keystone 的地址

运行服务
========

先手动创建 log 文件

::

   touch /var/log/barbican/api.log
   chmod 664 /var/log/barbican/api.log
   chown barbican:barbican /var/log/barbican/api.log

创建 `/run/barbican`

::

   mkdir /run/barbican
   chown barbican:barbican /run/barbican

运行服务

::

   systemctl enable openstack-barbican-api
   systemctl start openstack-barbican-api

测试
====

生成证书

参考：

- http://www.ruanyifeng.com/blog/2011/08/what_is_a_digital_signature.html
- http://blog.csdn.net/liuchunming033/article/details/48470575

命令如下

::

   openssl genrsa -out ca.key
   openssl req -new -key ca.key -out ca.csr
   openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt

   openssl genrsa -out server.key
   openssl req -new -key server.key -out server.csr
   touch /etc/pki/CA/index.txt
   echo 00 > /etc/pki/CA/serial
   openssl ca -in server.csr -out server.crt -cert ca.crt -keyfile ca.key

   openssl crl2pkcs7 -nocrl -certfile ca.crt -out ca-chain.p7b


操作过程的LOG，包括需要手动输入的一些参数

::

   /ssh:ocon: #$ openssl genrsa -out ca.key
   Generating RSA private key, 1024 bit long modulus
   ..............++++++
   ....++++++
   e is 65537 (0x10001)
   /ssh:ocon: #$ openssl req -new -key ca.key -out ca.csr
   You are about to be asked to enter information that will be incorporated
   into your certificate request.
   What you are about to enter is what is called a Distinguished Name or a DN.
   There are quite a few fields but you can leave some blank
   For some fields there will be a default value,
   If you enter '.', the field will be left blank.
   -----
   Country Name (2 letter code) [XX]:CN
   State or Province Name (full name) []:Shanghai
   Locality Name (eg, city) [Default City]:Shanghai
   Organization Name (eg, company) [Default Company Ltd]:Huayun
   Organizational Unit Name (eg, section) []:NeoCU
   Common Name (eg, your name or your server's hostname) []:ocon
   Email Address []:

   Please enter the following 'extra' attributes
   to be sent with your certificate request
   A challenge password []:
   An optional company name []:
   /ssh:ocon: #$ openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
   Signature ok
   subject=/C=CN/ST=Shanghai/L=Shanghai/O=Huayun/OU=NeoCU/CN=ocon
   Getting Private key
   /ssh:ocon: #$ openssl genrsa -out server.key
   Generating RSA private key, 1024 bit long modulus
   ...++++++
   ...........++++++
   e is 65537 (0x10001)
   /ssh:ocon: #$ openssl req -new -key server.key -out server.csr
   You are about to be asked to enter information that will be incorporated
   into your certificate request.
   What you are about to enter is what is called a Distinguished Name or a DN.
   There are quite a few fields but you can leave some blank
   For some fields there will be a default value,
   If you enter '.', the field will be left blank.
   -----
   Country Name (2 letter code) [XX]:CN
   State or Province Name (full name) []:Shanghai
   Locality Name (eg, city) [Default City]:Shanghai
   Organization Name (eg, company) [Default Company Ltd]:Huayun
   Organizational Unit Name (eg, section) []:NeoCU
   Common Name (eg, your name or your server's hostname) []:ocon
   Email Address []:

   Please enter the following 'extra' attributes
   to be sent with your certificate request
   A challenge password []:
   An optional company name []:
   /ssh:ocon: #$ openssl ca -in server.csr -out server.crt -cert ca.crt -keyfile ca.key
   Using configuration from /etc/pki/tls/openssl.cnf
   Check that the request matches the signature
   Signature ok
   Certificate Details:
   Serial Number: 1 (0x1)
   Validity
   Not Before: Jul  4 08:27:20 2017 GMT
   Not After : Jul  4 08:27:20 2018 GMT
   Subject:
   countryName               = CN
   stateOrProvinceName       = Shanghai
   organizationName          = Huayun
   organizationalUnitName    = NeoCU
   commonName                = ocon
   X509v3 extensions:
   X509v3 Basic Constraints:
   CA:FALSE
   Netscape Comment:
   OpenSSL Generated Certificate
   X509v3 Subject Key Identifier:
   24:A7:AA:36:BA:79:CE:90:2B:8F:6D:89:C8:B3:C1:01:37:37:2F:73
   X509v3 Authority Key Identifier:
   DirName:/C=CN/ST=Shanghai/L=Shanghai/O=Huayun/OU=NeoCU/CN=ocon
   serial:FC:3D:C4:70:75:C4:77:36

   Certificate is to be certified until Jul  4 08:27:20 2018 GMT (365 days)
   Sign the certificate? [y/n]:y


   1 out of 1 certificate requests certified, commit? [y/n]y
   Write out database with 1 new entries
   Data Base Updated
   /ssh:ocon: #$ openssl crl2pkcs7 -nocrl -certfile ca.crt -out ca-chain.p7b

使用 LBaasV2 测试

::

    openstack secret store --name='cert1' --payload-content-type='text/plain' --payload="$(cat server.crt)"
    openstack secret store --name='key1' --payload-content-type='text/plain' --payload="$(cat server.key)"
    openstack secret store --name='intermediates1' --payload-content-type='text/plain' --payload="$(cat ca-chain.p7b)"
    openstack secret container create --name='tls_container1' --type='certificate' --secret="certificate=$(openstack secret list | awk '/ cert1 / {print $2}')" --secret="private_key=$(openstack secret list | awk '/ key1 / {print $2}')" --secret="intermediates=$(openstack secret list | awk '/ intermediates1 / {print $2}')"

    openstack acl user add -u admin_id $(openstack secret list | awk '/ cert1 / {print $2}')
    openstack acl user add -u admin_id $(openstack secret list | awk '/ key1 / {print $2}')
    openstack acl user add -u admin_id $(openstack secret list | awk '/ intermediates1 / {print $2}')
    openstack acl user add -u admin_id $(openstack secret container list | awk '/ tls_container1 / {print $2}')

    neutron lbaas-loadbalancer-create --name gd_lb_1 $lb_subnet
    neutron lbaas-listener-create --name gd_listener_1 --loadbalancer gd_lb_1 --protocol-port 443 --protocol TERMINATED_HTTPS --default-tls-container=$(openstack secret container list | awk '/ tls_container1 / {print $2}')
    neutron lbaas-pool-create --name gd_pool_1 --lb-algorithm ROUND_ROBIN --listener gd_listener_1 --protocol HTTP

    neutron lbaas-member-create --subnet $lb_subnet --address 20.0.0.121 --protocol-port 80 gd_pool_1 --name gd_lb_member_1
    neutron lbaas-member-create --subnet $lb_subnet --address 20.0.0.122 --protocol-port 80 gd_pool_1 --name gd_lb_member_2
