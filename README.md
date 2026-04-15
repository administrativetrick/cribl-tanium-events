# Tanium
----

This pack is targeted for Collection and Processing of Tanium Events. Use this pack to help normalize Tanium Connect Data being sent by Tanium Connect Jobs, and reduce Tanium All Asset Reports by approximately 20%. Every time we solve interesting use cases our intent is to add them to this knowledge pack.


## Requirements Section

Before you begin, ensure that you have met the following requirements:

* A Tanium Connect job pointing to your Cribl Logstream instance
* Highly recommend sending data from your Cribl Connect job as TCP and using a TCP Source in Cribl. There is an experimental function in the TaniumSplunkApp pipeline. However this is not supported at this time. If anyone is able to send in sample data from an environment sending Tanium data as syslog I can work on making this into a supported feature.
* Source must set an index field to Tanium for data to be picked up, or filter in routes must be changed to match your preferred filter
* If using TaaS a ticket must be created to open up ports in your TaaS Environement's Firewall or Self Service used if that is available in your TaaS instance.

## Enrichment
Use the application_approval_status.csv Lookup Knowledge Object to enrich your Tanium Data with a new field approved. You can give that field a value of True of False, or Yes or No. If an Md5 or Product name matches a name given in this Lookup, a new field named approved will be created in the event with a value that matches the value present in the Lookup file.

## Got a problem?

We will solve it, put it in the pack and add you to our contributors list! What to Expect Knowledge of LogStream Power of the Pipeline Requirements A insatiable thirst for knowledge.


## Pipelines

| Pipeline | Use case | Default index / sourcetype |
|---|---|---|
| `TaniumAsset` | All Asset Reports normalization, optional location enrichment, optional PII mask | `tanium` / `tanium:asset:allassets` |
| `TaniumDiscover` | Discover (managed/unmanageable hosts), null/empty field stripping | `tanium` / `tanium:discover:*` |
| `TaniumSoftwareUsage` | Software Usage reports | `tanium` / `tanium:asset:softwareusage` |
| `TaniumSplunkApp` | Pass-through for Tanium Splunk App users (preserves Splunk app field names) | `tanium` / `tanium` |
| `TaniumThreatResponse` | EDR / Threat Response events, mapped to Splunk CIM Endpoint and ECS | `tanium_security` / `tanium:threatresponse` |
| `TaniumComply` | Vulnerability scan data + CVE extraction, mapped to Splunk CIM Vulnerabilities | `tanium_security` / `tanium:comply:vulnerability` |
| `TaniumPatch` | Patch installation/applicability status, mapped to CIM Updates | `tanium_ops` / `tanium:patch:status` |
| `TaniumClientStatus` | Tanium client health (last_registration, version) for stale-agent alerting | `tanium_ops` / `tanium:client:status` |
| `TaniumDeploy` | Software deployment status events | `tanium_ops` / `tanium:deploy:status` |

### Lookups

* `application_approval_status.csv` — match on `MD5_Hash` or `product_usage_name`, returns `approved` (Yes/No). Seeded with example browser/p2p/remote-access entries.
* `locations.csv` — CIDR match on `subnet`, returns `site_name`, `region`, `country`, `timezone`. Wired into the `TaniumAsset` pipeline by default; populate with your own subnets.

## Release Notes

### Version 0.7.0 - 2026-04-15
* Added `TaniumThreatResponse` pipeline for Tanium Threat Response / EDR data with Splunk CIM Endpoint and ECS field mappings, optional PII masking
* Added `TaniumComply` pipeline for vulnerability/compliance data with regex-based CVE extraction and CIM Vulnerabilities mapping
* Added `TaniumPatch` pipeline for patch status normalization (CIM Updates) with optional drop of fully-installed/non-applicable noise
* Added `TaniumClientStatus` pipeline that computes `hours_since_registration` for stale-agent alerting
* Added `TaniumDeploy` pipeline for software deployment status events with optional drop of old succeeded deployments
* Added `Security` and `Operations` route groups; new routes auto-detect ThreatResponse, Comply, Patch, ClientStatus, and Deploy traffic
* Enhanced `TaniumAsset` with CIDR-based location enrichment from `locations.csv` (adds `site`, `region`, `country`, `timezone`) and an optional disabled-by-default username masking function
* Seeded `application_approval_status.csv` with example approved/denied software entries
* Seeded `locations.csv` with a CIDR → site/region/country/timezone schema and example rows
* Added sample data for all five new pipelines
* Updated pack manifest tags (`security`, `edr`, `vulnerability`, `patch`, `ops`, `health`, `deploy`) and use cases (`security`, `compliance`)
* Documented codebase conventions in `CLAUDE.md`

### Version 0.6.6 - 2022-03-23
* Changed naming convention of pack to follow Cribl standard naming convention for Cribl Community Packs
* Changed names of sample files to match pipelines to follow Cribl best practices
* Changed name of taniumsplunkapp pipeline to match camel case of other pipelines
* Added sample data for Tanium Software Usage
* Added explanation of what sample data to use in TaniumSplunkApp pipeline

### Version 0.6.5 - 2022-07-06
Added additional comments into taniumsplunkapp pipeline to make their uses clear
Added AutoTimestamp function for use with Tanium data sent via syslog. This function is disabled by default but can be enabled for those sending via syslog.
Removed filter to replace spaces with _'s to better match Splunk App


### Version 0.6.4 - 2022-03-23
Added a lookup function in the taniumsplunkapp pipeline that can be used to enrich Tanium data with an approval status for events that have a MD5_Hash or product_usage_name that is matches a value in the lookup.

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

