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
```
Now check the API request
```bash
SET SERVEROUTPUT ON;
DECLARE
  lo_req  UTL_HTTP.req;
  lo_resp  UTL_HTTP.resp;
BEGIN
  UTL_HTTP.SET_WALLET (
       'file:D:\OracleDB\OracleWallet',
       'Apex#123');
    lo_req := UTL_HTTP.begin_request ('https://smsplus.sslwireless.com/api/v3/send-sms/dynamic');
    lo_resp := UTL_HTTP.get_response (lo_req);
    DBMS_OUTPUT.put_line (lo_resp.status_code);
    UTL_HTTP.end_response (lo_resp);
END;
```

Create a database procedure to send sms
```bash
CREATE PROCEDURE prc_send_sms(
        p_msisdn        IN VARCHAR2,
        p_message_body  IN VARCHAR2,
        p_result        OUT VARCHAR2
    ) AS
        l_api_token CONSTANT VARCHAR2(100) := 'your-api-token';
        l_sid        CONSTANT VARCHAR2(50)  := 'YOUR-SID';
        l_domain     CONSTANT VARCHAR2(200) := 'https://smsplus.sslwireless.com';
        l_url            VARCHAR2(500);
        l_request_body   CLOB;
        l_cleaned_msisdn VARCHAR2(20);
        l_csms_id        NUMBER := 10001;
        l_escaped_message VARCHAR2(4000);
    BEGIN
        -- Validate input
        IF p_msisdn IS NULL OR SUBSTR(p_msisdn, 1, 3) != '880' OR p_message_body IS NULL THEN
            p_result := '{"status":"ERROR","message":"Invalid or missing parameters"}';
            RETURN;
        END IF;

        l_cleaned_msisdn := p_msisdn;

        -- Build URL
        l_url := RTRIM(l_domain, '/') || '/api/v3/send-sms/dynamic';
        -- Build JSON Request Body
        l_escaped_message := p_message_body;
        -- Step 1: Escape backslashes FIRST
        l_escaped_message := REPLACE(l_escaped_message, '\', '\\');
        -- Step 2: Escape double quotes
        l_escaped_message := REPLACE(l_escaped_message, '"', '\"');
        -- Step 3: Handle different line break formats (Windows: CRLF, Unix/Mac: LF)
        -- Replace CRLF (CHR(13)||CHR(10)) first, then LF (CHR(10))
        l_escaped_message := REPLACE(l_escaped_message, CHR(13)||CHR(10), '\n'); -- Windows line breaks
        l_escaped_message := REPLACE(l_escaped_message, CHR(10), '\n');          -- Unix/Mac line breaks
        l_escaped_message := REPLACE(l_escaped_message, CHR(13), '\n');          -- Old Mac line breaks
        -- Step 4: Escape tabs (if any)
        l_escaped_message := REPLACE(l_escaped_message, CHR(9), '\t');

        -- Make request body
        l_request_body := '{"api_token":"' || l_api_token || '","sid":"' || l_sid || '","sms":[{"msisdn":"' || p_msisdn || '","text":"' || l_escaped_message || '","csms_id":' || l_csms_id || '}]}';
    
        -- Set Headers
        APEX_WEB_SERVICE.SET_REQUEST_HEADERS(
            p_name_01  => 'Content-Type',
            p_value_01 => 'application/json',
            p_name_02  => 'Accept',
            p_value_02 => 'application/json'
        );

        -- Make API Request
        p_result := APEX_WEB_SERVICE.MAKE_REST_REQUEST(
            p_url         => l_url,
            p_http_method => 'POST',
            p_body        => l_request_body,
            p_wallet_path => 'file:D:\OracleDB\OracleWallet',
            p_wallet_pwd  => 'Apex#123'
        );
    EXCEPTION
        WHEN OTHERS THEN
            p_result := '{"status":"ERROR","message":"' ||
                        REPLACE(SQLERRM, '"', '\"') || '"}';
            DBMS_OUTPUT.PUT_LINE('SEND_SMS ERROR => ' || SQLERRM);
    END prc_send_sms;
```

Execute procedure
```bash
SET SERVEROUTPUT ON;
DECLARE
  l_result varchar2(4000);
BEGIN
   prc_send_sms('8801764000000', 'Test sms', l_result);
   DBMS_OUTPUT.PUT_LINE(l_result);
END;
```
