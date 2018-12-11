## keystone 安装

本次安装环境信息：

```shell
Linux version 4.9.93-linuxkit-aufs (root@XXX) (gcc version 6.4.0 (Alpine 6.4.0) )
```

1. 安装数据库。这里选择了mysql数据库
2. 创建keystone数据库

```mysql
mysql> CREATE DATABASE keystone;
```

3. 创建一个用户，并将keystone数据库授权给该用户，这里创建的用户名是keystone

```mysql
mysql> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'KEYSTONE_DBPASS';
mysql> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'KEYSTONE_DBPASS';
```



```mysql
mysql> use keystone
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+-----------------------------+
| Tables_in_keystone          |
+-----------------------------+
| access_token                |
| application_credential      |
| application_credential_role |
| assignment                  |
| config_register             |
| consumer                    |
| credential                  |
| endpoint                    |
| endpoint_group              |
| federated_user              |
| federation_protocol         |
| group                       |
| id_mapping                  |
| identity_provider           |
| idp_remote_ids              |
| implied_role                |
| limit                       |
| local_user                  |
| mapping                     |
| migrate_version             |
| nonlocal_user               |
| password                    |
| policy                      |
| policy_association          |
| project                     |
| project_endpoint            |
| project_endpoint_group      |
| project_tag                 |
| region                      |
| registered_limit            |
| request_token               |
| revocation_event            |
| role                        |
| sensitive_config            |
| service                     |
| service_provider            |
| system_assignment           |
| token                       |
| trust                       |
| trust_role                  |
| user                        |
| user_group_membership       |
| user_option                 |
| whitelisted_config          |
+-----------------------------+
44 rows in set (0.01 sec)
```

