Thanks for stopping by!  Continue reading to understand what we covered in our Extending APEX Workflows.  

This session was designed to showcase how users can interact with APEX outside of an application.


#### Database: Autonomous Data Warehouse (ADW) 23ai
#### APEX Version: 24.2
___
#### Code Snippets

### PL/SQL used to initiate workflow from outside of application

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

### PL/SQL used for....

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
 - Listing of available Workflow related APIs [Documentation](https://docs.oracle.com/en/database/oracle/apex/23.2/aeapi/APEX_WORKFLOW.START_WORKFLOW-Function.html#GUID-EC513C91-8A56-46FD-A25D-16A9BF071804)

**Note:** *We utilized the START_WORKFLOW function*

#### APEX In-Email Approvals
 - Blog posting describing alternative method for configuring in-email approvals [Blog](https://blogs.oracle.com/apex/post/accelerate-decisionmaking-with-inemail-approvals-in-oracle-apex-workflows?source=:so:fb:or:awr:odv:::&SC=:so:fb:or:awr:odv:::&pcode=)

___
### Code Innovate Program

More about the Code Innovate Program [Oracle Developers](https://www.oracle.com/developer/community/code-innovate-developers/)  
Code Innovate videos on [YouTube](https://www.youtube.com/watch?v=zW1uo1LhU7g)  

If you're interested in learning more about the program, email us at **codeinnovate_us_grp@oracle.com** and one of our engineers will get in touch with you.
