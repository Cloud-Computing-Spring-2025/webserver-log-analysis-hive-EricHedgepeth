# Web-Server-Log-Analysis

# Hands-on overview

This project went over how to analyze wen server logs that are in a CSV format using teh Apache Hive. We use Docker to stand up a Hadoop/Hive environment in codespaces, load the logs into HDFS, create Hive tables, and run queries to extract information about it. 

## Implementation Approach

- Docker for Environment
  - We use Docker Compose to spin up a container named "hive-server" that contains both Hive and HDFS.

- Data Loading
  - We copy our web_server_logs.csv file into the container and then place it into HDFS under /user/hive/logs.


- Hive External Table
  - We create an external table pointing to /user/hive/logs.
  - This lets Hive read the raw CSV without moving or duplicating files.

- Queries
  - We use standard HiveQL to group by columns, filter by status codes, and count or aggregate as needed.

- Partitioning  
  - We create an additional table web_logs_partitioned partitioned by status for improved performance when filtering by status.




## Steps 

# Start Docker 
```bash
Docker compose up -d 
```

# Copy CSV into container 
```bash
docker cp web_server_logs.csv hive-server:/tmp/
```

# Open container shell 
```bash
docker exec -it hive-server /bin/bash
```

# Create HDFS Directory 
```bash
hadoop fs -mkdir -p /user/hive/logs 
```

# Put file in HDFS Diretory
```bash
hadoop fs -put /tmp/web_server_logs.csv /user/hive/logs
```

# Open Hive
```bash
hive
```


## I switched to HUE at this point to use the editor there 


# Make a database and use it 
```bash
CREATE DATABASE IF NOT EXISTS web_logs_db;

USE web_logs_db;
```

# Create an External Table 
```bash
CREATE EXTERNAL TABLE IF NOT EXISTS web_logs (
  ip STRING,
  time_used STRING,
  url STRING,
  status INT,
  user_type STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/user/hive/logs';
```

# Check to make sure it was created and works 
```bash
SHOW TABLES;
SELECT * FROM web_logs LIMIT 5;
```
realized that I was counting the header row :(

# Count total requests
```bash
SELECT COUNT(ip)
FROM web_logs
WHERE ip != 'ip';
```
# Status code analysis
```bash
SELECT 
  status,
  COUNT(*) AS status_count
FROM web_logs
WHERE ip != 'ip'
GROUP BY status
ORDER BY status_count DESC;
```
# 3 most visited pages
```bash
SELECT 
  url,
  COUNT(*) AS visit_count
FROM web_logs
WHERE ip != 'ip'
GROUP BY url
ORDER BY visit_count DESC
LIMIT 3;
```

# Traffic source
```bash
SELECT 
  user_type, 
  COUNT(*) AS usage_count
FROM web_logs
WHERE ip != 'ip'
GROUP BY user_type
ORDER BY usage_count DESC;
```

# Suspicious IP
```bash
SELECT
    ip,
    COUNT(*) AS failed_requests
FROM web_logs
WHERE ip != 'ip'
  AND status IN (404, 500)
GROUP BY ip
HAVING COUNT(*) > 3;
```

# Traffic trends 
```bash
SELECT 
    SUBSTR(time_used, 1, 16) AS minute_window,
    COUNT(*) AS request_count
FROM web_logs
WHERE ip != 'ip'
GROUP BY SUBSTR(time_used, 1, 16)
ORDER BY minute_window;
```

# Create a Partitioned Table
```bash
CREATE TABLE IF NOT EXISTS web_logs_partitioned (
  ip STRING,
  time_used STRING,
  url STRING,
  user_type STRING
)
PARTITIONED BY (status INT)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/user/hive/logs_partitioned';

```

# Insert Data 
```bash
INSERT OVERWRITE TABLE web_logs_partitioned 
PARTITION (status=200)
SELECT ip, time_used, url, user_type
FROM web_logs
WHERE status = 200;



INSERT OVERWRITE TABLE web_logs_partitioned 
PARTITION (status=404)
SELECT ip, time_used, url, user_type
FROM web_logs
WHERE status = 404;



INSERT OVERWRITE TABLE web_logs_partitioned 
PARTITION (status=500)
SELECT ip, time_used, url, user_type
FROM web_logs
WHERE status = 500;

```


## Challenges Faced 

-   Header Row Parsing  
    - The CSV file had a header row and I didn't notice that before I loaded it and created the table. I used the "WHERE ip != 'ip' " to get around it but that threw me off for a bit. 
- Loading the CSV to start  
  - Loading the CSV file took me the longest because I was very confused on how to upload it to hive to work with. I eventually figured it out. 

- SQL coding  
  - It has been several years since I touched SQL and it took a good bit of searching to figure out how to use the tables and database. 

## Sanple Input and Output

- Sample Input 

```bash 
ip,timestamp,url,status,user_agent
192.168.1.1,2024-02-01 10:15:00,/home,200,Mozilla/5.0
192.168.1.2,2024-02-01 10:16:00,/products,200,Chrome/90.0
192.168.1.3,2024-02-01 10:17:00,/checkout,404,Safari/13.1
192.168.1.10,2024-02-01 10:18:00,/home,500,Mozilla/5.0
192.168.1.15,2024-02-01 10:19:00,/products,404,Chrome/90.0
```

# Sample Query
```bash 
SELECT COUNT(ip) AS total_requests
FROM web_logs
WHERE ip != 'ip';
```

# Sample Output 
```bash 
total_requests
100
```
