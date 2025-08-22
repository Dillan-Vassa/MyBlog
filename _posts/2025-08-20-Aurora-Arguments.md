---
layout: post
title: "Aurora Forecast Tool - Command-Line Arguments (Part 4)"
data: 2025-08-20
---
### **Configuring the Script with Command-Line Arguments**

In this post, we will cover how users are able to parse arguments at command line, providing an easier approach when using the script.
<br><br>
To make our script flexible and more user-friendly, we use Python’s built in argparse. It allows users to specify inputs such as API key, timezone, email etc at command-line, when running the program. This prevents users having to modify the code to meet their needs.
<br><br>
**Here is a snippet of the script:**

```python
parser = argparse.ArgumentParser(description="Aurora forecast")
parser.add_argument('smtp_apikey', help='SMTP2GO API key')
parser.add_argument('email', help='Your SMTP2GO email')
```
<br>
This captures command-line inputs, storing them in `args`:

```python
args = parser.parse_args()
```
<br>
Once the parsed arguments are stored in `args`, we can easily assign them to variables within the script:

```python
email = args.email
smtp_apikey = args.smtp_apikey
```

This means:

- `email` holds the email address that the user parsed
- `smtp_apikey` holds the user's API key for SMTP2GO

By assigning these to variables, the program can use them to fetch and process location-specific data, convert to specific timezones and send notifications to specified email addresses.
