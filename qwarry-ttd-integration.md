# Qwarry TTD Integration

Qwarry Hoppscotch is the best place to manipulate segments and categories.
<https://hoppscotch.qwarry.co/>

## Taxonomy Synchronization

The lambda `qwarry-serverless-production-generateTtdTaxonomy` runs every Sunday at midnight.

### Global Overview

The integration with TTD is very specific and depends on many concepts and terms.

We have a `chargeGroupId` at the global level for a specific pricing behavior: **QwarryChargeAll**

Either charge once for the most expensive category that the advertiser uses in the group (charge-max group) or charge for the sum of each individual category that the advertiser uses in the group (charge-all group).

We have two main data element types: **custom** or **standard**.

Identifies the type of categories assigned to a data element: custom or standard. There are two parent element IDs automatically created for you, one for standard data elements and another for custom data elements.

Specifically for Qwarry (`BrandId: qwarry-ctx`), the two parent IDs are:
- **-329** (custom)
- **-328** (standard)

### Data Element (or "Charges")

As we only use standard, all our root categories ("charges" or "data element" on TTD side) have been created having their `parentElementId` set to `-328` and their `ChargeGroupId` set to `QwarryChargeAll`.

`/contextualdata/qwarry/charge/query`

> *TTD says: The pre-bid tactic determines how advertisers discover your categories in The Trade Desk platform and indicates how advertisers can use your data for pre-bid targeting and blocking.*

Adding a data element is a rare action but may be sometimes useful.

### Standard Categories

Because we create them with the path `/contextualdata/qwarry/standardcategory`, they automatically belong to `-328` parentElementId category.

`/contextualdata/qwarry/standardcategory/query`

```json
{
    "ViewStatus": "LiveView",
    "Id": "q_29063817",
    "Name": "News and Politics",
    "Path": "News and Politics (q_29063817)",
    "ParentId": null,
    "CategoryBidListType": "Both"
}
```

But they are **not linked at creation** to any data element (or charge) listed above. We need to create a **category charge** to create that link.

"Category Charge" refers to the mapping of a contextual category to a data element. That is how the price assignment of the segment (category on TTD side) is done.

`/contextualdata/qwarry/categorycharge/query`

```json
{
    "BrandId": "qwarry-ctx",
    "CategoryId": "q_29063819",
    "ProviderElementId": -100139
}
```

---

## Segments Data Synchronization

Segments traffic is distributed with the Augmentor in Frankfurt on the "ttd" cluster.
<https://eu-central-1.console.aws.amazon.com/ec2/home?region=eu-central-1#Instances:tag:Cluster=ttd-cluster>

As of 20 Feb 2026, 2 proxies and 3 nodes, we reply at a rate of ~15k RPS.

### Proxy Instances (c7g.xlarge, 4 cores ARM64)

| Instance ID | Public IP | Name | Traffic |
|---|---|---|---|
| i-07015fa0344c843f4 | 3.66.29.61 | proxy-server-ttd-cluster-3-66-29-61 | ~9k RPS |
| i-018b8d7dd4f107f61 | 63.178.0.96 | proxy-server-ttd-cluster-63-178-0-96 | ~6k RPS |

### Node Instances (c7g.2xlarge, 8 cores ARM64)

| Instance ID | Public IP | Name |
|---|---|---|
| i-077fcdb25955ae079 | 18.199.143.243 | augmentor-server-ttd-cluster-18-199-143-243 |
| i-0e7bee88d1501d90b | 18.185.91.30 | augmentor-server-ttd-cluster-18-185-91-30 |
| i-0f4f23b1b25876c63 | 3.72.6.214 | augmentor-server-ttd-cluster-3-72-6-214 |

### Bypass Instances

TTD asks us to reply a "blank" answer in Asia and US even if we don't have any activity there. There is no local subscription.

| Instance ID | Public IP | Name | Region | Type | Traffic |
|---|---|---|---|---|---|
| i-02b4a75a4dce2b30c | 98.93.135.205 | bypass-proxy-us-east-1 | us-east-1c | c6g.large, 2 cores ARM64 | ~5k RPS |
| i-05a45d75f9429d1aa | 47.129.144.86 | bypass-proxy-ap-southeast-1 | ap-southeast-1b | c6g.large, 2 cores ARM64 | ~4k RPS |

### Traffic with Domain Names (qwarry.co)

**Main TTD domain:**
`ttd-augmentor.qwarry.co` — Weighted round-robin between:
- 3.124.196.173 (proxy-1, weight 100)
- 3.120.129.39 (proxy-2, weight 100)

**Regional TTD subdomains:**

| Domain | Region | Targets |
|---|---|---|
| `eu.ttd-augmentor.qwarry.co` | EU | 3.66.29.61 (proxy-1, weight 100), 63.178.0.96 (proxy-2, weight 100) |
| `us.ttd-augmentor.qwarry.co` | US | 98.93.135.205 (bypass-proxy-1, weight 100) |
| `ap.ttd-augmentor.qwarry.co` | Asia | 47.129.144.86 (bypass-proxy-1, weight 100) |

The bypass instances are smaller (c6g.large with 2 cores) and handle lower traffic volumes — US at ~5k RPS and Asia at ~4k RPS.

### Contacts at TTD

All those people are in the loop regarding the endpoint configurations:

- Preston Kelley — preston.kelley@thetradedesk.com
- Gauthier Peter — gauthier.peter@thetradedesk.com
- Dean Peart — dean.peart@thetradedesk.com
- Jeffrey Li — jeffrey.li@thetradedesk.com
- Alex Lukashev — alex.lukashev@thetradedesk.com
- Matthew McMahon — matthew.mcmahon@thetradedesk.com
- Lavinia Stanescu — lavinia.stanescu@thetradedesk.com
- Chris Makin — chris.makin@thetradedesk.com

### Monitoring

Neogmentor TTD on Grafana:
<https://g-d05d5b6d05.grafana-workspace.eu-central-1.amazonaws.com/d/dez5bhap19a0wf/neogmentor-ttd?orgId=1>

---

## Audience Synchronization

### TTD Audience Ingestion — Overview

Daily job that sends audience data to The Trade Desk (TTD) platform.

**Job Details:**
- **Job Name:** `firstids-ttd-ingestion-job-definition`
- **Region:** eu-west-3 (Paris)
- **Schedule:** Daily (processes previous day's data)
- **Console:** <https://eu-west-3.console.aws.amazon.com/batch/home?region=eu-west-3#jobs/list>
- **Monitoring:** CloudWatch Logs

### What It Does

The job reads audience segment files from S3, matches user IDs with their audience segments, and uploads this data to TTD's platform. It batches the data to stay within TTD's size limits and retries failed uploads automatically.
