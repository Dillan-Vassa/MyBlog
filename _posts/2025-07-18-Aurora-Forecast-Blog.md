---
layout: post
title: "Aurora Borealis Forecast Tool"
date: 2025-07-18
---
## Introduction

An aurora is natural vibrant light glowing in the sky caused by electrically charged particles from the sun colliding with the gas mollecules in Earth's atmosphere.

In order to collect data on the percentage of an aurora at a specified location, I used NOAA's (National Oceanic and Atmospheric Administration) 30 minute aurora Forecast. This is powererd by the OVATION (Oval, Variation, Assessment, Tracking, Intensity, and Online Nowcasting) Prime model, which uses real-time data from satellites to estimate how much charged particle activity is hitting the Earth's atmosphere. The OVATION Prime model uses this data to calculate the probability as a percentage of observing an aurora at different locations around the World.

When predicting the liklihood of an aurora, cloud cover is one important factor to consider. Despite a high percentage of observing an aurora at a location, there may be too much cloud cover at that time, obscuring the view. 

As well as cloud cover, light intensity is another important factor. Again, despite a high percentage of an aurora, if the sun has not set, the aurora may become difficult to view or not visible at all. This is why I collected data on the time of dusk, to ensure that it is dark enough to view an aurora.

Once considering these factors, I created a Python script which fetches aurora data of a specific location. It collects cloud cover data, the time of dusk at the location, and the percentage of observing an aurora. I then containerised my script using Docker and scheduled it to run on my Raspberry Pi every day at a certain time using `cron`. Once it has collected all the data, it sends me an email containing all necessary information.

Note that I used SMTP2GO for sending emails as this website allows emails to be sent without charge and it also has a Python API allowing my to send emails easily using Python.

### Project Goals

- Fetch real-time aurora data from the NOAA
- Fetch data on cloud cover from OpenWeatherMap - a service that provides global weather data
- Fetch data on time of dusk from Sunrise-Sunset - a service that provides global data on sunset and sunrise times
- Extract the probability of an aurora for the selected coordinates of a location
- Send an email containing all fetched data using SMTP2GO (a service that allows emails to be sent)
- Containerise the script using Docker
- Schedule the script to run every 21:00 to 23:00 UTC using cron ( a scheduler) on a Raspberry Pi

### Technology Used

- Python
- Libraries: `pytest`, `json`, `requests`, `datetime`, `argparse`, `zoneinfo`, `smtp2goClient`
- Docker
- `cron`

## Fetching JSON Data

- This fetches a JSON response containing aurora data from the URL as shown below
- The JSON data is converted to a Python dictionary using `.json()` which can be manipulated easily
- The function returns all data and is accessible elsewhere in the script.
- At first, I specified the URL, but I figured that I needed to reuse the function to fetch other data. So, instead of specifying the URL, I provided an argument which allowed me to reuse this function for fetching any other data that was needed.

```python
def fetch_data(url):
    jsontext = requests.get(url)
    data = jsontext.json()
    return data
```

## Fetching Nautical Dusk Data

This uses the Sunrise-Sunset API to get the time when dusk starts (the end of twilight). This is the time when the sky becomes dark enough for aurora and star viewing.

The Sunrise-Sunset API provides data on two types of dusk:

- Civil dusk
- Nautical dusk

I found out that civil dusk is when the sun's position is 6 degrees below the horizon, however, nautical dusk is when the sun is 12 degrees below the horizon. Therefore, the end of nautical dusk (nautical twilight) was the best time to observe an aurora.

This API response as well as the others, are returned in JSON format, which Python handles as dictionaries and lists, making it easy to extract data.

```python
url = f"https://api.sunrise-sunset.org/json?lat={lat}&lng={long}&date=today&formatted=0"
jsontext = requests.get(url)
data = jsontext.json()
raw_time = data['results']['nautical_twilight_end']
```

Before returning the time, I converted the `raw_time` into UTC time to allow me to convert the time to a different timezone as shown later.

For programmers, it is best practice to keep date/time values in UTC because UTC provides a standard which is recognised internationally.

Keeping all date/time values in UTC ensures others in different regions are able to convert date/time values into their timezones without errors.

```python
utc_time = datetime.fromisoformat(raw_time).strftime("%Y-%m-%dT%H:%M:%SZ")
```

## Fetching Cloud Cover Data

This uses the OpenWeatherMap API to collect data on cloud cover for the next 3 hours.

Here is an snippet of a JSON response from OpenWeatherMap:

