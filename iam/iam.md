##### 项目背景

Go应用的安全问题

- 服务自身的安全
  - 禁止非法用户访问服务
    - 服务器层面
      - 物理隔离
      - 网络隔离
      - 防火墙
    - 软件层面
      - HTTPS
      - 用户认证
- 服务资源的安全
  - 资源授权

为了保障Go应用的安全，需要对访问认证，对资源授权



##### IAM Identity and Access Management 身份识别与访问管理

- 三种资源
  - User
  - Secret
  - Policy

- 五个核心组件
  - iam-apiserver
    - 资源的crud
  - iam-authz-server
    - 授权服务
  - iam-pump
    - 从Redis中拉取缓存的授权日志，存入mongo数据库中
  - marmotedu-sdk-go
    - sdk
  - iamctl
    - 命令行
- 三个数据库
  - Redis
  - MySQL
  - MongoDB



eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2NDA4MjkwNTYsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2NDA3NDI2NTYsInN1YiI6ImFkbWluIn0.gEwDxxG_LGjmPPEjYaaFKP0DqcxMggPdo0k7rZ_Al0k



创建用户

$ curl -s -XPOST -H'Content-Type: application/json' -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2NDA4MjkwNTYsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2NDA3NDI2NTYsInN1YiI6ImFkbWluIn0.gEwDxxG_LGjmPPEjYaaFKP0DqcxMggPdo0k7rZ_Al0k' -d'{"password":"User@2021","metadata":{"name":"colin"},"nickname":"colin","email":"colin@foxmail.com","phone":"1812884xxxx"}' http://127.0.0.1:8080/v1/users



列出用户

$ curl -s -XGET -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2NDA4MjkwNTYsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2NDA3NDI2NTYsInN1YiI6ImFkbWluIn0.gEwDxxG_LGjmPPEjYaaFKP0DqcxMggPdo0k7rZ_Al0k' 'http://127.0.0.1:8080/v1/users?offset=0&limit=10'



获取 colin 用户的详细信息

$ curl -s -XGET -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2NDA4MjkwNTYsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2NDA3NDI2NTYsInN1YiI6ImFkbWluIn0.gEwDxxG_LGjmPPEjYaaFKP0DqcxMggPdo0k7rZ_Al0k' http://127.0.0.1:8080/v1/users/colin



修改 colin 用户

$ curl -s -XPUT -H'Content-Type: application/json' -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2NDA4MjkwNTYsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2NDA3NDI2NTYsInN1YiI6ImFkbWluIn0.gEwDxxG_LGjmPPEjYaaFKP0DqcxMggPdo0k7rZ_Al0k' -d'{"nickname":"colin","email":"colin_modified@foxmail.com","phone":"1812884xxxx"}' http://127.0.0.1:8080/v1/users/colin



删除 colin 用户

$ curl -s -XDELETE -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2NDA4MjkwNTYsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2NDA3NDI2NTYsInN1YiI6ImFkbWluIn0.gEwDxxG_LGjmPPEjYaaFKP0DqcxMggPdo0k7rZ_Al0k' http://127.0.0.1:8080/v1/users/colin



批量删除用户

$ curl -s -XDELETE -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2NDA4MjkwNTYsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2NDA3NDI2NTYsInN1YiI6ImFkbWluIn0.gEwDxxG_LGjmPPEjYaaFKP0DqcxMggPdo0k7rZ_Al0k' 'http://127.0.0.1:8080/v1/users?name=colin&name=mark&name=john'



创建 secret0 密钥

$ curl -s -XPOST -H'Content-Type: application/json' -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2NDA4MjkwNTYsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2NDA3NDI2NTYsInN1YiI6ImFkbWluIn0.gEwDxxG_LGjmPPEjYaaFKP0DqcxMggPdo0k7rZ_Al0k' -d'{"metadata":{"name":"secret0"},"expires":0,"description":"admin secret"}' http://127.0.0.1:8080/v1/secrets



列出所有密钥

$ curl -s -XGET -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2NDA4MjkwNTYsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2NDA3NDI2NTYsInN1YiI6ImFkbWluIn0.gEwDxxG_LGjmPPEjYaaFKP0DqcxMggPdo0k7rZ_Al0k' http://127.0.0.1:8080/v1/secrets



获取 secret0 密钥的详细信息

