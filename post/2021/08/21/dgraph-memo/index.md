# Dgraph备忘录

> 不定时补充更新

## 部署

```shell
docker pull dgraph/dgraph
# HOSTIPADDR=10.0.0.10 # 用以后续告知raft模块通信地址,确保集群连通性, 我这儿后面直接写了
cd ..
mkdir -p Dgraph
# 创建服务用户
groupadd dgraph
useradd -M -g dgraph -s /usr/sbin/nologin dgraph
chown -R dgraph:dgraph Dgraph
# 生成TLS证书
# client证书用于mtls, SAN(Subject Alternative Name) 取决于访问时需要用到的标识(localhost, 公网IP...)
docker run --rm -v ~/Dgraph:/dgraph dgraph/dgraph:latest dgraph cert -n $SAN -c zero1
docker run --rm -v ~/Dgraph:/dgraph dgraph/dgraph:latest dgraph cert -c alpha1
# zero
docker run -d --name dgraph-zero-1 --network dgraph_default -p 6080:6080 -p 5080:5080 -v ~/Dgraph:/dgraph dgraph/dgraph:latest dgraph zero --my=10.0.0.10:5080 --tls "ca-cert=/dgraph/tls/ca.crt;client-auth-type=REQUIREANDVERIFY;server-cert=/dgraph/tls/node.crt;server-key=/dgraph/tls/node.key;internal-port=true;client-cert=/dgraph/tls/client.zero1.crt;client-key=/dgraph/tls/client.zero1.key"
# alpha
docker run -d --name dgraph-alpha1 --network dgraph_default -p 7080:7080 -p 9080:9080 -p 8080:8080 -v ~/Dgraph:/dgraph dgraph/dgraph:latest dgraph alpha --zero=10.0.0.10:5080 --my=10.0.0.10:7080 --badger "compression=zstd:1" --security "whitelist=0.0.0.0/0" --tls "ca-cert=/dgraph/tls/ca.crt;client-auth-type=REQUIREANDVERIFY;server-cert=/dgraph/tls/node.crt;server-key=/dgraph/tls/node.key;internal-port=true;client-cert=/dgraph/tls/client.alpha1.crt;client-key=/dgraph/tls/client.alpha1.key"
# 给浏览器签发证书(自己导入ca.crt和.p12文件)
# 同样的,可以给需要连接的客户端签发证书
docker run --rm -v ~/Dgraph:/dgraph dgraph/dgraph:latest dgraph cert -c browser
openssl pkcs12 -export -out Dgraph/browser.p12 -in Dgraph/tls/client.browser.crt -inkey Dgraph/tls/client.browser.key
```
端口占用情况:

| Dgraph Node Type | gRPC-internal-private | gRPC-external-private | gRPC-external-public | HTTP-external-private | HTTP-external-public |
| :--------------: | :-------------------: | :-------------------: | :------------------: | :-------------------: | :------------------: |
|       zero       |         5080          |         5080          |                      |         6080          |                      |
|      alpha       |         7080          |                       |         9080         |                       |         8080         |
|      ratel       |                       |                       |                      |                       |         8000         |

## Triples

`Dgraph`采用的图数据模型是`Triples`(三元组), 也即`S P O`, 展开表示为: `<Subject> <Predicate> <Object>`.

也即图中由`subject`标识的节点通过有向边`predicate`连接到`object`.

