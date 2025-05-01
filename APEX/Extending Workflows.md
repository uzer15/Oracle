Thanks for stopping by!  Continue reading to understand what we covered in our Extending APEX Workflows.  

This session was designed to showcase how users can interact with APEX outside of an application.


#### Database: Autonomous Data Warehouse (ADW) 23ai
#### APEX Version: 24.2
___
#### Code Snippets
<details>
 <summary> PL/SQL used to initiate workflow from outside of application </summary>

        SET SERVEROUTPUT ON;
        
        DECLARE
            
            --API Endpoint and Criteria
            api_Domain_URL      varchar2(500)  := '{Domain_URL}';
            api_Endpoint        varchar2(500)  := '{API_Endpoint}';
            group_ocid          varchar2(500)  := 'ocid1.group.oc1..';
            
            --Sets database Credentials File
            cred_name           varchar2(4000) := '{enter credential name}';
        
            --Capture API Response
            resp_group          dbms_cloud_types.RESP;
            l_clob_group        CLOB;
            
            --To store User's Email Address
            l_username          varchar2(150);
            
        BEGIN
        --Make REST call to get members of a (domain) group
            resp_group := dbms_cloud.send_request(
                credential_name => cred_name,
                uri => api_Domain_URL || api_Endpoint || group_ocid || '?attributes=members',
                method => dbms_cloud.METHOD_GET
            );
        
        --Set JSON response
            l_clob_group := dbms_cloud.get_response_text(resp_group);
            
        --Displays raw JSON response
            --dbms_output.put_line(l_clob_group); 
        
        --Determine which users are not in the Control Table (SYSTEM_ACCESS_REQUESTS)
            FOR i IN (
                SELECT g.name as User_ID
                FROM
                JSON_TABLE(l_clob_group,'$.members[*]'
                COLUMNS
                    (row_number FOR ORDINALITY,
                    name VARCHAR2(150) PATH '$.name')) 
                AS g
                WHERE g.name NOT IN (SELECT USER_NAME FROM WKSP_APRCC.SYSTEM_ACCESS_REQUESTS)
            ) LOOP
        
        --Set Current User
            SELECT i.User_ID INTO l_username FROM dual;
        
        --Display User Name 
            dbms_output.put_line(l_username);
        
        --Trigger APEX Workflow Start
                DECLARE
                   l_workflow_id    number;
                   l_app_id         number:= 500;
            
                BEGIN
                --Create Session
                    apex_session.create_session (
                        p_app_id   => l_app_id,
                        p_page_id  => 1,
                        p_username => '{Enter Authorized APEX User Name}' );
                        
                --print current App ID and Session ID
                    --sys.dbms_output.put_line ('App is '||v('APP_ID')||', session is '||v('APP_SESSION'));        
        
                    l_workflow_id := apex_workflow.start_workflow (
                        p_application_id => l_app_id,
                        p_static_id      => 'User_Access_Workflow',
                        p_detail_pk      => l_username,
                        p_parameters     => apex_workflow.t_workflow_parameters(
                            1 => apex_workflow.t_workflow_parameter(static_id => 'USER_NAME',   string_value => l_username)
                       ));
                       
                    --NOTE: APEX Workflow MUST be activated when using this API
        
                --Delete Session
                    apex_session.delete_session (
                        p_session_id   => v('APP_SESSION') );
                        
                END;
        
            END Loop;
        
        END;
        /
</details>

