### 132B Database Apps Review Notes

***

**Part I: Database Theory**

*Mapping Cardinalities*

Set A: {$a_1$, $a_2$, ..., $a_i$, ... $a_n​$} 

Set B: {$b_1$, $b_2$, ..., $b_i$, ... $b_n​$} 

One to Many: one $a_i$ can map to many $b_i$. E.g. one person can have many properties. 

Many to One: several $a_i$ can map to one $b_i$. E.g. many properties can belong to one person. 

Many to Many: 

injection from A -> B is one to many; 

injection from B -> A is also one to many;

*Functional Dependencies*

A -> B means: A can determine B. More formally: if tuples (t1, t2) have the same A, then they have the same B.

Functional dependency is trivial if B is a subset of A.

*3NF*

A scheme R is in 3NF iff X -> A for a non-trivial functional dependency, then X is a key or A is a prime. 

Basically translates into:

A key in table determines every attribute. 

3NF Checking Algorithm 

*Concept of Closure*: Denote closure as $F^+$ by using Armstrong's Axioms:

1) reflectivity 

2) augmentation $A->B$ then cA -> cB

3) transitivity 

*Computing the Attribute Closure*

Input: An attribute set 

Output: An attribute set that has applied all functional dependencies 

If the output is R, then the input attribute set is the key.

```
Possible Exam Question on 3NF and Closure:
- Given a relation and a set of functional dependencies, check if the relation is in 3NF
- Test if a table is 3NF 
- Decompose a table into 3NF Form
```



**Example for 3NF**

```
R1(ABCD)
ACD -> B   AC -> D   D -> C   AC -> B 

R2(ABCD)
AB -> C   ABD -> C   ABC -> D   AC -> D

R3(ABCD)
C -> B   A -> B   CD -> A   BCD -> A

R4(ABCD)
C -> B   B -> A   AC -> D   AC -> B
```

```
R1:
keys: AC, AD
prime attributes: A, C, D
It is in 3NF. 

R2:
keys: AB
prime attributes: A, B
AC -> D: RHS is not a prime, so it is not in 3NF.

R3:
keys: CD, 
prime attributes: C, D
C -> B: LHS is not a superkey, RHS is not a prime, so R3 is not in 3NF.

R4:
keys: C
prime attributes: C
B -> A: violates 3NF condition so R4 is not in 3NF.
```



***

**Part II: SQL Queries**

(1) Multi-way joins 

Tables: 

classes (**id**, name, number, date_code, start_time) primary key is id 

Example entry for classes table: (id, 'Database Apps', 'CSE132B', 'W', 3:00pm)

students(**id**, pid, first, last) primary key is id 

enrollments (**class_id**, **student_id**, credits) primary key is class_id and student id 

1. find first, last name of all students 

```sql
SELECT first, last FROM students
```

2. find pid, name, credits of CSE132B enrollees

```SQL
-- using implicit join 
SELECT pid, first, last, credits FROM students, enrollments, classes
WHERE classes.number = 'CSE132B' AND classes.id = enrollments.class_id
AND students.id = enrollments.student_id 

-- using explicit join 
SELECT student.pid, student.first, student.last, enrollment.credits
FROM (classes INNER JOIN enrollments ON classes.id = enrollments.class_id)
INNER JOIN students ON students.id = enrollments.student_id 
WHERE classes.number = 'CSE132B'
```

3. find other classes that CSE 132B students take 

```SQL 
-- soln 1
SELECT s.pid, s.first, s.last, cother.number, eother.credits
FROM student s, enrollments e132, classes c132, enrollments eother, classes cother
WHERE c132.number = 'CSE132B' 
	AND c132.id = e132.class_id
	AND s.id = e132.student_id 
	AND cother.id = eother.class_id
	AND s.id = eother.student_id
	AND eother.id <> c132.id 

-- soln 2
-- step 1: create table to store cse132 students 
SELECT student.id, student.pid, student.first, student.last INTO CSE132BStudents
FROM (classes INNER JOIN enrollments ON classes.id = enrollments.class_id)
INNER JOIN students ON students.id = enrollments.student_id 
WHERE classes.number = 'CSE132B';

-- step 2: INNER JOIN CSE132BStudents with enrollments and classes to find out about other classes
SELECT CSE132BStudents.pid, CSE132BStudents.first, CSE132BStudents.last, classes.number, enrollments.credits
INNER JOIN enrollments ON CSE132BStudents.id = enrollments.student_id 
INNER JOIN classes ON enrollments.class_id = classes.id 
WHERE classes.number <> 'CSE132B';
```

