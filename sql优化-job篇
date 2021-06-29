# sql优化-job篇
## 背景
一直以来，慢sql都是比较重要，又比较难处理的问题。重要是因为处理不好，容易引发数据库不可用，进而导致线上故障。难处理是因为在各个团队中，慢sql总是像韭菜一样，割一茬长一茬，时不时就会冒出来。根据笔者的经验，慢sql的重灾区往往在2处：其一是复杂查询；其二就是job。本文将针对job进行阐述，讨论一种通用的sql模式，来避免慢sql。

## job中的慢sql
一般而言，业务中会有各种各样的job（定时跑批或者兜底job），用来对数据进行一些批处理。这种场景往往需要在大量数据中根据条件筛选出待处理数据，然后进行业务处理。而筛选“待处理数据”最容易生成慢sql,其主要原因在于数据量大, 一个job往往需要处理大批量数据，少则几千，多则千万，这样的sql很难做到有效查询，往往需要对数据进行分页，主要解决方案有如下。

## 解决方案
### 方案1 offset+limit分页
```java
int begin=0;
while (true) {
	List<Object> pageData = select * from t where {{condition}} offset {{begin}}, limit 100
	if (isEmpty(pageData)) {
		break;//结束，跳出循环。
	}
	process(pageData)
	begin+=100;
}
```
点评：实现简单，但是主要过滤逻辑在数据库，大部分情况下需要回表，并且当begin值增大时查询效率会降低，对数据库有较大压力，不适合数据量比较大的场景。

### 方案2 使用id进行分页
```java
int maxId = select max(id) from t where {{condition}}
int minId = select min(id) from t where {{condition}}

int currentId = minId;
while(true) {
	List<Object> pageData = select * from t where {{condition}} and id >= {{minId}} and id< {{maxId}} limit 100
	if (isEmpty(pageData)) {
		break;//结束，跳出循环。
	}
	process(pageData)
	currentId = getMaxId(pageData)+1;
}

```
点评：实现复杂，sql个数从维护原来的1个变成了3个, 任何一个都可能成为慢sql, 而且需要处理 getMaxId(pageData)的逻辑。

### 推荐方案 覆盖索引+内存分页
```java
List<Long> candidateIds = select id from t where {{part of condition}}; //通过覆盖索引查询符合条件或符合部分条件的id
List<List<Long>> pageIds = partion(candidateIds);//内存分页
for(List<Long> pageId: pageIds) {
	List<Object> pageData = select * from t where id in {{pageId}} and {{other condition}}; //通过主键查询+条件过滤, 也可在内存进行过滤。
	process(pageData)
}

```
点评：简单粗暴的三板斧：1.通过覆盖索引查询所有id 2.对id进行分页 3.对每页id进行主键查询。 使用覆盖索引不仅可以避免回表，而且由于二级索引的数据量小可以减少磁盘页读取次数，进而提高搜索效率。当条件简单时，直接通过覆盖索引即可拿到目标id; 当条件复杂时可以分为两步：第一步走覆盖索引进行初筛，第二步对初筛后的ID进行in查询+条件过滤。而且8M内存即可支持百万级的id个数（以bigint计）， 对大多数场景下都已够用。当然，当查询的id数量比较多的时候（百万级）查询的耗时也会比较长，此时可以结合方案2中对步骤1的查询进行分页。

## 结论
job场景下的慢sql主要难点在于数据量比较大，在各种解决方案中推荐使用覆盖索引+内存分页的方式。
