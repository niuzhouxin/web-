## 前言
云服务器的实例元数据是指在实例内部通过访问元数据服务（Metadata Service）获取的实例属性等信息，如实例 ID、VPC 信息、网卡信息。元数据包含实例角色所对应的临时凭据的情况下，如果服务器所搭载的对外业务服务存在 SSRF 漏洞，将造成云服务器可被攻击者接管的风险。本文来具体实践、学习下 SSRF 漏洞给云服务元数据带来的安全威胁。
## Metadata元数据
用阿里云 ECS 服务进行学习和演示。
**ESC实例元数据**：是指在 ECS 实例内部通过访问元数据服务（Metadata Service）获取的实例属性等信息，如实例 ID、VPC 信息、网卡信息。通过元数据服务，您无需登录控制台或调用 API，在实例内部即可获取实例信息，可以更便捷、安全地配置或管理正在运行的实例或实例上的程序。例如，在基于 ECS 实例的应用程序中，通过元数据获取实例 RAM 角色的临时访问凭证，以安全地访问相关授权资源（比如访问 OSS 下载文件）。
### 使用限制
- 仅网络类型为专有网络 VPC 的 ECS 支持实例元数据功能。
- 仅支持在实例内部访问元数据服务器来获取实例元数据，且实例需处于运行中状态。
- 单实例高频访问元数据服务器获取元数据，可能会导致限流。在高频访问场景下，建议缓存已获取的实例元数据。以 RAM 临时安全凭证为例，获取到凭证后建议将其缓存，并在凭证接近过期时间前重新获取新的凭证，以避免因频繁访问实例元数据服务器而导致的访问限流。

### 获取实例元数据
```bash
# 普通模式下获取，其中<metadata>：需替换为具体要查询的实例元数据，可查阅实例元数据列表
curl http://100.100.100.200/latest/meta-data/<metadata>

# 加固模式下获取
# 获取访问元数据服务器访问凭证，需设置有效期，不可包含标头X-Forwarded-For
TOKEN=`curl -X PUT "http://100.100.100.200/latest/api/token" -H "X-aliyun-ecs-metadata-token-ttl-seconds:<元数据服务器访问凭证有效期>"`
# 获取实例元数据
curl -H "X-aliyun-ecs-metadata-token: $TOKEN" http://100.100.100.200/latest/meta-data/<metadata>

```
各个云厂商都有自己的元数据 metadata，下面给出部分云厂商的元数据访问地址：
```
阿里云元数据地址：http://100.100.100.200/latest/meta-data/ram/security-credentials/${role-name}
腾讯云元数据地址：http://metadata.tencentyun.com/latest/meta-data/cam/security-credentials/${role-name}
华为云元数据地址：http://169.254.169.254/
亚马云元数据地址：http://169.254.169.254/
微软云元数据地址：http://169.254.169.254/
谷歌云元数据地址：http://metadata.google.internal/
```
由于元数据服务部署在链路本地地址上，云厂商默认没有进一步设置安全措施来检测或阻止由实例内部发出的恶意的对元数据服务的未授权访问。攻击者可以通过实例上应用的 SSRF 漏洞对实例的元数据服务进行访问。

因此，如果实例中应用中存在 SSRF 漏洞，那么元数据服务将会完全暴露在攻击者面前。在实例元数据服务提供的众多数据中，有一项数据特别受到攻击者的青睐，那就是角色的临时访问凭据。这将是攻击者由 SSRF 漏洞到获取实例控制权限的桥梁。
## RAM资源管理角色
RAM 角色是一种虚拟用户，可以被授予一组权限策略。与 RAM 用户不同，RAM 角色没有永久身份凭证（登录密码或访问密钥），需要被一个可信实体扮演。扮演成功后，可信实体将获得 RAM 角色的临时身份凭证，即安全令牌（STS Token），使用该安全令牌就能以 RAM 角色身份访问被授权的资源。
在 Metadata 元数据中危害最大的莫过于能够获取到云服务器对应的 sts（Security Token Service），通过 sts 最大危害能够接管控制台，如从云服务器 SSRF 漏洞到接管阿里云控制台。但在使用该攻击方法之前，我们需要知道 ECS 本身是不存在 RAM 的，通过具体的 ECS 云服务器实例来看下。
登录 ECS 服务器实例，请求访问元数据：
```
curl http://100.100.100.200/latest/meta-data/
```
![](image/384.png)
从返回的路径可以看出当前 Metadata 元数据资源中不包含 ram 子路径，此时请求 `http://100.100.100.200/latest/meta-data/ram/security-credentials/` 自然也是不包含 RAM 凭据信息的：
```
curl http://100.100.100.200/latest/meta-data/ram/security-credentials/
```
![](image/385.png)
如果我们需要让 ECS 拥有 RAM，并且让我们去利用 metadata 进行渗透，那么就首先需要去给对应的服务器授予一个 RAM 角色，我们访问阿里云控制台，搜索访问控制或直接访问https://ram.console.aliyun.com/roles，并点击创建角色，选择“阿里云服务角色”：
在配置角色上，选择普通服务角色，并选择受信服务为云服务器：

## 实战应用














## 参考文章
https://blog.csdn.net/weixin_39190897/article/details/136049642
https://cloudsec.huoxian.cn/docs/articles/aws/aws_ec2
https://www.freebuf.com/articles/network/276957.html