4. find 132B students who take 11:00 AM classes on Friday 

```SQL
SELECT s.pid, s.first, s.last, cother.number, eother.credits
FROM student s, enrollments e132, classes c132, enrollments eother, classes cother
WHERE c132.number = 'CSE132B' 
	AND c132.id = e132.class_id
	AND s.id = e132.student_id 
	AND cother.id = eother.class_id
	AND s.id = eother.student_id
	AND cother.date_code = 'F'
	AND cother.start_time = 11:00
```



(2) Building recommendation system in SQL

table:

likes(uid, vid, stars)

1. compute the similarity (inner product) of 2 users 

```SQL
CREATE VIEW similarity AS 
	SELECT l1.uid, l2.uid, SUM(l1.stars * l2.stars) AS rank 
	FROM likes l1, likes l2 
	WHERE l1.vid = l2.vid 
	GROUP BY l1.uid, l2.uid
```

2. recommend top 20 videos by overall stars, do not recommend videos watched by the user 

define liked videos as >= 4 stars

```SQL
-- create ranked_video table that stores qualified movies 
CREATE VIEW ranked_video AS 
	SELECT vid, count(*)
	FROM likes WHERE stars >= 4
	GROUP BY vid
	
-- recommend top 20 movies to user u for movies u did not watch 
SELECT vid, rank
FROM ranked_video 
WHERE NOT ranked_video.vid NOT IN (
	SELECT vid FROM likes 
    WHERE uid = u
)
ORDER BY ranked_video.rank DESC 
LIMIT 20 
```

3. recommend top 20 movies liked by user x's friend; similarly, liked is defined by a review of 4 stars and above 

tables: likes(uid, vid, stars); friends (uid1, uid2)l users(uid)

```SQL
-- create view to store movies liked by user u's friends 
CREATE VIEW liked_by_friends
	SELECT u.uid, l.vid, count (*) AS rank 
	FROM likes l, friends f, users u 
	WHERE f.uid1 = u.uid AND f.uid2 = l.uid 
	AND l.stars >= 4 
	GROUP BY u.uid, l.vid 

-- recommendation query when u logs in 
SELECT lbf.vid 
FROM liked_by_friends lbf
WHERE lbf.uid = x
AND NOT EXISTS (SELECT * FROM watched WHERE uid = x AND vid = lbf.vid)
ORDER BY lbf.rank DESC
LIMIT 20 
```

4. create a view to store the similarity between each pair of users 

user 1 vector = [1, 0, 0, 1, 1, .... 1 ]. Each position represents the index of the movie. 1 represents user 1 likes this movie (gives it a rating for stars >= 4) and 0 otherwise. Therefore the similarity score sim(u1, u2) = bitwise AND of user 1 vector and user 2 vector. 

```sql
CREATE VIEW sim AS 
	SELECT l1.uid, l2.uid, count(*) AS sim_score
	FROM likes l1, likes l2 
	WHERE l1.vid = l2.vid 
	AND l1.uid <> l2.uid 
	AND l1.stars >= 4 AND l2.stars >= 4 
	GROUP BY l1.uid, l2,uid 
```



(3) Trigger

**Average GPA Example **

GPA table: GPA(sid, sum, count) -- sum is the total for GPA and count is the # of courses 

Enrollment table: (sid, courseid, grade)

create a trigger that incrementally updates the GPA table when there is an insertion at enrollment table. 

```SQL
CREATE OR REPLACE FUNCTION check_enrollment()
	RETURN trigger AS
	$$
	BEGIN
		-- if student does not exist in the GPA table 
		-- add an entry 
		IF NOT EXISTS(SELECT sid FROM GPA WHERE sid = NEW.sid)
		THEN 
			INSERT INTO GPA VALUES (NEW.sid, NEW.grade, 1)
		ELSE
		-- if student already exists in the GPA table, update sum and count 
			UPDATE GPA SET GPA.sum = GPA.sum + NEW.grade
			WHERE GPA.sid = NEW.sid; 
			UPDATE GPA SET GPA.count = GPA.count + 1
			WHERE GPA.sid = NEW.sid;
		END IF 
	END
	$$
```

***

**Part III: Row Storage vs. Column Storage**









***

