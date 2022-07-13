# Crowdstrike FDR pack
----

This pack processes data originating from a 'Crowdstrike source' or 'Amazon S3 pull source' through Cribl. This is not to be confused with a Cribl 'S3 Collector'.

Crowdstrike FDR logs are rich with data, but much of the data is considered noisy and not necesssary for continuous security monitoring. This pack reduces Crowdstrike logs by up to 80% by:
- dropping unwanted event_simpleNames, 
- removing unwanted fields from events 
- aggregating network communication logs
- sampling ExternalApiEvent
- dropping DNS events matching Majestic 100K top URLs 

Additionally, the pack includes the option to use Redis for the aggregation, as well as enrichment of the ComputerName and other asset data from an aid lookup. This is because Crowdstrike populates an aimaster directory in the S3 bucket with asset data regularly. Streaming events only have an aid field. Retrieving ComputerName and other inventory info requires a lookup from the aimaster logs. This type of lookup requires Redis. 

This pack includes a set of CSV lookup files including:
- **event_simpleNames_DROP.csv** (list of event_simpleName values to drop)
- **event_simpleName_NetworkEvents.csv** (list of network event_simpleName values to apply aggregations to)
- **event_simpleName_Processes.csv** (list of process event_simpleName values to treat identically)

# Pack Workflow
![Crowdstrike Pack Workflow Diagram](https://drive.google.com/uc?id=1G8Z31Txft8GSv854nfe0ZpooJ3i7K8hl)

## Requirements Section

Optional:
- Redis is required for enrichment. This is due to Crowdstrike sending the data to enrich as events in the fdr/aimaster directory. If you don't want to use Redis, disable both the enrichment route and enrichment chain functions within the Crowdstrike_General pipeline
- Redis is optional but recommended for stateful aggregation


## Instructions:
1. Create a new 'Crowdstrike' source or 'Amazon S3 pull source' within Cribl Stream. This leverage SQS first to obtain a list of newly added files to the S3 bucket, which are subsequently downloaded.   
**Do NOT confuse those with 'S3 Collector'.  The 'S3 Collector' is used for replays, adhoc pulls, and/or scheduled pulls from S3. It is not applicable here.**

2. Download and install this Event Breaker Ruleset. Install through the 'Knowledge' dropdown menu at the very top, and select Event Breakers.

   **[`Right Click to download Cribl Crowdstrike Event Breaker Ruleset`](https://drive.google.com/uc?id=1GoVKO8y_9AlbFXalvsNqF6per96hUDQ3)**

3. Associate the event breaker with the Crowdstrike FDR S3 source

4. Create a Route with a filter to the FDR S3 source, or QuickConnect from the FDR S3 Source. Select the `Crowdstrike Pack` pack as the pipeline.

5. Capture data on the wire to ensure your event formats match those in the pack. 

6. If using Redis (optional, but recommended):
    A. Edit all pipelines with Redis functions including: Inventory_Events_Redis, Inventory_Enrich_FromRedis, and Crowdstrike_Network. Perform the following:
        a. Edit each pipeline
        b. Click on the gear icon next to the 'function' button
        c. Select 'edit as JSON'
        d. Perform bulk replace of the 3 fields: Redis URL, Redis user (if applicable), and Redis password (if applicable)
    B. Enable the Redis_Inventory_Population route
    C. Disable the Inventory_Population_passthru route
    D. Within the Crowdstrike_General pipeline, enable function #5: 'Enrich from Redis'
    E. Within the Crowdstrike_Network pipeline, enable the 'Aggregation via Redis' group, and disable the 'Native Aggregation' group
    F. For Redis Network Aggregation, adjust max events and aggregation period by editing Function 5 in the Crowdstrike_Network pipeline. Default values are 6000 seconds and 100 max events in the period.

7. Optional modifications:
    1. To change the list of dropped event_simpleNames update the ...DROP.csv file. Remove or add entries as necesssary
    2. To change the list of fields removed from every event, edit the parser functions within the master 'Crowdstrike_General' and all children chained pipelines.

## Release Notes

### Version 0.65 - 2022-07-13
- Removed value==0 from filter of fields to drop in various pipelines
- Added new route for non-streaming events and no Redis
- Clarified the source to configure in the documentation (Crowdstrike or Amazon S3 pull source, not S3 Collector)

### Version 0.63 - 2022-06-29
- Moved the Redis enrichment section to the end of the Crowdstrike_General pipeline, so you only enrich events that will sent out, not other ones that are dropped
- Additional fixes in Network Aggregation pipeline
- Added a !cribl_pipe filter to the 'Other Events' function to accommodate Redis being moved to the end

### Version 0.62 - 2022-06-14
- Added ExternalApiEvent Pipeline and chain function
- Fixed Aggregations in Network Pipeline
    - Group by fields are now consistent across both functions
    - Arrays returning single value are converted to single fields
- Added Auto-timestamp to the core Crowdstrike_General pipeline as a backup to Event Breaker Ruleset

### Version 0.50 - 2022-05-06
---
In this release, we have added a number of great features. We've goat you covered!
- Redis population of aimaster lookup data including ComputerName
- Enrichment from Redis of ComputerName and other Inventory fields
- Pipelines for Network Events, DNS Request Events, and Process Events

## Contributing to the Pack
---
Discuss this pack on our Community Slack channel [#packs](https://cribl-community.slack.com/archives/C021UP7ETM3).

## Contact
---
The author of this pack is Ahmed Kira and can be contacted at <akira@cribl.io>.  
Special thanks to Igor Gifrin for also contributing to the pack: <igifrin@cribl.io>.

## License
---
This Pack uses the following license: [`Apache 2.0`](https://github.com/criblio/appscope/blob/master/LICENSE).

This Pack includes portions of the majestic million URLs found here: [`Majestic Million Domains`](https://majestic.com/reports/majestic-million?majesticMillionType=2&tld=&oq=)

Per the web page, the list uses the [`Creative Commons Attribution 3.0 Unported License`](https://creativecommons.org/licenses/by/3.0/)