$ curl -s -XGET -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2NDA4MjkwNTYsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2NDA3NDI2NTYsInN1YiI6ImFkbWluIn0.gEwDxxG_LGjmPPEjYaaFKP0DqcxMggPdo0k7rZ_Al0k' http://127.0.0.1:8080/v1/secrets/secret0



修改 secret0 密钥

$ curl -s -XPUT -H'Content-Type: application/json' -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2NDA4MjkwNTYsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2NDA3NDI2NTYsInN1YiI6ImFkbWluIn0.gEwDxxG_LGjmPPEjYaaFKP0DqcxMggPdo0k7rZ_Al0k' -d'{"metadata":{"name":"secret0"},"expires":0,"description":"admin secret(modified)"}' http://127.0.0.1:8080/v1/secrets/secret0



删除 secret0 密钥

$ curl -s -XDELETE -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2NDA4MjkwNTYsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2NDA3NDI2NTYsInN1YiI6ImFkbWluIn0.gEwDxxG_LGjmPPEjYaaFKP0DqcxMggPdo0k7rZ_Al0k' http://127.0.0.1:8080/v1/secrets/secret0





创建策略

$ curl -s -XPOST -H'Content-Type: application/json' -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2NDA4MjkwNTYsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2NDA3NDI2NTYsInN1YiI6ImFkbWluIn0.gEwDxxG_LGjmPPEjYaaFKP0DqcxMggPdo0k7rZ_Al0k' -d'{"metadata":{"name":"policy0"},"policy":{"description":"One policy to rule them all.","subjects":["users:<peter|ken>","users:maria","groups:admins"],"actions":["delete","<create|update>"],"effect":"allow","resources":["resources:articles:<.*>","resources:printer"],"conditions":{"remoteIPAddress":{"type":"CIDRCondition","options":{"cidr":"192.168.0.1/16"}}}}}' http://127.0.0.1:8080/v1/policies



列出所有策略

$ curl -s -XGET -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2NDA4MjkwNTYsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2NDA3NDI2NTYsInN1YiI6ImFkbWluIn0.gEwDxxG_LGjmPPEjYaaFKP0DqcxMggPdo0k7rZ_Al0k' http://127.0.0.1:8080/v1/policies



获取 policy0 策略的详细信息

$ curl -s -XGET -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2NDA4MjkwNTYsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2NDA3NDI2NTYsInN1YiI6ImFkbWluIn0.gEwDxxG_LGjmPPEjYaaFKP0DqcxMggPdo0k7rZ_Al0k' http://127.0.0.1:8080/v1/policies/policy0



修改 policy0 策略

$ curl -s -XPUT -H'Content-Type: application/json' -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2NDA4MjkwNTYsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2NDA3NDI2NTYsInN1YiI6ImFkbWluIn0.gEwDxxG_LGjmPPEjYaaFKP0DqcxMggPdo0k7rZ_Al0k' -d'{"metadata":{"name":"policy0"},"policy":{"description":"One policy to rule them all(modified).","subjects":["users:<peter|ken>","users:maria","groups:admins"],"actions":["delete","<create|update>"],"effect":"allow","resources":["resources:articles:<.*>","resources:printer"],"conditions":{"remoteIPAddress":{"type":"CIDRCondition","options":{"cidr":"192.168.0.1/16"}}}}}' http://127.0.0.1:8080/v1/policies/policy0



删除 policy0 策略

$ curl -s -XDELETE -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2NDA4MjkwNTYsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2NDA3NDI2NTYsInN1YiI6ImFkbWluIn0.gEwDxxG_LGjmPPEjYaaFKP0DqcxMggPdo0k7rZ_Al0k' http://127.0.0.1:8080/v1/policies/policy0





$ curl -s -XPOST -H'Content-Type: application/json' -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsImtpZCI6ImRxQk00Zmo1SXljVjNWYmRrRVN3ckFYSmpsYXVha0JjMnJ5TSIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXV0aHoubWFybW90ZWR1LmNvbSIsImV4cCI6MTY0MDc1MDY0MiwiaWF0IjoxNjQwNzQzNDQyLCJpc3MiOiJpYW1jdGwiLCJuYmYiOjE2NDA3NDM0NDJ9.1aT2qXpBbV30o3u7abXftCtyqUCtM_X3P5JVHndFKxk' -d'{"subject":"users:maria","action":"delete","resource":"resources:articles:ladon-introduction","context":{"remoteIPAddress":"192.168.0.5"}}' http://127.0.0.1:9090/v1/authz
{"allowed":true}