---
layout: post
title: "Aurora Forecast Tool - Emails (Part 3)"
date: 2025-08-20
---
### **Sending Emails**

In this post, we will cover how emails containing the forecast data are dispatched.
<br><br><br>
**1. Create the email client:**

You initialist the `SMTP2GO` client with your API key (recieved when setting up an account). To authenticate your account:

```python
client = Smtp2goClient(apikey)
```
<br>
**2. We prepare the email content:**

```python
payload = {
        'sender': smtp_email,
        'recipient': [recipient],  
        'subject': f'{location_name} Aurora data',
        'text': output_text,
    }
```

The `payload` dictionary sets:

- `sender`: who the email is from (your SMTP2GO email address)
- `recipient`: one of the recipient email addresses
- `subject`: the email's subject line containing the location name
- `text`: the main body of the email, containing forecast data

<br>
**3. Send the email**

Using `client.send(**payload)`, the email is dispatched.
<br><br>
Note that when calling the `send_email` function, a `for` loop is used allowing an email to be dispatched to multiple recipients:

```Python
for recipient in args.recipients:
            send_email(smtp_email, recipient, smtp_apikey, output, location_name)
```