同时借助[RDF](https://www.w3.org/RDF/)(资源描述框架, 基于语义网的知识表示框架)来增强其数据关联性.

## Blank UID

```DQL
{
	set {
		_:alice <name> "Alice" .
		_:bob <name> "Bob"
		_:bob <follows> _:alice .
		# assign type
		_:alice <dgraph.type> "Person" .
    	_:bob <dgraph.type> "Person" .
	}
}
```

`_:alice` 并不是真正的`uid`, 其在这里就是一个`uid`的`placeholder`(数据还未真正创建), 可以在`set` 阶段去引用某对象的`uid`. 

## Schema里的<>

简单来说, 如果你的`predicate`名并不是由数字及字母组成, 在执行`mutation`时就需要用`<>`括起来.

> If your predicate is a URI or has language-specific characters, then enclose it with angle brackets `<>` when executing the schema mutation.

## Expand a type

```DQL
{
	data(func: type(Person)) {
		expand(_all_) {
			expand(_all_)
		}
	}
}
```

通过指定`type`函数参数为某`type`, 会自动展开结果中该`type`的字段(`predicate`).

## Reverse edges

在`predicate`定义里加上`@reverse`, 可通过`~`来取反.

## Password type

`Dgraph`提供了`password`类型, 表现为加密后的`string`, 其实是使用`bcrypt`加密生成的密文(60字节).

## @upsert

给`predicate`添加此`directive`, 以使提交事务时`Dgraph`可以检查索引是否有冲突, 从而达到`unique index`的效果.

```DQL
<email>: string @index(hash) @upsert .
```

使用`upsert block`:

- 发起一个新事务
- 先`query`, 如果成功返回了`uid`则说明该结点已经存在, 反则说明不存在
- 若不存在, 则会创建新结点(紧接着的`mutation`)

```DQL
upsert {
  query {
    q(func: eq(email, "user@company1.io")) {
      v as uid
      name
    }
  }

  mutation {
    set { # 若uid有值
      uid(v) <name> "first last" .
      uid(v) <email> "user@company1.io" .
    }
  }
}
```

## GraphQL & DQL

说实话, `Dgraph`的文档还是有些糟糕, 你所需的信息可能藏在你根本想不到的角落(尽量将所有文档概览一下), 这个主题来自这篇[Blog](https://dgraph.io/blog/post/graphql-vs-dql/).

早期`Dgraph`是直接使用`GraphQL`作为数据库查询语言, 但很显然, `GraphQL`设计的初衷是用于`API`的, 只是语法看起来很"graph", 所以很快就展现出了其局限性: 

> A DQL schema is predicate-first focused and supports some aspects not yet supported by the GraphQL syntax such as multi-lingual predicates and facets (data information stored on edges between nodes). 
>
> A GraphQL schema is type-first focused and only supports spec-compliant elements.

`DQL`是`Dgraph`基于`GraphQL`魔改的一套适于图数据库原生的查询语言, 事实上`Dgraph`核心只使用`DQL`, 任何传入的`GraphQL`请求都会先被重写为`DQL`.

简单来说, 直接使用`GraphQL Endpoint`也未尝不可, `Dgraph`本身也可以是`GraphQL Native`, 但无论如何, 其内部还是使用的`DQL`, 其会据`GraphQL`的`schema'`生成一份等同的`DQL schema`, 反之则不然. 

`GraphQL Endpoint`可以配备访问权限控制(`@auth directive... 等`), `DQL`更适合传统的后端开发(先查询数据, 做点自定操作然后再返回).

## 数据存储

`Dgraph`数据使用`Badger`存储, 一个基于`LSM-Tree`的`KV`数据库, 其设计使得它理论上能够很好的降低读写放大的问题.

`Dgraph`的图数据模型是`Triples`, 在存储时: 所有相同`predicate`的记录组成一个`shard`, 在该`shard`中, 具有同一`<subject - predicate>`关系的的记录会被组合浓缩成一条`K-V对`存储在`Badger中`. 该`value`又叫做`posting list`(搜索引擎术语, 根据搜索的term找到的排序好的文档ids). `posting list`作为一个`value`存储在`Badger`中, 而其`key`由`subject`及`predicate`共同推导而得. 

```
<0x01> <follower> <0xab> .
<0x01> <follower> <0xbc> .
<0x01> <follower> <0xcd> .
...
key = <follower, 0x01>
value = <0xab, 0xbc, 0xcd, ...>
```

在`Dgraph`中你可以使用`debug`子命令来观察`posting list`结构:

```shell
docker stop dgraph-alpha # 或复制一份p文件夹
dgraph debug -p Dgrpah/p
# 具体使用方法
dgraph debug -h
```

