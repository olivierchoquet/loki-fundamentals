# Loki Quickstart

This repository provides a quickstart guide to running Grafana Loki locally using Docker Compose. This setup includes Grafana, Loki, and Grafana Alloy for log collection and visualization.

![Loki Stack](https://grafana.com/media/docs/loki/getting-started-loki-stack-3.png)

## Getting Started

📖 **[Loki Quickstart Documentation](https://grafana.com/docs/loki/latest/get-started/quick-start/)**

### Prerequisites
Ensure you have the following installed on your system:
- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/)

### Running the Loki Stack

1. Clone this repository:
   ```sh
   git clone https://github.com/grafana/loki-fundamentals.git -b getting-started
   ```
2. Navigate into the repository:
   ```sh
   cd loki-fundamentals
   ```
3. Start Loki, Grafana, and Alloy using Docker Compose:
   ```sh
   docker compose up -d
   ```
4. Verify the components are running:
   - **Grafana:** [http://localhost:3000](http://localhost:3000)
   - **Loki:** [http://localhost:3100/metrics](http://localhost:3100/metrics)
   - **Alloy:** [http://localhost:12345/graph](http://localhost:12345/graph)

### Deploying the Sample Application

![Sample Microservice Architecture](https://grafana.com/media/docs/loki/get-started-architecture.png)

To test log collection, deploy the Carnivorous Greenhouse application:

```sh
  docker compose -f greenhouse/docker-compose-micro.yml up -d --build
```

You can then interact with the sample app at [http://localhost:5005](http://localhost:5005) to generate logs.

## View containers Logs (all)

You can view all logs using Grafana’s **Explore** feature at [http://localhost:3000](http://localhost:3000):
1. Open **Explore**.
2. Select the Loki data source.
3. Use label browser

## Querying Logs

You can query logs using Grafana’s **Explore** feature at [http://localhost:3000](http://localhost:3000):
1. Open **Explore**.
2. Select the Loki data source.
3. Use LogQL queries, e.g.,:
   ```logql
   {container="greenhouse-user_service-1"} |= "ERROR"
   ```

**USE DRILLDOW to view container services and logs !**

For more advanced queries and troubleshooting, refer to the full documentation.


