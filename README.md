# CrowdStrike FDR Pack
----

The CrowdStrike Falcon Data Replicator (FDR) Pack processes data originating from a **CrowdStrike** Source or **Amazon S3** Pull Source through Cribl Stream. Do not confuse it with a Cribl **S3** Collector.

CrowdStrike FDR logs are rich with data, but much of the data is considered noisy and not necesssary for continuous security monitoring. This pack reduces CrowdStrike logs by up to 80% by:
- Dropping unwanted process events based on process hash and/or process name
- Dropping of unwanted events based on `event_simpleNames` values.
- Removing unwanted fields from events.
- Aggregating network communication logs.
- Sampling `ExternalApiEvent`.
- Dropping DNS events matching Majestic 5K top URLs. 
- Trimming whitespaces and tabs from `ScriptControlScanInfo` events, which show the entire script being executed
- Convert Hex to ASCII for `ScriptContentBytes` events

Additionally, this Pack includes the option to use Redis for the aggregation, as well as enrichment of the `ComputerName` and other asset data from a Lookup. This is because CrowdStrike regularly populates an `aimaster` directory in the S3 bucket with asset data. Streaming events only have an aid field. Retrieving `ComputerName` and other inventory information requires a Lookup from the `aimaster` logs. This type of Lookup requires Redis. 

This pack includes a set of CSV lookup files including:
- `event_simpleNames_DROP.csv` - lList of `event_simpleName` values to drop
- `event_simpleName_NetworkEvents.csv` - List of network `event_simpleName` values to apply aggregations to
- `Process_SHA256HashData_Drop.csv.csv` - List of process names and corresponding SHA256 hash values to drop
- `event_simpleName_Processes.csv` - List of process `event_simpleName` values to drop

# Pack Workflow



