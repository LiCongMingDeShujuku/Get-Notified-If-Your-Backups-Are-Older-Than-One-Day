![CLEVER DATA GIT REPO](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/0-clever-data-github.png "李聪明的数据库")

# 关于如何做到当你的备份超过一天，你会收到通知。
#### Get Notified If Your Backups Are Older Than One Day
**发布-日期: 2015年09月09日 (评论)**


## Contents

- [中文](#中文)
- [English](#English)
- [SQL Logic](#Logic)
- [Build Info](#Build-Info)
- [Author](#Author)
- [License](#License) 


## 中文
这里有一些sql逻辑，你可以放入一个作业并在每晚运行它。基本上它会检查哪些数据库没有最近的备份。在这种情况下，它是任何没有最后一天的完整数据库备份的数据库。它会创建一个这些数据库的列表，然后发送一封电子邮件通知你。

这是电子邮件的样子：

以下是此逻辑将执行的步骤列表。

1. 检查是否已配置SQL数据库邮件。
2. 完全配置SQL数据库邮件。
3. 发送测试电子邮件，以便知道它是否正常工作。 （测试电子邮件包括服务器/实例的名称，以便知道电子邮件的来源）
4. 获取缺少最近完整数据库备份的所有数据库的列表。
5. 查找最近30天的备份及其位置，以便了解以前备份的时间和地点。
6. 发送一个格式化电子邮件，显示上次备份的时间长度，并提取最近30天备份的列表以供参考。

此作业将忽略镜像配置的数据库。将检查主体是否有当前备份，但如果将此作业部署到镜像服务器，则只需忽略镜像伙伴数据库，因为它们不需要备份。如果他们切换到主题就会被检查。



## English
Here’s some sql logic you can put in a Job and run it nightly. Basically it will check to see which databases haven’t had a recent backup. In this case it’s any database that doesn’t have a full database backup for the last day. It creates a list of those databases, and then shoots off an email to notify you.

Here’s what the email will look like:


![#](images/Get-Notified-If-Your-Backups-Are-Older-Than-One-Day-01.png?raw=true "#")

Here’s a list of steps that this logic will carry out.

1. Check to see if SQL Database Mail has been configured.
2. Completely configures SQL Database Mail for you.
3. Sends off a test email so you know if it’s working or not. (Test email includes name of server/instance so you know where the email came from)
4. Gets a list of all databases that are missing a recent full database backup.
5. Finds the last 30 days backups, and their locations so you’ll know when and where the former backups were going.
6. Sends off a nifty formatted email showing you how old the last backup is, and pulls a list of the last 30 days backups for reference.


This Job will ignore Mirror configured databases. Principals will be checked for current backup, but if this Job was deployed to a Mirrored server the Mirror partner databases will simply be ignored as they don’t require backups. If they were switched to the Principal they of course would be checked.

---
## Logic
```SQL
use msdb;
set nocount on
set ansi_nulls on
set quoted_identifier on
 
----------------------------------------------------------------------
-- Configure SQL Database Mail if it's not already configured.
--配置SQL数据库邮件（如果尚未配置）。
if (select top 1 name from msdb..sysmail_profile) is null
    begin
        ----------------------------------------------------------------------
        -- Enable SQL Database Mail
--启用SQL数据库邮件

        exec master..sp_configure 'show advanced options',1
        reconfigure;
        exec master..sp_configure 'database mail xps',1
        reconfigure;
 
        ----------------------------------------------------------------------
        -- Add a profile
--添加个人资料

        execute msdb.dbo.sysmail_add_profile_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @description        = 'SQLDatabaseMail';
 
        ----------------------------------------------------------------------
        -- Add the account names you want to appear in the email message.
--添加要在电子邮件中显示的帐户名称。
        execute msdb.dbo.sysmail_add_account_sp
            @account_name       = 'sqldatabasemail@your_domain.com'
        ,   @email_address      = 'sqldatabasemail@your_domain.com'
        ,   @mailserver_name    = 'your_smtp_server.your_domain.com'  
        --, @port           = ####  --optional
        --, @enable_ssl     = 1 --optional
        --, @username       ='MySQLDatabaseMailProfile' --optional
        --, @password       ='MyPassword' --optional
 
        -- Adding the account to the profile     
--将帐户添加到配置文件
        execute msdb.dbo.sysmail_add_profileaccount_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @account_name       = 'sqldatabasemail@your_domain.com'
        ,   @sequence_number    = 1;
 
        -- Give access to new database mail profile 
--授予对新数据库邮件配置文件的访
 (DatabaseMailUserRole)
        execute msdb.dbo.sysmail_add_principalprofile_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @principal_id       = 0
        ,   @is_default     = 1;
 
        ----------------------------------------------------------------------
        -- Get Server info for test message
--获取测试消息的服务器信息
        declare @get_basic_server_name              varchar(255)
        declare @get_basic_server_name_and_instance_name        varchar(255)
        declare @basic_test_subject_message         varchar(255)
        declare @basic_test_body_message                varchar(max)
        set @get_basic_server_name          = (select cast(serverproperty('servername') as varchar(255)))
        set @get_basic_server_name_and_instance_name    = (select  replace(cast(serverproperty('servername') as varchar(255)), '\', '   SQL Instance: '))
        set @basic_test_subject_message     = 'Test SMTP email from SQL Server: ' + @get_basic_server_name_and_instance_name
        set @basic_test_body_message            = 'This is a test SMTP email from SQL Server:  ' + @get_basic_server_name_and_instance_name + char(10) + char(10) + 'If you see this.  It''s working perfectly :)'
 
        ----------------------------------------------------------------------
        -- Send quick email to confirm email is properly working.

--发送快速电子邮件以确认电子邮件正常运行
 
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name   = 'SQLDatabaseMailProfile'
        ,   @recipients = 'SQLJobAlerts@your_domain.com'
        ,   @subject        = @basic_test_subject_message
        ,   @body       = @basic_test_body_message;
 
        -- Confirm message send

--确认邮件发送
        -- select * from msdb..sysmail_allitems
    end
 
----------------------------------------------------------------------------------------
-- get basic server info.
--获取基本服务器信息
declare @server_name_basic      varchar(255)
declare @server_name_instance_name  varchar(255)
declare @server_time_zone           varchar(255)
set @server_name_basic      = (select cast(serverproperty('servername') as varchar(255)))
set @server_name_instance_name  = (select  replace(cast(serverproperty('servername') as varchar(255)), '\', '   SQL Instance: '))
 
----------------------------------------------------------------------------------------
-- set message subject.
--设置消息主题。
declare @message_subject        varchar(255)
set @message_subject        = 'A current database backup is missing for:  ' + @server_name_instance_name
 
----------------------------------------------------------------------------------------
-- create table to hold database list without a recent full database backup
--创建表以保存数据库列表而不进行最近的完整数据库备份
set nocount on
if object_id('tempdb..#missing_current_backup') is not null
    drop table #missing_current_backup
 
create table #missing_current_backup
(
    [server]        varchar(255)
,   [database]  varchar(255)
,   [time_of_backup]    varchar(255)
,   [last_backup]   varchar(255)
)
 
----------------------------------------------------------------------------------------
-- populate table with databases that have no recent full database backup
--使用没有近期完整数据库备份的数据库填充表
insert into #missing_current_backup ([server], [database],[time_of_backup],[last_backup])
select
    'server'        = upper(@@servername)
,   'database'  = upper(bs.database_name)
,   'time_of_backup'    = replace(replace(replace(left(max(bs.backup_finish_date), 19),':', '-'), 'AM', 'am'), 'PM', 'pm') + ' ' + datename(dw, max(bs.backup_finish_date))
,   'last_backup'   = cast (datediff(day, max(bs.backup_finish_date), getdate()) as varchar(10)) + ' Days Old'
from
    msdb.dbo.backupset bs join master.sys.database_mirroring sdm on bs.database_name = db_name(sdm.database_id)
where
    type = 'd'
    and bs.database_name in (select name from sys.databases)
    and sdm.mirroring_role_desc is null
    or sdm.mirroring_role_desc != 'mirror'
group by
    bs.database_name
having
    (max(bs.backup_finish_date) < dateadd(hour, -24, getdate()))
 
----------------------------------------------------------------------------------------
-- create table to hold list of the last known backups and their locations
--创建表以保存上次已知备份及其位置的列表
if object_id('tempdb..#get_last_known_backups') is not null
    drop table #get_last_known_backups
 
create table #get_last_known_backups
(
    [database]  varchar(255)
,   [time_of_backup]    varchar(255)
,   [location]      varchar(255)
,   [backup_file]   varchar(255)
)
 
----------------------------------------------------------------------------------------
-- populate table with a list of the last known backups and their locations
--使用最近已知备份及其位置的列表填充表
insert into #get_last_known_backups ([database], [time_of_backup], [location], [backup_file])
select
    'database'  = upper(bs.database_name)
,   'time_of_backup'    = replace(replace(replace(left(max(bs.backup_finish_date), 19),':', '-'), 'AM', 'am'), 'PM', 'pm') + ' ' + datename(dw, max(bs.backup_finish_date))
,   'location'      = reverse(right(reverse(upper(bmf.physical_device_name)), len(bmf.physical_device_name) - charindex('\',reverse(bmf.physical_device_name),1) + 1))
,   'backup_file'   = right(bmf.physical_device_name, charindex('\', reverse('\' + bmf.physical_device_name)) - 1)
from
    msdb.dbo.backupset bs join msdb.dbo.backupmediafamily bmf on bs.media_set_id = bmf.media_set_id
where
    bs.database_name in (select [database] from #missing_current_backup)
    and bs.backup_finish_date > (select getdate()- 30)
group by
    bs.database_name, bs.backup_finish_date, bmf.physical_device_name
order by
    bs.database_name, bs.backup_finish_date desc
 
----------------------------------------------------------------------------------------
-- create conditions for html tables in top and mid sections of email.
--在电子邮件的顶部和中间部分为html表创建条件。
 
declare @xml_top            NVARCHAR(MAX)
declare @xml_mid            NVARCHAR(MAX)
declare @body_top           NVARCHAR(MAX)
declare @body_mid           NVARCHAR(MAX)
 
----------------------------------------------------------------------------------------
-- set xml top table td's
-- create html table object for: #agent_job_step_error_report
--设置xml上表td的
--为#agent_job_step_error_report创建html表对象
set @xml_top = 
    cast(
        (select
            [server]        as 'td'
        ,   ''
        ,   [database]  as 'td'
        ,   ''
        ,   [time_of_backup]    as 'td'
        ,   ''
        ,   [last_backup]   as 'td'
        ,   ''
        from  #missing_current_backup 
        order by [database] asc
        for xml path('tr')
    ,   elements)
    as NVARCHAR(MAX)
        )
 
----------------------------------------------------------------------------------------
-- set xml mid table td's
-- create html table object for: #agent_job_information
--设置xml 中表 td's
--为# agent_job_information创建html表对象
set @xml_mid = 
    cast(
        (select
            [database]  as 'td'
        ,   ''
        ,   [time_of_backup]    as 'td'
        ,   ''
        ,   [location]      as 'td'
        ,   ''
        ,   [backup_file]   as 'td'
 
        from  #get_last_known_backups 
        order by [database], [time_of_backup] desc 
        for xml path('tr')
    ,   elements)
    as NVARCHAR(MAX)
        )
 
----------------------------------------------------------------------------------------
-- format email
--格式化电子邮件
set @body_top =
        '<html>
        <head>
            <style>
                    h1{
                        font-family: sans-serif;
                        font-size: 110%;
                    }
                    h3{
                        font-family: sans-serif;
                        color: black;
                    }
 
                    table, td, tr, th {
                        font-family: sans-serif;
                        border: 1px solid black;
                        border-collapse: collapse;
                    }
                    th {
                        text-align: left;
                        background-color: gray;
                        color: white;
                        padding: 5px;
                    }
 
                    td {
                        padding: 5px;
                    }
            </style>
        </head>
        <body>
        <H3>' + @message_subject + '</H3>
        <h1>Note: Last known backup info</h1>
        <table border = 1>
        <tr>
            <th> Server Name  </th>
            <th> Database </th>
            <th> Time Of Backup   </th>
            <th> Last Backup  </th>
        </tr>'
         
set @body_top = @body_top + @xml_top + '</table>
 
<h1>Last 30 days backups and their locations</h1>
最近30天备份及其位置

 
<table border = 1>
        <tr>
            <th> Database </th>
            <th> Time Of Backup   </th>
            <th> Location </th>
            <th> Backup File  </th>
        </tr>'        
         
+ @xml_mid + '</table>
        <h1>Go to the server by pasting in the following text under: Start-Run, or (Win + R)</h1>
        <h1>mstsc -v:' + @server_name_basic + '</h1>'
+ '</body></html>'
 
----------------------------------------------------------------------------------------
-- send email.
--发送邮件
 
 
if exists(select top 1 server from #missing_current_backup)
    begin
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @recipients         = 'SQLJobAlerts@your_domain.com'
        ,   @subject            = @message_subject
        ,   @body               = @body_top
        ,   @body_format        = 'HTML';
 
    end
 
drop table #missing_current_backup
drop table #get_last_known_backups


```



[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

- **李聪明的数据库 Lee's Clever Data**
- **Mike的数据库宝典 Mikes Database Collection**
- **李聪明的数据库** "Lee Songming"

[![Gist](https://img.shields.io/badge/Gist-李聪明的数据库-<COLOR>.svg)](https://gist.github.com/congmingshuju)
[![Twitter](https://img.shields.io/badge/Twitter-mike的数据库宝典-<COLOR>.svg)](https://twitter.com/mikesdatawork?lang=en)
[![Wordpress](https://img.shields.io/badge/Wordpress-mike的数据库宝典-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

---
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Lee Songming](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/1-clever-data-github.png "李聪明的数据库")

