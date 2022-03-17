# Tanium
----

This pack is targeted for Collection and Processing of Tanium Events. Use this to help pack to help normalize Tanium Connect Data, and reduce Tanium All Asset Reports by approximately 20%. Every time we solve interesting use cases our intent is to add them to this knowledge pack.


## Requirements Section

Before you begin, ensure that you have met the following requirements:

* A Tanium Connect job pointing to your Cribl Logstream instance
* If using TaaS a ticket must be created to open up ports in your TaaS Environement's Firewall or Self Service used if that is available in your TaaS instance.


## Got a problem?

We will solve it, put it in the pack and add you to our contributors list! What to Expect Knowledge of LogStream Power of the Pipeline Requirements A insatiable thirst for knowledge.


## Release Notes

### Version 0.6.3 - 2022-03-17
Added comments to Routes to better explain their purpose. Added space saving routes into their own groups. Added route for use with the Tanium Splunk App.

### Version 0.6.2 - 2021-11-18
Fixed Sample Logs file size

### Version 0.6.1 - 2021-11-18
Fixed syntax error in sample file that prevented sample from displaying correctly.

### Version 0.6.0 - 2021-11-18
Added in an additional pipeline for Tanium Discover Connect Jobs runs.

### Version 0.5.0 - 2021-11-18
Added in an additional pipeline for Tanium Asset Software Connect Jobs runs. Edited field names to make them more readable. Field names now roll back into _raw to reduce event size. Added in event samples. Renamed routes and pipelines to be more accurate now that there is more than one. Changed route filters to be more specific.

### Version 0.0.1 - 2021-09-24
Initial release!



## Contributing to the Pack
Discuss this pack on the Community Slack channel #packs, or fork this repo and submit a merge request after your changes are complete.

## Contact

The authors and contributors to this pack are:

James Curtis james.l.curtis@gmail.com



## License
This Pack uses the following license: [`Apache 2.0`](https://github.com/criblio/appscope/blob/master/LICENSE).

