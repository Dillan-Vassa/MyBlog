---
layout: post
title: "Aurora Forecast Tool - Containerisation (Part 5)"
data: 2025-08-20
---
### **Containerising the Script Using Docker**

In this post, we will cover what Docker is, and how to containerise our script, allowing us to transfer the script between different machines with ease. I will also cover how to transfer the container to the Raspberry Pi, allowing us to run the script on the Raspberry Pi.
<br><br>
#### **What is Docker**

Docker is a software that lets you package your script and everything it needs, into a portable container. Containers ensure your script runs exactly the same way on different machines, preventing any issues when running a script on different machines.

I created a `Dockerfile` inside my Northern Lights folder which contained my script as shown below:

#### **Dockerfile:**

```dockerfile
FROM python:3.11-slim
```

- This sets the base image for our container, using Python's version 3.11-slim
- `slim` is used to exclude unnecessary tools and libraries, keeping our image small and lightweight.

<br>
```dockerfile
WORKDIR /Aurora
```

- Sets the working directory inside the container to `/Aurora`
- When the container starts, this is the directory the script runs from

<br>
```dockerfile
COPY northern_lights.py .
```

- This copies our Python script (`northern_lights.py`) from your local machine to the container’s working directory (`/Aurora`)
- The `.` means copy the script into the current working directory in this case `/Aurora`

<br>
```dockerfile
RUN apt-get install -y tzdata
```

- Installs `tzdata` inside the container which is one of our libraries needed for handing timezones
- The `-y` ensures the installation process does not stop and wait for you to manually confirm "y/n"

<br>
```dockerfile
RUN pip install --no-cache-dir requests smtp2go
```

- Installs the `requests` library for HTTP get requests and `smtp2go` for sending emails
- The `--no-cache-dir` ensures that download cache is not stored, keeping the image smaller

<br>
```dockerfile
ENTRYPOINT ["python", "northern_lights.py"]
```

- This tells Docker to run `python northern_lights.py`
- I used `ENTRYPOINT` instead of `CMD` so that when I parsed argumets at command line, the arguments did not replace `python northern_lights.py`, which would cause an error

<br>
### **Build and Transfer Docker Image to Raspberry Pi**
<br>
#### **Building the Image:**

At first, when building the docker image, I used:

```bash
docker build -t northern_lights .
```

However, after transfering this image to the Pi, I came across an error telling me that the container was built for the wrong CPU architecture.

So, I used:

```bash
docker buildx build --platform linux/arm64 -t northern_lights --load .
```

This built an image that was compatible for the Raspberry Pi’s arm64 CPU architecture, allowing me to run the container smoothly without any issues.
<br><br>
#### **Transfering the Image:**
<br>
**1. Save the image as a .tar file (on your machine):**

```bash
docker save -o aurora.tar northern_lights:latest
```
<br>
**2. Copy the image to the Pi:**

```bash
scp aurora.tar pi@<IP address of Pi>:/home/pi/
```
<br>
**3. SSH into Raspberry Pi from your machine:**

```bash
ssh pi@<IP address of Pi>
```
<br>
**4. Load the image onto the Pi:**

```bash
docker load -i aurora.tar
```
<br>
**5. Run the container:**

```bash
docker run --rm northern_lights arg1 arg2 arg3
```
