```python
%pip install dotenv
%pip install boto3
```

    Requirement already satisfied: dotenv in /opt/conda/lib/python3.11/site-packages (0.9.9)
    Requirement already satisfied: python-dotenv in /opt/conda/lib/python3.11/site-packages (from dotenv) (1.2.4)
    Note: you may need to restart the kernel to use updated packages.
    Requirement already satisfied: boto3 in /opt/conda/lib/python3.11/site-packages (1.44.9)
    Requirement already satisfied: botocore<1.45.0,>=1.44.9 in /opt/conda/lib/python3.11/site-packages (from boto3) (1.44.9)
    Requirement already satisfied: jmespath<2.0.0,>=0.7.1 in /opt/conda/lib/python3.11/site-packages (from boto3) (1.1.0)
    Requirement already satisfied: s3transfer<0.20.0,>=0.19.0 in /opt/conda/lib/python3.11/site-packages (from boto3) (0.19.3)
    Requirement already satisfied: python-dateutil<3.0.0,>=2.1 in /opt/conda/lib/python3.11/site-packages (from botocore<1.45.0,>=1.44.9->boto3) (2.8.2)
    Requirement already satisfied: urllib3!=2.2.0,<3,>=1.25.4 in /opt/conda/lib/python3.11/site-packages (from botocore<1.45.0,>=1.44.9->boto3) (2.0.7)
    Requirement already satisfied: six>=1.5 in /opt/conda/lib/python3.11/site-packages (from python-dateutil<3.0.0,>=2.1->botocore<1.45.0,>=1.44.9->boto3) (1.16.0)
    Note: you may need to restart the kernel to use updated packages.



```python
import os
from dotenv import load_dotenv

# Carga las variables del fichero .env en el entorno del sistema
load_dotenv()

# Recupera las variables con os.getenv()
api_key = os.getenv("AEMET_API_KEY")
bucket_name = os.getenv("S3_BUCKET_NAME")
aws_access_key_id = os.getenv("AWS_ACCESS_KEY_ID")
aws_secret_access_key = os.getenv("AWS_SECRET_ACCESS_KEY")
aws_session_token = os.getenv("AWS_SESSION_TOKEN")
aws_default_region = os.getenv("AWS_DEFAULT_REGION")

if not api_key or not bucket_name:
    raise ValueError("Faltan variables de entorno críticas en el archivo .env")
```


```python
print(api_key)
print(bucket_name)
print(aws_access_key_id)
```

    eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOi-----lzcyI6IkFFTUVUIiwiaWF0IjoxNzk0Mzg3Nzk5LCJ1c2VySWQiOiI5YjEwYzVkZC1hMjZkLTQ0YWEtYmI3My0wNmQ3NjJmYmE3ZDIiLCJyb2xlIjoiIn0.H0YgbCa9gQcKBp_kNSozOApPnuvQD6FJ6QQNIVFYXOA
    datalake-turismo-sgt
    ASIAY2LUUB67RFPGLMQ7



```python
from pathlib import Path
import requests

url_csv = "https://vgonzalez165.github.io/apuntes_bda/practicas/ut03/data/playas.csv"
ruta_destino = Path("./data/playas.csv")

# Crea la carpeta ./data si no existe
ruta_destino.parent.mkdir(parents=True, exist_ok=True)

response = requests.get(url_csv, timeout=15)
response.raise_for_status()  # Lanza una excepción si hay error HTTP (404, 500...)

# Guardamos el contenido binario directamente en disco
with open(ruta_destino, "wb") as f:
    f.write(response.content)

url_csv = "https://vgonzalez165.github.io/apuntes_bda/practicas/ut03/data/Playas_codigos.csv"
ruta_destino = Path("./data/playas_codigos.csv")
response = requests.get(url_csv, timeout=15)
response.raise_for_status()  # Lanza una excepción si hay error HTTP (404, 500...)

# Guardamos el contenido binario directamente en disco
with open(ruta_destino, "wb") as f:
    f.write(response.content)
```


```python
import pandas as pd

# Lectura del catálogo MITECO (separado por comas, UTF-8)
df_miteco = pd.read_csv("./data/playas.csv", sep=",", encoding="utf-8", dtype={"Código_IN": str})
#print(df_miteco)
# Aseguramos que el código INE tenga siempre 5 dígitos rellenando con ceros a la izquierda
df_miteco["Código_IN"] = df_miteco["Código_IN"].str.zfill(5)
#print(df_miteco["Código_IN"])
df_codigos_playas = pd.read_csv("./data/playas_codigos.csv", sep=";", encoding="latin1", dtype={"ID_PLAYA": str})
print(df_codigos_playas["ID_PLAYA"])
```

    0      0301101
    1      0301401
    2      0301408
    3      0301410
    4      0301808
            ...   
    586    4807701
    587    4808502
    588    5100104
    589    5100108
    590    5200103
    Name: ID_PLAYA, Length: 591, dtype: object



```python
import json
import time
import requests

id_playa = "4807701"
url_endpoint = f"https://opendata.aemet.es/opendata/api/prediccion/especifica/playa/{id_playa}"
headers = {"api_key": api_key, "Accept": "application/json"}

# PASO 1: Petición inicial para obtener la URL de los datos
resp_control = requests.get(url_endpoint, headers=headers, timeout=10)
resp_control.raise_for_status()
info_control = resp_control.json()

# Verificamos el estado interno que devuelve AEMET (200 = éxito, 404 = playa sin datos)
if info_control.get("estado") == 200:
    url_datos_finales = info_control["datos"]

    # PASO 2: Descarga de los datos reales desde la URL temporal
    resp_datos = requests.get(url_datos_finales, timeout=10)
    resp_datos.raise_for_status()

    # Forzamos la codificación correcta antes de parsear el JSON para conservar tildes y eñes
    resp_datos.encoding = "ISO-8859-15"
    datos_prediccion = resp_datos.json()

    # OPCIÓN A: Guardar en un fichero local
    with open(f"./data/pred_{id_playa}.json", "w", encoding="utf-8") as f:
        json.dump(datos_prediccion, f, ensure_ascii=False, indent=2)

# Pausa para no saturar la cuota de la API (evita error HTTP 429)
time.sleep(1.5)
```