<details>
 <summary> Email Template </summary>

        <html>
            <head>
                <meta charset="UTF-8">
                <title>Task Assignment Notification</title>
            </head>
            <body style="font-family: Arial, sans-serif; background-color: #f4f4f4; padding: 20px; margin: 0;">
        
                <table align="center" width="600" style="background-color: #ffffff; padding: 20px; border-radius: 8px; box-shadow: 0px 0px 10px rgba(0, 0, 0, 0.1);">
                    <tr>
                        <td align="center" style="padding-bottom: 20px;">
                            <h2 style="color: #333;"> New Task Assigned to You!</h2>
                        </td>
                    </tr>
        
                    <tr>
                        <td style="color: #555; font-size: 16px; line-height: 1.6; padding: 10px 20px;">
                            <p>Hello <strong>#USERNAME#</strong>,</p>
                            <p>You have been assigned a new task: <strong>#SHORT_TASK_DESC#</strong>.</p>
                            <p><em>#TASK_DESC#</em></p>
                            <p>Please review the task and click one of the buttons below to approve or reject it.</p>
                        </td>
                    </tr>
        
                    <tr>
                        <td align="center" style="padding: 20px;">
                            <a href="#APPROVE_LINK#" 
                               style="background-color: #28a745; color: #ffffff; padding: 12px 24px; text-decoration: none; border-radius: 5px; font-size: 16px; display: inline-block;">
                               Approve Task
                            </a>
                            &nbsp;&nbsp;
                            <a href="#REJECT_LINK#" 
                               style="background-color: #dc3545; color: #ffffff; padding: 12px 24px; text-decoration: none; border-radius: 5px; font-size: 16px; display: inline-block;">
                               Reject Task
                            </a>
                        </td>
                    </tr>
        
                    <tr>
                        <td style="text-align: center; padding: 20px; font-size: 14px; color: #888;">
                            <p>If you have any questions, please contact your administrator.</p>
                            <p>Due Date: <strong>#DUE_DATE#</strong></p>
                        </td>
                    </tr>
        
                </table>
        
            </body>
            </html>
</details>

<details>
 <summary> Task Definition Action </summary>
        
        Declare l_task_id varchar(500);
        l_token VARCHAR2(500);
        l_raw_token RAW(32);
        l_hash raw(32);
        l_base_link VARCHAR2(500);
        l_approval_link VARCHAR2(500);
        l_reject_link VARCHAR2(500);
        l_placeholders clob;
        Begin
        
        --create 32 random bytes.Very long secret code string
        l_raw_token := DBMS_CRYPTO.RANDOMBYTES(32);
        
        --random bytes are converted into a hexadecimal string.express data using numbers and letters (0-9 and A-F)
        --string of characters
        l_token := RAWTOHEX(l_raw_token);
        
        --calculates a SHA‑256 hash of the random bytes
        --digital fingerprint
        l_hash := DBMS_CRYPTO.HASH(l_raw_token, DBMS_CRYPTO.HASH_SH256);
        
        --get unique task ID for the newly created task
        l_task_id := :APEX$TASK_ID;
        INSERT INTO
            EMAIL_TASK_APPROVALS (
                TASK_ID,
                USERNAME,
                TOKEN_HASH,
                EXPIRES_AT,
                USED_FLAG,
                WORKFLOW_ID
            )
        VALUES
            (
                l_task_id,
                'USER_DEV',
                l_hash,
                sysdate + 14,
                'N',
                TO_CHAR(:APEX$WORKFLOW_ID) --unique identifier of the workflow that initiated this task
            );
        
        l_base_link := 'https://{host}.adb.{region}.oraclecloudapps.com/ords/{Endpoint}';
        
        l_approval_link := l_base_link || '?task_id=' || l_task_id || '&token=' || l_token || '&outcome=APPROVED';
        
        l_reject_link := l_base_link || '?task_id=' || l_task_id || '&token=' || l_token || '&outcome=REJECTED';
        
        l_placeholders := '{ "USERNAME": "USER_DEV", "SHORT_TASK_DESC": "User Creation Request",
            "TASK_DESC": "Approval with automatically create a new user in the database","APPROVE_LINK": "#APPROVAL_LINK#",
            "REJECT_LINK": "#REJECT_LINK#","DUE_DATE": "#DUE_DATE#","MY_APPLICATION_LINK": "{APEX App Link}"}';
        
        l_placeholders := REPLACE(l_placeholders, '#APPROVAL_LINK#', l_approval_link);
        
        l_placeholders := REPLACE(l_placeholders, '#REJECT_LINK#', l_reject_link);
        
        l_placeholders := REPLACE(l_placeholders, '#DUE_DATE#', sysdate + 14);
        
        apex_mail.send (
            p_template_static_id = > 'USER_REQUEST_TEMPLATE',
            p_placeholders = > l_placeholders,
            p_to = > '{RECIPIENT}',
            p_from = > '{YOUR APPROVED EMAIL ADDRESS}'
        );
        
        --Send email out of queue
        APEX_MAIL.PUSH_QUEUE;
        
        End;
