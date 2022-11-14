## SQL - SEQUEL
Structured English Query Language 

## Aggregates 聚合函数
> Functions that return a single value from a bag of tuples.  
> Aggregate functions can only be used in the SELECT output list.

- AVG(col)
- MIN(col)
- MAX(col)
- SUM(col)
- COUNT(col)

* SELECT COUNT(*) AS cnt FROM student WHERE login like '@cs'; 等价于 COUNT(login), COUNT(1)

- COUNT, SUM, AVG support DISTINCT
* SELECT COUNT(DISTINCT login) FROM student;

- GROUP BY
- Project tuples into subsets and calculate aggregates against each subset.
* SELECT AVG(s.gpa), e.cid FROM enrolled AS e, student AS s WHERE e.sid = s.sid GROUP BY e.cid;

- Non-aggregated values in SELECT output clause must appear in GROUP BY clause.
* SELECT AVG(s.gpa), e.cid, s.name FROM enrolled AS e, student AS s WHERE e.si = s.sid GROUP BY e.cid, s.name;

- HAVING
- Filters results based on aggregation computation.
* Like a WHERE clause for a GROUP BY.
	SELECT AVG(s.gpa) AS avg_gpa, e.cid FROM enrolled AS e, student AS s WHERE e.sid = s.sid AND avg_gpa > 3.9 GROUP BY e.cid;
	- 这句sql是错误的因为进行where时还没有avg_gpa的存在

* SELECT AVG(s.gpa) AS avg_gpa, e.cid FROM enrolled AS e, student AS s WHERE e.sid = s.sid GROUP BY e.cid HAVING avg_gpa > 3.9;

## String operations
- MySQL不区分大小写，且允许使用单/双引号

- LIKE is used for string matching.
	- '%' Matches any substring
	- '_' Matches any one character
* SELECT * FROM enrolled AS e WHERE e.cid LIKE '15-%';
* SELECT * FROM student AS s WHERE s.login LIKE '%@c_';

- '+' concatenate two or more strings to together.
* SELECT name FROM student WHERE login = LOWER(name) + '@cs';

## DATE/TIME operations
- Operations to manipulate and modify DATE/TIME attributes.
- Can be used in either output and predicates.

* SELECT NOW();
* SELECT CURRENT_TIMESTAMP();
* SELECT CURRENT_TIMESTAMP;
* SELECT EXTRACT(DAY FROM DATE('2022-11-08'));  // 8
* SELECT DATE('2022-11-08') - DATE('2022-01-01') AS DAYS; // 1007
* SELECT ROUND((UNIX_TIMESTAMP(DATE('2022-11-08')) - UNIX_TIMESTAMP(DATE('2022-01-01'))) / (60*60*24), 0) AS DAYS;  // 311,UNIX_TIMESTAMP从1970年1月开始计算

## Output redirection
> Store query results in another table

- Table must not already be defined.
- Table will have the same # of columns with the same types as the input.
* CREATE TABLE CourseIds (SELECT DISTINCT cid FROM enrolled);

- Insert tuples from query into another table:
	- Inner SELECT must generate the same columns as the target table.
	- DBMSs have different options/syntax on what to do with duplicates.
* INSERT INTO CourseIds (SELECT DISTINCT cid FROM enrolled);

- ORDER BY <column*> [ASC|DESC]
	- Order the output tuples by the values in one or more of their columns.
* SELECT sid, grade FROM enrolled WHERE cid = '15-721' ORDER BY grade DESC, sid ASC;

- LIMIT <count> [offset]
	- Limit the # of tuples returned in output.
	- Can set an offset to return a "range".
* SELECT sid, name FROM student WHERE login LIKE '%@cs' LIMIT 20 OFFSET 10;

## Nested queries 嵌套查询

- 先从外部查询开始构建
* SELECT name FROM student WHERE sid IN (SELECT sid FROM enrolled where cid = '15-445');

- ALL: must satisfy expression for all rows in sub-query
- ANY: must satisfy expression for at least one row in sub-suery
- IN:  equivalent to '=ANY()'
- EXISTS: at least one row is returned.
* SELECT name FROM student WHERE sid = ANY(SELECT sid FROM enrolled where cid = '15-445'); // 从student表中找出所有sid，再和enrolled表中sid集合进行比较，返回相等的部分(即求这两个表的交集)

* SELECT (SELECT S.name FROM student AS S WHERE S.sid = E.sid) AS sname FROM enrolled AS E WHERE cid = '15-445' // 在select中做了一次join，与上述处理表的过程相反

Ex: Find student name, sid with the highest sid that is enrolled in at least one course.
* SELECT sid, name FROM student WHERE sid >= ALL(SELECT sid FROM enrolled);
* SELECT sid, name FROM student WHERE sid IN (SELECT MAX(sid) FROM enrolled);
* SELECT MAX(sid), name FROM student WHERE sid IN (SELECT sid FROM enrolled); // error
* SELECT sid, name FROM student WHERE sid IN (SELECT sid FROM enrolled ORDER BY sid DESC LIMIT 1);

Ex: Find all courses that has no students enrolled in it.
* SELECT * FROM course WHERE NOT EXISTS(SELECT * FROM enrolled WHERE course.cid = enrolled.cid); // 内部查询才可以引用外部查询的东西


## Window functions
- Performs a calculation across a set of tuples that related to a single row
- Like an aggregation but tuples are not grouped into a single output tuples

- ROW_NUMBER(): # of the current row
- RANK(): order position of the current row
* SELECT *, ROW_NUMBER() OVER() AS row_num FROM enrolled;

- OVER keyword specifies how to group together tuples when computing the window function. Use PARTITION BY to specify group, use ORDER BY to sort entires in each group.
* SELECT cid, sid, ROW_NUMBER() OVER(PARTITION BY(cid)) FROM enrolled ORDER BY cid;
* SELECT *, ROW_NUMBER() OVER (ORDER BY cid) FROM enrolled ORDER BY cid;

Ex: Find the student with the highest grade for each course.
* SELECT * FROM (SELECT *, RANK() OVER (PARTITION BY cid ORDER BY grade ASC)) as r FROM enrolled) AS ranking WHERE ranking.r = 1;


## CTE (Common table expressions 公用表表达式)
> Provides a way to write auxiliary statements for use in a larger query.
> Think of it like a temp table just for one query.

* WITH cteName (col1, col2) AS (SELECT 1, 2) SELECT col1 + col2 FROM cteName;

Ex: Find student record with the highest id that is enrolled in at least one course.
* WITH cteSource (maxId) AS (SELECT MAX(sid) FROM enrolled) SELECT name FROM student, cteSource WHERE student.sid = cteSource.maxId;

- 和嵌套查询的区别在于，CTE可以递归，但嵌套查询不可以
Ex:输出1到10
* WITH RECURSIVE cteSource (counter) AS ((SELECT 1) UNION ALL (SELECT counter + 1 FROM cteSource WHERE counter < 10)) SELECT * FROM cteSource;