```python
import boto3

try: 
    s3 = boto3.client("s3")
    s3.create_bucket(
    Bucket=bucket_name,
)
    buckets = s3.list_buckets()  
    print(buckets)

except Exception as e:
    print(f"Error-> {e}")
    
 
```

    {'ResponseMetadata': {'RequestId': 'QR69N8YA50K873G6', 'HostId': 'wUByj8RxEZwmBUZijsdSyTB+74xOZ438hFDEzq7l+IfBt9cITtcgCnVxYpshtyLvLZlhPTyG8ec=', 'HTTPStatusCode': 200, 'HTTPHeaders': {'x-amz-id-2': 'wUByj8RxEZwmBUZijsdSyTB+74xOZ438hFDEzq7l+IfBt9cITtcgCnVxYpshtyLvLZlhPTyG8ec=', 'x-amz-request-id': 'QR69N8YA50K873G6', 'server': 'AmazonS3', 'date': 'Wed, 25 Nov 2026 19:34:13 GMT', 'content-type': 'application/xml', 'transfer-encoding': 'chunked'}, 'RetryAttempts': 0}, 'Buckets': [{'Name': 'buck-bda-606371844031-us-east-1-an', 'CreationDate': datetime.datetime(2026, 11, 10, 9, 8, 43, tzinfo=tzlocal()), 'BucketArn': 'arn:aws:s3:::buck-bda-606371844031-us-east-1-an'}, {'Name': 'datalake-turismo-sgt', 'CreationDate': datetime.datetime(2026, 11, 25, 17, 20, 27, tzinfo=tzlocal()), 'BucketArn': 'arn:aws:s3:::datalake-turismo-sgt'}], 'Owner': {'ID': '84f26677f66758494d1057c0276f3b261d8829a35683529ed32b86aa65e9e515'}}



```python
from datetime import datetime, timezone
# Generamos marca de tiempo actual en UTC (ISO 8601) y variables de particionado
ahora = datetime.now(timezone.utc)
timestamp_iso = ahora.strftime("%Y-%m-%dT%H:%M:%SZ")
ruta_particion = f"year={ahora.year}/month={ahora.month:02d}/day={ahora.day:02d}"

# -------------------------------------------------------------------------
# CASO A: Subir un fichero que ya existe en tu disco local (upload_file)
# -------------------------------------------------------------------------
s3.upload_file(
    Filename="./data/playas.csv",
    Bucket=bucket_name,
    Key="bronce/catalogos/miteco_playas/v1/playas.csv",
    ExtraArgs={"Metadata": {"source": "MITECO_OpenData", "ingestion_timestamp": timestamp_iso}},
)
s3.upload_file(
    Filename="./data/playas_codigos.csv",
    Bucket=bucket_name,
    Key="bronce/catalogos/aemet_codigos/v1/playas_codigos.csv",
    ExtraArgs={"Metadata": {"source": "AEMET_Catalogo", "ingestion_timestamp": timestamp_iso}},
)

# -------------------------------------------------------------------------
# CASO B: Subir un JSON directamente desde una variable en RAM (put_object)
# -------------------------------------------------------------------------
# Convertimos la lista/diccionario de Python a texto JSON en UTF-8 (bytes)
json_en_bytes = json.dumps(datos_prediccion, ensure_ascii=False, indent=2).encode("utf-8")
s3_key_json = f"bronce/meteorologia/aemet_prediccion_playas/{ruta_particion}/pred_playa_{id_playa}_{ahora.strftime('%H%M%S')}.json"

s3.put_object(
    Bucket=bucket_name,
    Key=s3_key_json,
    Body=json_en_bytes,
    ContentType="application/json",
    Metadata={"source": "AEMET_API", "ingestion_timestamp": timestamp_iso},
)

```




    {'ResponseMetadata': {'RequestId': 'MZ47VCC0358KE3XG',
      'HostId': 'ZZg1TSNcMGAqWb0goKqS0B2D6r9eXiikL/lG25+JzJtyM9b3fzGnEO2vIusz2XvSN9ytsjg7f8Bxg8sdc3uNZxXos69OwKsy',
      'HTTPStatusCode': 200,
      'HTTPHeaders': {'x-amz-id-2': 'ZZg1TSNcMGAqWb0goKqS0B2D6r9eXiikL/lG25+JzJtyM9b3fzGnEO2vIusz2XvSN9ytsjg7f8Bxg8sdc3uNZxXos69OwKsy',
       'x-amz-request-id': 'MZ47VCC0358KE3XG',
       'server': 'AmazonS3',
       'date': 'Wed, 25 Nov 2026 19:34:15 GMT',
       'x-amz-server-side-encryption': 'AES256',
       'etag': '"64300b59a72522358f0a5b4a0811c771"',
       'x-amz-checksum-crc32': 'bQkdGw==',
       'x-amz-checksum-type': 'FULL_OBJECT',
       'content-length': '0'},
      'RetryAttempts': 0},
     'ETag': '"64300b59a72522358f0a5b4a0811c771"',
     'ChecksumCRC32': 'bQkdGw==',
     'ChecksumType': 'FULL_OBJECT',
     'ServerSideEncryption': 'AES256'}




```python

```
