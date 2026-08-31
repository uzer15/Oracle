Thanks for stopping by!  Continue reading to understand what we covered in our From Documents to Intelligence - 26ai Through the lens of APEX.  

This session was designed to showcase how users can build out the different stages common to document process pipelines.

![Database Actions](</APEX/Images/AppHome.png> "Demo Application Home Page")

#### Database: Autonomous AI Data Lakehouse - 26ai
#### APEX Version: 26.1

[YouTube](https://docs.oracle.com/en/database/oracle/apex/24.2/htmdb/workflow-substitution-strings.html#GUID-110A2DE8-0586-45E3-A439-D3D56425FE10)
___
#### Working With Files
<details>
 <summary> Uploading Local Files </summary>

      DECLARE
        l_blob      BLOB;
        l_mimetype  VARCHAR2(100);
        l_filename  VARCHAR2(200);
      
      BEGIN
        SELECT BLOB_CONTENT,
               MIME_TYPE,
               FILENAME
          INTO l_blob,
               l_mimetype,
               l_filename
          FROM APEX_APPLICATION_TEMP_FILES  --Temporary system view to hold uploaded files into APEX
         WHERE NAME = :P57_UPLOAD_DOCUMENT;  --File Upload page item name
      
        INSERT INTO DOCUMENT_STAGING (
            DOCUMENT_FILE,
            DOCUMENT_MIME_TYPE,
            DOCUMENT_FILE_NAME
        )
        VALUES (
            l_blob,
            l_mimetype,
            l_filename
        )
        RETURNING RECORD_ID INTO :P57_DOCUMENT_ID; --Optional method for returning a column value from the row that was just inserted
      END;
</details>

<details>
  <summary> Object Storage </summary>
 
    SELECT * FROM DBMS_CLOUD.LIST_OBJECTS('<Credential Name>','<Object Storage URL>');
    
    SELECT * FROM DBMS_CLOUD.LIST_OBJECTS('OCI_Credentials','https://objectstorage.us-ashburn-1.oraclecloud.com/n/<NameSpace>/b/<Bucket Name>/o/') WHERE object_name like '%.pdf';

   ##### Requirements

    grant execute on DBMS_CLOUD to '<Schema Name>'

   ##### Optional Step.  Add File(s) to Table

    DECLARE
    
       l_object   blob := null;
       l_bucket varchar2(4000) := 'https://objectstorage.us-ashburn-1.oraclecloud.com/n/<NameSpace>/b/<Bucket Name>/o/';
       
    BEGIN
       for i in (select * from dbms_cloud.list_objects('OCI_Credentials', l_bucket) where object_name like '%.png') 
       
       loop
          
          l_object := dbms_cloud.get_object(credential_name => 'OCI_Credentials', object_uri => l_bucket || i.object_name);
    
          insert into object_storage_files ( object_storage_file ) values ( l_object );
    
       end loop;
    
    END;
    
</details>

<details>
  <summary> REST </summary>

##### OCI List Compartments REST endpoint: https://identity.us-ashburn-1.oraclecloud.com/20160918/compartments/
##### OCI List Buckets REST endpoint: https://objectstorage.us-ashburn-1.oraclecloud.com/n/namespace/b/ (compartmentID as URL query String)
   

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
