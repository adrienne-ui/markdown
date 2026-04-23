# Pubmatic

![Pubmatic Segment Synchronization](images/pubmatic_segment_synchro.png)

## Data Inventory Segments

### Taxonomy Synchronization

Need to go to the partner dashboard and perform the update **manually**.

---

## Contextual Segments

**Dashboard:** [Pubmatic Audience Manager - Contextual](https://apps.pubmatic.com/v3/connect/demand/audience-manager/audience/contextual)

### Audience Synchronization

> ⚠️ We don't send our audience data as of **13 Feb 2026**.

---

## Segments Data Synchronization

Segments traffic is distributed with the **Augmentor** in Frankfurt (`eu-central-1`) on the **"tiers"** cluster.

### Infrastructure (as of 13 Feb 2026)

| Component | Details |
|-----------|---------|
| Proxy | 1 |
| Nodes | 2 |
| Rate | ~13k RPS |
| HAProxy Stats | [http://3.78.56.94/haproxy?stats](http://3.78.56.94/haproxy?stats) |

### Monitoring

Grafana dashboard — **Neogmentor Tiers**:

[https://g-d05d5b6d05.grafana-workspace.eu-central-1.amazonaws.com/d/cf4snfzbvs7i8e/neogmentor-tiers?orgId=1&var-Partner=pubmatic](https://g-d05d5b6d05.grafana-workspace.eu-central-1.amazonaws.com/d/cf4snfzbvs7i8e/neogmentor-tiers?orgId=1&var-Partner=pubmatic)

### Response Format

For each URL we send back:

```json
{
  "vendor": "qwarry",
  "sources": [
    {
      "dpid": 806,
      "segments": [...]
    }
  ]
}
```

### Endpoint IP

We should **not** need to update the endpoint IP because we have allocated an **Elastic IP**.

If we need to change anything, we need to contact **Marc Canal** by mail.
