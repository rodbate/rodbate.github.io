---
title: 对象存储服务MinIO介绍
date: 2020-02-05 09:45:14
tags: 
    - OSS
    - MinIO
---

## 对象存储服务MinIO

> `Open Source, S3 Compatible, Enterprise Hardened and Really, Really Fast`
>
> MinIO是一种高性能并兼容Amazon S3 Api的对象存储服务
>
> [官网地址](https://min.io/)  [Github项目地址](https://github.com/minio/minio)





### MinIO简介

#### 1. Erasure Coding(纠删码)

MinIO数据存储数据保护数据采用的是Erasure Coding(纠删码)技术，与之对应人们熟知的是Replication(数据副本)技术, 两者之间的区别就不再赘述了。

引用官方文档纠删码的应用解析

>MinIO protects data with per-object, inline erasure coding which is written in assembly code to deliver the highest performance possible. MinIO uses Reed-Solomon code to stripe objects into n/2 data and n/2 parity blocks - although these can be configured to any desired redundancy level. This means that in a 12 drive setup, an object is sharded across as 6 data and 6 parity blocks. Even if you lose as many as 5 ((n/2)–1) drives, be it parity or data, you can still reconstruct the data reliably from the remaining drives. MinIO’s implementation ensures that objects can be read or new objects written even if multiple devices are lost or unavailable. Finally, MinIO's erasure code is at the object level and can heal one object at a time.

说白了就是纠删码技术空间占用率低,容错率高。MinIO使用[Reed-Solomon](https://en.wikipedia.org/wiki/Reed–Solomon_error_correction)编码, 原始数据块=n/2, 校验数据块=n/2,一个对象条形存储在这些快上,容错率为(n/2 - 1)。

![](/blog/images/minio/erasure-code.svg)



###  MinIO服务部署

![](/blog/images/minio/architecture_diagram.svg)



>  `分布式部署适合生产场景`

**`分布式部署注意点：`**

1. 所有参与部署的节点都必须设置相同访问秘钥(即access key和secret key相同);
2. MinIO以`4, 6, 8, 10, 12, 14, 16`的最大公约数创建EC数据集合(每个对象的存储MinIO采用不可逆的哈希算法分到相应的数据集中), 因此MinIO单个集群的硬盘数必须是`4, 6, 8, 10, 12, 14, 16`其中一个数的倍数;
3. MinIO会选择最大的EC数据集合,比如8块硬盘,那么MinIO就会创建1个大小为8的EC数据集合;
4. 一个对象只会写到一个EC数据集合中, 也就是说一个对象不会存储超过16个硬盘;
5. 所有参与节点推荐为同构系统
6. MinIO数据目录要求为独占
7. MinIO节点之间的时间差异不能超过15分钟, 可以使用NTP时间同步
8. 设置环境变量`MINIO_DOMAIN`可以支持DNS风格的Bucket Federation模式



引用官方示例

![](/blog/images/minio/Architecture-diagram_distributed_32.png)


单集群分布式部署示例

```bash
export MINIO_ACCESS_KEY=<ACCESS_KEY>
export MINIO_SECRET_KEY=<SECRET_KEY>
minio server http://host{1...32}/export{1...32}
```



MinIO分布式模式支持集群扩展

```bash
export MINIO_ACCESS_KEY=<ACCESS_KEY>
export MINIO_SECRET_KEY=<SECRET_KEY>
minio server http://host{1...32}/export{1...32} http://host{33...64}/export{1...32}
```



#### 多集群Federation模式

Federation模式适用于MinIO多集群扩展, 在已有集群容量不够的情况下才会采取此模式进行扩展

> MinIO单集群是不支持动态扩缩容(增减节点或者硬盘数量)
>
> [官方ISSUE](https://github.com/minio/minio/issues/7986) 

![](/blog/images/minio/bucket-lookup.png)




假设MinIO集群拥有四个节点, 每个节点拥有一块硬盘

```bash
export MINIO_ACCESS_KEY=minio
export MINIO_SECRET_KEY=minio123
minio server http://192.168.10.11/data/minio http://192.168.10.12/data/minio http://192.168.10.13/data/minio http://192.168.10.14/data/minio
```

以上集群构成了一个EC数据集合的MinIO的服务集群。



#### Nginx反向代理和负载均衡

在生产分布式模式下, 可以采用nginx以获得以下能力

1. HTTPS
2. 反向代理
3. 负载均衡
4. 限流...

![](/blog/images/minio/minio-instances-load-balanced.png)

```nginx
#示例配置
upstream minio_servers {
    server 192.168.10.11:9000;
    server 192.168.10.12:9000;
  	server 192.168.10.13:9000;
  	server 192.168.10.14:9000;
}

server {
    listen 80;
    server_name www.example.com;

    location / {
        proxy_set_header Host $http_host;
        proxy_pass       http://minio_servers;
    }
}
```





### MinIO监控

#### 健康检查(Health Check)

1. /minio/health/live  服务是否运行中
2. /minio/health/ready  服务是否已就绪



#### Promethus监控

`/minio/prometheus/metrics` 指标Exported URL  详见[官方文档](https://github.com/minio/minio/blob/master/docs/metrics/prometheus/README.md)







### MinIO API

> 引用JAVA SDK

| Bucket operations (Bucket操作)                               | Object operations(对象操作)                                  | Presigned operations(预签名操作)                             | Bucket Policy/LifeCycle Operations(Bucket权限及生命周期操作) |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [`makeBucket`](https://docs.min.io/docs/java-client-api-reference.html#makeBucket) | [`getObject`](https://docs.min.io/docs/java-client-api-reference.html#getObject) | [`presignedGetObject`](https://docs.min.io/docs/java-client-api-reference.html#presignedGetObject) | [`getBucketPolicy`](https://docs.min.io/docs/java-client-api-reference.html#getBucketPolicy) |
| [`listBuckets`](https://docs.min.io/docs/java-client-api-reference.html#listBuckets) | [`putObject`](https://docs.min.io/docs/java-client-api-reference.html#putObject) | [`presignedPutObject`](https://docs.min.io/docs/java-client-api-reference.html#presignedPutObject) | [`setBucketPolicy`](https://docs.min.io/docs/java-client-api-reference.html#setBucketPolicy) |
| [`bucketExists`](https://docs.min.io/docs/java-client-api-reference.html#bucketExists) | [`copyObject`](https://docs.min.io/docs/java-client-api-reference.html#copyObject) | [`presignedPostPolicy`](https://docs.min.io/docs/java-client-api-reference.html#presignedPostPolicy) | [`setBucketLifeCycle`](https://docs.min.io/docs/java-client-api-reference.html#setBucketLifeCycle) |
| [`removeBucket`](https://docs.min.io/docs/java-client-api-reference.html#removeBucket) | [`statObject`](https://docs.min.io/docs/java-client-api-reference.html#statObject) |                                                              | [`getBucketLifeCycle`](https://docs.min.io/docs/java-client-api-reference.html#getBucketLifeCycle) |
| [`listObjects`](https://docs.min.io/docs/java-client-api-reference.html#listObjects) | [`removeObject`](https://docs.min.io/docs/java-client-api-reference.html#removeObject) |                                                              | [`deleteBucketLifeCycle`](https://docs.min.io/docs/java-client-api-reference.html#deleteBucketLifeCycle) |
| [`listIncompleteUploads`](https://docs.min.io/docs/java-client-api-reference.html#listIncompleteUploads) | [`removeIncompleteUpload`](https://docs.min.io/docs/java-client-api-reference.html#removeIncompleteUpload) |                                                              |                                                              |
| [`listenBucketNotification`](https://docs.min.io/docs/java-client-api-reference.html#listenBucketNotification) | [`composeObject`](https://docs.min.io/docs/java-client-api-reference.html#composeObject) |                                                              |                                                              |
| [`setBucketNotification`](https://docs.min.io/docs/java-client-api-reference.html#setBucketNotification) | [`selectObjectContent`](https://docs.min.io/docs/java-client-api-reference.html#selectObjectContent) |                                                              |                                                              |
| [`getBucketNotification`](https://docs.min.io/docs/java-client-api-reference.html#getBucketNotification) |                                                              |                                                              |                                                              |
| [`removeAllBucketNotification`](https://docs.min.io/docs/java-client-api-reference.html#removeAllBucketNotification) |                                                              |                                                              |                                                              |
| [`enableVersioning`](https://docs.min.io/docs/java-client-api-reference.html#enableVersioning) |                                                              |                                                              |                                                              |
| [`disableVersioning`](https://docs.min.io/docs/java-client-api-reference.html#disableVersioning) |                                                              |                                                              |                                                              |
| [`setDefaultRetention`](https://docs.min.io/docs/java-client-api-reference.html#setDefaultRetention) |                                                              |                                                              |                                                              |
| [`getDefaultRetention`](https://docs.min.io/docs/java-client-api-reference.html#getDefaultRetention) |                                                              |                                                              |                                                              |

常用的API一般就是对象操作API, 比如

- getObject: 下载对象
- putObject: 上传对象
- removeObject: 删除对象



#### 预签名(Presigned)操作的应用场景

上传(`presignedPutObject`)或下载(`presignedGetObject`)时, 客户端可以向服务器获取对应的Presigned Url(预签名URL)的方式来完成操作。

优点：

- MinIO秘钥统一由服务器管理, 客户端不需要也没必要管理
- 获取的预签名URL是有时效性的，安全，时间长短可以由服务器设置

详见[官方文档介绍](https://docs.min.io/docs/upload-files-from-browser-using-pre-signed-urls.html)







### MinIO客户端秘钥收敛(STS模式)

客户端访问MinIO的方式可以采用STS(`Security Token Service`)模式。类似下图

![STS Mode](/blog/images/minio/identity-management.svg)

三种角色

- IDP 身份校验服务
- Application 应用服务
- MinIO



目前MinIO支持以下四种身份校验服务

- [**Client grants**](https://github.com/minio/minio/blob/master/docs/sts/client-grants.md)
- [**WebIdentity**](https://github.com/minio/minio/blob/master/docs/sts/web-identity.md)
- [**AssumeRole**](https://github.com/minio/minio/blob/master/docs/sts/assume-role.md)
- [**AD/LDAP**](https://github.com/minio/minio/blob/master/docs/sts/ldap.md)





