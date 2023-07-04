---
layout: post
title:  ¡Hola Pegasus!
subtitle: Vigilando a la vigilancia aérea
tags: [python, aws, REST API]
comments: true
image1: /assets/img/hello_pegaso_post/heatmap.png
image2: /assets/img/hello_pegaso_post/trayectory.png
---
## 1. Introducción
Hace unos meses, conduciendo por la autopista, vi un cartel de la DGT que advertía de la presencia de helicópteros en la zona. Por lo general, cuando voy a realizar un viaje largo, utilizo un navegador como Waze que me informa de radares o incidentes en la carretera. Sin embargo, no me avisa si hay helicópteros [PEGASUS](https://www.xataka.com/movilidad/casi-indetectable-supercamara-mx15-asi-funcionan-helicopteros-pegasus-dgt) de la DGT cerca de mí. Eso me hizo preguntarme si habría alguna forma de localizarlos en tiempo real y así poder evitar algún susto.

Creo que los controles de velocidad fijos son legítimos y ayudan a reducir la siniestralidad en las carreteras. Sin embargo, considero que los controles de velocidad móviles solo tienen fines recaudatorios y, en mi opinión, son un abuso por parte de las autoridades de este país. Sobretodo cuando usan un helicoptero que tiene un [coste de vuelo por hora](https://www.autobild.es/noticias/exclusiva-calculan-cuanto-gasta-dgt-cada-ano-helicopteros-pegasus-277887) de unos 1500€.  Pero bueno, manifiestos políticos aparte, vayamos al lio.


## 2. Investigando
Lo primero que tenía que averiguar era la matrícula de estas aeronaves, y mi primer instinto fue usar Google. En una primera búsqueda, no encontré nada más que la mención del número de helicópteros que tiene la DGT, pero ningún dato sobre sus matrículas. Sin embargo, me di cuenta de que, en imágenes, se podían ver muchas fotos que los famosos spotters de aviación habían hecho. Por si no lo sabéis, generalmente las aeronaves suelen tener su matrícula pintada, y gracias a eso pude encontrar todas las matrículas de las aeronaves.

Para asegurarme de que seguían volando, me descargué el [registro de aeronaves](https://www.seguridadaerea.gob.es/sites/default/files/aeronaves_inscritas.pdf) de la Agencia Estatal de Seguridad Aérea (AESA) y me aseguré que tanto el modelo y la matrícula aparecían. Tuve que descartar un par de helicópteros, pero el resto seguía en regla. Con esto ya podía empezar a intentar localizar las aeronaves. Otra cosa que tuve que tener en cuenta fue el [accidente que hubo hace unos meses](https://www.elmundo.es/madrid/2023/03/05/6404e60de4d4d8285b8b45a5.html) para no incluirlo entre los helicopteros en activo. 

## 3. Localizando las moscas
La aviación moderna se caracteriza por tener muchos sensores y sistemas sofisticados para poder realizar los viajes o tareas de la manera más segura posible. Uno de estos dispositivos es el [ADS-B](https://es.wikipedia.org/wiki/Sistema_de_Vigilancia_Dependiente_Autom%C3%A1tica).

El ADS-B es un sistema de vigilancia dependiente que se encarga de transmitir información de localización GPS, altitud, velocidad, rumbo y otros datos a las aeronaves del alrededor y a tierra. Cualquier persona con una antena y un lugar alto puede crear su propio receptor ADS-B y empezar a espiar a los aviones que les sobrevuelan. Es más, existen multitud de servicios como FlightRadar24 que tienen multitud de sensores por todo el mundo y centralizan todos esos datos en su web para que puedas ver el tráfico aéreo por encima del mundo; y, además, existen APIs que permiten integrar todo esto de manera sencilla con solo la matrícula de la aeronave.

Investigando un poco, me decidí por [ADSB-X](https://es.wikipedia.org/wiki/Sistema_de_Vigilancia_Dependiente_Autom%C3%A1tica)dado que su API era relativamente barata y me permitía realizar todo esto de manera sencilla con las matrículas.

## 4. Cazando las moscas
Con la API y las matrículas solo tenía que picar unas cuantas líneas de código para empezar a monitorizar las aeronaves. Usando AWS Lambda y AWS RDS (Postgress SQL) cree una pequeña aplicación capaz de obtener los datos ADSB de los helicópteros y añadirlos a una base de datos para poder hacer algún análisis posteriormente.

La función de AWS es muy sencilla, solo tenemos que añadir la biblioteca de “requests” para comunicarme con la REST API de ADBS-X y “psycopg2” para conectarme a la base de datos e insertar los datos nuevos. Aquí abajo podéis encontrar mi función:
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

## 5. Monitorizando las aeronaves
Con todo esto, ya era hora de realizar un análisis de los helicópteros. Durante 3 semanas, cada 10 minutos de 8:00 a 19:00, recopilé la información de estos helicópteros e hice un pequeño Dashboard con Streamlit. Si os interesa ver el código fuente lo teneís [aquí](https://github.com/jaimebw/hello_pegaso).

<iframe
  src = "https://jaimebw-hello-pegaso.streamlit.app/?embed=true"
  height="450"
  style="width:100%;border:none;"
></iframe>


De los 13 helicópteros, solo pude monitorizar uno. Creo que esto debe ser porque el resto no vuela o no tiene el ADSB encendido, aun así, el que volaba se puede observar en varios momentos. Para ver por dónde le gustaba volar más, hice un mapa de densidad y sectoricé usando la librería [h3](https://h3geo.org/) de Uber. El dashboard muestra un mapa con los sectores que más pasan los helicópteros. Tengo malas noticias para ti si te gusta conducir por la A-4, y si vives por la zona de Boadilla/Brunete. 
![Heli heatmp]({{page.image1 | relative_url }})

También se puede filtar por día para ver las trayectorias de los helicopteros por día.
![trayectory]({{page.image2 | relative_url }})
## 6. Las buenas noticias y futuro proyectos
Este experimento demuestra que se pueden localizar estas aeronaves en tiempo real. También demuestra que la mayoría no vuelan, y la que vuela, vuela poco(y menos mal, dudo que pongan suficiente multas para poder justificar los 1500€/h de cada helicóptero)

Además, estos mosquitos solo pueden picarte siempre y cuando tenga una separación de 1000m como máximo a tu vehículo. Con la latitud, longitud y la altitud de la aeronave sería trivial calcular la distancia entre un vehículo con GPS y la aeronave, por lo que implementar esto en una app sería bastante sencillo. 
