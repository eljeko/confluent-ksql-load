# Confluent KSQL load demo


# Start docker

This tutorial uses [jq](https://jqlang.github.io/jq/) to format some outputs of the Confluent Platform components

Star the docker clster with:

    ./start-cluster.sh

# Ksql setup

Now we create the ksql items we need to merge the information from two different databases and 

    docker exec -it ksqldb-cli ksql http://ksqldb-server:8088 

```sql   
SET 'auto.offset.reset' = 'earliest';
```

## Create the powermeters table

```sql
CREATE TABLE powermeters ( 
    id INT PRIMARY KEY,
    uuid string, 
    brand string, 
    devicegroup string, 
    code string, 
    rate string ,
    cutrate string ,
    stowner string ,
    email string ,
    contractor string ,
    COD string ,
    RAT string ,
    XZY string ,
    ABECAD string ,
    AV_COD string ,
    AV_RAT string ,
    AV_XZY string ,
    BBV_ABECAD string ,
    CCV_ABECAD string ,
    DDV_ABECAD string ,
    EEV_ABECAD string ,
    BCOD string ,
    BRAT string ,
    BXZY string ,
    BABECAD string ,
    BAV_COD string ,
    BAV_RAT string ,
    BAV_XZY string ,
    ZBBV_ABECAD string ,
    ZCCV_ABECAD string     
) WITH (
    KAFKA_TOPIC = 'powermeters', 
    KEY_FORMAT='json', 
    VALUE_FORMAT='json',
    PARTITIONS = 50
); 
```

## Create the readings stream

```sql
    CREATE STREAM powerreadings (
        powermeterid INT KEY, 
        reading int, 
        timestamp int    
    ) WITH (
        KAFKA_TOPIC = 'powerreadings', 
        KEY_FORMAT='json', 
        VALUE_FORMAT='json',
        PARTITIONS = 50
    );
```

## Merge Powermeters info with readings

```sql
CREATE stream powermeters_readings as
    SELECT powerreadings.powermeterid AS powermeterid, 
        reading, 
        timestamp, 
        uuid,
        stowner
    FROM powerreadings
    LEFT JOIN powermeters ON powerreadings.powermeterid = powermeters.id;
```
# Populate the powermeters stations

```
jr run --embedded '{{counter "mycountek" 0 1}}*{ "id": "{{counter "mycounterv" 0 1}}", "uuid": "{{uuid}}", "brand": "{{from "cool_name"}}", "devicegroup": "{{randoms "SFB|SK3|ANF2|PTIE3|BOP|WJ32"}}", "code": "{{randoms "AB|CF|EI"}}", "rate": "{{format_float "%.2f" (floating 1 5)}}", "cutrate": "{{format_float "%.2f" (floating 1 5)}}", "stowner": "{{name }} {{surname }}", "email": "{{email}}", "contractor":"{{ company}}", "COD": "{{integer 1 100}}", "RAT": "{{integer 100 200}}", "XZY":"{{integer 200 300}}", "ABECAD":"{{integer 100 999}}-{{integer 100 999}}-{{integer 100 999}}-{{integer 100 999}}", "AV_COD": "{{integer 1 100}}", "AV_RAT": "{{integer 100 200}}", "AV_XZY":"{{integer 200 300}}", "BBV_ABECAD":"{{integer 100 999}}-{{integer 100 999}}-{{integer 100 999}}-{{integer 100 999}}", "CCV_ABECAD":"{{integer 200 999}}-{{integer 100 999}}-{{integer 100 999}}-{{integer 100 999}}", "DDV_ABECAD":"{{integer 300 999}}-{{integer 100 999}}-{{integer 100 999}}-{{integer 100 999}}", "EEV_ABECAD":"{{integer 400 999}}-{{integer 100 999}}-{{integer 100 999}}-{{integer 100 999}}", "BCOD": "{{integer 1 100}}", "BRAT": "{{integer 100 200}}", "BXZY":"{{integer 200 300}}", "BABECAD":"{{integer 100 999}}-{{integer 100 999}}-{{integer 100 999}}-{{integer 100 999}}", "BAV_COD": "{{integer 1 100}}", "BAV_RAT": "{{integer 100 200}}", "BAV_XZY":"{{integer 200 300}}", "BBBV_ABECAD":"{{integer 100 999}}-{{integer 100 999}}-{{integer 100 999}}-{{integer 100 999}}", "ZBBV_ABECAD":"{{integer 100 999}}-{{integer 100 999}}-{{integer 100 999}}-{{integer 100 999}}", "ZCCV_ABECAD":"{{integer 200 999}}-{{integer 100 999}}-{{integer 100 999}}-{{integer 100 999}}" }' -n 2000000| kafka-console-producer --bootstrap-server broker:9092 --topic powermeters --property "key.separator=*" --property "parse.key=true"
```

# generate some readings

```
jr run -f 1s --embedded '{{$powerid:= (integer 1999000 )}}{{$powerid}}*{ "powermeterid": "{{$powerid}}", "reading": "{{integer 100 999}}", "timestamp": "{{unix_time_stamp 10}}" }' -n 1|kafka-console-producer --bootstrap-server broker:9092 --topic powerreadings --property "key.separator=*" --property "parse.key=true"
```

# Check the POWERMETERS_READINGS topic for results
docker exec -it broker kafka-console-consumer --bootstrap-server broker:9092  --topic POWERMETERS_READINGS  --property "print.key=true" --from-beginning