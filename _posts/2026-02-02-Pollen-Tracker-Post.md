---
layout: post
title: "Pollen Tracker"
date: 2026-02-02
---
## Introduction

### Project Goals

- Fetch real-time pollen data from OpenMeteo
- Fetch user data
- Containerise the scripts using Docker
- Schedule the script to run every hour using cron (a scheduler) on a Raspberry Pi.
- Store all pollen data in a postgres database on the Raspberry Pi
- Write unittests to mock API calls and test methods

### Technology used

- Python
- Libraries:
    - `requests` - HTTP get requests
    - `datetime` - create timestampms for when data was collected
    - `argparse` - command line arguments
    - `psycopg2` - store data in postgres database
- Docker
- PostgreSQL
- `cron`

## Setting Up Database

To track pollen data, I used a PostgreSQL database. The reason I chose PostgreSQL was due to being able to pull an image onto the Raspberry Pi through docker. 

The three tables in our database:

1. `pollen_type`
2. `pollen_data`
3. `user_data`

### Setting up PostgreSQL

Ensure docker is installed on your machine. Check using:

```bash
docker --version
```

Pulling Postgres Docker image:

```bash
docker pull postgres
```

Create and run container:

```bash
docker run --name pollen-postgres \
    -e POSTGRES_DB=pollen_db \
    -e POSTGRES_USER=username \
    -e POSTGRES_PASSWORD=password \
    -p 5432:5432 \
    -d postgres
```

- Creates container called `pollen-postgres`
- It sets up:
    - a database called pollen_db
    - a user called username and its password
    - maps ports between machine and container
- `-d` runs the container in detached mode meaning it runs in the background

### Creating the `pollen_type` table

The `pollen_type` table is a reference table for all pollen types that need to be tracked.

```sql
CREATE TABLE pollen_type (
    pollen_type_id SERIAL PRIMARY KEY,
    name TEXT NOT NULL UNIQUE
);
```

- `pollen_type_id` - an automatically assigned unique ID
- `name` - text based name of pollen type. This must be unique, to prevent any duplicate pollen types

Next, insert pollen types that are going to be tracked:

```sql
INSERT INTO pollen_type (name) VALUES
('grass'),
('birch'),
('alder'),
('ragweed'),
('mugwort'),
('olive');
```

Note this may change depending on user preferences.

### Creating the `pollen_data` table

The `pollen_data` table stores the pollen counts.

```sql
CREATE TABLE pollen_data (
    pollen_data_id SERIAL PRIMARY KEY,
    pollen_type_id INT NOT NULL
        REFERENCES pollen_type(pollen_type_id),
    date TIMESTAMP NOT NULL,
    pollen_count DECIMAL(10,2) NOT NULL
);
```

- `pollen_data_id` - again, a unique ID for each entry
- `pollen_type_id` - links each record to a pollen type in the `pollen_type` table that was created earlier
- `date`- timestamp for when measurement was taken
- `pollen_count` - numeric pollen level with accuracy of 2dp

By having a separate table pollen_type, I am able to easily add more pollen types if needed.

## Fetching Pollen Types

`get_pollen_type_dict` connects to a PostgreSQL database and fetches a list of pollen types, returning them all as a dictionary. This dictionary will be used as reference in other functions as shown later.

### Connect to database

```python
conn = psycopg2.connect(
    host="localhost",
    port=5432,
    database=db_name,
    user=db_user,
    password=db_pass
)
```

- Creates a connection to the database using same host, port and credentials used when creating the PostgreSQL container earlier
- Note that db_name etc are parsed as command line arguments

### Execute SQL

```python
cur = conn.cursor()
sql = """
SELECT pollen_type_id, name FROM pollen_type
"""
cur.execute(sql)
```

- Creates a cursor `cur` which allows SQL to be executed
- Collects `pollen_type_id` and the readable `name` from `pollen_type`

### Creating reference dictionary

```python
# key:value
pollen_dict = {}
rows = cur.fetchall()

for pollen_type_id, name in rows:
    pollen_dict[name.lower()] = pollen_type_id
```

`cur.fetchall()` returns all query results as a list of tuples:

```python
[
    (1, 'grass'),
    (2, 'birch'),
    (3, 'alder')
]
```

I then iterated over each row in the returned list, converted all pollen names to lowercase for consistoncy and added them to `pollen_dict`.

Example dictionary:

```python
{
    "grass": 1, # Key:Value
    "birch": 2,
    "alder": 3
}
```

## Fetching JSON Data

`get_pollen_data` fetches current pollen levels for a specific location based on coordinates of the location and the pollen data the user requests. It returns the pollen count for the current hour.

### Construct API URL

```python
    url = (
        "https://air-quality-api.open-meteo.com/v1/air-quality?"
        f"latitude={lat}&longitude={lon}&"
        "hourly=grass_pollen,birch_pollen,alder_pollen,ragweed_pollen,mugwort_pollen,olive_pollen&"
        "timezone=Europe/London&forecast_days=1"
    )
```

