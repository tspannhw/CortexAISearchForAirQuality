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


