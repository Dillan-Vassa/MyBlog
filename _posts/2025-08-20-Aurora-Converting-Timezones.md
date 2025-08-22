---
layout: post
title: "Aurora Forecast Tool - Timezones (Part 2)"
date: 2025-08-20
---
### **Converting UTC To Other Timezones**

In this post, we will be covering how UTC time can be changed to any other timezone using `zoneinfo`.
<br><br>
APIs often give timestamps in UTC (Coordinated Universal Time), which is a standard reference time internationally. However, for practical use, you want to be able to see the time, (e.g. time of dusk), in your local timezone.

These snippets convert UTC to any local timezone specified by the user:
<br><br>
**1. Parsing the UTC timestamp:**

UTC time strings usually end in `“Z”`, which means UTC time. `fromisoformat()` doesn’t recognise `“Z”` directly so we replace it with `“+00:00”`, to indicate no time offset:

```python
dt_utc = datetime.fromisoformat(utc_time.replace("Z", "+00:00"))
```
<br>
**2. Converting to local time:**

Using the library `ZoneInfo`, we convert the `dt_utc` datetime object into the timezone you specify (e.g. `“Europe/London”`):

```python
dt_local = dt_utc.astimezone(ZoneInfo(timezone_name))
```
<br>
**3. Formatting the time:**

Finally, we format the datetime into a readable string including data, time and timezone abbreviation:

```python
return dt_local.strftime("%Y-%m-%d %H:%M:%S %Z")
```
<br>
**4. Example output:**
```python
2025-07-02 23:45:00 BST
```
<br>
