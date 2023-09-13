# Time-Sorted Unique Identifiers as Primary Key

## 业务主键(Business ID)
如何选择数据库主键是一个经典问题，一般我们需要思考的第一步，就是使用 "业务主键" 还是 "逻辑主键"？我个人一般无脑选择后者。其中最主要的理由是，我们希望主键是不可变的，因为主键常常作为"外键"被其他表引用，一旦我们需要修改一条记录的主键，我们就需要同时修改所有引用到这条记录的表，这往往是麻烦且易错的。而**一旦主键同时具有业务上的意义，它就存在着需要被修改的风险。**  
举个例子，如果某个产品经理让我设计一个员工管理系统，每个员工有一个唯一的"工号"，该号码按照员工进入公司的时间递增，老板是一号员工。产品经理信誓旦旦的保证"员工的工号一旦确定，永远不需要再更改，可以唯一标识员工"。这样一来，直接把工号设为主键是一个非常诱人的选择，但我的建议是不要这么做，而是使用一个额外的逻辑主键列。因为一旦"工号"成为产品设计的一部分，就已经不再"纯粹"了，它可能会被打印在工号牌上，继而某个领导可能会想用他的幸运数字作为工号；某个领导可能觉得他的工号很晦气，想要换个号码转转运；甚至于可能公司招募到了某个著名球星，几位创始人一高兴，决定把该球星的球衣号码让给他作为工号。归根结底，一些起初不会变的东西一旦有了业务含义，就有了需要被改变的风险。  
除了可变性的原因之外，使用"业务主键"还需要考虑其他的一些因素，同样是上面的例子，使用员工的身份证号作主键是可以解决上面的问题，既唯一，又不变，但员工的身份证号成了创建员工记录的前置需求。这可能不是产品经理想要的结果，或者说即时现在是，以后也可能不是，换句话说，创建员工记录多了"收集员工身份证号"这个额外的"业务流程"。
综上所述，数据库主键需要满足多条在我看来很"苛刻"的性质，倒不是说不存在完美的"业务主键"，只是判断一个业务上的属性是否可以用来做主键需要设计者对业务有足够的理解和有大量的经验。即便如此，在设计一个数据库数百张表时也难免失误。因此我总是无脑的使用逻辑主键，毕竟不思考，就不会犯错。

## 复合业务主键 (Composite Key)
如果说有什么比"业务主键"更让我排斥的，就是所谓的"复合业务主键"，即通过组合多个属性来保证"唯一性"。它不仅有"业务主键"的所有问题，还有以下缺点:
- "唯一性" 可能因为业务逻辑变化而被破坏
- 引用该表的记录需要冗余的保存复合主键的所有组成部分
- 无法使用 `select * from table_name where primary_key in ("id1", "id2")` 来简化查询语句

## 数据库自增ID (Auto Increment ID)
数据库自增 ID 是一个非常廉价的逻辑主键生成方式，但它也同样有很多缺点：
1. 暴露商业信息，当需要提供对外接口时，接口的使用者能够轻易的通过新增记录来得到一张表当前的记录数量。  
  e.g. 如果有一个 RESTFUL 的注册新用户接口，接口的调用者可以每个月注册一个新用户，根据返回的 ID 得知你这个有多少个新用户，当前你的系统一共有多少用户等信息。一种可能的解决方法是服务端屏蔽返回的ID，但这样一来，一些正常的用户在反馈 bug 时，就没办法帮程序员定位到出问题的数据。这导致我经常不得不为一些数据生成一个额外的，隐藏商业信息的唯一ID。
2. 能够轻易预测, 容易泄露数据  
  e.g. 如果一个客户关系系统有一个 `https://{baseUrl}/clients/{id}` 的 http 接口，根据客户 id 查询客户信息。如果使用自增 id，有该接口权限的用户可以通过脚本使用极少的调用次数获取数据库中全部的客户信息。
