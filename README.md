# countries-scheduled-job

### Overview
A Solution to Replicate data from System A(Snowflake) to System B(Postgres)

### Assumptions
* Ordering on the replicated data does not need to be maintained
* The Case Table structure is identical in both databases
* Table only contains new/fresh data. Table will not contain data from the previous scheduled job

### Metadata
Table: ```case```
| Column Name | Type    |
|-------------|---------|
| country_id  | int     |
| country     | varchar |

Error Queue Message:
```
{
  "country_id": Number,
  "country" : String
}
```
DLQ Message:
```
{
  "country_id": Number,
  "country" : String
}
```


### Design Decisons
* Country column uses a 3-letter code to represent the country
* Our Case Table has two columns/fields (CountryId, Country)
* CountryId is the primary key of the Case Table
* Utilize Mulesoft and Anypoint MQ
* Use regulare queues (non-FIFO) and Dead Letter Queues (DLQ)
* Retry on failed records
* Scale applications independently

## Conceptual Design
[Diagram](https://lucid.app/lucidchart/4a0a08ca-3611-4a41-b03a-b9f84e4fed61/edit?view_items=iLtehS8NDRNI&invitationId=inv_946e58a3-b027-499f-ae6c-84cac0415964)
<p>Our system will run on a scheduled job. Schedule and frequency will be set based on business needs. We will have a process application (proc-countries-scheduler-app) which will pull data from the database in System A and insert that data in batches into our database in System B. This application can be scaled horizontally and vertically if needed.<p>
<p>If any errors occur during the process of replicating data over to System B, proc-countries-scheduler-app will publish messages of failed records onto our countries-schedule-error-queue in Anypoint MQ. This allows us to process any failed records asyncronously. This also allows us to have a more sophisticated fallback strategy as we can set a number of retries on those records/messages.<p>
<p>To implement the error stategy using an error queue we will need to have another process application (proc-countries-error-app). This application will subsrcibe to the queue and attempt to insert the failed records into the database in System B. If records fail again after a certain amount of retries, they will be dropped off into our countries-schedule-dlq DLQ.<p>
<p>Our proc-countries-dlq-app will take any messages off of the DLQ and append those failed records on to a failure report file as our lowest state of fallback.<p>

### Dead Letter Queues
<p>Anypoint will allow us to assign a DLQ to any queue (countries-schedule-error-queue). After our set amount of redeliveries/retries, the message will be dropped into the assign DLQ. DLQs are meant to be subscribed in the same fashion as other queues. The benefit of subscribing to the queue is the application can handle messages in real-time.<p>
  
### Limitations
* Anypoint MQ supports up to 120,000 in-flight messages per standard queue.
* The amount of resources allocated to our application are limited to Mulesoft.

### Considerations
* Limiting the amount of records pulled in per scheduled job
* Only remove processed records from System A database
* Could move all the logic in proc-countries-error-app into proc-countries-scheduler-app. Should allocate more workers to handle the extra load put onto the app.
* If we want real time data, consider having another application which will publish the new records found onto a seperate queue and then having an application subscribe/listen on that queue. [Diagram](https://lucid.app/lucidchart/d61e1b34-69ff-4696-bc56-a40f2fe38638/edit?viewport_loc=-94%2C11%2C2351%2C1094%2C0_0&invitationId=inv_f56a58c2-ef24-44ab-bdd5-1d20470a6730)
* Have a system to process the failure report and determine a strategy to recover or compensate. If customers are impacted, send notifications.
