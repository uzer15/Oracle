Thanks for stopping by!  Continue reading to understand what we covered in the: From Documents to Intelligence - 26ai Through the lens of APEX.  

This session was designed to showcase how users can build out the different stages common to document process pipelines.

![Home](</APEX/Images/AppHome.png> "Demo Application Home Page")

#### Database: Autonomous AI Data Lakehouse - 26ai
#### APEX Version: 26.1
#### Recorded on: 08/27/26
#### Presented by: Pete Midura - Data & AI Architect

___
### Working With Files
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
    
    SELECT * FROM DBMS_CLOUD.LIST_OBJECTS('OCI_Credentials','https://objectstorage.us-ashburn-1.oraclecloud.com/n/<Namespace>/b/<Bucket Name>/o/') WHERE object_name like '%.pdf';

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

##### [OCI List Compartments API](https://docs.oracle.com/en-us/iaas/api/#/en/identity/20160918/Compartment/ListCompartments)
##### [OCI List Buckets API](https://docs.oracle.com/en-us/iaas/api/#/en/objectstorage/20160918/Bucket/ListBuckets)
##### [OCI List Objects API](https://docs.oracle.com/en-us/iaas/api/#/en/objectstorage/20160918/Object/ListObjects)

</details>

___
### Extraction

<details>
  <summary> Extract Text from Text based documents - Database Feature </summary>
    
    DECLARE
      l_document_id   NUMBER;
      l_document_txt  CLOB;
    
    BEGIN
      l_document_id := :P58_DOCUMENT_ID;
    
      UPDATE DOCUMENT_STAGING
         SET DOCUMENT_TEXT = DBMS_VECTOR_CHAIN.UTL_TO_TEXT(
             DOCUMENT_FILE,
             JSON('{
               "plaintext": "true",
               "charset"  : "UTF8"
             }')
           )
       WHERE RECORD_ID = l_document_id;
    
      SELECT DOCUMENT_TEXT
        INTO :P58_EXTRACTED_TEXT
        FROM DOCUMENT_STAGING
       WHERE RECORD_ID = l_document_id;
    END;

##### [DBMS_VECTOR_CHAIN Package](https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/dbms_vector_chain-vecse.html)

</details>

<details>
  <summary> Document Understanding </summary>

##### Example Request Body template

    {
     
      "features": [
        {
          "featureType": "#FEATURETYPE#"
        }
      ],
    
    "document": {
      "source": "OBJECT_STORAGE",
      "namespaceName": "<Namespace>",
      "bucketName": "#BUCKET#",
      "objectName": "#FILENAME#"
    },
    "compartmentId": "#COMPARTMENT#"
    }

Note:  Any element surrounded by # indicates it's an Operational Parameter that will be sent in when the request is made.


##### [Document Understanding API](https://docs.oracle.com/en-us/iaas/api/#/en/document-understanding/20221109/datatypes/AnalyzeDocumentDetails)

</details>


___
### Working With JSON

<details>
  <summary> JSON_TABLE </summary>

    SELECT
      inv.INVOICE_NUM,
      inv.INVOICE_DATE,
      inv.INVOICE_AMOUNT,
      inv.INVOICED_BY,
      li.sku,
      li.description,
      li.amount,
      li.quantity
    
    FROM INVOICE_JSON_COLLECTIONS inv,
         JSON_TABLE(
           inv.INVOICE_LINE_ITEMS,
           '$[*]'
           COLUMNS (
             sku         VARCHAR2(30)  PATH '$.itemSkuNumber',
             description VARCHAR2(100) PATH '$.itemDescription',
             amount      NUMBER       PATH '$.itemAmount',
             quantity    NUMBER(10,2) PATH '$.itemQuantity'
           )
         ) li;
##### [JSON_TABLE Documentation](https://docs.oracle.com/en/database/oracle/oracle-database/19/adjsn/function-JSON_TABLE.html#GUID-0172660F-CE29-4765-BF2C-C405BDE8369A)

</details>


<details>
  <summary> JSON_VALUE </summary>

    SELECT 
        JSON_VALUE(:P59_JSON_RESPONSE_RAW, '$.modelId'),
        JSON_VALUE(:P59_JSON_RESPONSE_RAW, '$.modelVersion'),
        JSON_VALUE(:P59_JSON_RESPONSE_RAW, '$.chatResponse.choices.message.content.text'),
        JSON_VALUE(:P59_JSON_RESPONSE_RAW, '$.chatResponse.usage.completionTokens'),
        JSON_VALUE(:P59_JSON_RESPONSE_RAW, '$.chatResponse.usage.promptTokens')
    INTO 
        :P59_MODEL_ID,
        :P59_MODEL_VERSION,
        :P59_RESPONSE_TEXT,
        :P59_RESPONSE_TOKENS,
        :P59_PROMPT_TOKENS;