3. 不利于整合数据  
  e.g. 继续以上面客户关系系统为例，如果把它部署在多个分公司，自增 id 会大面积重复。一个常见的做法是，使用 id 外加生成该 id 的 region 来唯一确定一个客户。站在程序维护者视角来看，这破坏了 id 的唯一性，当你听到某用户报告说 "id 是 xxx 的客户数据出错了"，你需要先反问 "这是哪家分公司的客户？"。实际操作中有些 bug 是由用户报告的，而与用户的沟通效率往往不可控。更严重的情况是，有时维护者也忘了 region 这回事，从而发生生产事故。比如说需求方可能要求在数据库中批量删除某个 id 客户相关的所有项目，操作者可能在登录错误的数据库操作。归根结底，id 能够单独准确定位数据是大家潜移默化都接受的规则，而 id + region 的方式破坏了这条规则。
4. 依赖集中式的 id 生成服务来保证 id 的唯一性，应用必须连接到数据库才能生成 id。

## Universally Unique Identifier(UUID)
UUID 与数据库自增 id 截然相反，它不暴露信息，全局唯一，还可以分布式生成。但它也丢掉了自增 id 所有的优点：
1. 使用 B+tree 索引 UUID 非常低效
2. 无序  
3. 需要 128 bits

## Time-Sorted Unique Identifiers(TSID)
[TSID](https://github.com/f4b6a3/tsid-creator) 结合了 UUID 和自增 ID 的优点，其格式如下:
```
                                            adjustable
                                           <---------->
|------------------------------------------|----------|------------|
       time (msecs since 2020-01-01)           node       counter
                42 bits                       10 bits     12 bits

- time:    2^42 = ~69 years or ~139 years (with adjustable epoch)
- node:    2^10 = 1,024 (with adjustable bits)
- counter: 2^12 = 4,096 (initially random)

Notes:
The node is adjustable from 0 to 20 bits.
The node bits affect the counter bits.
The time component can be used for ~69 years if stored in a SIGNED 64 bits integer field.
The time component can be used for ~139 years if stored in a UNSIGNED 64 bits integer field.
```

- time component(42 bits)
  前42位存储当前时间戳，与自增 id 一样，可以直接通过 `order by id` 来按照创建时间排序，比自增 id 更好的是，不需要额外的索引一个 `created_at` 列，就可以高效的对记录的生成时间进行筛选（`where id between tsid1 and tsid2`使用主键索引）。
- node(0 to 20 bits, default 10 bits)
  node 为节点 id，这样多个服务同时生成 id 也不会重复。以上面分公司的例子来看，只要不同分公司使用不同的 node_id 或者 node_id 段，即时同一时刻生成多个 TSID 也不会重复。对于单个服务，或大部分唯一性要求不高的场景，随机生成一个 node_id 也可以。
- counter(2 to 22bits, default 12 bits)
  随机数，用于保证在单个节点，同一时刻生成多个 id 的唯一性。如果某节点同一时刻生成多个 id，则首个 counter 随机，后续 counter 递增，如果超过了最大范围，则 time + 1。

一共 64 bits，刚好可以用 Java 的 Long 类型，MySQL 的 BigInt 存储。遗憾的是 Javascript number 类型只有 52 bits 用于表示整数，因此直接使用 JS 序列化和反序列化 TSID 会丢失精度。
```javascript
JSON.stringify(3141339522892735723)
// => '3141339522892735500'
```
为此，我们可以使用 [Crockford's base32](https://www.crockford.com/base32.html) encode 之后再传给前端，encode 后为13个字符，大小写不敏感，可以直接用在 URL 中。

## Summary
TSID 结合了自增 id 和 UUID 的优点，可以作为首选的逻辑主键

## Reference
- [Natural Key](https://en.wikipedia.org/wiki/Natural_key)
- [Surrogate key](https://en.wikipedia.org/wiki/Surrogate_key)
- [The great primary-key debate](https://www.techrepublic.com/article/the-great-primary-key-debate/)
- [What's the best practice for primary keys in tables?](https://stackoverflow.com/questions/337503/whats-the-best-practice-for-primary-keys-in-tables)
- [How to choose my primary key?](https://stackoverflow.com/questions/4291990/how-to-choose-my-primary-key)  
- [The best UUID type for a database Primary Key](https://vladmihalcea.com/uuid-database-primary-key/)
- [Announcing Snowflake](https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake)
- [TSID Creator](https://github.com/f4b6a3/tsid-creator)
- [ulid](https://github.com/ulid/spec)
