# Flask Task

## File Structure

```
flask-task/
├── app
│   └── app.py
│   └── storeInfo.py
│   └── crontab
│   └── requirements.txt
│   └── start.sh
│   └── Dockerfile
│   └── templates
└── docker-compose.yml
```

## app.py

```
from flask import Flask, render_template
import psutil
import datetime
import time
import mysql.connector

app = Flask(__name__)


config = {
    'user': 'root',
    'password': 'root',
    'host': 'db',
    'port': '3306',
    'database': 'statistics'
}



def getCpuInfo():
    connection = mysql.connector.connect(**config)
    cursor = connection.cursor()
    cursor.execute("SELECT * FROM cpu order by id desc limit 24")
    cpuInfo = cursor.fetchall()
    cursor.close()
    connection.close()
    return cpuInfo

def getMemInfo():
    connection = mysql.connector.connect(**config)
    cursor = connection.cursor()
    cursor.execute("SELECT * FROM memory order by id desc limit 24")
    memoryInfo = cursor.fetchall()
    cursor.close()
    connection.close()
    return memoryInfo


def getDiskInfo():
    connection = mysql.connector.connect(**config)
    cursor = connection.cursor()
    cursor.execute("SELECT * FROM disk order by id desc limit 24")
    diskInfo = cursor.fetchall()
    cursor.close()
    connection.close()
    return diskInfo

def getCurrentCpuUsage():
    cpuUsage = f"{psutil.cpu_percent(interval=0.5)} %"
    ts = time.time()
    timestamp= datetime.datetime.fromtimestamp(ts).strftime('%Y-%m-%d %H:%M:%S')
    currentCpuUsage=(cpuUsage,timestamp)
    return currentCpuUsage

def getCurrentMemoryUsage():
    ramUsage = f"{psutil.virtual_memory().percent} %"
    ramFree = f"{int(psutil.virtual_memory().available / 1024 / 1024)} MB"
    ts = time.time()
    timestamp= datetime.datetime.fromtimestamp(ts).strftime('%Y-%m-%d %H:%M:%S')
    currentMemoryUsage=(ramUsage,ramFree,timestamp)
    return currentMemoryUsage

def getCurrentDiskUsage():
    diskUsage = f"{psutil.disk_usage('/').percent}%"
    diskFree = f"{int(psutil.disk_usage('/').free/1024/1024)} MB"
    ts = time.time()
    timestamp= datetime.datetime.fromtimestamp(ts).strftime('%Y-%m-%d %H:%M:%S')
    currentDiskUsage=(diskUsage,diskFree,timestamp)
    return currentDiskUsage


@app.route('/')
def index():
    return render_template('index.html')

@app.route('/cpu')
def cpu():
    return render_template('cpu.html', data= getCpuInfo())

@app.route('/memory')
def memory():
    return render_template('memory.html', data= getMemInfo())

@app.route('/disk')
def disk():
    return render_template('disk.html', data= getDiskInfo())


@app.route('/currentCpuUsage')
def currentCpuUsage():
    return render_template('currentCpuUsage.html', data= getCurrentCpuUsage())


@app.route('/currentMemoryUsage')
def currentMemoryUsage():
    return render_template('currentMemoryUsage.html', data= getCurrentMemoryUsage())

@app.route('/currentDiskUsage')
def currentDiskUsage():
    return render_template('currentDiskUsage.html', data= getCurrentDiskUsage())





if __name__ == '__main__':
    app.run(host='0.0.0.0')

```

-   app.py — contains the Flask app which connects to the database and exposes REST API endpoints.

## storeInfo.py

```
import psutil
import os
import mysql.connector

config = {
    'user': 'root',
    'password': 'root',
    'host': 'db',
    'port': '3306',
    'database': 'statistics'
}



mydb = mysql.connector.connect(**config)

mycursor = mydb.cursor()

sqlInsertCpuInfo = "INSERT INTO cpu (cpuUsage) VALUES (%s)"

sqlInsertMemInfo = "INSERT INTO memory (memUsage, memFree) VALUES (%s, %s)"

sqlInsertDiskInfo = "INSERT INTO disk (diskUsage, diskFree) VALUES (%s, %s)"

cpuUsage = f"{psutil.cpu_percent(interval=0.5)} %"

ramUsage = f"{psutil.virtual_memory().percent} %"

ramFree = f"{int(psutil.virtual_memory().available / 1024 / 1024)} MB"

diskUsage = f"{psutil.disk_usage('/').percent}%"

diskFree = f"{int(psutil.disk_usage('/').free/1024/1024)} MB"

mycursor.execute(sqlInsertCpuInfo, (cpuUsage,))
mycursor.execute(sqlInsertMemInfo, (ramUsage, ramFree))
mycursor.execute(sqlInsertDiskInfo, (diskUsage, diskFree))

mydb.commit()

```

-   The script above will be used by the crontab.

## crontab

```
*/1 * * * * /usr/local/bin/python /app/store.py
```

-   The crontab above will start when the application container startup

## requirements.txt

```
mysql-connector
psutil
Flask
```

## start.sh

```
# runs 2 commands simultaneously:

cron -f & # your first application
P1=$!
python app.py & # your second application
P2=$!
wait $P1 $P2
```

