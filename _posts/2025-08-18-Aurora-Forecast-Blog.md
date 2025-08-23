---
layout: post
title: "Aurora Borealis Forecast Tool"
date: 2025-08-18
---
## Introduction

I created a Python script which fetches Aurora data of a specific location. It collects cloud cover data, time of dusk at the location, and the percentage of observing an Aurora. I containerised my script using Docker and scheduled it to run on my Raspberry Pi every day at a certain time using `cron`.

When predicting the liklihood of a Aurora, cloud cover and time of dusk are both important factors to consider. Despite a high percentage of observing an Aurora, there ay be too much cloud cover, obscuring the view of the Aurora. Light intensity is another important factor, which is why I collected data on the time of dusk, ensuring that it is dark enough to view an Aurora.

The Aurora data source and model I used was NOAA's (National Oceanic and Atmospheric Administration) 30 minute Aurora Forecast, which is powererd by the OVATION (Oval, Variation, Assessment, Tracking, Intensity, and Online Nowcasting) Prime model, which uses real-time data from satellites to estimate how much charged particle activity (which produces an Aurora) is hitting the Earth's atmosphere. The OVATION Prime model uses this data to calculate the probability as a percentage of observing an Aurora at different locations around the World.

### Project Goals

- Fetch real-time Aurora data from the NOAA
- Fetch data on cloud cover from OpenWeatherMap - a service that provides global weather data
- Fetch data on time of dusk from Sunrise Sunset - a service that provides global data on sunset and sunrise times
- Extract the probability of an Aurora for the selected coordinates of a location
- Send an email containing all fetched data
- Containerise the script using Docker
- Schedule the script to run every 21:00 to 23:00 UTC using cron ( a scheduler) on a Raspberry Pi

### Technology Used

- Python
- Libraries: `requests`, `datetime`, `argparse`, `zoneinfo`, `smtp2goClient`
- Optional libraries: `pytest`, `json`
- Docker
- cron

## Fetching Real-Time Aurora Data

- This fetches a JSON file containing Aurora data from the URL as shown
- The JSON data is converted to a Python dictionary using `.json()` which can be manipulated
- The function returns all data and is accessible elsewhere in the script.
- Instead of specifying the URL, I provided an argument which allowed me to reuse this function for fetching any other data that was needed.

```python
def fetch_data(url):
    jsontext = requests.get(url)
    data = jsontext.json()
    return data
```

## Fetching Nautical Dusk Data

This uses the Sunrise-Sunset API to get the time when nautical twilight ends. This is the time when the sky becomes dark enough for Aurora and star viewing. This is important as an Aurora is much easier to view in low light conditions.

This API response as well as the others, are returned in JSON format, which Python handles as dictionaries and lists, making it easy to extract data.

```python
url = f"https://api.sunrise-sunset.org/json?lat={lat}&lng={long}&date=today&formatted=0"
jsontext = requests.get(url)
data = jsontext.json()
raw_time = data['results']['nautical_twilight_end']
```

Before returning the time, I converted the `raw_time` into UTC time to allow us to convert the time to a different timezone as shown later.

```python
utc_time = datetime.fromisoformat(raw_time).strftime("%Y-%m-%dT%H:%M:%SZ")
```

## Fetching Cloud Cover Data

This uses the OpenWeatherMap API to collect data on cloud cover for the next 3 hours.

Forecast time data is collected as well as the percentage of cloud cover:

```python
next_forecast = data['list'][0]
timestamp = next_forecast['dt']
cloud_cover = next_forecast['clouds']['all']
```

`timestamp` was also converted to UTC as shown below so it can be easily converted to another timezone as shown later:

```python
utc_time = datetime.fromtimestamp(timestamp, tz=timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")
```

## NOAA's Aurora Forecast Coordinate System

When I first began working with the NOAA Aurora 30 minute Forecast data, I assumed that the coordinate format was the same as most other weather forecast API’s - using decimal degrees with negative values for longitudes and negative and positive signs for latitude and longitude. I used this standard format in order to extract the probability of an Aurora at a specified location.

However, my initial attempts at extracting Aurora probabilities produced unexpected results. When I compared my percentage probability to the Aurora 30 minute forecast clip, the probabilities didn’t make sense for the places I collected data from.

