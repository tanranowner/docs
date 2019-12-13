[toc]

# 1. 存储引擎

## 1. Innodb存储引擎

InnoDB，是MySQL的数据库引擎之一，现为MySQL的默认存储引擎。

innodb_page_size：根据局部性原理，每次IO以页为单位（操作系统默认4KB，innodb为16KB）。

```sql
show variables like 'innodb_page_size';
```

![image-20191213115909444](mysql.assets/image-20191213115909444.png)

其中页的数据结果如下所示。

![image-20191212185947575](mysql.assets/image-20191212185947575.png)

在查找数据时（单页），可以根据二分法，检索页目录，再查找对应的数据。

### 1. 索引

- 主键索引

主键索引为聚集索引，主键索引会默认排序（插入的数据会按照主键进行排序后保存在磁盘）。入插入主键字段数据3，2，1，则在保存时，会按照1，2，3进行保存，select出来的数据也是这个数据。如果没有自定义主键，则选取一个unique字段为主键，否则innodb会新增加一列rowid（隐藏列，由其它主键的情况下，不会添加该列）作为主键。

```sql
CREATE TABLE `demo` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  `c` int(11) DEFAULT NULL,
  `d` varchar(45) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_demo_a_b_c` (`a`,`b`,`c`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

insert into demo(id,a,b,c,d) value(5,4,3,2,'e');
insert into demo(id,a,b,c,d) value(4,3,2,1,'b');
insert into demo(id,a,b,c,d) value(3,2,1,5,'c');
insert into demo(id,a,b,c,d) value(2,1,5,4,'d');
insert into demo(id,a,b,c,d) value(1,5,4,3,'a');
select * from demo;
```

全表查询结果会自动按照主键（聚集）索引排序。

![image-20191213121548488](mysql.assets/image-20191213121548488.png)

```sql
select * from demo where a >= 3;
```

过滤查询的默认结果也会按照主键（聚集）索引排序。

![image-20191213121846023](mysql.assets/image-20191213121846023.png)

- 聚集索引

索引和数据都是集中在一起，如果表有主键，则默认聚集索引为主键索引，否则找下一个非NULL索引，如果没有其它索引，则默认以rowid为聚集索引进行保存数据。

- B+树索引

主键索引默认为B+树。Innodb默认为三层深度，如果行数据超过2000w行，则升级为四层深度（影响效率）。

聚集索引的B+树。可以看到索引和数据在一起。

![image-20191212200725219](mysql.assets/image-20191212200725219.png)

非聚集索引的B+树。可以看到索引和数据不在一起，是和主键在一起，所以要再通过主键回表，到聚集索引再查询一次，所以效率不高。

![image-20191212200749034](mysql.assets/image-20191212200749034.png)

## 2. 查询优化

### 1. 开启查询优化器跟踪

由于查询优化器跟踪默认是关闭的，所以首先需要开启。

```sql
show variables like 'optimizer_trace';
set optimizer_trace = 'enabled=on';
```

![image-20191213124047207](mysql.assets/image-20191213124047207.png)

开启完毕之后，就可以通过下面的sql查询日志（在执行完毕业务查询sql之后才能执行查询日志sql）。

```sql
select * from demo where id = 3 or a = 2;
select * from information_schema.optimizer_trace;
```

![image-20191213124510791](mysql.assets/image-20191213124510791.png)

其中的trace就是下面部分。

```json
{
	"steps": [
		{
			"join_preparation": {
				"select#": 1,
				"steps": [
					{
						"expanded_query": "/* select#1 */ select `demo`.`id` AS `id`,`demo`.`a` AS `a`,`demo`.`b` AS `b`,`demo`.`c` AS `c`,`demo`.`d` AS `d` from `demo` where ((`demo`.`id` = 3) or (`demo`.`a` = 2)) limit 0,1000"
					}
				]
			}
		},
		{
			"join_optimization": {
				"select#": 1,
				"steps": [
					{
						"condition_processing": {
							"condition": "WHERE",
							"original_condition": "((`demo`.`id` = 3) or (`demo`.`a` = 2))",
							"steps": [
								{
									"transformation": "equality_propagation",
									"resulting_condition": "(multiple equal(3, `demo`.`id`) or multiple equal(2, `demo`.`a`))"
								},
								{
									"transformation": "constant_propagation",
									"resulting_condition": "(multiple equal(3, `demo`.`id`) or multiple equal(2, `demo`.`a`))"
								},
								{
									"transformation": "trivial_condition_removal",
									"resulting_condition": "(multiple equal(3, `demo`.`id`) or multiple equal(2, `demo`.`a`))"
								}
							]
						}
					},
					{
						"substitute_generated_columns": {}
					},
					{
						"table_dependencies": [
							{
								"table": "`demo`",
								"row_may_be_null": false,
								"map_bit": 0,
								"depends_on_map_bits": []
							}
						]
					},
					{
						"ref_optimizer_key_uses": []
					},
					{
						"rows_estimation": [
							{
								"table": "`demo`",
								"range_analysis": {
									"table_scan": {
										"rows": 5,
										"cost": 4.1
									},
									"potential_range_indexes": [
										{
											"index": "PRIMARY",
											"usable": true,
											"key_parts": [
												"id"
											]
										},
										{
											"index": "idx_demo_a_b_c",
											"usable": true,
											"key_parts": [
												"a",
												"b",
												"c",
												"id"
											]
										}
									],
									"setup_range_conditions": [],
									"group_index_range": {
										"chosen": false,
										"cause": "not_group_by_or_distinct"
									},
									"analyzing_range_alternatives": {
										"range_scan_alternatives": [],
										"analyzing_roworder_intersect": {
											"usable": false,
											"cause": "too_few_roworder_scans"
										}
									},
									"analyzing_index_merge_union": [
										{
											"indexes_to_merge": [
												{
													"range_scan_alternatives": [
														{
															"index": "PRIMARY",
															"ranges": [
																"3 <= id <= 3"
															],
															"index_dives_for_eq_ranges": true,
															"rowid_ordered": true,
															"using_mrr": false,
															"index_only": true,
															"rows": 1,
															"cost": 1.21,
															"chosen": true
														}
													],
													"index_to_merge": "PRIMARY",
													"cumulated_cost": 1.21
												},
												{
													"range_scan_alternatives": [
														{
															"index": "idx_demo_a_b_c",
															"ranges": [
																"2 <= a <= 2"
															],
															"index_dives_for_eq_ranges": true,
															"rowid_ordered": false,
															"using_mrr": false,
															"index_only": true,
															"rows": 1,
															"cost": 1.21,
															"chosen": true
														}
													],
													"index_to_merge": "idx_demo_a_b_c",
													"cumulated_cost": 2.42
												}
											],
											"cost_of_reading_ranges": 2.42,
											"cost_of_mapping_rowid_in_non_clustered_pk_scan": 0.1,
											"cost_sort_rowid_and_read_disk": 1,
											"cost_duplicate_removal": 0.1881,
											"total_cost": 3.7081
										}
									],
									"chosen_range_access_summary": {
										"range_access_plan": {
											"type": "index_merge",
											"index_merge_of": [
												{
													"type": "range_scan",
													"index": "PRIMARY",
													"rows": 1,
													"ranges": [
														"3 <= id <= 3"
													]
												},
												{
													"type": "range_scan",
													"index": "idx_demo_a_b_c",
													"rows": 1,
													"ranges": [
														"2 <= a <= 2"
													]
												}
											]
										},
										"rows_for_plan": 2,
										"cost_for_plan": 3.7081,
										"chosen": true
									}
								}
							}
						]
					},
					{
						"considered_execution_plans": [
							{
								"plan_prefix": [],
								"table": "`demo`",
								"best_access_path": {
									"considered_access_paths": [
										{
											"rows_to_scan": 2,
											"access_type": "range",
											"range_details": {
												"used_index": "sort_union(idx_demo_a_b_c,PRIMARY)"
											},
											"resulting_rows": 2,
											"cost": 4.1081,
											"chosen": true
										}
									]
								},
								"condition_filtering_pct": 100,
								"rows_for_plan": 2,
								"cost_for_plan": 4.1081,
								"chosen": true
							}
						]
					},
					{
						"attaching_conditions_to_tables": {
							"original_condition": "((`demo`.`id` = 3) or (`demo`.`a` = 2))",
							"attached_conditions_computation": [],
							"attached_conditions_summary": [
								{
									"table": "`demo`",
									"attached": "((`demo`.`id` = 3) or (`demo`.`a` = 2))"
								}
							]
						}
					},
					{
						"refine_plan": [
							{
								"table": "`demo`"
							}
						]
					}
				]
			}
		},
		{
			"join_execution": {
				"select#": 1,
				"steps": []
			}
		}
	]
}
```

equality_propagation：等值传递。如，where a=b and b=c and c=1，优化后则为where a=1 and b=1 and c=1。

constant_propagation：常量传递。如，where a>b and b=1，优化后则为where a>1 and b=1。

trivial_condition_removal：删除无用的条件。如，where 1=1 and a=1，优化后则为where a=1。

### 2. 基于成本

mysql查询优化会选择一个成本最低的执行方案。包含CPU和IO成本。

>Innodb存储引擎规定读取一个页的成本默认为1.0，读取以及检测一条记录是否符合搜索条件的成本默认为0.2。优化步骤

### 3. 优化步骤

通过下面的sql可以查询出表的统计信息。

```sql
show table status like 'demo' --该表的数据默认不是实时更新，默认变动记录数超过10%才更新。也可以手动更新
analyze table demo;
```

1. 全表扫描（table_scan）

- IO：主键（聚集）索引页数（即全表页数）=表数据大小/16KB=m页。则IO成本=m页x1.0。
- CPU：CPU成本=全表的行数x0.2=n行x0.2。

```sql
select * from demo where id = 3 or a = 2;
```

上面sql的优化器追踪。

![image-20191213193037431](mysql.assets/image-20191213193037431.png)

2. 索引成本

- 主键（聚集）索引成本

主键索引成本=范围区间数IO成本+范围记录数CPU成本。

范围区间数IO成本：一个范围区间成本为1，=，<，>，in等。如，where id>1 and id<3成本为1。

范围记录数CPU成本（range_scan_alternatives）：该成本=范围记录数x0.2。

- 非聚集索引成本

非聚集索引成本=非聚集索引查询成本+回表成本。

回表成本：通过非聚集索引查询到的主键，再到主键索引中查询的成本。如查询非聚集索引字段where a>2，在非聚集索引中查到对应主键为1，4，5，则回表成本为where id in (1,4,5)。

最终的索引将有可能会使用成本最小的索引。最终的优化器也会比较索引成本和全表扫描成本，选择最小的执行路径。

## 3. 连接

所有表连接都需要一个驱动表和一个被驱动表。对于内连接，选取那个驱动表都是一样的；对于外连接，驱动表和被驱动表都是固定的，左连接的驱动表是左边的表，右连接的驱动表是右边的表。

连接的大致原理：

- 选取驱动表，使用与驱动表相关的过滤条件，选择最低成本的查询方式对驱动表查询；

- 对上面步骤中的查询结果集中的每条结果，都分别到被驱动表中查找匹配的记录。对应的伪代码如下所示。

  ```sql
  foreach row i in driveTableRows    // 遍历满足条件的驱动表每条记录
  	foreach row j in drivemTableRows // 遍历被驱动表中满足驱动表中的每条记录（on 条件）
  		if rowi.id == rowj.id 
  			result.add rowj
  ```

  如，select * from t1 join t2 on t1.id=t2.id，等价于下面的sql的合集。

  ```sql
  select * from t2 where t2.id=1;
  select * from t2 where t2.id=2;
  select * from t2 where t2.id=3;
  select * from t2 where t2.id=4;
  select * from t2 where t2.id=5;
  ```

### 1. 基于块的嵌套连接算法

基于上面的原理，mysql由join_buffer_size的概念。可以每次从驱动表中获取256KB（16页）的数据，再从驱动表中匹配。这样就是基于块的嵌套查询，减少IO降低成本。如下所示。

```sql
select * from t2 where t2.id in (1,2,3,4,5);
```

> 默认join_buffer_size的大小为256KB，最小可以设置为128B。

### 2. 外连接消除

由于外连接的驱动表和被驱动表是固定的，而内连接的表的驱动表和被驱动表可以根据成本调整。所以外连接无法优化表的连接顺序。

外连接和内连接的本质就是：**对于外连接的驱动表来说，如果无法在被驱动表中匹配on条件中的记录，那么该记录会以全NULL值填入到结果集。而内连接驱动表中的记录如果无法在被驱动表中找到匹配on条件中的记录，那么该记录就会丢弃。**

如，select * from t1 left join t2 where t1.id = t2.id结果为

| a    |      | a    |      |
| ---- | ---- | ---- | ---- |
| 1    | ……   | 1    | ……   |
| 2    | ……   | 2    | ……   |
| 3    | ……   | null | null |

如果该sql可以从**业务**上优化为select * from t1 left join t2 where t1.id = t2.id where t2.b is not null，则会减少不匹配的记录，并且优化器可以消除外连接，优化改为内连接方式（可以从执行计划看出）。

## 4. 子查询







