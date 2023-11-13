# Finnhub Streaming Data Pipeline

The project is a streaming data pipeline based on Finnhub.io API/websocket real-time trading data created for a sake of my master's thesis related to stream processing.
It is designed with a purpose to showcase key aspects of streaming pipeline development & architecture, providing low latency, scalability & availability.

## Architecture

![finnhub_streaming_data_pipeline_diagram drawio (4)](https://user-images.githubusercontent.com/75480707/218998119-12d514ef-8e10-40e7-a638-afaa728e6b4f.png)

The diagram above provides a detailed insight into pipeline's architecture. 

All applications are containerized into **Docker** containers, which are orchestrated by **Kubernetes** - and its infrastructure is managed by **Terraform**.

**Data ingestion layer** - a containerized **Python** application called **FinnhubProducer** connects to Finnhub.io websocket. It encodes retrieved messages into Avro format as specified in schemas/trades.avsc file and ingests messages into Kafka broker.

**Message broker layer** - messages from FinnhubProducer are consumed by **Kafka** broker, which is located in kafka-service pod and has **Kafdrop** service as a sidecar ambassador container for Kafka. On a container startup, **kafka-setup-k8s.sh** script runs to create topics. The **Zookeeper** pod is launched before Kafka as it is required for its metadata management.

**Stream processing layer** - a **Spark** Kubernetes cluster based on spark-k8s-operator is deployed using Helm. A **Scala** application called **StreamProcessor** is submitted into Spark cluster manager, that delegates a worker for it. This application connects to Kafka broker to retrieve messages, transform them using Spark Structured Streaming, and loads into Cassandra tables. The first query - that transforms trades into feasible format - runs continuously, whereas the second - with aggregations - has a 5 seconds trigger.

**Serving database layer** - a **Cassandra** database stores & persists data from Spark jobs. Upon launching, the **cassandra-setup.cql** script runs to create keyspace & tables.

**Visualization layer** - **Grafana** connects to Cassandra database using HadesArchitect-Cassandra-Plugin and serves visualized data to users as in example of Finnhub Sample BTC Dashboard. The dashboard is refreshed each 500ms.

## Dashboard

![ezgif com-crop](https://user-images.githubusercontent.com/75480707/219054392-5cc6a3e6-b034-4e75-8cb5-3baafe001149.gif)

You can access Grafana with a dashboard on localhost:3000 by running following command:
```
kubectl port-forward -n pipeline service/grafana 3000:3000
```
You can also modify it for your liking from UI - but if you want to save anything, you will need to export json and load it into Docker image.
Remember that if you change namespace name in Terraform variables you need to apply it into that command as well.

## Setup & deployment

The application is designed to be deployed on a local Minikube cluster. However, the deployment into EKS/GKE/AKS should be quite straight-forward, with tweaking deployment settings for providers, volumes etc.

Running the application requires you to have a Finnhub API token. You can retrieve it once you have created a Finnhub account. To include it in final deployment, insert it into proper fields in terraform-k8s/config.tf, along with Cassandra database username & password of choice. While setting Cassandra credentials remember to verify them with Grafana dashboard settings (the issue is referenced in config.tf file).

There is also an old setup that relies solely on docker-compose. To reach that, navigate to the docker-compose-old branch.

I was running this cluster on Windows with Minikube, Helm, Docker Desktop and Terraform pre-installed. I have utilized local Docker registry to apply custom images into deployment. I was launching it with no vtx enabled, using VirtualBox as VM engine.
