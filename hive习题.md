1.现有一个成绩表t_score，列名分别为uid、course和score (如图1)，每个uid在每门课程上只有一个成绩，统计学生在Chinese、English和Math3门课程上的成绩,展现形式如图2。

图1：t_score

| uid  | course  | score |
| ---- | ------- | ----- |
| 1    | Chinese | 90    |
| 2    | Math    | 80    |
| 3    | English | 88    |

图2：

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

2.