-   The script above allow the application container to run two services (cronjob, flask application) simultaneously.

## Dockerfile

```
FROM python:3.6
RUN apt-get update && apt-get -y install cron

EXPOSE 5000

WORKDIR /app
COPY crontab /etc/cron.d/crontab
COPY storeInfo.py /app/store.py
RUN chmod +x /etc/cron.d/crontab
RUN /usr/bin/crontab /etc/cron.d/crontab

COPY . /app
RUN pip install -r requirements.txt

CMD bash start.sh
```

## docker-compose.yml

```
version: "1"
services:
  app:
    build: ./app
    links:
      - db
    ports:
      - "5000:5000"

  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
    ports:
      - '3307:3306'
    entrypoint:
      sh -c "
        echo 'CREATE DATABASE IF NOT EXISTS statistics;use statistics;CREATE TABLE cpu(id INT NOT NULL AUTO_INCREMENT,cpuUsage varchar(255) NOT NULL,timestamp timestamp NOT NULL DEFAULT current_timestamp()
        ON UPDATE current_timestamp(),primary key (id));CREATE TABLE memory(id INT NOT NULL AUTO_INCREMENT,memUsage varchar(255) NOT NULL,memFree varchar(255) NOT NULL,timestamp timestamp NOT NULL DEFAULT
        current_timestamp() ON UPDATE current_timestamp(),primary key (id));CREATE TABLE disk(id INT NOT NULL AUTO_INCREMENT,diskUsage varchar(255) NOT NULL,diskFree varchar(255) NOT NULL,timestamp timestamp NOT        NULL DEFAULT current_timestamp() ON UPDATE current_timestamp(),primary key (id));' > /docker-entrypoint-initdb.d/init.sql;
        /usr/local/bin/docker-entrypoint.sh --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci"
```

## Building Docker Image and Run Containers

```
docker-compose up
```

-   You can see the image being built, the packages installed according to the requirements.txt, etc. If everything went right, you will see the following line:

```
flask-task_app_1  |  * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
```

-   We can find out that everything is running as expected by typing this url in a browser

You can access the database directly using the mysql client and following command

```
mysql --host=127.0.0.1 --port=3307 -u root -p
```

## Final Result

![CPU Usage](https://videos-storing.s3.ap-south-1.amazonaws.com/linux/cpuU.PNG)
![Current CPU Usage](https://videos-storing.s3.ap-south-1.amazonaws.com/linux/cpuCurrent.PNG)

![Memory Usage](https://videos-storing.s3.ap-south-1.amazonaws.com/linux/memU.PNG)

![Current Memory Usage](https://videos-storing.s3.ap-south-1.amazonaws.com/linux/memCurrent.PNG)

![Disk Usage](https://videos-storing.s3.ap-south-1.amazonaws.com/linux/diskU.PNG)

![Cureent Disk Usage](https://videos-storing.s3.ap-south-1.amazonaws.com/linux/diskCurrent.PNG)

## Pushing Docker Image to Docker Hub

```
docker image tag b06 radwanalboom/flask-task_app:v1
docker login
docker push radwanalboom/flask-task_app:v1
```

## Unit Test

-   Here we try to test the functions that deals with database, and these functions are getCpuInfo, getMemInfo and getDiskInfo.

```
from unittest.mock import patch, MagicMock, Mock
import app
from unittest import TestCase


class TestOperation(TestCase):
  @patch('mysql.connector')
  def test_get_cpu_info(self, mock_sql):
    conn = Mock()
    mock_sql.connect.return_value = conn

    cursor      = MagicMock()
    mock_result = MagicMock()

    cursor.__enter__.return_value = mock_result
    cursor.__exit___              = MagicMock()

    conn.cursor.return_value = cursor
    app.getCpuInfo()
    mock_sql.connect.assert_called_with(database='statistics', host='localhost', password='Aboalrood123@', user='root')
    cursor.execute.assert_called_with("SELECT * FROM cpu order by id desc limit 24")

  @patch('mysql.connector')
  def test_get_memory_info(self, mock_sql):
    conn = Mock()
    mock_sql.connect.return_value = conn

    cursor      = MagicMock()
    mock_result = MagicMock()

    cursor.__enter__.return_value = mock_result
    cursor.__exit___              = MagicMock()

    conn.cursor.return_value = cursor
    app.getMemInfo()
    mock_sql.connect.assert_called_with(database='statistics', host='localhost', password='Aboalrood123@', user='root')
    cursor.execute.assert_called_with("SELECT * FROM memory order by id desc limit 24")


  @patch('mysql.connector')
  def test_get_disk_info(self, mock_sql):
    conn = Mock()
    mock_sql.connect.return_value = conn

    cursor      = MagicMock()
    mock_result = MagicMock()

    cursor.__enter__.return_value = mock_result
    cursor.__exit___              = MagicMock()

    conn.cursor.return_value = cursor
    app.getDiskInfo()
    mock_sql.connect.assert_called_with(database='statistics', host='localhost', password='Aboalrood123@', user='root')
    cursor.execute.assert_called_with("SELECT * FROM disk order by id desc limit 24")
```