##### [JSON_VALUE Documentation](https://docs.oracle.com/en/database/oracle/oracle-database/19/adjsn/function-JSON_VALUE.html)

</details>


___
### Processing & Validation

<details>
  <summary> Agent Development </summary>

##### During this portion of the webinar, we focused on database agent development and monitoring
 
##### [DBMS_CLOUD_AI_AGENT Package](https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/dbms-cloud-ai-agent-package.html#GUID-39C4A94B-C07A-4A76-8412-BEEA667C259B)

##### [Practical Examples](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/examples-using-select-ai-agent.html#GUID-DA69F925-A3F4-4990-A229-7A1D368C6001)

</details>

<details>
  <summary> Create a Tool </summary>
 
      BEGIN
        DBMS_CLOUD_AI_AGENT.CREATE_TOOL(
          tool_name   => 'SALES_SQL_TOOL',
          attributes  => '{
            "tool_type": "SQL",
            "tool_params": {
              "profile_name": "SALES_NL2SQL_PROFILE"
            }
          }',
          description => 'Answers sales-data questions using the approved NL2SQL profile'
        );
      END;

REMINDER: These are the same tools that would be exposed to a MCP server

</details>

<details>
  <summary> Create a Task </summary>
 
    BEGIN
      DBMS_CLOUD_AI_AGENT.CREATE_TASK(
        task_name  => 'ANALYZE_SALES_TASK',
        attributes => q'~{
          "instruction": "Answer the user's sales question: {query}. Use the sales SQL tool. Summarize the result clearly and identify any material trends.",
          "tools": ["SALES_SQL_TOOL"],
          "enable_human_tool": true
        }~',
        description => 'Analyzes sales data and explains the results'
      );
    END;
    /

</details>

<details>
  <summary> Create an Agent </summary>
 
    BEGIN
      DBMS_CLOUD_AI_AGENT.CREATE_AGENT(
        agent_name  => 'SALES_ANALYST_AGENT',
        attributes  => q'~{
          "profile_name": "SALES_AGENT_PROFILE",
          "role": "You are a sales analyst. Provide accurate, concise, business-friendly answers grounded in approved sales data.",
          "enable_human_tool": true,
          "short_term_memory_length": 10
        }~',
        description => 'Explains sales performance using approved database data'
      );
    END;
    /

</details>

<details>
  <summary> Create an Agent Team </summary>
 
    BEGIN
      DBMS_CLOUD_AI_AGENT.CREATE_TEAM(
        team_name  => 'SALES_ANALYTICS_TEAM',
        attributes => '{
          "agents": [
            {
              "name": "SALES_ANALYST_AGENT",
              "task": "ANALYZE_SALES_TASK"
            }
          ],
          "process": "sequential"
        }',
        description => 'Answers sales questions using the sales analyst agent'
      );
    END;
    /

</details>

<details>
  <summary> Create a Conversation </summary>

    SELECT DBMS_CLOUD_AI.CREATE_CONVERSATION(
                   attributes => '{"title":"Conversation 1",
                                   "description":"this is a description",
                                   "retention_days":5,
                                   "conversation_length":5}')
         AS conversation_id FROM dual;

</details>

<details>
  <summary> Run an Agent Team </summary>

 DECLARE

    l_conversation_id varchar2(200);
    l_final_answer clob;
  
  
    BEGIN
  
      if :P75_CONV_ID IS Null THEN
          l_conversation_id :=DBMS_CLOUD_AI.create_conversation();
          SELECT l_conversation_id INTO :P75_CONV_ID;
          else
          l_conversation_id := :P75_CONV_ID;
  
      END if;
  
      l_final_answer := DBMS_CLOUD_AI_AGENT.RUN_TEAM(
        team_name => :P75_SELECT_TEAM,
        user_prompt => :P75_PROMPT,
        params => '{"conversation_id": "' ||l_conversation_id || '"}'
      );
  
      :P75_PROMPT := null;
  
    END;

</details>

<details>
  <summary> Multi-Agent Patterns </summary>
 
##### [Documentation](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/multi-agent-pattern-ai-agents.html)

</details>

<details>
  <summary> Observability </summary>
 