I observed the first and last coordinate sets in NOAA’s latest JSON file, I noticed:

- The longitudes ranged from 0 to 359, and were never negative
- The latitudes ranged from -90 to +90, as expected

This pattern revealed that the NOAA was using a 0-359 longitude system instead of the standard -180 to +180.

From this, I realised:

- All negative longitudes, (west of Greenwich) were converted by adding 360
- E.g. -115 longitude became 360 + (-115) = 245
- Latitudes remaind standard so no change was needed
- Also, all coordinate values were integers, so I had to round all parsed coordinates using Python’s built in `round()` function

## Extracting Probability Data

Here, I looped through NOAA’s `coordinates` data to find a matching latitude and longitude.

If there is a match, it fetches the associated probability, located at index 2 - `[long, lat, probability]`.

I was able to loop through NOAA’s data easily, because the original JSON had been converted to a Python dictionary through the `fetch_data` function.

```python
for coordinate in data["coordinates"]:
    if coordinate[0] == long and coordinate[1] == lat:
        probability = coordinate[2]
```

## Converting UTC to Other Timezones

Usually, APIs give timestamps in UTC (Coordinated Universal Time), which is a standard reference time known internationally. However, you want to be able to see the time, (e.g. time of dusk), in your local timezone, without the need of a conversion.

These snippets of code convert UTC to any local timezone which is specified by the user:

#### 1. Parsing the UTC timestamp:

UTC time strings usually end in `“Z”`, which means UTC time. `fromisoformat()` can't recognise `“Z”` directly so I replaced it with `“+00:00”`, the equivilent of `“Z”`:

```python
dt_utc = datetime.fromisoformat(utc_time.replace("Z", "+00:00"))
```

#### 2. Converting to local time:

Using the library `ZoneInfo`, I converted the `dt_utc` into the timezone you specify (e.g. `“Europe/London”`):

```python
dt_local = dt_utc.astimezone(ZoneInfo(timezone_name))
```

#### 3. Formatting time:

Finally, I formatted the string making it more readable, which included the date, time and the timezone as an abbreviation:

```python
return dt_local.strftime("%Y-%m-%d %H:%M:%S %Z")
```

#### 4. Example Output:

```python
2025-07-02 23:45:00 BST
```

## Sending Emails:

#### 1. Create the email client:

I initialised the SMTP2GO client with the API key (received when setting up an accout), to authenticate your account, allowing me to freely send emails:

```python
client = Smtp2goClient(apikey)
```

#### 2. Prepare the email content:

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
- `recipient`: one of the recipient email addresses (the address the email is sent to)
- `subject`: the email's subject line containing the location name by default
- `text`: the content of the email, containing all collected data

#### 3. Send the email:

Using `client.send(**payload)`, the email is sent.

## Configuring the Script with Command Line Arguments:

To make the script more flexible, I used Python’s built in `argparse`. It allows users to specify inputs such as API key, timezone, email etc, at command-line, when running the program. This prevents users having to modify the code to meet their own needs.

#### Here is a snippet of the script:

```python
parser = argparse.ArgumentParser(description="Aurora forecast")
parser.add_argument('smtp_apikey', help='SMTP2GO API key')
parser.add_argument('email', help='Your SMTP2GO email')
```

This takes command line inputs, storing them in `args`:

```python
args = parser.parse_args()
```

Once the parsed arguments are stored in `args`, I could easily assign them to variables in the script:

```python
email = args.email
smtp_apikey = args.smtp_apikey
```

This code above means:

- `email` holds the email address that the user parsed
- `smtp_apikey` holds the user's API key for SMTP2GO

By assigning each input to variables, the program can use them to fetch and process data from specific locations, convert to specific timezones and send emails to specified email addresses.

## Containerising the Script Using Docker

### What is Docker?

Docker is a software that lets you package your script and everything it needs, into a container which can be moved between other machines easily. Containers ensure your script runs exactly the same way on different machines, preventing any unnecessary issues.

I created a Dockerfile inside my Northern Lights folder which contained my script as shown below:

#### Dockerfile:

```dockerfile
FROM python:3.11-slim
```

- This sets up the image for the container, using Python's version 3.11-slim
- `slim` is used to exclude unnecessary tools e.g. libraries, to keep the image small.

