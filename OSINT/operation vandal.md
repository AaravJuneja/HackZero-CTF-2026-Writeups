# Operation Vandal

## Description

A threat intelligence brief reveals suspicious infrastructure and an operator footprint. Analyze the provided artifact, pivot across identities and infrastructure, and attribute the operation.
A threat intelligence team has intercepted fragments of an intrusion campaign involving suspicious infrastructure and developer activity. Initial findings suggest the presence of a staging environment and an operator-linked identity. The activity appears coordinated but attribution remains incomplete.

```md
Threat Intelligence Brief
Classification: Internal Use Only
Report ID: SL-17-INT
Summary: Recent analysis has identified anomalous outbound traffic patterns originating from multiple endpoints within the monitored network. Initial investigation suggests the use of staging infrastructure likely associated with coordinated intrusion activity. Indicators of Interest (IOCs):
- Suspicious domain: sync-node[.]net
- Suspicious URLs
Recovered Artifact:
Additional Notes: Development traces indicate possible local deployment references associated with the operator environment.
Analyst Remark: Further correlation required to attribute activity to a known threat actor.
ref: vandal
```

`Flag Format: hackzero{<alias>_<ip_without_dots>}`

## Writeup

My first thought was trying to look up dns records for the domain `sync-node.net` but it didn't yield any results and `https://api-sync-node.net/` didn't give any useful results either. `https://web.archive.org/web/20260315163914/https://www.reddit.com/r/homelab/comments/1rudelt/update_something_on_my_home_network_is_making/` again was a dead end.

Upon rechecking the exif data of pdf:

```yaml
Title                           : Incident Report – Internal Use Only
Producer                        : Skia/PDF m148 Google Docs Renderer
Create Date                     : 0000:01:01 00:00:00
Keywords                        : apt, osint, infrastructure, incident
Author                          : vandal_ops_99@gmail.com
Creator                         : Vandal Research Unit
```

The username `vandal_ops_99` / `vandal-ops-99` instantly gave a hit via a hit on [username lookup](https://instantusername.com/?q=vandal-ops-99) which led to a [github profile](https://github.com/vandal-ops-99/) which had a [single repository](https://github.com/vandal-ops-99/portal-sync) which had `primary endpoint: sync-node[.]net/logs backup node: 185.225.17.23`

This concludes the OSINT portion of the challenge and gives us the flag hackzero{vandal_ops_99_1852251723}

## Flag

`hackzero{vandal_ops_99_1852251723}`