</details>

<details>
<summary> REST API Endpoint </summary>
        
        Declare
            l_task_id varchar2(50) := :task_id; --task_id passed via URL Parameter
            l_outcome varchar2(50) := :outcome; --outcome passed via URL Parameter
            l_token varchar2(4000) := :token;   --token passed via URL Parameter
        
            l_used_flag varchar2(10);           --token used?
            l_username varchar2(500);           -- current owner of task
            l_hash raw(32);                     --secure summary of the original token, stored in DB.
        
            --l_activity_params WWV_FLOW_GLOBAL.VC_MAP;   --(send parameters to workflow) data type to store key–value pairs (bundle multiple related values together in one )
            l_workflow_id varchar2(50);                 --ID stored in DB table
        
        Begin
        
        select USED_FLAG,WORKFLOW_ID, TOKEN_HASH into l_used_flag, l_workflow_id, l_hash  from EMAIL_TASK_APPROVALS where TASK_ID = l_task_id;
        select ACTUAL_OWNER into l_username from APEX_TASKS where TASK_ID = l_task_id;
        
        
        
        CASE WHEN l_used_flag = 'N' and l_hash = DBMS_CRYPTO.HASH(l_token, DBMS_CRYPTO.HASH_SH256)then    
            --process workflow. You need a valid APEX session to take action on tasks
                APEX_SESSION.CREATE_SESSION (
                    p_app_id =>500,
                    p_page_id =>1,
                    p_username =>l_username);
        
                CASE WHEN l_outcome = 'APPROVED' THEN
                        apex_human_task.APPROVE_TASK(       --approve task
                        p_task_id => l_task_id);
                        --l_activity_params('TASK_OUTCOME') := APEX_APPROVAL.C_TASK_OUTCOME_APPROVED; --set workflow parameter
                    WHEN l_outcome = 'REJECTED' THEN
                        apex_human_task.REJECT_TASK( --reject task
                        p_task_id => l_task_id);
                        --l_activity_params('TASK_OUTCOME') := APEX_APPROVAL.C_TASK_OUTCOME_REJECTED; --set workflow parameter
                    ELSE 
                        null;
                        --Do something
                END CASE;
        
                --l_user_agent := OWA_UTIL.get_cgi_env('HTTP_USER_AGENT');
        	    --l_ip := OWA_UTIL.get_cgi_env('REMOTE_ADDR');
        
                UPDATE EMAIL_TASK_APPROVALS
                    SET USED_FLAG = 'Y', 
                    USED_AT = systimestamp
                    WHERE task_id = l_task_id
                    AND token_hash = DBMS_CRYPTO.HASH(l_token, DBMS_CRYPTO.HASH_SH256);
        
                
                -- MIME type to html
            owa_util.mime_header('text/html', FALSE);
            htp.p('Cache-Control: no-cache');
            owa_util.http_header_close;
            htp.p('<!DOCTYPE html>');
                htp.p('<html lang="en">');
                htp.p('<head>');
                htp.p('  <meta charset="UTF-8">');
                htp.p('  <meta name="viewport" content="width=device-width, initial-scale=1.0">');
                htp.p('  <title>Task Approved</title>');
                htp.p('  <style>');
                htp.p('    body { font-family: "Helvetica Neue", Helvetica, Arial, sans-serif; background: #f4f4f4; margin: 0; padding: 0; }');
                htp.p('    .container { max-width: 600px; margin: 50px auto; background: #fff; padding: 30px; box-shadow: 0 0 10px rgba(0,0,0,0.1); border-radius: 8px; }');
                htp.p('    h1 { color: #333; }');
                htp.p('    p { color: #555; line-height: 1.6; }');
                htp.p('    a { color: #007bff; text-decoration: none; }');
                htp.p('    a:hover { text-decoration: underline; }');
                htp.p('    .message {padding: 16px;font-size: 18px;font-weight: bold;color: #155724;background-color: #d4edda;border: 1px solid #c3e6cb;border-radius: 8px;box-shadow: 0px 2px 4px rgba(0, 0, 0, 0.1);}');
                htp.p('    .messageRejected {padding: 16px;font-size: 18px;font-weight: bold;color: #721c24;background-color: #f8d7da;border: 1px solid #f5c6cb;border-radius: 8px;box-shadow: 0px 2px 4px rgba(0, 0, 0, 0.1);}');
                htp.p('  </style>');
                htp.p('</head>');
                htp.p('<body>');
                htp.p('  <div class="container">');
                htp.p('    <h1>Thank you '|| l_username||'!</h1>');
                IF l_outcome = 'APPROVED' THEN
                        htp.p('    <div class="message"> &#10004; You have approved the task. We have captured your response. </div>');
                ELSIF l_outcome = 'REJECTED' THEN
                        htp.p('    <div class="messageRejected">&#10004; You have Rejected the task. We have captured your response.</div>');
                ELSE
                        htp.p('    <div class="messageRejected">Invalid task status provided.</div>');
                END IF;
                htp.p('    <p>You can now close this window or click the link below to return to the application.</p>');
                htp.p('    <p><a href="https://your-apex-url">Return to Application</a></p>');
                htp.p('  </div>');
                htp.p('  <script>');
                htp.p('//JavaScript Code Here');
                htp.p('//window.close();');
                htp.p('  </script>');
                htp.p('</body>');
            htp.p('</html>');
             
        WHEN l_used_flag = 'Y' or l_hash != DBMS_CRYPTO.HASH(l_token, DBMS_CRYPTO.HASH_SH256)  then
            owa_util.mime_header('text/html', FALSE);
            htp.p('Cache-Control: no-cache');
            owa_util.http_header_close;
            htp.p('<!DOCTYPE html>');
                htp.p('<html lang="en">');
                htp.p('<head>');
                htp.p('  <meta charset="UTF-8">');
                htp.p('  <meta name="viewport" content="width=device-width, initial-scale=1.0">');
                htp.p('  <title>One-Time Link Used</title>');
                htp.p('  <style>');
                htp.p('    body { font-family: "Helvetica Neue", Helvetica, Arial, sans-serif; background: #f4f4f4; margin: 0; padding: 0; }');
                htp.p('    .container { max-width: 600px; margin: 50px auto; background: #fff; padding: 30px; box-shadow: 0 0 10px rgba(0,0,0,0.1); border-radius: 8px; }');
                htp.p('    h1 { color: #333; }');
                htp.p('    .messageError { color: #d9534f; font-size: 18px; margin: 20px 0; }');
                htp.p('    p { color: #555; line-height: 1.6; }');
                htp.p('    a { color: #007bff; text-decoration: none; }');
                htp.p('    a:hover { text-decoration: underline; }');
                htp.p('    .message {padding: 16px;font-size: 18px;font-weight: bold;color: #155724;background-color: #d4edda;border: 1px solid #c3e6cb;border-radius: 8px;box-shadow: 0px 2px 4px rgba(0, 0, 0, 0.1);}');
                htp.p('    .messageRejected {padding: 16px;font-size: 18px;font-weight: bold;color: #721c24;background-color: #f8d7da;border: 1px solid #f5c6cb;border-radius: 8px;box-shadow: 0px 2px 4px rgba(0, 0, 0, 0.1);}');
                htp.p('  </style>');
                htp.p('</head>');
                htp.p('<body>');
                htp.p('  <div class="container">');
                htp.p('    <h1>Notice</h1>');
                htp.p('    <div class="messageError">');
                htp.p('      The one-time use token you provided has already been used or there was a system error.');
                htp.p('    </div>');
                htp.p('    <p>If you believe this is a mistake, please contact your system administrator.</p>');
                htp.p('    <p>You can close this window or <a href="https://your-apex-url">return to the application</a>.</p>');
                htp.p('  </div>');
                htp.p('</body>');
            htp.p('</html>');
        
        ELSE
            null;
            --Do Something else
        END CASE;
        
        End;
