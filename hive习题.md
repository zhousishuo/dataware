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

3.某网站需要计算连续3天及3天以上登录该网站的用户id，现有表id, log_time，用户每天只会登录该网站一次。数据如下表。

表1：t_log

| id   | log_time |
| ---- | -------- |
| 1    | 20180304 |
| 4    | 20180308 |
| 4    | 20180309 |
| 1    | 20180303 |
| 1    | 20180305 |
| 2    | 20180304 |
| 2    | 20180307 |
| 3    | 20180305 |
| 4    | 20180307 |

```sql
方法1：
SELECT  id
  FROM
      (
      	SELECT id
              ,log_time
              ,count(id) over(partition by id order by log_time range BETWEEN 2 preceding and current row) rn
         FROM t_log
      )t 
WHERE rn = 3
GROUP BY id
```

```sql
方法2：
SELECT  id
  FROM
      (
      	SELECT  id
               ,log_time
               ,lead(log_time,2) over(partition by id order by log_time) next_log_time
          FROM  t_log
      )t 
  WHERE log_time = next_log_time-2
  GROUP BY id
```

```sql
方法3：
select id
  from 
      (
        select  id
               ,(log_time - rn)  new_time
               ,count(1)         log_cnt
          from
              (
                select  id
                       ,log_time
                       ,row_number() over(partition by id order by log_time) rn
                  from  t_log
              )t 
          group by id, (log_time-rn)
      )t 
 where log_cnt >= 3
 group by id
```

方法分析：方法1比较简单，主要用了count() over()窗口函数，如果理解了over()中 range between and 的含义，这道题就能理解了，建议多尝试几次，看看中间结果；方法2在理解上比较简单，主要使用了lead() over()函数获取之后的第二次登录时间next_log_time，在比较next_log_time减2是否和log_time相等；方法3比较巧妙，它的精髓主要在于如果是连续登录，那么对于用户来说log_time-rn 将是相同的值，那么只需要count(id)>=3即可。

4.大学里学生进行体育成绩测试，规定学生可以测试一次或多次，学生每次体育成绩记录在表t_score，现需要将学生的多次成绩变成一列，并将成绩进行排序，结果如表t_new。

表t_score

| id   | score |
| ---- | ----- |
| 1    | 90    |
| 1    | 80    |
| 1    | 90    |
| 2    | 90    |
| 2    | 88    |

表t_new

| id   | score      |
| ---- | ---------- |
| 1    | [80,80,90] |
| 2    | [88,90]    |

```sql
select  id
       ,sort_array(collect_list(score))  score
  from  t_score
  group by id
```

如果想去掉重复的得分，比如id = 1的学生，两次得分重复了，想要将score变成[80,90]，那么可以使用collect_set()函数

```sql
select  id
       ,sort_array(collect_set(score))  score
  from  t_score
  group by id
```

结果如下：

| id   | score   |
| ---- | ------- |
| 1    | [80,90] |
| 2    | [88,90] |

注意：collect_set()和collect_list() 的返回类型是array，也就是返回的是数组类型，如果需要将结果存储到一张表t_new里，建表语句中score是string类型，那么直接存储就会报错，因为array类型不能直接转换成string，可以使用concat_ws()转一下，因为concat_ws(string SEP, array<string>)返回值类型是string，实现逻辑如下。

```sql
select  id
       ,concat_ws(',', collect_list(cast(score as STRING)))
  from  t_score
  group by id
```

5.有一张天气表Weather，需要找出今天相比于昨天温度更高的日期的Id。

表Weather

| id(BIGINT) | record_date(STRING) | temperature(BIGINT) |
| ---------- | ------------------- | ------------------- |
| 1          | 2019-03-01          | 10                  |
| 2          | 2019-03-02          | 25                  |
| 3          | 2019-03-03          | 20                  |
| 4          | 2019-03-04          | 30                  |
| 5          | 2019-03-06          | 40                  |

符合条件的Id如下：

| id   |
| ---- |
| 2    |
| 4    |

```sql
方法一：使用lag()函数来实现
select id
  from
      (
      	select  id
      	       ,record_date
      	       ,temperature
               ,lag(temperature) over(order by record_date) temperature_front
               ,lag(record_date) over(order by record_date) record_date_front
          from Weather
      )t 
  where temperature > temperature_front
  and date_sub(record_date,1) = record_date_front
```

```sql
方法二：使用Join的方式，Join的时候on后面的条件只能用来判断相等的情况，不能用来判断大于小于这种情况，这种判断需要放在where条件中。
select  t1.id
  from
      (
      	select  id
               ,record_date
               ,temperature
          from Weather
      )t1 
 inner join
      (
      	select  record_date
               ,temperature
          from Weather
      )t2 
    on date_sub(t1.record_date,1) = t2.record_date
 where t1.temperature > t2.temperature
;
```

