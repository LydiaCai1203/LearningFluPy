# 第四章：Schema 与数据类型优化

Clare Churcher 的 《Beginning Database Design》帮助学习数据库设计。

## 4.1 选择优化的数据类型

几条简单的原则：

+ **更小的通常更好**

  占用更小的内存、更少的磁盘、CPU缓存、并且处理时需要的 CPU 周期也更少。

+ **简单就好**

  简单数据类型的操作通常需要更少的 CPU周期

  + 整型比字符的操作代价更低，因为字符集的校对规则(排序规则)是 *字符比较*  要比 *整型比较* 更复杂。
  + 应该使用 MySQL 内建的类型而不是字符串来存储日期和时间。
  + 应该使用整型存储 IP 地址

+ **尽量避免 NULL**

  通常情况下最好指定列为 NOT NULL，除非真的有必要存 NULL。*可为 NULL 的列* 使得索引、索引统计、值比较，都**更复杂**。并且会使用**更多的存储空间**，在 MySQL 里也**需要特殊处理**。通常将可为  NULL 的列都变为 NOT NULL 带来的性能提升比较小。**InnoDB 使用 bit 来存储 NULL 值，对于稀疏数据(很多为 NULL, 很少为 NOT NULL)有很好的空间效率。** 另外，MySQL 为了兼容还支持很多的别名，如 INTEGER、BOOL、NUMERIC。

  + 当可为 NULL 的列被索引时，每个索引记录需要一个额外的字节。
  + 在 MyISAM 里甚至还可能导致固定大小的索引变成可变大小的索引。
  + DATETIME 和 TIMESTAMP 列都可以存储相同类型的数据：时间和日期，并且都是精确到秒。但是 TIMESTAMP 只使用 DATETIME 一半的存储时间。


### 4.1.1 整数(whole number)

​		TINYINT(8bit)、SAMLLINT(14bit)、MEDIUMINT(24bit)、INT(32bit)、BIGINT(64bit)。类型有可选的 UNSIGNED 属性，意为不允许负值。有符号和无符号的存储空间相同，性能相同。只是表示范围不同。

  		MySQL 可以为整数类型指定宽度，如INT(11)，对于大多数应用来说没有意义，**它并不会限制值的合法范围，只是规定了 MySQL 的一些交互工具用来显示字符的个数，只是对于存储和计算来说，INT(1) 和 INT(20) 是相同的。**

### 4.1.2 实数(real number)

​		实数就是带有小数部分的数字。MySQL 既支持精确类型(DECIMAL)，也支持不精确类型(FLOAT、DOUBLE)。

CPU 直接支持原生浮点计算，5.0+ 版本中 MySQL 服务器自身实现了 DECIMAL 的高精度计算，浮点计算肯定更快。Decimal 最多允许 65 个数字。MySQL 使用 DOUBLE 作为内部浮点计算的类型。

+ **Decimal(18, 9)**

  18 是精度(值存储的有效位数)，9是位数(小数点后可以存储的位数)。

  5.0+ 版本将数字打包保存到一个二进制字符串中(每4个字节存储9个数字)。上述例子中，小数点的前后各存储9个数字，也就是8个字节，小数点单独占一个字节，所以总共9个字节。

  Decimal(5, 2) 表达的范围则是 -999.99 ~ 999.99。
  

​       浮点类型在存储同样范围的值时，通常比 Decimal 使用更少的空间。比如 FLOAT 使用 4 个字节存储，DOUBLE 使用 8 个字节存储。因此尽可能只在对小数进行精确计算时才用 DECIMAL。但是当数据量较大的情况下可以使用 BIGINT 来代替，只要根据小数的位数*相应的倍数即可。**这样可以同时避免浮点存储计算不精确和 DECIMAL 精确计算代价的问题。**

### 4.1.3 字符串类型（默认引擎 InnoDB / MyISAM）

https://dev.mysql.com/doc/refman/8.0/en/char.html

**-- VARCHAR 和 CHAR 类型**

