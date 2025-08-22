---
layout: post
title: "Aurora Forecast Tool - Introduction"
data: 2025-08-17
---
I created a Python script that fetches real-time Aurora forecast data, put it in a Docker container, and scheduled it to run automatically on a Raspberry Pi using cron. It sends an email between certain hours every day, containing information on the likelihood of an Aurora.

In this post, I will introduce the project goals, technology used and the Aurora data source model.

### **Project Goals**
- Fetch real-time Aurora data from NOAA (National Oceanic and Atmospheric Administration) - an American scientific and regulatory agency
- Fetch forecast data on cloud cover from OpenWeatherMap - a service that provides global weather data
- Fetch data on time of dusk from Sunrise Sunset - a service that provides global data on sunset and sunrise times
- Extract the probability for the selected coordinates of a location
- Output the percentage liklihood of an Aurora
- Send an email containing all fetched data
- Containerise the script using Docker
- Schedule the script to run every 20:00 to 22:00 BST using Cron (scheduler) on a Raspberry Pi (a single-board computer)

### **Technology Used**

- Python
- Libraries: `requests`, `datetime`, `argparse`, `zoneinfo`, `smtp2goClient`
- Optional libraries: `pytest`, `json`
- Docker (containerisation)
- Cron (scheduling)

### **Aurora Data Source and Model**

NOAA’s 30-Minute Aurora Forecast is powered by the OVATION (Oval, Variation, Assessment, Tracking, Intensity, and Online Nowcasting) Prime model, which uses real-time data from satellites, including solar wind speed and the interplanetary magnetic field, to estimate how much charged particle activity (mainly electrons and protons) is hitting Earth’s atmosphere.

When these particles collide with gases in the upper atmosphere e.g. oxygen and nitrogen, they excite the atoms, causing them to release light. This process creates the colourful glow we see as the Aurora (northern lights).

The OVATION Prime model uses this data to calculate the probability (as a percentage) of visible Auroral activity at different locations around the world.
