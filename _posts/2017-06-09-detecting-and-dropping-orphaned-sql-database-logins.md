---
layout: post
title: Detecting and dropping orphaned SQL database logins
date: 2017-06-09 12:23:26.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- SQL
tags:
- SQL 2016 Availability Group Orphaned Users
meta:
  _publicize_job_id: '5937885631'
  _rest_api_client_id: "-1"
  _rest_api_published: '1'
author:
  login: jrudlingmailcom
  email: jrudlin@gmail.com
  display_name: jrudlin
  first_name: ''
  last_name: ''
permalink: "/2017/06/09/detecting-and-dropping-orphaned-sql-database-logins/"
---
Sometimes it might be useful to find all SQL and Windows logins inside all databases within an ┬áinstance of SQL that do not have associated logins. If the database is not in Contained mode, then instance level logins will probably be used to access the database.

This scripts finds all orphaned users in all databases in the instance and then drops those users accounts from the respective database. Note, use with caution and perform a database backup before testing.

```sql
CREATE TABLE ##ORPHANUSER
(
RowID int not null primary key identity(1,1) ,
DBNAME VARCHAR(100),
USERNAME VARCHAR(100),
CREATEDATE VARCHAR(100),
USERTYPE VARCHAR(100)
)

USE Test_Database
INSERT INTO ##ORPHANUSER
SELECT DB_NAME() DBNAME, NAME,CREATEDATE,
(CASE
WHEN ISNTGROUP = 0 AND ISNTUSER = 0 THEN 'SQL LOGIN'
WHEN ISNTGROUP = 1 THEN 'NT GROUP'
WHEN ISNTGROUP = 0 AND ISNTUSER = 1 THEN 'NT LOGIN'
END) [LOGIN TYPE] FROM sys.sysusers
WHERE SID IS NOT NULL AND SID 0X0 AND ISLOGIN =1 AND
SID NOT IN (SELECT SID FROM sys.syslogins) AND sys.sysusers .name != 'dbo'

SELECT * FROM ##ORPHANUSER

Declare @SQL as varchar (200)
Declare @DDBName varchar (100)
Declare @Orphanname varchar (100)
Declare @DBSysSchema varchar (100)
Declare @From int
Declare @To int
Select @From = 0, @To = @@ROWCOUNT
from ##ORPHANUSER
--Print @From
--Print @To
While @From < @To
Begin
Set @From = @From + 1

Select @DDBName = DBNAME, @Orphanname = USERNAME from ##ORPHANUSER
Where RowID = @From

Set @DBSysSchema = '[' + @DDBName + ']' + '.[sys].[schemas]'
print @DBsysSchema
Print @DDBname
Print @Orphanname
set @SQL = 'If Exists (Select * from ' + @DBSysSchema
+ ' where name = ''' + @Orphanname + ''')
Begin
Use ' + @DDBName
+ ' Drop Schema [' + @Orphanname + ']
End'
print @SQL
Exec (@SQL)

Begin Try
Set @SQL = 'Use ' + @DDBName
+ ' Drop User [' + @Orphanname + ']'
Exec (@SQL)
End Try
Begin Catch
End Catch

End

DROP TABLE ##ORPHANUSER
```