- Latitude and longitude tell the API the coordinates of the location I am interested in
- I specified which pollen types that are needed
- A days worth of hourly data is requested

### API call and response

```python
response = requests.get(url)
data = response.json()
```

- This fetches a JSON response containing pollen data from a URL as shown below
- The JSON data is converted to a Python dictionary using `.json()` which can be manipulated easily

Access specific pollen data:

```python
hourly = data['hourly']
times = hourly['time']
```

- The API returns lots of data so `data['hourly']` is used to extract hourly pollen counts for that day
- `hourly['time']` provides the list of timestamps for each hour of the day

### Recording time of collection

```python
current_time = datetime.now().replace(minute=0, second=0, microsecond=0)
current_iso = current_time.isoformat(timespec="minutes")
```

- Rounds the current time down to the nearest hour
- Converts the timestamp into an ISO-8601 string which is the format the API uses to store timestamps

### Find correct data index

```python
try:
    index = times.index(current_iso)
except ValueError:
    return []
```

- This tries to find a time in the JSON that matches the current time
- If it's missing e.g. due to the API not updating, it returns an empty array, preventing the program from crashing

### Extracting pollen values

```python
for pollen_type in pollen_dict.keys():
    pollen_count = hourly[f"{pollen_type}_pollen"][index]
    pollen_data.append((pollen_type, current_time, pollen_count))

return pollen_data
```

- `pollen_dict.keys()` returns the corresponding keys for each row in `pollen_dict`. E.g. `grass`
- Here, the correct pollen string is formed e.g. `"grass_pollen"`
- The pollen count is collected for the current hour
- Pollen type, time and pollen count are stored as a tuple and returned as an array

## Store Pollen Data

`store_data` takes the collected data from `pollen_data` list and stores it in the `pollen_data` table in the database.

### SQL queries

After connecting to the database as shown earlier, I created two SQL queries:

```python
sql_check = """
    SELECT 1 FROM pollen_data
    WHERE pollen_type_id = %s AND date = %s
"""
```

- Checks if a record already exists for:
    - a specific pollen type
    - a specific timestamp
- Returns a row only if data is already stored

Note that `%s` is a placeholder for variables within the program.

```python
sql_insert = """
    INSERT INTO pollen_data
    (pollen_type_id, date, pollen_count)
    VALUES (%s, %s, %s)
"""
```

- Inserts a new row containing `pollen_count`, `pollen_type_id` and `date`

### Store data

```python
for pollen_type, time, pollen_count in pollen_data:
    pollen_type_id = pollen_dict[pollen_type]

    cur.execute(sql_check, [pollen_type_id, time])
    if cur.fetchone() is None:
        cur.execute(sql_insert, [pollen_type_id, time, pollen_count])
```

- Iterate through list `pollen_data` and looks up corresponding `pollen_type_id` using `pollen_dict`
- The first `cur.execute` line runs the `sql_check` query:
    - If no matching row exists, it inserts new data
    - If matching row exists, it skips insertion, preventing duplicate data

## Collecting User Data

The `user_input` script will collect daily data and store it in another table `user_data` in the database. This data, as well as the pollen data will be used to see if there is a correleation between certain pollen types and severity of user symptoms. 

### Creating `user_data` table

The `user_data` table stores data about the user, including symptoms and the severity of symptoms.

```sql
CREATE TABLE user_data (
    user_data_id SERIAL PRIMARY KEY,
    date DATE NOT NULL,
    antihistamine_taken BOOLEAN NOT NULL,
    allergy_problems BOOLEAN NOT NULL,
    ill BOOLEAN NOT NULL,
    symptom_severity INT NOT NULL,
    CONSTRAINT check_symptom CHECK (symptom_severity BETWEEN 0 AND 10)
);
```

- `antihistamine_taken` - boolean value, whether or not the user has taken allergy medication
- `ill` - whether the user is ill, as this may cause some similar allergy symptoms
- `symptom_severity` - scale between 0 and 10 of how severe the user reacted that day

### SQL queries

Again, once establishing a connecting with the database as shown earlier, I created two SQL queries:

```python
sql_check = """
    SELECT 1 FROM user_data
    WHERE date = %s
"""
```

This checks if a row already exists for todays date preventing duplicate data.

```python
sql_insert = """
INSERT INTO user_data (antihistamine_taken, allergy_problems, date, symptom_severity, ill) VALUES (%s, %s, %s, %s, %s)
"""
```

This query inserts a new row into user_data table.

### Inserting data

```python
cur.execute(sql_check, [date])
if cur.fetchone() is None:
    cur.execute(sql_insert, (antihistamine, allergy, date, severity, ill))
    print("Data Stored")
else:
    print("Data already collected")
```

- This runs the `sql_check` query
- If `fetchone()` returns `None` (no matching row with same date exists), data is inserted
- If `fetchone()` returns a record, a message is output telling the user they have already submitted data for the day