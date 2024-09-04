# Daily_Airline_Data_Ingestion
Designed End-to-End pipeline using different AWS services to ingest the data on daily basis in target redshift table.

# Tech Stack:
1. AWS S3 : Used as a storage.
2. AWS S3 Cloud Trial : Used for sending notifications for events related to S3.
3. AWS EventBridge Pattern Rule.
4. AWS Glue Crawler.
5. AWS Glue ETL.
6. AWS SNS : used for sending notification on E-mail
7. Redshift : Used as a warehouse.
8. AWS Step Function: Used for Orchestration.

**Steps to design Pipeline:**

1. Create a S3 bucket and inside it create 2 folders - i] for dimension table and upload the airport.csv (input file) ii] for daily incoming raw file.
2. Create and start Redshift cluster->using query editor->create schema and tables for dimension table and fact table (commands are provided in input files).
   Here COPY command is used for creating the dimension table.
3. Create Glue database, then create Crawlers for dimension table and fact table with the source as JDBC connection and provide paths for the dimension and fact table from redshift->run the crawlers->meta deta tables will be created under Database.
4. In S3 bucket -> inside the folder for daily incoming raw files -> create a new folder as (date=yyyy-mm-dd) and upload the flights.csv file (input file) into that folder.
5. Now create one more crawler for this s3 bbucket with data source as S3 and folder for daily raw files-> here we need to enable crawl new sub-folder option-> save and run the crawler.
6. Now using Glue Visual ETL we are creating glue job: First add data source ->Glue data catalog->select Database and Table of daily row.
7. add one more source ->Glue data catalog->now with dimension table.
8. Select Join->in source select both of the nodes-> it will be inner join -> add condition as (originairportid = airportid)
9. Now add Change schema ->drop these columns: originairportid,airportid,date -also update the name of fields as per the schema.
10. 2nd join->source will be Change schema and destination -> condition (destinationairportid=airportid).
11. One more time add change schema -> drop both airportid -> add target-> select database->table will be fact table -> add the role as redshift-role.
12. In Upper navigation go to Job Details -> in worker node change number to 2 -> Enable Job Bookmark.
  
**Now comes the Orchestration part means orchestrating different AWS services based on SDK's and API calls.**

13. Search for Step Function-> State machines-> creat ->search StartCrawler (we need to drag and drop it) -> on RHS in API parameter update the name feild with crawler name which was created lastly for daily row files.
14. also add the policies for Glue ->create inline-> type Glue -> select all.
15. Now search for GetCrawler-> update the name as previous.
16. Now add choice-> in rules edit Rule1-> add condition as ($.Crawler.State - matches string - RUNNING)
17. Now add wait-> update seconds to 10 -> chnage the next state to GetCrawler.
18. Add JobRunStart after wait and select Wait for Complete -> in choice add default rule -> StartJobRun and in wait->GetCrawler.
19. Drag and drop SNS publish -> select the SNS topic-> in messages you can  also provide custom message.

**Error Handling:**

20. On StartJobRun -> in error handlin -> error-> select task.failed -> then for below option choose-> SNS topic for failed notification.
21. Again add choice after StartRunJob->add SNS publish on both sides.
22. In choice->default->failed notification and in Rule1-> add condition as ($.JobRunState-matches-SUCCEEDED)
23. Now go to S3 bucket -> in properties->scroll down to AWS cloudTrail event ->Configure in CloudTrail ->create->give name -> in encryption key provide some random name->enable cloudwatch log -> next-> select Data Events -> next-> in Data events->S3->create.
24. Now go to AWS EventBridge Rules-> in event patterns -> add service as S3 -> event type->AWS API Call Via Cloud Trial -> in pattern edit and enter the provided pattern.->in Target->Step Function-> name of state machine and create.
25. In s3-> inside dail raw folder -> create folder with (date=yyyy-mm-dd) and upload flights.csv file.
  **[After this CloudTrail will be generated and EventBridge Rule will be capture that and based on that it will trigger Step Function]**