</details>

___
### Scheduling a Database Job

<details>
 
<summary> Step-by-step instructions on how to create a database job </summary>
 
To anyone that stumbled across this section of the repo, apologies for not covering this during the demo.  That was part of the plan, but was running short on time.  I've outlined the steps necessary to setup a re-occurring job that will run your PL/SQL and initiate APEX workflows.

I'll be using the Scheduling service within Database Actions of an Autonomous Data Warehouse 23ai.

Start by logging into the database within OCI.  Open up Database Actions and select 

![Database Actions](</Analytics Cloud (OAC)/Images/ImportDataset.png> "Database Actions")

</details>

___

### Mentioned Links
Below are some links to documentation that we mentioned during the webinar

#### Database Credentials
 - Required when working with OCI resources to authenticate the request. [Documentation](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/dbms-cloud-subprograms.html#GUID-742FC365-AA09-48A8-922C-1987795CF36A)

#### JSON Functions
 - Very useful when working with JSON files and responses. We utilized the JSON_TABLE function, but many other JSON functions are available. [Documentation](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/JSON_TABLE.html)

#### APEX Workflow Runtime Views
 - APEX provides views to help understand every aspect of the Workflow engine. [Documentation](https://docs.oracle.com/en/database/oracle/apex/24.2/htmdb/workflow-views.html#GUID-851AB064-5B41-432F-9CAF-00CF78D975E4)

Here are the example SQL statements for APEX views we utilized

        SELECT * FROM APEX_WORKFLOWS WHERE APPLICATION_ID = 500
        SELECT * FROM APEX_WORKFLOW_ACTIVITIES WHERE WORKFLOW_ID = 21648550992363173 ORDER BY START_TIME


#### APEX Workflow Substitution Strings
 - Substitution Strings are used to pass information about a workflow to an Oracle APEX page. [Documentation](https://docs.oracle.com/en/database/oracle/apex/24.2/htmdb/workflow-substitution-strings.html#GUID-110A2DE8-0586-45E3-A439-D3D56425FE10)

##### APEX Workflow APIs
 - Listing of available Workflow related APIs. [Documentation](https://docs.oracle.com/en/database/oracle/apex/23.2/aeapi/APEX_WORKFLOW.START_WORKFLOW-Function.html#GUID-EC513C91-8A56-46FD-A25D-16A9BF071804)

**Note:** *We utilized the START_WORKFLOW function*

#### APEX In-Email Approvals
 - Blog posting describing alternative method for configuring in-email approvals. [Blog](https://blogs.oracle.com/apex/post/accelerate-decisionmaking-with-inemail-approvals-in-oracle-apex-workflows?source=:so:fb:or:awr:odv:::&SC=:so:fb:or:awr:odv:::&pcode=)

#### Scheduling
 - To learn more about scheduling database jobs. [Documention](https://docs.oracle.com/en/database/oracle/sql-developer-web/sdwad/scheduling-page.html)
___
### Code Innovate Program

More about the Code Innovate Program [Oracle Developers](https://www.oracle.com/developer/community/code-innovate-developers/)  
Code Innovate videos on [YouTube](https://www.youtube.com/playlist?list=PLPIzp-E1msrZMCfSHbKgLK3KWsNM9JB9a)  

If you're interested in learning more about the program, email us at **codeinnovate_us_grp@oracle.com** and one of our engineers will get in touch with you.