##### DBMS_CLOUD_AI_AGENT views

    -- DBA views
    SELECT * FROM DBA_AI_AGENTS;  --The view display agents created by all users inside their database.
    SELECT * FROM DBA_AI_AGENT_ATTRIBUTES; --The view displays attributes of the agents created by all users inside their database. 
    SELECT * FROM DBA_AI_AGENT_TOOLS;  --The view displays details of the tools created by all users inside their database.
    SELECT * FROM DBA_AI_AGENT_TOOL_ATTRIBUTES;  --The view displays attributes of tools created by all users inside their database. 
    SELECT * FROM DBA_AI_AGENT_TASKS;  --The view display tasks created by all users inside their database.
    SELECT * FROM DBA_AI_AGENT_TASK_ATTRIBUTES;  --The view displays attributes of tasks created by all users inside their database.
    SELECT * FROM DBA_AI_AGENT_TEAMS;  --The view displays details of the teams created by all users inside their database.
    SELECT * FROM DBA_AI_AGENT_TEAM_ATTRIBUTES;  --The view displays attributes of teams created by all users inside their database. 
    
    --  USER views 
    SELECT * FROM USER_AI_AGENTS;  --The view displays details on conversations in your schema.
    SELECT * FROM USER_AI_AGENT_ATTRIBUTES;  --The view displays attributes of agents created by the current user inside their schema
    SELECT * FROM USER_AI_AGENT_TOOLS;  --The view displays details tools created by the current user inside their schema. 
    SELECT * FROM USER_AI_AGENT_TOOL_ATTRIBUTES;  --The view displays attributes of tools created by the current user inside their schema.
    SELECT * FROM USER_AI_AGENT_TASKS;  --The view displays details on tasks created by the current user inside their schema
    SELECT * FROM USER_AI_AGENT_TASK_ATTRIBUTES;  --The view displays attributes of tasks created by the current user inside their schema. 
    SELECT * FROM USER_AI_AGENT_TEAMS;  --The view displays details on teams created by the current user inside their schema.
    SELECT * FROM USER_AI_AGENT_TEAM_ATTRIBUTES;  --The view displays details of attributes of teams created by the current user inside their schema.
    
    /* DBMS_CLOUD_AI_AGENT history views */
    
    --  DBA views
    SELECT * FROM DBA_AI_AGENT_TEAM_HISTORY;  --The view displays all agent team-runs across the system.
    SELECT * FROM DBA_AI_AGENT_TASK_HISTORY;  --This view shows agent task parameters within a team.
    SELECT * FROM DBA_AI_AGENT_TOOL_HISTORY;  --This view lists calls to tools across the system.
    
    --  USER views
    SELECT * FROM USER_AI_AGENT_TEAM_HISTORY;  --The view displays all agent team runs for teams owned by the current user.
    SELECT * FROM USER_AI_AGENT_TASK_HISTORY;  --This view shows agent task parameters for the current user’s teams.
    SELECT * FROM USER_AI_AGENT_TOOL_HISTORY;  --This view lists calls to tools the current user owns.
    
    -- Conversation Views
    select * from user_cloud_ai_conversations ORDER BY CREATED ASC; --The view displays details on conversations in your schema.
    select * from user_cloud_ai_conversation_prompts ORDER BY CREATED ASC; --The view displays details on prompts used in conversations in your schema.

</details>

___
### Querying

<details>
  <summary> SELECT AI </summary>

##### Requirements
    grant execute on DBMS_CLOUD_AI to 'Schema Name'

![Home](</APEX/Images/SELECT AI Actions.png> "Demo Application Home Page")

##### Profile Example

    BEGIN
        DBMS_CLOUD_AI.CREATE_PROFILE(
            profile_name => 'SALES',
            attributes =>
                '{"provider": "oci",
                "credential_name": "My Credential", 
                "comments":true,
                "conversation":true,
                "region":"us-chicago-1",
                "oci_compartment_id":"ocid1.compartment.oc1..aaaaaaaac......",
                "model":"meta.llama-3.1-405b-instruct",
                "object_list": [{"owner": "SH", "name": "SALES"},
                    {"owner": "SH", "name": "TIMES"},
                    {"owner": "SH", "name": "CUSTOMERS"},
                    {"owner": "SH", "name": "PRODUCTS"},
                    {"owner": "SH", "name": "PROMOTIONS"},
                    {"owner": "SH", "name": "CHANNELS"}
                    ]
                }'
        );
    END;

##### [Documentation](https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/select-ai-manage-profiles.html)

##### Run Profile

    BEGIN
      SELECT DBMS_CLOUD_AI.GENERATE(:P72_SELECTAI_PROMPT, -- User Prompt
               profile_name => :P72_SELECTAI_PROFILES, -- SELECT AI Profile
               action       => :P72_SELECTAI_ACTION -- SELECT AI Action
             )
        INTO :P72_SELECTAI_RESPONSE --Insert Response into APEX page item
        FROM DUAL;
    END;