+ **varchar**

  ​		**用于存储可变长字符串。比定长类型更节省存储空间**。除了一种情况，MySQL 的表使用的是 ROW_FORMAT=FIXED 创建。这样每一行都会使用定长存储。**MySQL 在存储或检索的时候会保留末尾的空格**。

  ​		varchar 需要使用 **1 或 2 个额外的字节来记录字符串的长度**, 如果列的最大长度 <= 255byte, 则只使用1 byte 来表示。否则用 2 byte 来表示。(因为 1byte=8bit, 能表示的最大正整数为 2^8 - 1 = 255, 所以更大的就要用 2 个字节表示了，超过 65535 的就要 3个字节表示咯～)。

  ​		**但是！！** 由于行是变长的，*UPDATE 可能导致行变得更长，这就导致需要做额外的工作*。比如，行占用的空间在增长，并且在页内没有更多的空间可以存储，这是 MyISAM 会将行拆成不同的片段存储。InnoDB 则需要分裂页来使行可以放进页内。

  ​		**适用以下情况**，字符串列的长度比平均长度大很多，列的更新很少，所以碎片不是问题。使用了像 UTF-8 这样的字符存储，每个字符都使用不同的字节数进行存储。

  ​		**MySQL 会把过长的 VARCHAR 转换成 BLOB。**

+ **char**

  ​		根据定义的字符串长度分配足够的空间。**存储时 MySQL 会将输入值的右边的空格截断，用空格向右填充直至定义长度(这样方便比较),检索到的结果 MySQL 会删除所有的末尾空格。**

  ​		**使用以下情况**，适合存储很短的字符串，或者所有的值都接近于一个长度。*char 就非常适合存储密码的 md5 值。* 对于经常变更的数据，char 类型不容易产生碎片。对于非常短的列，char 值比 varchar 在存储空间上也更有效率。

+ **binary 和 varbinary**

  ​		存储的是二进制字符串，二进制字符串存的是字节码而不是字符，填充的是 \0 而不是 空格，检索的时候也不会去掉填充值。二进制比较的优势有：**1. 大小写敏感 2. 在做比较的时候每次按一个字节，并且根据该字节的数值进行比较，二进制比较比字符串的更简单，更快。**

**-- BLOB 和 TEXT 类型**

​		**blob** 以二进制存储。类别有 TINYBLOB、SMALLBLOB、BLOB、MEDIUMBLOB、LONGBLOB。**text** 以字符串存储。类别有 TINYTEXT、SMALLTEXT、TEXT、MEDIUMTEXT、LONGTEXT。当存的值太大时，InnoDB 会使用*专用的外部存储区域* 进行存储，所以每个值在行内需要 1～4 个字节存储一个指针，然后在外部存储区域存储实际的值。

​		**MySQL 只对每个列的最前 max_sort_length 字节做排序。或者使用 ORDER BY SUBSTRING(column, length)。同样不能将这两种类型的列的全部字符串做索引，也不能使用这些索引消除排序。**

​		**应尽量避免使用 BLOB 和 TEXT 类型**，如果实在无法避免，在所有使用 BLOB 的地方都用 SUBSTRING(column, length) 将列值转换为字符串，这样就可以使用内存临时表。例如，有一个 1000W 行的表，占用几个G 的磁盘空间，其中有一个 utf-8 字符集的 varchar(1000)列，每个字符最多使用 3个字节，最坏情况下需要 3K字节的空间。如果在 ORDER BY 中用到这个列，就要查询扫描整个表，因此为了排序就需要超过 30GB 的临时表。

