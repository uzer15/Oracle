Thanks for stopping by!  Continue reading to understand what we covered in our From Documents to Intelligence - 26ai Through the lens of APEX.  

This session was designed to showcase how users can build out the different stages common to document process pipelines.

[YouTube](https://docs.oracle.com/en/database/oracle/apex/24.2/htmdb/workflow-substitution-strings.html#GUID-110A2DE8-0586-45E3-A439-D3D56425FE10)

#### Database: Autonomous AI Data Lakehouse - 26ai
#### APEX Version: 26.1
___
#### Working With Files
<details>
 <summary> Uploading Local Files </summary>

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

___
#### Extraction

___
#### Working With JSON

___
#### Processing & Validation

___
#### Querying - SELECT AI

___
#### Querying - Vectors


___
#### Querying - Agents

___
#### Previous Examples
___
### Scheduling a Database Job

<details>
 
<summary> Step-by-step instructions on how to create a database job </summary>
 
To anyone that stumbled across this section of the repo, apologies for not covering this during the demo.  That was part of the plan, but was running short on time.  I've outlined the steps necessary to setup a re-occurring job that will run your PL/SQL and initiate APEX workflows.

I'll be using the Scheduling service within Database Actions of an Autonomous Data Warehouse 23ai.

Start by logging into the database within OCI.  Open up Database Actions and select Scheduling

![Database Actions](</APEX/Images/DA - Scheduling.png> "Database Actions")

Select Create Job

![Database Actions](</APEX/Images/DA - Scheduling-Home.png> "Database Actions")

This will open the job details page

![Database Actions](</APEX/Images/Create Job.png> "Database Actions")

Give the job a name and paste your PL/SQL script

![Database Actions](</APEX/Images/Enter Details.png> "Database Actions")

Select Execution Mode

![Database Actions](</APEX/Images/Select Execution Mode.png> "Database Actions")

Configure Schedule

![Database Actions](</APEX/Images/Set up Mode.png> "Database Actions")

At this point, you should be able to save the job and you use the same area to monitor the job to ensure it's running as expected.

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

>**Note:** *We utilized the START_WORKFLOW function*

#### APEX In-Email Approvals
 - Blog posting describing alternative method for configuring in-email approvals. [Blog](https://blogs.oracle.com/apex/post/accelerate-decisionmaking-with-inemail-approvals-in-oracle-apex-workflows?source=:so:fb:or:awr:odv:::&SC=:so:fb:or:awr:odv:::&pcode=)

#### Scheduling
 - To learn more about scheduling database jobs. [Documention](https://docs.oracle.com/en/database/oracle/sql-developer-web/sdwad/scheduling-page.html)
___
### Code Innovate Program

More about the Code Innovate Program [Oracle Developers](https://www.oracle.com/developer/community/code-innovate-developers/)  
Code Innovate videos on [YouTube](https://www.youtube.com/playlist?list=PLPIzp-E1msrZMCfSHbKgLK3KWsNM9JB9a)  

If you're interested in learning more about the program, email us at **codeinnovate_us_grp@oracle.com** and one of our engineers will get in touch with you.
