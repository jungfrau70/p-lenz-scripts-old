
#########################################################################################
# 1. Start kafka-cluster
#########################################################################################

cd ~/lens/kafka/

## Create docker-compose.yml

## docker-compose up
docker-compose -f docker-compose-kafka.yml up --build


#########################################################################################
# 2. Create Topics
#########################################################################################


## Create Topic, telegraf-kr1-metric
docker exec -it kafka1 kafka-topics --bootstrap-server=kafka1:19091 \
                                    --create \
                                    --topic telegraf-kr1-metric \
                                    --partitions 2 \
                                    --replication-factor 2

## Create Topic, telegraf-kr2-metric
docker exec -it kafka1 kafka-topics --bootstrap-server=kafka1:19091 \
                                    --create \
                                    --topic telegraf-kr2-metric \
                                    --partitions 2 \
                                    --replication-factor 2


#########################################################################################
# 3. Collect metrics, and publich to KR1-Kafka
#########################################################################################

cd ~/lens/telegraf/
kubectl apply -f external.yaml
kubectl apply -f daemonset.yaml


#########################################################################################
# 4. Relay to KR2-Kafka
#########################################################################################

cd ~/lens/kafka/

## Create Dockerfile
cat <<EOF | tee Dockerfile
# syntax=docker/dockerfile:1
FROM python:3.6.15-slim-buster

WORKDIR /app

COPY requirements-docker.txt requirements.txt
RUN pip3 install -r requirements.txt

COPY config/hosts /etc/hosts
COPY telegraf-relay.py .

CMD [ "python3", "telegraf-relay.py"]
EOF

## Create custom image
docker build -t jungfrau70/telegraf-relay:0.1 .
docker build -t jungfrau70/telegraf-relay:0.2 .

## Create docker-compose.yml

## docker-compose up
cd ~/lens/kafka/
docker-compose -f docker-compose-relay.yml up --build


#########################################################################################
# 5. Start Influxdb2 and Grafana
#########################################################################################

## Create docker-compose.yml

## docker-compose up
cd ~/lens/influxdb2/
docker-compose -f docker-compose-influxdb2.yml up --build


#########################################################################################
# 6. Ingest to Influxdb2
#########################################################################################

## Create docker-compose.yml

## docker-compose up
cd ~/lens/telegraf/
docker-compose -f docker-compose-ingestion.yml up -d


#########################################################################################
# 7. Open Grafana
#########################################################################################

## userid: admin / Tl******
http://localhost:3000/


#########################################################################################
# 8. Push codes to github
#########################################################################################
cd ~/lens
bash git-push.sh 
