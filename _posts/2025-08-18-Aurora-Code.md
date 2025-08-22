---
layout: post
title: "Aurora Forecast Tool - Code (Part 1)"
date: 2025-08-18
---
In this post, we will briefly cover how each part of the code functions and its purpose in providing us data on the probability of an Aurora.- Send an email containing all fetched data
<br><br>
### **Fetching Real-Time Aurora Data**

- This fetches Aurora data in the format of JSON from the URL.
- JSON data is converted to a Python dictionary to be manipulated - `.json()`
- All data is returned and is accessible elsewhere in the script.
- Instead of specifying the url, I provided an argument which allowed me to re-use this function for fetching any other data needed.

```python
def fetch_data(url):
    jsontext = requests.get(url)
    data = jsontext.json()
    return data
```
<br>
### **Fetching Nautical Dusk Data**

This uses the Sunrise-Sunset API to get the time when nautical twilight ends - the point when the sky becomes dark enough for Aurora viewing. This is important as an Aurora is much easier to view in low light conditions.

This API response as well as the others, are returned in JSON format, which Python handles as nested dictionaries and lists, making it easy to extract data.

```python
url = f"https://api.sunrise-sunset.org/json?lat={lat}&lng={long}&date=today&formatted=0"
    jsontext = requests.get(url)
    data = jsontext.json()
    raw_time = data['results']['nautical_twilight_end']
```
<br>
Before returning the time, we convert the `raw_time` into UTC to allow us to convert the time to a different timezone as shown later.

```python
utc_time = datetime.fromisoformat(raw_time).strftime("%Y-%m-%dT%H:%M:%SZ")
```
<br>
### **Fetching Cloud Cover Data**

This uses the OpenWeatherMap API to collect data on cloud cover forecast for the next 3 hours.

Cloud cover is another key factor to consider before looking outside. With a high percentage of cloud cover (e.g. >80%), an Aurora may not be visible despite a high percentage value at that time, which is why we include cloud cover data in our emails.

Here, the forecast time and the cloud cover data are collected as percentages:

```python
next_forecast = data['list'][0]
timestamp = next_forecast['dt']
cloud_cover = next_forecast['clouds']['all']
```
<br>
`timestamp` was also converted to UTC as shown below so it can be easily converted to another timezone (specified by the user):

```python
utc_time = datetime.fromtimestamp(timestamp, tz=timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")
```
<br>
### **NOAA's Aurora Forecast Coordinate System**

When I first began working with the NOAA Aurora 30-Minute Forecast data, I assumed that the coordinate format was the same as most weather API’s - using decimal degrees with negative values for western longitudes and explicit negative/positive signs for latitude and longitude. I used this standard format in order to extract the probability of an Aurora at a specified location.

However, my initial attempts at extracting Aurora probabilities produced unexpected results. When I compared my percentage probability to the Aurora 30 minute forecast clip, the probabilities didn’t make sense for the places I was interested in.

By carefully observing the first and last coordinate sets in NOAA’s latest JSON file, I noticed:

- The longitudes ranged from 0 to 359, never negative
- The latitudes ranged from -90 to +90, as expected

This pattern revealed that the NOAA was using a 0-359 longitude system instead of the standard -180 to +180.

From this, I realised:

- All negative longitudes, (west of Greenwich) were converted by adding 360
- E.g. -115 longitude became 245 (360 - 115)
- Latitudes remaind standard so no change was needed
- Also, all coordinate values were integers, so I had to round all parsed coordinates using Python’s built in `round()`

<br>
### **Extracting Probability Data**

Here, I looped through NOAA’s `coordinates` data to find a match on longitude and latitude that’s provided by the user.

If there is a match, it fetches the associated probability, located at index 2 - `[long, lat, probability]`.

I was able to loop through NOAA’s data easily, as the original JSON had been converted to a Python dictionary through the `fetch_data` function.

```python
for coordinate in data["coordinates"]:
    if coordinate[0] == long and coordinate[1] == lat:
        probability = coordinate[2]
```
