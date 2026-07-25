---
title: "OpenWRT UCI scripting cheat sheet"
last_modified_at: 2026-05-10T19:01:00+02:00
categories:
  - OpenWRT
tags:
  - scripting
  - automation
  - networking
---

## Notes
I use UCI together with `bash`. When using other shells, you might need to adjust variable handling and loop syntax accordingly. This cheatsheet was originally created as one of my GitHub [gists](https://gist.github.com/bsmcn/dd4743fc2199a80cbe6638bdd2e7334c).

## New config entry - indexing

### Negative indexes
Reference the newly added item with `[-1]`:
    
```text
uci add firewall rule
uci set firewall.@rule[-1].name=test
```
Each newly added entry is always accessible via `[-1]`.

### Variables
Assign the new entry to a shell variable:
    
```text
rule=$(uci add firewall rule)
uci set firewall.$rule.src='lan'

pbr_policy=$(uci add pbr policy)
uci set pbr.$pbr_policy.name='test'
```
When you store the result in a variable, you capture the unique identifier of the new configuration entry. This approach is generally more predictable in scripts and less error-prone.
You can combine the commands into one line:
```text
uci set pbr.$(uci add pbr policy).name='test'
```

## Modifying existing entries

### Unnamed sections

`[0]` lets you hook to the first unnamed section in the config file:

```
config dnsmasq
        option domain 'lan'
```

`uci set dhcp.@dnsmasq[0].domain='home'`

### Named sections

```
config adblock 'global'
        option adb_fetchutil 'curl'
```
`uci set adblock.global.adb_fetchutil='uclient-fetch'`

## Deleting
### Pending changes
All pending changes (`uci changes`) are stored in the temporary file system. You can nuke them with `rm -rfv /tmp/.uci/*`.

### Quirks
When deleting entries with `uci delete`, the indexes of remaining configuration entries are updated immediately, even before committing with `uci commit`.

For example, entry [1] becomes [0] as soon as the original [0] entry is deleted.

#### Real-World Example

After installing the PBR package, you may want to remove two default policies from the configuration.

A common approach might be: `for i in {0..1}; do uci -q delete pbr.@policy[$i]; done`

However, this will only delete the first policy.

Correct approach: `for i in {0..1}; do uci -q delete pbr.@policy[0]; done`

This works because the list is reindexed after each deletion.