##### System Views

    SELECT * FROM USER_CLOUD_AI_PROFILES
    SELECT * FROM USER_CLOUD_AI_PROFILE_ATTRIBUTES 
    
    --Joining both views
    
    select a.profile_name, a.status, b.attribute_name, b.attribute_value 
    from user_CLOUD_AI_PROFILES a, USER_CLOUD_AI_PROFILE_ATTRIBUTES b
    where a.profile_id = b.profile_id;

</details>

<details>
  <summary> Vectors </summary>

 ##### Requirements

    grant execute on DBMS_CLOUD_AI to 'Schema Name';
    grant execute on DBMS_VECTOR to 'Schema Name';
    grant execute on DBMS_VECTOR_CHAIN to 'Schema Name';

##### DBMS_Vector
[Documentation](https://docs.oracle.com/en/database/oracle/oracle-database/26/arpls/dbms_vector1.html)

##### DBMS_VECTOR_CHAIN
[Documentation](https://docs.oracle.com/en/database/oracle/oracle-database/26/arpls/dbms_vector_chain1.html)

##### Create VECTOR Column

    ALTER TABLE <Table Name>
      ADD (
        <Column Name> VECTOR
      );

##### Create Vector Example

    select dbms_vector_chain.utl_to_embedding('Enter text to be embedded here!', 
     json('{
       "provider": "<Provider>",
       "credential_name": "<Credential Name>",
       "url": "https://inference.generativeai.us-chicago-1.oci.oraclecloud.com/20231130/actions/embedText",
       "model": "cohere.embed-v4.0",  --Embedding Model
       "batch_size":10 
     }')) embed_vector

##### Vector Distance Search

    SELECT
        <column1>,
        VECTOR_DISTANCE(
            description_vector,  --Vector Column or Variable
            Convert_prompt_to_Vector(:P73_VECTOR_PROMPT),  --User prompt which will be converted into vector for comparison using VECTOR_DISTANCE function
            COSINE
        ) AS distance
    FROM report_directory
    ORDER BY VECTOR_DISTANCE(
        description_vector,
        Convert_prompt_to_Vector(:P73_VECTOR_PROMPT),
        COSINE
    )
    FETCH FIRST 3 ROWS ONLY;

##### Example function to convert objects into Vectors.  

Note: There are several methods for converting text/files into Vectors, but I opted to create a function.

    create or replace FUNCTION Convert_prompt_to_Vector (p_prompt varchar2)
        RETURN VECTOR IS
    BEGIN
        RETURN dbms_vector_chain.utl_to_embedding(p_prompt, json('{
                      "provider": "<provider>",
                      "credential_name": "<Credentials>",
                      "url": "https://inference.generativeai.us-chicago-1.oci.oraclecloud.com/20231130/actions/embedText",
                      "model": "cohere.embed-v4.0",
                      "batch_size":50
                    }'));
    END;
    /

</details>

<details>
  <summary> Agents </summary>

 During this portion of the webinar, I wanted to show how to monitor Agent Team conversations that have a Human in the loop.

 ##### Create Agent example w/ Human in the Loop
  <summary> Create an Agent </summary>
 
    BEGIN
      DBMS_CLOUD_AI_AGENT.CREATE_AGENT(
        agent_name  => 'SALES_ANALYST_AGENT',
        attributes  => q'~{
          "profile_name": "SALES_AGENT_PROFILE",
          "role": "You are a sales analyst. Provide accurate, concise, business-friendly answers grounded in approved sales data.",
          "enable_human_tool": true,
          "short_term_memory_length": 10
        }~',
        description => 'Explains sales performance using approved database data'
      );
    END;

Use the following system views to monitor 'Open' conversations.

    USER_AI_AGENT_TEAM_HISTORY
    USER_AI_AGENT_TASK_HISTORY

##### Example of how this is displayed

 ![Home](</APEX/Images/SELECT AI Actions.png> "Demo Application Home Page")

</details>

___
### Code Innovate Program

More about the Code Innovate Program [Oracle Developers](https://www.oracle.com/developer/community/code-innovate-developers/)  
Code Innovate videos on [YouTube](https://www.youtube.com/playlist?list=PLPIzp-E1msrZMCfSHbKgLK3KWsNM9JB9a)  

If you're interested in learning more about the program, email us at **codeinnovate_us_grp@oracle.com** and one of our engineers will get in touch with you.