```json
{
  "coord": {
    "lon": -0.263,
    "lat": 51.334
  },
  "clouds": {
    "all": 90
  },
  "dt": 1756635038,
  "sys": {
    "sunrise": 1756617158,
    "sunset": 1756666232
  }
}
```

This JSON response can be manipulated as a dictionary when using Python, making it easy to extract the percentage of `clouds` at specified coordinates.

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

When I first began working with the NOAA aurora 30 minute Forecast data, I assumed that the coordinate format was the same as most other weather forecast API’s - using decimal degrees with negative values for longitudes and negative and positive signs for latitude and longitude. I used this standard format in order to extract the probability of an aurora at a specified location.

However, initially, the extracted aurora data gave unexpected results. When I compared my percentage probability to the aurora 30 minute forecast clip by the NOAA, the probabilities didn’t make sense for the places I collected data from.

So, I observed the first and last coordinate sets in NOAA’s latest JSON file, I noticed:

- The longitudes ranged from 0 to 359, and were never negative
- The latitudes ranged from -90 to +90, as expected

This pattern had revealed that the NOAA was using a 0-359 longitude system instead of the standard -180 to +180.

From this, I found out that:

- All negative longitudes, (west of Greenwich) were converted by adding 360 degrees
- E.g. -115 longitude became 360 + (-115) = 245
- Latitudes remained standard so no change was needed
- Also, all coordinate values were integers, so I had to round all parsed coordinates using Python’s built in `round()` function

## Extracting Probability Data

Here, I looped through NOAA’s `coordinates` data to find a matching latitude and longitude.

If there is a match, it fetches the associated probability, located at index 2 - `[long, lat, probability]`.

I was able to loop through NOAA’s data easily, because the original JSON had been converted to a Python dictionary through the `fetch_data` function.

Here is a snippet of NOAA's aurora data:

```json
{
  "Observation Time": "2025-09-02T09:31:00Z",
  "Forecast Time": "2025-09-02T10:16:00Z",
  "Data Format": "[Longitude, Latitude, Aurora]",
  "coordinates": [
    [0, -90, 4],
    [0, -89, 0],
    [0, -88, 5]
  ]
}
```

This JSON response contains the time to observe the aurora (observation time), time of data collection (forecast time), the format of the data and all the different locations around the world.

```python
for coordinate in data["coordinates"]:
    if coordinate[0] == long and coordinate[1] == lat:
        probability = coordinate[2]
```

I also ensured to convert any negative longitudes to positive to match NOAA's coordinate format, as well as rounding any decimal values:

```python
if long < 0:
    long = 360 + long
long = math.floor(long)
lat = math.floor(lat)
```

## Converting UTC to Other Timezones

Usually, APIs give timestamps in UTC (Coordinated Universal Time), which is a standard reference time known internationally. However, I wanted to be able to see the time, (e.g. time of dusk), in my local timezone, with ease.

These snippets of code convert UTC to any local timezone which is specified by the user:

#### 1. Parsing the UTC timestamp:

UTC time strings usually end in `“Z”`, which means UTC time. However, `fromisoformat()` can't recognise `“Z”` directly, so I replaced it with `“+00:00”`, the equivilent of `“Z”`, indicating UTC time:

```python
dt_utc = datetime.fromisoformat(utc_time.replace("Z", "+00:00"))
```

#### 2. Converting to local time:

Using the library `ZoneInfo`, I converted `dt_utc` into the timezone you specify (e.g. `“Europe/London”`):

```python
dt_local = dt_utc.astimezone(ZoneInfo(timezone_name))
```

#### 3. Formatting time:

Finally, I formatted the string making it more readable. This included the date, time and the timezone as an abbreviation:

```python
return dt_local.strftime("%Y-%m-%d %H:%M:%S %Z")
```

#### 4. Example Output:

```python
2025-07-02 23:45:00 BST
```

## Sending Emails:

#### 1. Create the email client:

I initialised the SMTP2GO client with the API key (received when setting up an accout). This is to authenticate the account, allowing me to freely send emails:

```python
client = Smtp2goClient(apikey)
```

#### 2. Prepare the email content:

