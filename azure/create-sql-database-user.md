1. create logins on Master. 
   
   Password generator is handly. http://passwordsgenerator.net/

  ```sql
  CREATE LOGIN StagingUser WITH PASSWORD=N'password'
  CREATE LOGIN MigratorUser WITH PASSWORD=N'password'
  ```

2. create users for those logins on your db

  ```sql
  create user StagingUser from LOGIN StagingUser
  create user MigratorUser from LOGIN MigratorUser
  ```

3. add them to the appropriate roles
  ```sql
  EXEC sp_addrolemember N'db_datareader', N'StagingUser'
  EXEC sp_addrolemember N'db_datawriter', N'StagingUser'
  EXEC sp_addrolemember N'db_datareader', N'MigratorUser'
  EXEC sp_addrolemember N'db_datawriter', N'MigratorUser'
  EXEC sp_addrolemember N'db_ddladmin',   N'MigratorUser'
  ```

