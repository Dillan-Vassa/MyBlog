---
layout: post
title: "Aurora Forecast Tool - Scheduling (Part 6)"
data: 2025-08-20
---
### **Scheduling the Script with Cron**

In this post, we will cover how we can schedule our script to run on the Raspberry Pi every day at a certain time using `cron`.
<br><br>
Once the Docker image could be run reliably on the Raspberry Pi, I needed a way to automate it. Specifically, to run the scipt at regular intervals without intervention.

This is where `cron` comes in…
<br><br>
#### **What is Cron?**

Cron is a built in Linux tool that allows you to schedule tasks to run automatically at specific times, dates, or intervals.

These tasks are created in a `crontab` (`cron` table).

I wanted my Python script to run automatically every hour between 21:00 and 23:00 UTC, daily.
<br><br>
#### **Cron Syntax:**

Cron syntax uses five fields to schedule tasks, representing minutes, minutes, hours, day of month, month, day of week, as shown below:

```bash
 # ┌───────────── minute (0 - 59)
 # │ ┌───────────── hour (0 - 23)
 # │ │ ┌───────────── day of the month (1 - 31)
 # │ │ │ ┌───────────── month (1 - 12)
 # │ │ │ │ ┌───────────── day of the week (0 - 6) (Sun to Sat)
 # │ │ │ │ │
 # * * * * * <command to execute>
```
<br>
#### **Setting Up Cron Jobs:**
<br>
**1. Open the crontab editor:**

```bash
crontab -e
```
<br>
**2. Add Cron Job**

```bash
0 21-23 * * * /usr/bin/docker run --rm northern_lights arg1 arg2 arg3
```

Note that in order to find the path to Docker (in my case `/usr/bin/docker`), type `which docker` into the Raspberry Pi’s terminal.

Also, `--rm` was used in order to automatically delete the container once it finishes running. This ensures Docker cleans up the container as soon as it is stopped, preventing the system from becoming cluttered.
