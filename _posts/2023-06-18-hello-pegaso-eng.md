---
layout: post
title:  ¡Hello Pegasus!
subtitle: Watching the aerial surveillance
tags: [python, aws, REST API]
comments: true
image1: /assets/img/hello_pegaso_post/heatmap.png
image2: /assets/img/hello_pegaso_post/trayectory.png
---
### Note
This is a translation from my original post in Spanish.

## 1. Introduction
A few months ago, while driving on the highway, I noticed a sign from the DGT warning of helicopters in the vicinity. Typically, when I embark on long journeys, I use a navigation system like Waze which alerts me of speed cameras or incidents on the road. However, it doesn't notify me if there are DGT [PEGASUS](https://www.xataka.com/movilidad/casi-indetectable-supercamara-mx15-asi-funcionan-helicopteros-pegasus-dgt) helicopters nearby. This made me wonder if there's a way to track them in real-time to avoid potential surprises.

I believe that fixed speed checks are valid and contribute to reducing accidents on roads. However, I think that mobile speed checks are merely revenue-generating tools. In my view, it's an overreach by the authorities in this country. Especially when they deploy a helicopter that has a [flight cost per hour](https://www.autobild.es/noticias/exclusiva-calculan-cuanto-gasta-dgt-cada-ano-helicopteros-pegasus-277887) of around 1500€. But setting political viewpoints aside, let's get into it.

## 2. Investigation
The first thing I needed to find out was the registration numbers of these aircraft. My initial instinct was to use Google. During my initial search, I only found mentions of the number of helicopters DGT has, but no specifics about their registration numbers. But then, browsing through images, I spotted numerous photos taken by aviation enthusiasts. For those unaware, aircraft typically have their registration numbers displayed, which helped me find all the registration numbers.

