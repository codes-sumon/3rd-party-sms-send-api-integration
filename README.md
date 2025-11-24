# 3rd party sms send API integration

Calling external (HTTP/HTTPS) APIs from Oracle PL/SQL requires two main configurations:<br>
**ACL (Access Control List)** – Security to allow OUTBOUND network access.<br>
**Wallet** – Only needed for HTTPS (SSL/TLS URL).<br>


At first, check the existing ACL
```bash
SELECT * FROM dba_network_acls;
```
Then check ACL privileges
```bash
SELECT * FROM dba_network_acl_privileges;
```

If ACL and ACL privileges do not exist, then create your own as you require.<br>
At first connect your database with sysdba privileges, then execute bellow code.<br>
Here I am creating for the host smsplus.sslwireless.com<br>
**Note:** Check and change parameter names as you require.
```bash
BEGIN
 SYS.dbms_network_acl_admin.create_acl(
   acl => 'sslwireless_api_access.xml', --ACL Name
   description => 'sslwireless_api_access',
   principal => 'HRMS', --Schema Name
   is_grant => TRUE,
   privilege => 'connect',
   start_date => null,
   end_date => null
 );
SYS.DBMS_NETWORK_ACL_ADMIN.ADD_PRIVILEGE(
   acl => 'sslwireless_api_access.xml',
   principal => 'HRMS', --Schema Name
   is_grant => true,
   privilege => 'connect'
 );
 SYS.DBMS_NETWORK_ACL_ADMIN.ADD_PRIVILEGE(
   acl => 'sslwireless_api_access.xml',
   principal => 'HRMS', --Schema Name
   is_grant => true,
   privilege => 'resolve'
 );
 SYS.dbms_network_acl_admin.assign_acl(
   acl => 'sslwireless_api_access.xml', --ACL Name
   host => 'smsplus.sslwireless.com', --Host Name
   lower_port => null, 
   upper_port => null
 );
END;

BEGIN
    SYS.DBMS_NETWORK_ACL_ADMIN.ADD_PRIVILEGE(
   acl => 'sslwireless_api_access.xml',
   principal => 'APEX_240200', --Schema Name for APEX(Version wise)
   is_grant => true,
   privilege => 'connect'
 );
 SYS.DBMS_NETWORK_ACL_ADMIN.ADD_PRIVILEGE(
   acl => 'sslwireless_api_access.xml',
   principal => 'APEX_240200', --Schema Name for APEX(Version wise)
   is_grant => true,
   privilege => 'resolve'
 );
END;
```
Now Create Wallet. It's only needed for HTTPS (SSL/TLS URL).<br>
Download the root certificate and the intermediate certificate from your host website.<br>

here wallet path is **D:\OracleDB\OracleWallet**<br>
The root certificate and the intermediate certificate path is **D:\root.crt** and **D:\int.crt**
Run the command below in PowerShell
```bash
orapki wallet create -wallet "D:\OracleDB\OracleWallet" -pwd Apex#123 -auto_login
orapki wallet add -wallet "D:\OracleDB\OracleWallet" -trusted_cert -cert "D:\root.crt" -pwd Apex#123
orapki wallet add -wallet "D:\OracleDB\OracleWallet" -trusted_cert -cert "D:\int.crt" -pwd Apex#123
```
Check the wallet
```bash
orapki wallet display -wallet "D:\OracleDB\OracleWallet"

