## CortexAISearchForAirQuality
Snowflake Cortex AI, SQL, Cortex Search, Tables, Air Quality, Apache NiFi



### DataFlow Pipeline


* ListenSlack
* EvaluateJsonPath
* RouteOnAttribute
* EvaluateJsonPath
* RouteOnAttribute:   ${inputs:toUpper():contains('AIRQ')}
* ExecuteSQLRecord: with JSON Writer and SnowflakeConnectonPool

````
 SELECT PARSE_JSON(
  SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
      'DEMO.DEMO.AIRQUALITY_SRVC',
      '{
        "query": "${inputs:trim()}",
        "columns":[
            "AIRQUALITY_TEXT",
            "REPORTINGAREA",
            "STATECODE",
            "DATEOBSERVED",
            "HOUROBSERVED",
            "PARAMETERNAME",
            "AQI"
        ],
        "limit":10
      }'
  )
)['results'] as results;
````
* SplitRecord
* EvaluateJsonPaH
* ExtractText
* ExecuteSQLRecord: with JSON Writer and SnowflakeConnectionPool

````
SELECT SNOWFLAKE.CORTEX.COMPLETE( 'llama2-70b-chat', 'You are an expert air quality assistant that extracts information from the CONTEXT provided between <context> and </context> tags.
When answering the question contained between <question> and </question> tags be concise, please provide a complete report and do not hallucinate. If you don´t have the information just say so.
Only answer the question if you can extract it from the CONTEXT provideed. Do not mention the CONTEXT used in your answer.<context>${flow:trim()}</context><question>${inputs:trim()}</question>Answer:' ) as aqchat;
````
* SplitRecord
* EvaluateJsonPath
* PublishSlack:   with  Thread Timestamp ${theadTs:isNull():ifElse(${ts},${threadTs})} and include flow file as attachment

````
======= AIRQUALITY ===========================================================================================

--- From Apache NiFi --- 
    UUID: ${uuid}
    Date: ${date}

--- Snowflake SQL Results

Row Query Duration: ${executesql.query.duration}
Row Execution Time: ${executesql.query.executiontime}
Row fetch time: ${executesql.query.fetchtime}
Row indx: ${executesql.resultset.index}
Row count: ${executesql.row.count}z

--- Cortex Search Results

Air Quality Search Single Result
${AIRQUALITY_TEXT}
AQI : ${AQI}
DATEOBSERVED : ${DATEOBSERVED}
HOUROBSERVED : ${HOUROBSERVED}
PARAMETERNAME : ${PARAMETERNAME}
REPORTINGAREA : ${REPORTINGAREA}
STATECODE : ${STATECODE}

--- flow: ${flow}

--- Cortex AI RAG Results

${content}
===========================================================================================
````  



### Snowflake SQL

````
--- Create a warehouse for search

CREATE OR REPLACE WAREHOUSE CORTEX_SEARCH_AQ_WH
WITH
     WAREHOUSE_SIZE='X-SMALL'
     AUTO_SUSPEND = 120
     AUTO_RESUME = TRUE
     INITIALLY_SUSPENDED=TRUE;


--- Create our air quality service with 1 hour lag

CREATE OR REPLACE CORTEX SEARCH SERVICE DEMO.DEMO.AIRQUALITY_SRVC
ON AIRQUALITY_TEXT
ATTRIBUTES REPORTINGAREA, STATECODE, LATITUDE, LONGITUDE, DATEOBSERVED, PARAMETERNAME, AQI
WAREHOUSE = CORTEX_SEARCH_AQ_WH
TARGET_LAG = '1 hour'
AS
        SELECT DATEOBSERVED,HOUROBSERVED,REPORTINGAREA, STATECODE, LATITUDE, LONGITUDE,PARAMETERNAME,AQI,
        ('Air Quality Report for ' || REPORTINGAREA || ' ' || STATECODE  || ' of ' ||  PARAMETERNAME || ' is ' || AQI || ' which is ' || CATEGORYNAME || '.  Observation date/time was ' || DATEOBSERVED || ' at the hour of ' || HOUROBSERVED  || '.\n\n' ) as AIRQUALITY_TEXT
    FROM DEMO.DEMO.AQ;

--- Check the Cortex Search Service for air quality

DESC CORTEX SEARCH SERVICE DEMO.DEMO.AIRQUALITY_SRVC;

--- Assign RBAC permission

GRANT USAGE ON CORTEX SEARCH SERVICE DEMO.DEMO.AIRQUALITY_SRVC TO ROLE ACCOUNTADMIN;


--- Test the cortex search service for air quality

SELECT *
FROM
  TABLE (
    CORTEX_SEARCH_DATA_SCAN (
      SERVICE_NAME => 'AIRQUALITY_SRVC'
    )
  ) LIMIT 5 ;


--- Build text report from air quality readings
    
        SELECT DATEOBSERVED,HOUROBSERVED,REPORTINGAREA, STATECODE, LATITUDE, LONGITUDE,
        ('Air Quality Report for ' || REPORTINGAREA || ' ' || STATECODE  || ' of ' ||  PARAMETERNAME || ' is ' || AQI || ' which is ' || CATEGORYNAME || '.  Observation date/time was ' || DATEOBSERVED || ' at the hour of ' || HOUROBSERVED  || '.\n\n' ) as AIRQUALITY_TEXT
    FROM DEMO.DEMO.AQ where STATECODE = 'MA';


--- Look at air quality for Mass.

select * FROM DEMO.DEMO.AQ where STATECODE = 'MA';


--- Parse PDF with Cortex Parse Document

SELECT SNOWFLAKE.CORTEX.PARSE_DOCUMENT(@DOCUMENTS, 'tspann-2024-nov-cloudx-addinggenerativeaitoreal-timestreamingpipelines-241114203020-192327c8.pdf', {'mode': 'LAYOUT'});


--- Test Cortex Complete with Pixtral Large against an image

  SELECT SNOWFLAKE.CORTEX.COMPLETE('pixtral-large',
    'Fully describe the image in at least 100 words',
    TO_FILE('@images', 'IMG_3116.jpg'));
    

--- Test Cortex Search Preview

    SELECT PARSE_JSON(
  SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
      'DEMO.DEMO.AIRQUALITY_SRVC',
      '{
        "query": "PM10 is Good",
        "columns":[
            "AIRQUALITY_TEXT",
            "REPORTINGAREA",
            "STATECODE",
            "DATEOBSERVED",
            "HOUROBSERVED",
            "PARAMETERNAME",
            "AQI"
        ],
        "filter": {"@eq": {"REPORTINGAREA": "Boston"} },
        "limit":10
      }'
  )
)['results'] as results;

--- Test Cortex Complete against llama 3.3 70b for our prompt

  SELECT SNOWFLAKE.CORTEX.COMPLETE( 'snowflake-llama-3.3-70b', 'You are an expert air quality assistant that extracs information from the CONTEXT provided between <context> and </context> tags.
When ansering the question contained between <question> and </question> tags
be concise and do not hallucinate. If you don´t have the information just say so. Only anwer the question if you can extract it from the CONTEXT provideed. Do not mention the CONTEXT used in your answer.
<context>Air Quality Report for Boston MA of O3 is 17 which is Good.  Observation date/time was 2025-05-14 at the hour of 7.</context><question>What is the air quality like right now in Boston?</question>Answer:' ) as aqchat;


--- Test Source Air Quality table

select * from demo.demo.aq where statecode = 'MA' and reportingarea ='Boston' order by dateobserved desc;

````


#### Resources

* https://docs.snowflake.com/en/user-guide/snowflake-cortex/complete-multimodal#process-images
* https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/tutorials/cortex-search-tutorial-1-search#step-3-create-the-search-service