**-- 使用枚举(ENUM)代替字符串类型**

	create table test_ip (ip_str ENUM ('value1', 'value2', 'value3');
​		有时候可以使用 枚举 列代替常用的字符串类型。枚举列可以把一些不重复的字符串压缩成一个预定义的集合。因为 MySQL 在存储枚举类型的时候非常紧凑，会根据列表值的数量压缩到一个或者两个字节中。MySQL 在内部会将每个值在列表中的位置保存为整数，并且在表的 .frm 文件中保存 '数字-字符串' 映射关系的一个'查找表'。

​		**PS：枚举字段的排序是按照内部存储的整数来排序的，而不是按照定义的字符串进行排序的。**

​		如果要指定按照定义枚举列进行排序，使用`select e from enum_test order by field(e, 'apple', 'dog', 'fish');` 不过如果定义的时候就是按照字符排序的话，就不必要这样做了。

​		**字符串列表是固定的，添加或删除字符串就必须用 alter table。又因为存储的是整型，所以存储空间小，如果这个字端还是索引，索引也会变小。但是查找的时候必须要进行转换，所以有一定的开销。另外就是枚举类型关联枚举类型，效率极高。**

#### 4.1.4 日期和时间类型

**datetime**

​		范围：1001 年 ～ 9999 年。精度到秒，**它把日期和时间封装到格式为 YYYYMMDDHHMMSS 的整数中**，和时区无关。始终使用 8 byte 存储空间。默认情况下是一种可排序、无歧义的格式显示 datetime 值。例如 "2001-01-01 00:00:00"，这是 ANSI 标准定义的日期和时间的表示方法。

​		将 unix 时间存储为整数值，这样的时间通常不方便处理，不推荐使用。(突然想到某人设计的表，就全是日期整数，的确是很不方便)。

**timestamp(应当尽量使用)**

​		保存了 1970-1-1 00:00:00 以来的秒数。只使用 4 byte 的存储空间。但是只能表示 1970 ~ 2038 年的时间。

​		`FORMAT_UNIXTINE()` 是 MySQL 提供用来将时间戳转换为日期的函数，当然还有 `FORMAT_TIMESTAMP()。`

**如果想要存储比秒更小粒度的日期和时间，就可以用 BIGINT 或者 DOUBLE 来实现。**

#### 4.1.5 位数据类型

**BIT**

​		可以在 BIT 列在一列中存储一个或者多个 true/false 值。 BIT(1) 定义一个包含单个单位的字段。最大长度是 64 位。其在 MySQL 中是呗当作字符串类型，而不是数字类型。但是如果在数字上下文检索的场景中，比如 `select a, a+0 from bittest;` 检索出来的就是十进制的数字了。*如果想要在一个位的存储空间中存储一个 true/false 的值，可以创建一个 char(0), 该列可以保存 NULL, 或者长度为0的字符串。*

​		**MyISAM 会打包存储 BIT 的所有列，所以 17 个单独的 BIT 列只需要 17个位存储，3个字节。**

​		**Memory 和 InnoDB 为每个 BIT 列使用一个足够存储的最小整数类型来存放，所以不能节省存储空间。**

**SET**

​		在 MySQL 内部是以一系列打包的位的集合来表示的，可以有效利用存储空间，并且 MySQL 有 `FIND_IN_SET()` 、`FIELD()` 这样的函数方便在查询中使用。但是缺点就是改变列的定义的代价较高。需要 `ALTER TABLE`。一般来说也无法在 SET 列上通过索引来查找。

#### 4.1.6 选择标识符(identifier)

​		自增长列，测试了一下，在 MySQL 8+ 在整型以外的类型字段上是无法添加 auto_increment的。加上 index 也是不行的，查阅了一下文档，文档里只说明了可以在 int 类型上使用 auto_increment。但是 MyISAM 表中可以有另一种独特的用法。[戳](https://dev.mysql.com/doc/refman/8.0/en/example-auto-increment.html)

​		[让我想到一道问烂了的面试题](https://database.51cto.com/art/201910/604595.htm)

#### 4.1.7 特殊类型数据

​		某些类型的数据并不直接与内置类型一致，比如：低于秒级精度的时间戳，IPv4地址也是(人们常用 VARCHAR(15) 来表示，但是它其实是 32 位无符号整数)。MySQL 提供了 `INET_ATON() 和 INET_NTOA() 在两种表示法之间进行转换。`

### 4.2 MYSQL schema 设计中的陷阱

**太多的列**

​		MySQL 的存储引擎 API 工作时需要在服务器层和存储引擎层之间进行缓冲格式拷贝数据，然后在服务器层将缓冲内容解码成各个列。**从行缓冲中将编码过的列转换成行数据结构的操作代价是非常高的**。

+ MyISAM 定长行结构：匹配，不需要转换。

+ MyISAM 变长行结构 和 InnoDB 的行结构：总是需要转换。且转换的代价依赖于列的数量。

  *这里指的变长行结构，比如说在 A 表中有一个 字段a, 类型是 VARCHAR, 第一行存储的值小于 255 个字符，则这一行在 InnoDB 中的行存储结构中的记录大小的字段值就为1。第二行存储的值大于 255 个字符，则这一行在 InnoDB 中的行存储结构中记录大小的字段值就为2。这就是变长行结构。（我觉得这里要去了解一下 InnoDB 层存储的表结构是怎么样的）*

**太多的关联**

​		MySQL 限制了每个关联操作最多只能有 61 张表。**单个查询最好在 12 张表内做关联**。

**全能的枚举**

​		防止过度使用枚举。

**变相的枚举**

​		`CREATE TABLE test (is_default set('Y', 'N') NOT NULL default 'N');` 如果真和假两种情况不会同时出现，应该使用枚举列代替集合列。

**NULL**

​		上面说了避免使用 NULL(MySQL 会在索引中存储 NULL 值), 现在说一下某些场景下，使用 NULL 比使用某些特定的魔法数字要好。

​		`CREATE TABLE test (dt DATETIME NOT NULL DEFAULT '0000-00-00 00:00:00')`

​		像这样的日期就是不可能的，不应该在设计中出现的。	

### 4.3 范式和反范式

​		*竟然让我在这本书中找到了一个反驳阿三使用关系表解决一切问题的方法。*

​		场景：用户权限关系表。将所有的用户信息，权限信息，用户权限关系，全放在一张表中。

​		坏处：假如需要删除某类用户，那么相关的权限信息也将会被删除。避免这个问题，应该拆表存储。

**范式：**

1. 数据库中所有的字段都是单一属性，不可拆分的。
2. 数据库表中不存在非关键字段对任一候选关键字段的部分函数依赖。即所有非关键字段都完全依赖于任意一组候选关键字。
3. 数据表中如果不存在非关键字段对任一候选关键字段的传递函数依赖则符合第三范式。所谓传递函数依赖，指的是如果存在”A → B → C”的决定关系，则C传递函数依赖于A。
4. 鲍依斯-科得范式（BCNF）在第三范式的基础上，数据库表中如果不存在任何字段对任一候选关键字段的传递函数依赖则符BCNF范式。

#### 4.3.1 范式的优点和缺点

**优点**

+ 范式化的更新操作通常比反范式化更快。
+ 当数据较好地范式化时，就只有很少或者没有重复数据，所以只需要修改更少的数据。
+ 范式化的表通常更小，可以更好地放在内存里，所以执行操作会更快。
+ 很少有多余的数据，意味着检索列表数据时更少需要 `distinct` 或者 `group by` 。(感觉阿三是真的不懂这些)

**缺点**

+ 通常需要关联，即时是稍微复杂一点的查询语句，这样会导致代价昂贵，也可能导致一些索引策略无效。

#### 4.3.2 反范式的优点和缺点

**优点**

+ 很好地避免关联

  即便对表没有使用索引，是全表扫描。当数据比内存大时，这可能比关联要快的多。这样避免了随机 I/O。*全表扫描基本上时顺序 I/O, 也不是 100% 的，跟存储引擎的实现有关。*

+ 更有效地使用索引策略

#### 4.3.3 混用范式和反范式化

​		最常见的反范式化数据的方法是复制或者缓存，在不同的表中存储相同的特定列(不过如果这样做的话，修改相同的列的话，就会涉及多张表)。这样并不是完全的反范式，而且避免了完全反范式化的插入和删除的问题。也不会把 user_message 表搞的太大，也有利于高效地获取数据。

### 4.4 缓存表和汇总表

​		**缓存表**是指存储那些可以比较简单地从 schema 其它表获取(但是每次获取的速度比较慢)数据的表。**汇总表**保存的是使用 GROUP BY 语句聚合数据的表。

​		**汇总表优点，**不严格的计数或者通过小范围查询填满间隙的严格计数, 都比计算 message 表的所有行要有效的多。实时计算统计值是非常昂贵的操作，因为要么需要扫描表中的大部分数据，要么查询语句只能在某些特定的索引上才能有效运行，而这类特定索引一般会对 UPTE 操作有影响，所以一般不希望创建这样的索引。

​		**缓存表优点，** 对其优化搜索和检索查询语句很有效，这些查询语句经常需要特殊的表和索引结构。

​		**缺点，**必须决定是实时维护数据还是定期重建，定期重建不仅节省资源，还可以保持表不会有很多碎片，以及有完全顺序组织的索引。

#### 4.4.1 物化视图

​		实际上是预先计算并存储在磁盘上的表，可以通过各种各样的策略刷新和更新。（没看懂 跳过）

#### 4.4.2 计数器表

​		如果应用在表中保存计数器，则在更新计数器的时候可能碰到并发问题。应用场景有：缓存一个用户的朋友数、文件的下载次数等。

**优点：**计数器表小且快，而且独立的表还可以帮助避免查询缓存的失效。

**缺点：**对于任何想要更新这一行的事务来说，这条记录上都有一个全局互斥锁，这会使得事务串行执行，要获得更高的并发更新性能，也可以将计数器保存在多行中，每次都随机选择一行进行更新。

```mysql
CREATE TABLE hit_counter(
	slot tinyint unsigned not null primary key, 
  cnt int unsigned not null
) ENGINE=InnoDB;
```
​		预先在这张表上增加 100 行数据，然后随机选择一个槽进行更新。

`UPDATE hit_counter SET cnt = cnt + 1 WHERE slot = RAND * 100;`

​		如果要获得统计结果，就需要进行聚合查询。
`SELECT SUM(cnt) FROM his_counter;`

​		假如是需要是每隔一段时间就开始一个新的计数器(例如每天一个)。

```mysql
CREATE TABLE daily_hit_counter (
  day date not null.
  slot tinyint unsigned not null primary key, 
  cnt int unsigned not null,
  primary key(slot, cnt)
) ENGINE=InnoDB;
```

```mysql
INSERT INTO daily_hit_counter(day, slot, cnt)
VALUES(CURRENT_DATE, RAND() * 100, 1)
ON DUPLICATE KEY UPDATE cnt = cnt + 1;
```
​		有时候可以牺牲写性能来提高读性能。这都是常见的优化技巧。

### 4.5 加快 ALTER TABLE 操作的速度

+ 表重建
  ​**方法一：**
  ​		先在一台不提供服务的机器上执行 ALTER TABLE, 然后和提供服务的主库进行切换。
  **方法二：**
  ​		影子拷贝，用要求的表结构创建一张和源表无关的新表，然后通过重命名和删表操作交换两张表。
  
+ 默认值更改
  **方法一：**
    	`ALTER TABLE test MODIFY COLUMN field1 TINYINT(3) NOT NULL DEFAULT 5;`
    	使用 `SHOW STATUS` 会发现这个操作执行了 1k 次的读操作和插入操作。也就是说 MySQL 拷贝了一张新表。但其实不用这样做，因为列的默认值实际上是存储在 .frm 文件中的，所以按理说只要更改这个文件即可。事实上所有的 `MODIFY COLUMN ` 操作都会导致表重建。(查了一下也没查到 show status 该怎么看好)

  **方法二:**
  
  ​	`ALTER TABLE test ALTER COLUMN field1 SET DEFAULT 5;`
  
  ​	这个操作只会修改 .frm 文件而不会重建表，因此是很快的。

#### 4.5.1 只修改 .frm 文件

​		`ALTER TABLE` 允许使用 `MODIFY COLUMN `、`ALTER COLUMN`、`CHANGE COLUMN` 修改列。三种操作都是不一样的。

​		**下面一些操作可能不需要重建表：**

+ 移除一个列的 `AUTO INCREMENT` 属性。

+ 增加、移除、更改 `ENUM` 和 `SET` 常量。

  **基本的技术是为想要的表结构创建一个新的 .frm 文件，然后用它替换掉已经存在的那张表的 .frm 文件。**

+ 创造一张有相同表结构的空表，并进行所需要的修改。比如想增加一个 ENUM 常量

+ 执行 `FLUSH TABLES WITH READ LOCK` 这会关闭所有正在使用的表，并且禁止任何表被打开

+ 交换 .frm 文件

+ 执行 `UNLOCK TABLES` 来释放第2步的读锁

+ 最后删除掉为这个操作而创建的辅助表即可

#### 4.5.2 快速创建 MyISAM 索引
+ 禁用索引 `ALTER TABLE test DISABLE KEYS;`
+ 载入数据(在 .MYD 文件中)
+ 创建索引 `ALTER TABLE test ENABLE KEYS;`
+ 构建索引的工作被延迟到了载入数据之后，之后可以利用排序来创建索引。
**这个方法对唯一索引无效，MyISAM 会在内存中为载入的每一行检查唯一性，并且一旦索引的大小超过了有效内存的大小，载入操作就会越来越慢。当然也可以先删除唯一索引，然后重复上面的操作，然后再创建唯一索引。**

**操作步骤：**

1. 用需要的表结构创建一张表，但是不包括索引。
2. 载入数据到表中以构建 .MYD 文件
3. 按照需要的结构创建另一张空表，这次要包含索引。这会创建需要的 .frm 和 .myi 晚间
4. 获取读锁并刷新表
5. 重命名第二张表的两个文件，让 MySQL 认为这是第一张表
6. 释放读锁
7. 使用 REPAIR TABLE 来重建表的索引。该操作会通过排序来构建所有的索引，当然也包括唯一索引。

### 4.6 总结

1. 使用小而简单的数据类型，尽可能避免使用 NULL 值
2. 尽量使用相同的数据类型来存储真实的值
3. 注意可变长字符串，其在临时表和排序时可能会按照最大长度进行内存分配
4. 尽量使用整型定义标识列
5. 尽量避免使用 MySQL 已经遗弃的特性。如整数的显示宽度。
6. 小心使用 ENUM 和 SET, 最好避免使用 BIT。