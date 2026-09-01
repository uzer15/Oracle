Thanks for stopping by!  Continue reading to understand what we covered in our From Documents to Intelligence - 26ai Through the lens of APEX.  

This session was designed to showcase how users can build out the different stages common to document process pipelines.

![Home](</APEX/Images/AppHome.png> "Demo Application Home Page")

#### Database: Autonomous AI Data Lakehouse - 26ai
#### APEX Version: 26.1

[YouTube](www.youtube.com)
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
  <summary> Working with JSON - JSON_TABLE </summary>

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
  <summary> Working with JSON - JSON_TABLE </summary>

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

___
### Querying - SELECT AI

___
### Querying - Vectors


___
### Querying - Agents

___
### Previous Examples



___
### Code Innovate Program

More about the Code Innovate Program [Oracle Developers](https://www.oracle.com/developer/community/code-innovate-developers/)  
Code Innovate videos on [YouTube](https://www.youtube.com/playlist?list=PLPIzp-E1msrZMCfSHbKgLK3KWsNM9JB9a)  

If you're interested in learning more about the program, email us at **codeinnovate_us_grp@oracle.com** and one of our engineers will get in touch with you.
