---
title: nodejs OOM in production
date: 2021-01-14 14:12:55
tags:
- CaseStudy
- Tips
- Code
excerpt: share a troubleshooting for OOM problem
---

### 1. Backgroud
API slow in production and a OOM error in log.

### 2. Research
#### 2.1 code review
check the code and debug the slow code looks like below:
```
const org = await DB.Org.findByPk(orgId, {
    include: [DB.Project, DB.User, DB.Subject],
  });
```
the above code use the ORM(Sequelize) find the org and relation entites(project/user/subject).

#### 2.2 find/format/simplify sql
```
SELECT *
FROM `Orgs` `Org`
	LEFT JOIN `Subjects` `Subjects` ON `Org`.`id` = `Subjects`.`orgId` # 1512
	LEFT JOIN `Projects` `Projects` ON `Org`.`id` = `Projects`.`orgId` # 635
	LEFT JOIN `Users` `Users` ON `Org`.`id` = `Users`.`orgId` # 14
WHERE `Org`.`id` = 1;
```
#### 2.3 separate the sql and exec
- only left join Subjects: 1512 rows, 35ms
```
SELECT *
FROM `Orgs` `Org`
	LEFT JOIN `Subjects` `Subjects` ON `Org`.`id` = `Subjects`.`orgId`
WHERE `Org`.`id` = 1;
```
- only left join Projects: 635 rows, 26ms
```
SELECT *
FROM `Orgs` `Org`
	LEFT JOIN `Projects` `Projects` ON `Org`.`id` = `Projects`.`orgId`
WHERE `Org`.`id` = 1;
```
- only left join Users: 14 rows, 7ms
```
# only for left join Users
# 14 rows, 7ms
SELECT *
FROM `Orgs` `Org`
	LEFT JOIN `Users` `Users` ON `Org`.`id` = `Users`.`orgId` # 14
WHERE `Org`.`id` = 1;
```

The reason is very clear now. If we join three tables, the total rows count is 1512 * 635 * 14 = **13441680**.
**13441680** rows with all columns are too big for v8 engine in nodejs.

### 3. How to resolve?
Separate query for Subject/User/Project is the solution. The three paralle query maybe a better solution.