The diagram below shows the workflow for this Pack, including optional paths of using Redis.
![Crowdstrike Pack Workflow Diagram](https://drive.google.com/uc?id=1G8Z31Txft8GSv854nfe0ZpooJ3i7K8hl)

## Requirements

Consider the following regarding the optional use of Redis:

- Redis is optional but recommended for stateful aggregation.
- Redis is required for enrichment - because CrowdStrike sends the data to enrich as events in the `fdr/aimaster` directory. 
- If you don't want to use Redis, disable both the enrichment Route and enrichment chain Functions within the `Crowdstrike_General` pipeline.

## Using this Pack:

1. Create a new CrowdStrike Source or Amazon S3 Pull Source. This leverage SQS first to obtain a list of newly added files to the S3 bucket, which are subsequently downloaded.   
**NOTE**: Do **not** confuse those Sources with an S3 Collector.  The S3 Collector is used for replays, adhoc pulls, and/or scheduled pulls from S3. It is not applicable here.

2. Download and install this Event Breaker ruleset. Install it by hovering over the **Processing/Packs** menu at the view top of the Cribl UI, then select **Knowledge**, then select **Event Breakers**. 

Right Click [here](https://drive.google.com/uc?id=1GoVKO8y_9AlbFXalvsNqF6per96hUDQ3) download Cribl CrowdStrike Event Breaker Ruleset.

**[`Right Click to download Cribl Crowdstrike Event Breaker Ruleset`](https://drive.google.com/uc?id=1GoVKO8y_9AlbFXalvsNqF6per96hUDQ3)**

3. Associate the Event Breaker with the CrowdStrike FDR S3 Source.

4. Create a Route with a filter to the FDR S3 Source, or QuickConnect from the FDR S3 Source and Select the `CrowdStrike Pack` pack as the Pipeline.

5. Capture data using **Capture** within the **Sample Data** right pane... to ensure your event formats match those in the Pack. 

If using Redis (optional, but recommended), edit all Pipelines with Redis Functions including: `Inventory_Events_Redis`, `Inventory_Enrich_FromRedis`, and `Crowdstrike_Network`. 
Perform the following:

- Edit each pipeline
- Click on the gear icon next to the 'function' button
- Select 'edit as JSON'

Perform bulk replace of the three fields: `Redis URL`, `Redis user` (if applicable), and `Redis password` (if applicable):

- Enable the `Redis_Inventory_Population` Route.
- Disable the `Inventory_Population_passthru` Route.
- Within the `Crowdstrike_General` Pipeline, enable Function 5: 'Enrich from Redis'.
- Within the `Crowdstrike_Network` Pipeline, enable the 'Aggregation via Redis' group, and disable the 'Native Aggregation' group.
- For Redis Network Aggregation, adjust max events and aggregation period by editing Function 5 in the `Crowdstrike_Network` Pipeline. Default values are 6,000 seconds and 100 max events in the period.

Optional modifications:

- To change list of dropped processes, updated the process_cmdline_DROP.csv or ideally, the process_SHA256_Drop.csv file with the hash of known unwanted processes.
- To change the list of dropped `event_simpleNames` update the `...DROP.csv` file. Remove or add entries as necesssary.
- To change the list of fields removed from every event, edit the parser Functions within the master `Crowdstrike_General` and all children chained Pipelines.


## Release Notes

## Version 1.0.0 - 2022-11-14
#### Enhancements:
- Introduced process matching based on SHA256 hash and optionally for process name. Hash matching is recommended.
- Updated 1st function in all chained pipelines with filter of `typeof _raw !== 'object'`. This allows testing within the chained pipelines without making any changes.
- Pipeline optimizations. Removed parser and eval functions within children pipelines. Referenced _raw.fieldnames instead in pipelines
- Reduced DNS lookup file to top 5,000 domains to improve performance. 
- Updated Network Event Redis aggregations to allow for possible events outside time window. In such a case, an Aggregate event will be created when 1st event is generated AFTER the time window expires. 
- Updated Network Event Redis aggregation to exclude tracking of event_simpleName values
#### Bug fixes
- Fixed wrong filter in the Inventory passthru Route


### Version 0.70 - 2022-08-14
- Updated Event breakers to be as large 768Kbytes and the breaker to include the beginning of a JSON event. Many events broke since the ScriptContent or ScriptedContentBytes made the events exorbitantly large. And those fields often included new line characters.
- Updated the first Eval function in the Crowdstrike_General pipeline to NOT perform JSON.parse if the event is not JSON
- Introduced chained pipelines for other high volume offender including ScriptControlScanInfo event_simpleName, and events with a ScriptContentBytes field
- Added EndOfProcess to list of events to drop, as found at multiple customers

### Version 0.6.5 - 2022-07-13

- Removed `value==0` from filter of fields to drop in various Pipelines.
- Added new Route for non-streaming events and no Redis.
- Clarified the Source to configure in the documentation (CrowdStrike or Amazon S3 Pull Source - not S3 Collector).

### Version 0.6.3 - 2022-06-29

- Moved the Redis enrichment section to the end of the `Crowdstrike_General` Pipeline, so you only enrich events that will be sent out - not other ones that are dropped.
- Additional fixes in `Crowdstrike_Network` aggregation Pipeline.
- Added a `!cribl_pipe` filter to the `Crowdstrike_OtherEvents` Function within the `Crowdstrike_General` pipeline to accommodate Redis being moved to the end. 


### Version 0.6.2 - 2022-06-14

- Added `ExternalApiEvent` Pipeline and chain Function.
- Fixed `Native Aggregations` in `Crowdstrike_Network` Pipeline.
    - Group by fields are now consistent across both Functions.
    - Arrays returning single value are converted to single fields.
- Added Auto-timestamp to the core `Crowdstrike_General` Pipeline as a backup to `Event Breaker Ruleset`.

### Version 0.5.0 - 2022-05-06

In this release, we have added a number of great features. We've goat you covered!

- Redis population of `aimaster` Lookup data, including `ComputerName`.
- Enrichment from Redis of `ComputerName` and other `Inventory` fields.
- Pipelines for Network Events, DNS Request Events, and Process Events.

## Contributing to the Pack

Discuss this Pack on our Community Slack [#packs](https://cribl-community.slack.com/archives/C021UP7ETM3) channel.

## Contact

The author of this Pack is Ahmed Kira. Contact Ahmed at <akira@cribl.io>.  
Special thanks to Igor Gifrin for also contributing to this Pack: <igifrin@cribl.io>.

## License
---
This Pack uses the [`Apache 2.0`](https://github.com/criblio/appscope/blob/master/LICENSE) license and includes portions of the [Majestic Million URLs](https://majestic.com/reports/majestic-million?majesticMillionType=2&tld=&oq=) - licensed under Creative Commons Attribution 3.0 Unported License [(CC BY 3.0)](https://creativecommons.org/licenses/by/3.0/).
