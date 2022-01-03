# Postgres Kafka Python code 

Dans cette patrie, on va depoloyer un pipline.
Ce dernier prend les donnï¿½es d'un bdd postgres, puis
les mettre dans un Kafka topic,
puis les consomï¿½ par un programme python flask . 

Pour ce tuto, on va utiliser **docker**.
## Etape 01 : *Télécharger* le repo
```bat
Your local machine> sudo git clone .....
```
## Etape 02 : *Crée *un network docker
Cela permit la communication entre les diffï¿½rentes contenaire docker. 
```bat
Your local machine> sudo docker network create mynetwork
```
## Etape 03 : Lancer votre Base de donnï¿½es Postgres
Si vous n'aviez pas de bdd postgres *déja* *déployer*. vous devez *créer* une :
```bat
Your local machine> sudo docker run -d --name postgres -p 5432:5432 -e POSTGRES_USER=start_data_engineer -e POSTGRES_PASSWORD=password --network mynetwork debezium/postgres
```
Puis, il faut *crées* une table et changer quelques configuration dans votre base. Pour cela il faut : 

- Renter dans le contenaire de la base de *données* 
```bat
Your local machine> sudo docker exec -it postgres bash
```
- Connecter au Postgres CLI 

```bat
docker postgres container>psql -U start_data_engineer
```
- Lancer le script suivant pour la creation de la table et les configs
```SQL
CREATE SCHEMA bank;
SET search_path TO bank,public;
CREATE TABLE bank.holding (
    holding_id int,
    user_id int,
    holding_stock varchar(8),
    holding_quantity int,
    datetime_created timestamp,
    datetime_updated timestamp,
    primary key(holding_id)
);
ALTER TABLE bank.holding replica identity FULL;
insert into bank.holding values (1000, 1, 'VFIAX', 10, now(), now());
```
> copyright : https://www.startdataengineering.com/post/change-data-capture-using-debezium-kafka-and-pg/
> copyright : https://debezium.io/



## Etape 04: Lancer Kafka 

Dans cette partie, nous alons lancer notre Kafka. Pour cela, nous avons perparer un docker compose qui sur le repo git. 

```bat
your local machine repo path>docker-compose up -d
```
*Aprés* cette *étape*, nous avons Kafka lancer sur http://127.0.0.1:29092.

> copyright : https://www.baeldung.com/ops/kafka-docker-setup
> copyright : https://www.confluent.io/

## Etape 05: Lancer le connecteur Postgres Kafka 
Nous allons utiser le connecteur de debezium. 
```bat
Your local machine> sudo docker run -d --name connect -p 8083:8083 --link kafka:kafka  --link postgres:postgres -e BOOTSTRAP_SERVERS=kafka:9092  -e GROUP_ID=sde_group -e CONFIG_STORAGE_TOPIC=sde_storage_topic  -e OFFSET_STORAGE_TOPIC=sde_offset_topic --network mynetwork debezium/connect:1.1
```
Puis, il faut le connecter à la bdd postgres et le serveur Kafka par l'instruction suivante: 
```bat
Your local machine> sudo curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json"  localhost:8083/connectors/ -d '{"name": "sde-connector", "config": {"connector.class": "io.debezium.connector.postgresql.PostgresConnector", "database.hostname": "postgres", "database.port": "5432", "database.user": "start_data_engineer", "database.password": "password", "database.dbname" : "start_data_engineer", "database.server.name": "bankserver1", "table.whitelist": "bank.holding"}}'
```
**La reponse**: 
```bat
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   747  100   389  100   358   1341   1234 --:--:-- --:--:-- --:--:--  2584HTTP/1.1 201 Created
Date: Mon, 03 Jan 2022 18:05:46 GMT
Location: http://localhost:8083/connectors/sde-connector
Content-Type: application/json
Content-Length: 389
Server: Jetty(9.4.20.v20190813)

{"name":"sde-connector","config":{"connector.class":"io.debezium.connector.postgresql.PostgresConnector","database.hostname":"postgres","database.port":"5432","database.user":"start_data_engineer","database.password":"password","database.dbname":"start_data_engineer","database.server.name":"bankserver1","table.whitelist":"bank.holding","name":"sde-connector"},"tasks":[],"type":"source"}
```
> copyright : https://www.startdataengineering.com/post/change-data-capture-using-debezium-kafka-and-pg/
> copyright : https://debezium.io/

## Etape 06: *Crée* un consomateur Python
Pour cette *étape*, il faut installer la lib kafka-python sur python . 
```bat
Your local machine>  pip install kafka-python
```
Puis, lancer le consomateur python par le code suivant: 
```python
import kafka
consumer = kafka.KafkaConsumer("bankserver1.bank.holding",bootstrap_servers=['127.0.0.1:29092'])
for msg in consumer: 
    print(msg.value)
```
