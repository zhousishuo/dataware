1.现有一个成绩表t_score，列名分别为uid、course和score (如表1)，每个uid在每门课程上只有一个成绩，统计学生在Chinese、English和Math3门课程上的成绩,展现形式如表2。

表1：t_score

| uid  | course  | score |
| ---- | ------- | ----- |
| 1    | Chinese | 90    |
| 2    | Math    | 80    |
| 3    | English | 88    |

表2：

| uid  | Chinese | English | Math |
| ---- | ------- | ------- | ---- |
| 1    | 90      | null    | null |
| 2    | null    | null    | 80   |
| 3    | null    | 88      | null |

```sql
方法1：
select  uid 
       ,sum(case course when 'Chinese' then score end) as 'Chinese'
       ,sum(case course when 'English' then score end) as 'English'
       ,sum(case course when 'Math'    then score end) as 'Math'
  from t_score 
  group by uid
```

```sql
方法2：
select uid
      ,sum(Chinese)  Chinese
      ,sum(English)  English
      ,sum(Math)     Math
  from
      (
          select  uid
                ,score  as Chinese
                ,0      as English
                ,0      as Math
           from  t_score
           where course = 'Chinese'
         union all
         select  uid
                ,0
                ,score  as English
                ,0
           from  t_score
           where course = 'English'
         select  uid
                ,0
                ,0
                ,score  as Math
           from  t_score
           where course = 'Math'
       )t 
  group by uid
;
```

分析：这两种方法更推荐方法1，因为方法1只用扫描一遍表，效率更快，而且代码也更简单一点。

2.给学生成绩排序，如果两个分数相同，那么它们有相同的排名，接下来的排名应该是连续的没有跳跃。

表1是排序前的顺序，表2是排序后的结果

表1：t_score

| id   | score |
| ---- | ----- |
| 1    | 90    |
| 2    | 88    |
| 3    | 90    |
| 4    | 87    |

表2：排序后

| id   | score | rank |
| ---- | ----- | ---- |
| 1    | 90    | 1    |
| 3    | 90    | 1    |
| 2    | 88    | 2    |
| 4    | 87    | 3    |

```sql
SELECT id
      ,score 
      ,dense_rank() over(order by score desc) rank
  FROM  t_score
```

拓展延伸：在hive里面，常用的排序函数还有row_number()和rank()函数，row_number()即使相同字段也会连续排序，连续计数；但是rank()函数是相同取值的字段排序一样，但是接下来的计数不是连续的。

```sql
SELECT id
      ,score 
      ,row_number() over(order by score desc) rn 
      ,rank() over(order by score desc) rank_rn
      ,dense_rank() over(order by score desc) dense_rank
  FROM  t_score
```

排序结果如下

| id   | score | rn   | rank_rn | dense_rank |
| ---- | ----- | ---- | ------- | ---------- |
| 1    | 90    | 1    | 1       | 1          |
| 3    | 90    | 2    | 1       | 1          |
| 2    | 88    | 3    | 3       | 2          |
| 4    | 87    | 4    | 4       | 3          |

3.