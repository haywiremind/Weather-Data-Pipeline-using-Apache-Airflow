# Weather Data Pipeline using Apache Airflow

## Overview
This Airflow DAG extracts weather data from the OpenWeatherMap API for the city of Portland, transforms it, and loads it into an AWS S3 bucket as a CSV file.

## Prerequisites
Before running this DAG, ensure you have the following:
- Apache Airflow installed and running
- An OpenWeatherMap API key
- AWS S3 credentials with access to a designated bucket
- Airflow HTTP connection set up for OpenWeatherMap API

## Installation
1. Clone this repository.
2. Install required Python packages if not already installed:
   ```sh
   pip install apache-airflow pandas
   ```
3. Set up an Airflow connection for OpenWeatherMap:
   ```sh
   airflow connections add 'weathermap_api' \
       --conn-type 'http' \
       --conn-host 'https://api.openweathermap.org' \
       --conn-extra '{"apikey": "your_api_key_here"}'
   ```
4. Add AWS credentials to Airflow or configure them using environment variables.

## DAG Structure
The DAG follows these steps:
1. **Check API Availability:** Uses `HttpSensor` to verify if the OpenWeatherMap API is reachable.
2. **Extract Weather Data:** Fetches weather data using `SimpleHttpOperator`.
3. **Transform and Load Data:** Processes and saves data to AWS S3 using `PythonOperator`.

## Code Explanation
```python
from airflow import DAG
from datetime import timedelta, datetime
from airflow.providers.http.sensors.http import HttpSensor
import json
from airflow.providers.http.operators.http import SimpleHttpOperator
from airflow.operators.python import PythonOperator
import pandas as pd

def kelvin_to_fahrenheit(temp_in_kelvin):
    return (temp_in_kelvin - 273.15) * (9/5) + 32

def transform_load_data(task_instance):
    data = task_instance.xcom_pull(task_ids="extract_weather_data")
    city = data["name"]
    weather_description = data["weather"][0]['description']
    temp_fahrenheit = kelvin_to_fahrenheit(data["main"]["temp"])
    feels_like_fahrenheit = kelvin_to_fahrenheit(data["main"]["feels_like"])
    min_temp_fahrenheit = kelvin_to_fahrenheit(data["main"]["temp_min"])
    max_temp_fahrenheit = kelvin_to_fahrenheit(data["main"]["temp_max"])
    pressure = data["main"]["pressure"]
    humidity = data["main"]["humidity"]
    wind_speed = data["wind"]["speed"]
    time_of_record = datetime.utcfromtimestamp(data['dt'] + data['timezone'])
    sunrise_time = datetime.utcfromtimestamp(data['sys']['sunrise'] + data['timezone'])
    sunset_time = datetime.utcfromtimestamp(data['sys']['sunset'] + data['timezone'])
    
    transformed_data = [{
        "City": city,
        "Description": weather_description,
        "Temperature (F)": temp_fahrenheit,
        "Feels Like (F)": feels_like_fahrenheit,
        "Minimum Temp (F)": min_temp_fahrenheit,
        "Maximum Temp (F)": max_temp_fahrenheit,
        "Pressure": pressure,
        "Humidity": humidity,
        "Wind Speed": wind_speed,
        "Time of Record": time_of_record,
        "Sunrise (Local Time)": sunrise_time,
        "Sunset (Local Time)": sunset_time
    }]
    
    df_data = pd.DataFrame(transformed_data)
    aws_credentials = {"key": "xxxxxxxxx", "secret": "xxxxxxxxxx", "token": "xxxxxxxxxxxxxx"}
    now = datetime.now()
    dt_string = now.strftime("%d%m%Y%H%M%S")
    file_name = f"s3://weatherapiairflowyoutubebucket-yml/current_weather_data_portland_{dt_string}.csv"
    df_data.to_csv(file_name, index=False, storage_options=aws_credentials)

default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'start_date': datetime(2023, 1, 8),
    'email': ['myemail@domain.com'],
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=2)
}

with DAG('weather_dag',
         default_args=default_args,
         schedule_interval='@daily',
         catchup=False) as dag:
    
    is_weather_api_ready = HttpSensor(
        task_id='is_weather_api_ready',
        http_conn_id='weathermap_api',
        endpoint='/data/2.5/weather?q=Portland&APPID=your_api_key_here'
    )
    
    extract_weather_data = SimpleHttpOperator(
        task_id='extract_weather_data',
        http_conn_id='weathermap_api',
        endpoint='/data/2.5/weather?q=Portland&APPID=your_api_key_here',
        method='GET',
        response_filter=lambda r: json.loads(r.text),
        log_response=True
    )
    
    transform_load_weather_data = PythonOperator(
        task_id='transform_load_weather_data',
        python_callable=transform_load_data
    )
    
    is_weather_api_ready >> extract_weather_data >> transform_load_weather_data
```

## Running the DAG
1. Ensure Airflow is running:
   ```sh
   airflow scheduler & airflow webserver
   ```
2. Trigger the DAG manually or wait for the scheduled run:
   ```sh
   airflow dags trigger weather_dag
   ```

## Expected Output
- A CSV file named `current_weather_data_portland_<timestamp>.csv` uploaded to the specified AWS S3 bucket.
- Airflow UI displaying successful task execution.

## Notes
- Replace `your_api_key_here` with your OpenWeatherMap API key.
- Ensure the `aws_credentials` dictionary contains valid AWS access credentials.

## License
This project is open-source and free to use.