To ensure they were still operational, I downloaded the [aircraft registry](https://www.seguridadaerea.gob.es/sites/default/files/aeronaves_inscritas.pdf) from the State Agency for Air Safety (AESA) and cross-referenced both the model and registration numbers. I had to eliminate a couple of helicopters, but the rest were operational. With this information, I could start trying to track the helicopters. I also had to consider a [recent accident](https://www.elmundo.es/madrid/2023/03/05/6404e60de4d4d8285b8b45a5.html) to exclude it from the list of active helicopters.

## 3. Tracking the Birds
Modern aviation boasts many sensors and sophisticated systems to ensure trips or tasks are completed as safely as possible. One such device is the [ADS-B](https://es.wikipedia.org/wiki/Sistema_de_Vigilancia_Dependiente_Autom%C3%A1tica).

The ADS-B is a dependent surveillance system that broadcasts GPS location, altitude, speed, direction, and other data to nearby aircraft and ground stations. Anyone with an antenna and a high vantage point can set up their ADS-B receiver and start tracking overhead flights. In fact, services like FlightRadar24 have sensors worldwide and consolidate this data on their website, allowing you to observe global air traffic. Moreover, there are APIs for easy integration using just the aircraft registration number.

After some research, I settled on [ADSB-X](https://es.wikipedia.org/wiki/Sistema_de_Vigilancia_Dependiente_Autom%C3%A1tica) due to its affordable API, which allowed straightforward tracking using the registration numbers.

## 4. Catching the Birds
With the API and registration numbers in hand, I only needed to code a few lines to start monitoring the helicopters. Using AWS Lambda and AWS RDS (PostgreSQL), I created a small app capable of fetching the ADSB data for the helicopters and storing it in a database for later analysis.

The AWS function is quite simple. I just had to add the "requests" library to communicate with the ADBS-X REST API and "psycopg2" to connect to the database and insert new data. Below you can find my function:

```python
try:
    from dotenv import load_dotenv
    load_dotenv()
except ImportError:
    pass


import os
import psycopg2
import requests
from datetime import datetime
from collections import defaultdict
from requests.exceptions import HTTPError


def transform_adsbx_dict(input_dict):
    # Outputs the json response from adsbx to the desired formats
    out_dict = defaultdict(lambda:None)
    out_dict["timestamp"] = int(input_dict["postime"])
    out_dict["registration"] = str(input_dict["reg"])
    out_dict["groundspeed"] = float(input_dict["spd"])
    out_dict["altitude"] = int(input_dict["alt"])
    out_dict["geo_altitude"] = int(input_dict["galt"])
    out_dict["lat"] = float(input_dict["lat"])
    out_dict["lon"] = float(input_dict["lon"])
    out_dict["callsign"] = str(input_dict["call"])
    out_dict["onground"] = bool(int(input_dict["gnd"]))
    return out_dict


def get_aircraft_postion_registration(registration:str):
    """
    GET method for getting an aircraft postion with the registration  
    """
    registration_number = registration.upper()
    url = "https://adsbexchange-com1.p.rapidapi.com/registration/{}/".format(registration)

    headers = {
        'x-rapidapi-host': "adsbexchange-com1.p.rapidapi.com",
        'x-rapidapi-key': os.environ["RAPIDAPIKEY"]
        }

    r = requests.request("GET", url, headers=headers)
    if r.status_code == requests.codes.ok:
        r = r.json()
        r["registration"] = registration
        if r["ac"] is None:
            r["timestamp"] = r["ctime"]
            r["lat"] = None
            r["lon"] = None
            r["altitude"] = None
            return r
        else:
            r = r["ac"] # gets ac, is a list of one element
            r = r[0] # gets the only element, which is a dict
            return transform_adsbx_dict(r)
    else:
        raise HTTPError("Fallo de la API de ADSBX")

HELI1 = "EC-LBD"
HELI2 = "EC-MMF"
HELI3 = "EC-MDO"
HELI4 = "EC-IKS"
HELI5 = "EC-ISB"
HELI6 = "EC-LDF"
HELI7 = "EC-KXU"
HELI8 = "EC-ISZ"
HELI9 = "EC-LAR"
# HELI10 ="EC-JMK" Helicoptero accidentado 
HELI11 ="EC-LGC"
HELI12 ="EC-LGD"
HELI13 ="EC-MHV"

heli_data= [HELI1,HELI2,HELI3,HELI4,HELI5,HELI6,HELI7,HELI8,HELI9,HELI11,HELI12,HELI13]


dbuser = os.environ["DBUSER"]
dbpassword = os.environ["DBPASSWORD"]
dbhost = os.environ["DBHOST"]
dbport = os.environ["DBPORT"]
dbname = os.environ["DBNAME"]

def lambda_handler(event,context):
# Connect to the PostgreSQL server
    conn = psycopg2.connect(
        dbname=dbname,
        user=dbuser,
        password=dbpassword,
        host=dbhost,
        port=dbport
    )



# Create a cursor object
    cur = conn.cursor()

# Loop through the aircraft data
    for aircraft in heli_data:
        print(aircraft)
        try:
            req = get_aircraft_postion_registration(aircraft)
        except HTTPError:
            return {
              "statusCode": 500,
              "errorType": "ADSBX connection failed",
              "errorMessage":"ADSBX rapidapi does not work"}

        req["timestamp"] =  datetime.fromtimestamp(req['timestamp'] / 1000.0)

        # Insert the data into the database
        cur.execute(
            """
            INSERT INTO AircraftInformation (registration, timestamp, lat, lon, altitude)
            VALUES (%s, %s, %s, %s, %s)
            """,
            (req['registration'], req['timestamp'], req['lat'], req['lon'], req['altitude'])
        )

    conn.commit()
    cur.close()
    conn.close()
    
```
## 5. Monitoring the Aircraft
With everything set up, it was time to analyze the helicopters. Over three weeks, every 10 minutes from 8:00 AM to 7:00 PM, I collected information on these helicopters and created a small dashboard using Streamlit. If you're interested in viewing the source code for the dashboard, you can find it [here](https://github.com/jaimebw/hello_pegaso).

<iframe
  src = "https://jaimebw-hello-pegaso.streamlit.app/?embed=true"
  height="450"
  style="width:100%;border:none;"
></iframe>

Out of the 13 helicopters, I was only able to monitor one. I suspect this is because the rest either don't fly or don't have their ADSB turned on. Still, the one that was airborne can be spotted at various times. To see its preferred flight paths, I generated a density map and segmented it using the [h3](https://h3geo.org/) library from Uber. The dashboard displays a map showing the sectors most frequently traversed by the helicopters. Bad news if you enjoy driving on the A-4 or live around the Boadilla/Brunete area.
![Heli heatmap]({{page.image1 | relative_url }})

You can also filter by day to see the helicopters' daily flight patterns.
![trajectory]({{page.image2 | relative_url }})

## 6. Good News and Future Projects
This experiment proves that it's possible to track these aircraft in real-time. It also shows that most of them don't fly, and the ones that do, fly infrequently (thankfully, I doubt they issue enough fines to justify the 1500€/h cost per helicopter).

Furthermore, these pests can only "bite" you if they maintain a maximum separation of 1000m from your vehicle. Given the latitude, longitude, and altitude of the aircraft, it would be straightforward to calculate the distance between a vehicle equipped with GPS and the aircraft, so integrating this into an app would be pretty simple.