```dockerfile
WORKDIR /Aurora
```

- Sets the working directory inside the container to `/Aurora`
- This means that when the container starts, this is the directory the script runs from

```dockerfile
COPY northern_lights.py .
```

- This copies the Python script (`northern_lights.py`) from my local machine to the container’s working directory in this case `/Aurora`
- The `.` means copy everything into the current working directory (`/Aurora`)

```dockerfile
RUN apt-get install tzdata
```

- Installs `tzdata` inside the container which is one of our libraries that is needed for handing timezones

```dockerfile
RUN pip install requests smtp2go
```

- This installs the `requests` library for HTTP get requests and `smtp2go` for sending emails

```dockerfile
ENTRYPOINT ["python", "northern_lights.py"]
```

- This means that Docker runs `python northern_lights.py`
- I used `ENTRYPOINT` instead of `CMD` so that when I parsed argumets at command line, the arguments did not replace `python northern_lights.py`, which would cause an error

## Build and Transfer Docker Image to Raspberry Pi

### Build Image:

At first, when I built the docker image, I used:

```bash
docker build -t northern_lights .
```

However, after transfering this image to the Raspberry Pi, I came across an error telling me that the container was built for the wrong CPU architecture.

So, instead I used:

```bash
docker buildx build --platform linux/arm64 -t northern_lights --load .
```

This meant that the image that was built was compatible for the Raspberry Pi’s arm64 CPU architecture, allowing me to run the container without any issues.

### Transfer Image:

#### 1. Save the image as a .tar file (on your own machine):

```bash
docker save -o aurora.tar northern_lights:latest
```

#### 2. Copy the image to the Raspberry Pi:

```bash
scp aurora.tar pi@<IP address of Raspberry Pi>:/home/pi/
```

#### 3. SSH into the Raspberry Pi from your terminal:

```bash
ssh pi@<IP address of Raspberry Pi>
```

#### 4. Load the .tar file onto the Raspberry Pi:

```bash
docker load -i aurora.tar
```

#### 5. Run the Container:

```bash
docker run --rm northern_lights arg1 arg2 arg3
```

Note that when I ran the container, nothing seemed to happen, and I was not sure if my script was working or not. So, I ran:

```bash
docker run --rm northern_lights arg1 arg2 arg3 >> /home/pi/northern.log 2>&1
```

This meant that the result of running the script e.g. any printed lines as well as errors, were stored in the file `northern.log`. This allowed me to tell whether my script was working or not.

Also, the line:

```bash
tail -f /home/pi/northern.log
```

Can be used to view the file live, allowing you to see exactly when and what errors occur.

## Scheduling the Script with Cron

Once the Docker image could be run reliably on the Raspberry Pi, I wanted a way to automate it. I wanted to run the scipt at regular intervals without the need for me to intervene.

So, I decided to use `cron`:

### What is Cron?

Cron is a built in Linux tool which allows you to schedule tasks to run automatically at a time or date, specified by the user.

These tasks, which are also known as `cronjob`'s are created in a `crontab` (`cron` table).

I wanted my script to run automatically each hour between 21:00 and 23:00 UTC, every day.

### Cron Syntax:

Cron syntax uses five different sections in order to schedule tasks. Each of the sections are minutes, hours, day of the month, month, day of the week, as shown below:

```bash
 # ┌───────────── minute (0 - 59)
 # │ ┌───────────── hour (0 - 23)
 # │ │ ┌───────────── day of the month (1 - 31)
 # │ │ │ ┌───────────── month (1 - 12)
 # │ │ │ │ ┌───────────── day of the week (0 - 6) (Sun to Sat)
 # │ │ │ │ │
 # * * * * * <command to execute>
```

### Setting up a Cronjob:

#### 1. Open the crontab editor:

```bash
crontab -e
```

#### 2. Add cronjob:

```bash
0 21-23 * * * /usr/bin/docker run --rm northern_lights arg1 arg2 arg3
```

Note that in order to find the path to Docker (in my case /usr/bin/docker), type `which docker` into the Raspberry Pi’s terminal.

Also, I used `--rm` in order to automatically delete the container once it has finished running. This ensures that Docker deletes the container as soon as it is stopped, preventing the system from becoming cluttered with multiple unused containers.