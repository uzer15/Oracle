Thanks for stopping by!  Continue reading to understand what we covered in our OCI AI Speech Live Transcription service integration with APEX.  

![Welcome](</APEX/Images/Speech_pptx.png> "Home Page")
___
### Recorded Session
[Oracle AI Speech Live Transcription - APEX Application Integration](https://www.youtube.com/watch?v=nZno-1-k2NY)  
___
### Download APEX Application
[Download](https://github.com/uzer15/Oracle/blob/main/APEX/Apps/OCI AI Speech App.sql)
___

#### Database: Autonomous AI Database 26ai (Lakehouse) 
#### APEX Version: 26.1.5
#### Recorded on: 10/14/26
#### Developed by: Pete Midura
___
#### Code Snippets
<details>
 <summary> DDL for the Segments Table </summary>

    CREATE TABLE "SPEECH_TRANSCRIPT_SEGMENT" 
     (	"RECORDING_ID" VARCHAR2(64 CHAR) NOT NULL ENABLE, 
    "SEGMENT_NO" NUMBER NOT NULL ENABLE, 
    "TRANSCRIPT_TEXT" VARCHAR2(4000 CHAR) NOT NULL ENABLE, 
    "RECOGNIZED_AT_CLIENT" VARCHAR2(40 CHAR) NOT NULL ENABLE, 
    "RECEIVED_AT" TIMESTAMP (6) WITH TIME ZONE DEFAULT systimestamp NOT NULL ENABLE, 
    "CREATED_BY" VARCHAR2(255 CHAR) NOT NULL ENABLE, 
     CONSTRAINT "SPEECH_TRANSCRIPT_SEGMENT_PK" PRIMARY KEY ("RECORDING_ID", "SEGMENT_NO")
    USING INDEX  ENABLE
     ) ;
     
</details>

<details>
 <summary> DDL for Database View to rollup segments into a single transcript </summary>

    CREATE OR REPLACE FORCE EDITIONABLE VIEW "SPEECH_RECORDING_V" ("RECORDING_ID", "CREATED_BY", "RECORDING_START_TIME", "RECORDING_END_TIME", "RECORDING_ELAPSED_TIME", "COMPLETE_TRANSCRIPT") AS 
    with recording_data as (
      select
          recording_id,
          max(created_by) as created_by,
          min(received_at) as recording_start_time,
          max(received_at) as recording_end_time,
          max(received_at) - min(received_at) as elapsed_interval,
  
          rtrim(
              xmlagg(
                  xmlelement(
                      e,
                      transcript_text || ' '
                  )
                  order by segment_no
              ).extract('//text()').getclobval(),
              ' '
          ) as complete_transcript
  
      from speech_transcript_segment
      group by recording_id
    )
    select
        recording_id,
        created_by,
        recording_start_time,
        recording_end_time,
  
      lpad(
          to_char(
              extract(day from elapsed_interval) * 24 +
              extract(hour from elapsed_interval),
              'FM9990'
          ),
          2,
          '0'
      )
      || ':' ||
      lpad(
          to_char(extract(minute from elapsed_interval), 'FM00'),
          2,
          '0'
      )
      || ':' ||
      lpad(
          to_char(floor(extract(second from elapsed_interval)), 'FM00'),
          2,
          '0'
      ) as recording_elapsed_time,
  
      complete_transcript
    from recording_data;
     
</details>
    
    
    
    
    
    
    
    
    