```python
payload = 
      {
        'sender': smtp_email,
        'recipient': [recipient],  
        'subject': f'{location_name} aurora data',
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

## Command Line Arguments:

To make the script more flexible, I used Python’s built in `argparse`. It allows users to specify inputs such as API key, timezone etc, at command-line, when running the program. This prevents users having to modify the code to meet their own needs.

### All Command Line Arguments Required:

- OpenWeatherMap API key
- SMTP2GO API key
- Location name (Paris)
- latitude of location (48.8575)
- longitude of location (2.3514)
- timezone (Europe/Paris)
- SMTP2GO email
- Recipient emails

#### Here is a snippet of the script:

```python
parser = argparse.ArgumentParser(description="Aurora forecast Script")
parser.add_argument('smtp_apikey', help='Your SMTP2GO API key')
parser.add_argument('email', help='Your SMTP2GO email')
```

This takes command line inputs, and stores them all in `args`:

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

By assigning each input to variables, the program can use them to fetch and process data from specific locations.

## Containerising the Script Using Docker

### What is Docker?

Docker is a software that lets you package your script and everything it requires to run, into a container which can be moved between other machines easily. Containers ensure the script runs exactly the same way on different machines, preventing any unnecessary issues.

I containerised my script using Docker because an older version of Python existed on my Raspberry Pi, so my script would not run. Docker allowed me to build an image containing all the necessary libraries and the correct Python version, allowing me to run a container containing my script on the Raspberry Pi. This meant that I did not have to install a new version of Python on my Raspberry Pi.

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
RUN apt-get update && apt-get install -y tzdata && pip install requests smtp2go
```

- `apt-get update` is used to fetch the latest version of tzdata, preventing any outdated software from being installed
- Installs `tzdata` inside the container which is one of our libraries that is needed for handing timezones
- Installs `requests` for HTTP get requests and `smtp2go` for dispatching emails 

Previously, I had two `RUN` lines in my `Dockerfile`:

```dockerfile
RUN apt-get install -y tzdata
```

```dockerfile
RUN pip install requests smtp2go
```

However, after some research I found out that it was best practice to have all installations on one line. This reduces the number of layers in the `Dockerfile` which helps to keep the image smaller, making them more efficient when installing on other machines.

```dockerfile
ENTRYPOINT ["python", "northern_lights.py"]
```

- This means that Docker runs `python northern_lights.py`

At first, when building the image, I used `CMD` isntead of `ENTRYPOINT`. However, this meant command line arguments would replace `python northern_lights.py`.

This meant that `arg1` `arg2` `arg3...` would run instead of `python northern_lights.py` `arg1` `arg2` `arg3...`, which lead to the script not running.

`ENTRYPOINT` did not replace `python northern_lights.py` with the command line arguments and instead added them afterwards.

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

#### 1. Save the image as a tar file (on your own machine):

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

#### 4. Load the tar file onto the Raspberry Pi:

```bash
docker load -i aurora.tar
```

#### 5. Run the Container:

```bash
docker run northern_lights <openweatherAPIkey> <smtpAPIkey> Paris 48.8575 2.3514 Europe/Paris <smtp email> <recipient emails>
```

Note that when I ran the container, nothing seemed to happen, and I was not sure if my script was working or not. So, I ran:

```bash
docker run northern_lights arg1 arg2 arg3 >> /home/pi/northern.log 2>&1
```

This meant that the result of running the script e.g. any printed lines as well as errors, were stored in the file `northern.log`. This allowed me to tell whether my script was working or not.

Also, to view the file live:

```bash
tail -f /home/pi/northern.log
```

This is a useful command as it allowed me to see exactly when and what errors occur.

## Scheduling the Script with Cron

Once the Docker image could run reliably on the Raspberry Pi, I wanted the script to run automatically. I wanted to run the scipt at regular intervals without the need for me to intervene.

So, I decided to use `cron`:

### What is Cron?

Cron is a built in Linux tool which allows you to schedule tasks to run automatically at a time or date, specified by the user.

These tasks, which are also known as `cronjob`'s are created in a `crontab` (`cron` table).

Note that `cron` may not be the best choice when using it to schedule a script to run at certain times each day. See below...

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

I wanted my script to run automatically each hour between 21:00 and 23:00 UTC, every day:

```bash
0 21-23 * * * /usr/bin/docker run --rm northern_lights arg1 arg2 arg3
```

Note that in order to find the path to Docker (in my case /usr/bin/docker), type `which docker` into the Raspberry Pi’s terminal.

Also, I used `--rm` in order to automatically delete the container once it has finished running. This ensures that the container is deleted as soon as it is stopped, preventing the system from becoming cluttered with multiple unused containers.

### Downsides of Cron:

The version of `cron` on my Raspberry Pi (Vixie cron) did not support scheduling a script to run in a different timezone other than the native one. As a result, when I wanted to run a script at a certain time in another timezone, I had to manually convert the time in the other timezone to the equivilent time in my native timezone.

In future, a better option would be scheduling inside the container using a Python library such as `time`. This would allow me to run the Python app as a continually running service.