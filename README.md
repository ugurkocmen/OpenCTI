
# OpenCTI Project

In this project, I took my first project step to CTI and SOC by installing OpenCTI. I continued using portainer stack in the installation.

I will share all the connector list and files I used in my project. You can use the APIs I used in this process.

When you import your .VDI file, it will come with only the portainer file installed. Portainer login information is as follows;

admin:admin1234


## System specifications used in the installation.


Ubuntu 20.04+ LTS
16GB RAM
250GB Storage
Ryzen 5 5600
## Screenshots

![App Screenshot](https://i.imgur.com/xbY4y6P.png)

![App Screenshot](https://i.imgur.com/31lgR5q.png)

## Installation

First, we need to update the repositories. 

```bash
sudo apt-get update
```

We need to install some prerequisites. Some of them may be installed, but don't forget to enter the commands. Update if an older version is available.

```bash
sudo apt-get install apt-transport-https

sudo apt-get install ca-certificates

sudo apt-get install curl

sudo apt-get install gnupg-agent

sudo apt-get install software-properties-common
```

We will need to add GPG Key for Docker. Let's add this too.

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

Now we need to add docker repository. Let's add it now.

```bash
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

We need to update the repositories. Next, we will install docker and docker-compose.

```bash
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose
```

We must manage Docker as a non-root user. (To make Docker user changes take effect, log out and log in again.)

```bash
sudo usermod -aG docker $USER
```
Now we need to docker swarm initialization. Let's do it now.

```bash
docker swarm init - advertise-addr <LOCAL_IP>
```

Now we will install portainer to manage docker as GUI.

```bash
sudo mkdir -p /opt/portainer && cd /opt/portainer

sudo curl -L https://downloads.portainer.io/portainer-agent-stack.yml -o portainer-agent-stack.yml
```

When starting Portainer, we need to change the port to avoid a port conflict. Let's do this immediately. (Hit CTRL-X to exit, “Y” to save changes, and drop back to the shell.)

```bash
sudo nano ./portainer-agent-stack.yml
```

```bash
  portainer:
    image: portainer/portainer-ce:2.11.1
    command: -H tcp://tasks.agent:9001 --tlsskipverify
    ports:
      - "9443:9443" ---> Don't change
      - "9000:9000" ---> Change first port
      - "8000:8000" ---> Change first port
    volumes:
      - portainer_data:/data
    networks:
      - agent_network
```

Now we can start the portainer. Remember, you will use the port you specified for login. (Example: http://192.168.1.21:9500)

```bash
sudo docker stack deploy - compose-file=portainer-agent-stack.yml portainer
```

Here we will ask you to create a user. After performing these steps, we go to OpenCTI's docker github page. Here we open the “docker.compose.yml” file in raw and copy all its contents. (Note: There is no version information here. Add Version: '3' at the top). We're entering the Portainer. We come to the Stack tab. Here we create a stack. You can name it as “OpenCTI”. Paste the copied data here.

```bash
version: '3'
services:
  redis:
    image: redis:7.4.2
    restart: always
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.17.3
    volumes:
      - esdata:/usr/share/elasticsearch/data
    environment:
      # Comment-out the line below for a cluster of multiple nodes
      - discovery.type=single-node
      # Uncomment the line below below for a cluster of multiple nodes
      # - cluster.name=docker-cluster
      - xpack.ml.enabled=false
      - xpack.security.enabled=false
      - thread_pool.search.queue_size=5000
      - logger.org.elasticsearch.discovery="ERROR"
      - "ES_JAVA_OPTS=-Xms${ELASTIC_MEMORY_SIZE} -Xmx${ELASTIC_MEMORY_SIZE}"
    restart: always
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    healthcheck:
      test: curl -s http://elasticsearch:9200 >/dev/null || exit 1
      interval: 30s
      timeout: 10s
      retries: 50
  minio:
    image: minio/minio:RELEASE.2024-05-28T17-19-04Z # Use "minio/minio:RELEASE.2024-05-28T17-19-04Z-cpuv1" to troubleshoot compatibility issues with CPU
    volumes:
      - s3data:/data
    ports:
      - "9000:9000"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}    
    command: server /data
    restart: always
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 10s
      timeout: 5s
      retries: 3
  rabbitmq:
    image: rabbitmq:4.0-management
    environment:
      - RABBITMQ_DEFAULT_USER=${RABBITMQ_DEFAULT_USER}
      - RABBITMQ_DEFAULT_PASS=${RABBITMQ_DEFAULT_PASS}
      - RABBITMQ_NODENAME=rabbit01@localhost
    volumes:
      - type: bind
        source: rabbitmq.conf
        target: /etc/rabbitmq/rabbitmq.conf 
      - amqpdata:/var/lib/rabbitmq
    restart: always
    healthcheck:
      test: rabbitmq-diagnostics -q ping
      interval: 30s
      timeout: 30s
      retries: 3
  opencti:
    image: opencti/platform:6.5.6
    environment:
      - NODE_OPTIONS=--max-old-space-size=8096
      - APP__PORT=8080
      - APP__BASE_URL=${OPENCTI_BASE_URL}
      - APP__ADMIN__EMAIL=${OPENCTI_ADMIN_EMAIL}
      - APP__ADMIN__PASSWORD=${OPENCTI_ADMIN_PASSWORD}
      - APP__ADMIN__TOKEN=${OPENCTI_ADMIN_TOKEN}
      - APP__APP_LOGS__LOGS_LEVEL=error
      - REDIS__HOSTNAME=redis
      - REDIS__PORT=6379
      - ELASTICSEARCH__URL=http://elasticsearch:9200
      - ELASTICSEARCH__NUMBER_OF_REPLICAS=0
      - MINIO__ENDPOINT=minio
      - MINIO__PORT=9000
      - MINIO__USE_SSL=false
      - MINIO__ACCESS_KEY=${MINIO_ROOT_USER}
      - MINIO__SECRET_KEY=${MINIO_ROOT_PASSWORD}
      - RABBITMQ__HOSTNAME=rabbitmq
      - RABBITMQ__PORT=5672
      - RABBITMQ__PORT_MANAGEMENT=15672
      - RABBITMQ__MANAGEMENT_SSL=false
      - RABBITMQ__USERNAME=${RABBITMQ_DEFAULT_USER}
      - RABBITMQ__PASSWORD=${RABBITMQ_DEFAULT_PASS}
      - SMTP__HOSTNAME=${SMTP_HOSTNAME}
      - SMTP__PORT=25
      - PROVIDERS__LOCAL__STRATEGY=LocalStrategy
      - APP__HEALTH_ACCESS_KEY=${OPENCTI_HEALTHCHECK_ACCESS_KEY}
    ports:
      - "8080:8080"
    depends_on:
      redis:
        condition: service_healthy
      elasticsearch:
        condition: service_healthy
      minio:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    restart: always
    healthcheck:
      test:  ["CMD", "wget", "-qO-", "http://opencti:8080/health?health_access_key=${OPENCTI_HEALTHCHECK_ACCESS_KEY}"]
      interval: 10s
      timeout: 5s
      retries: 20
  worker:
    image: opencti/worker:6.5.6
    environment:
      - OPENCTI_URL=http://opencti:8080
      - OPENCTI_TOKEN=${OPENCTI_ADMIN_TOKEN}
      - WORKER_LOG_LEVEL=info
    depends_on:
      opencti:
        condition: service_healthy
    deploy:
      mode: replicated
      replicas: 3
    restart: always
  connector-export-file-stix:
    image: opencti/connector-export-file-stix:6.5.6
    environment:
      - OPENCTI_URL=http://opencti:8080
      - OPENCTI_TOKEN=${OPENCTI_ADMIN_TOKEN}
      - CONNECTOR_ID=${CONNECTOR_EXPORT_FILE_STIX_ID} # Valid UUIDv4
      - CONNECTOR_TYPE=INTERNAL_EXPORT_FILE
      - CONNECTOR_NAME=ExportFileStix2
      - CONNECTOR_SCOPE=application/json
      - CONNECTOR_LOG_LEVEL=info
    restart: always
    depends_on:
      opencti:
        condition: service_healthy
  connector-export-file-csv:
    image: opencti/connector-export-file-csv:6.5.6
    environment:
      - OPENCTI_URL=http://opencti:8080
      - OPENCTI_TOKEN=${OPENCTI_ADMIN_TOKEN}
      - CONNECTOR_ID=${CONNECTOR_EXPORT_FILE_CSV_ID} # Valid UUIDv4
      - CONNECTOR_TYPE=INTERNAL_EXPORT_FILE
      - CONNECTOR_NAME=ExportFileCsv
      - CONNECTOR_SCOPE=text/csv
      - CONNECTOR_LOG_LEVEL=info
    restart: always
    depends_on:
      opencti:
        condition: service_healthy
  connector-export-file-txt:
    image: opencti/connector-export-file-txt:6.5.6
    environment:
      - OPENCTI_URL=http://opencti:8080
      - OPENCTI_TOKEN=${OPENCTI_ADMIN_TOKEN}
      - CONNECTOR_ID=${CONNECTOR_EXPORT_FILE_TXT_ID} # Valid UUIDv4
      - CONNECTOR_TYPE=INTERNAL_EXPORT_FILE
      - CONNECTOR_NAME=ExportFileTxt
      - CONNECTOR_SCOPE=text/plain
      - CONNECTOR_LOG_LEVEL=info
    restart: always
    depends_on:
      opencti:
        condition: service_healthy
  connector-import-file-stix:
    image: opencti/connector-import-file-stix:6.5.6
    environment:
      - OPENCTI_URL=http://opencti:8080
      - OPENCTI_TOKEN=${OPENCTI_ADMIN_TOKEN}
      - CONNECTOR_ID=${CONNECTOR_IMPORT_FILE_STIX_ID} # Valid UUIDv4
      - CONNECTOR_TYPE=INTERNAL_IMPORT_FILE
      - CONNECTOR_NAME=ImportFileStix
      - CONNECTOR_VALIDATE_BEFORE_IMPORT=true # Validate any bundle before import
      - CONNECTOR_SCOPE=application/json,text/xml
      - CONNECTOR_AUTO=true # Enable/disable auto-import of file
      - CONNECTOR_LOG_LEVEL=info
    restart: always
    depends_on:
      opencti:
        condition: service_healthy
  connector-import-document:
    image: opencti/connector-import-document:6.5.6
    environment:
      - OPENCTI_URL=http://opencti:8080
      - OPENCTI_TOKEN=${OPENCTI_ADMIN_TOKEN}
      - CONNECTOR_ID=${CONNECTOR_IMPORT_DOCUMENT_ID} # Valid UUIDv4
      - CONNECTOR_TYPE=INTERNAL_IMPORT_FILE
      - CONNECTOR_NAME=ImportDocument
      - CONNECTOR_VALIDATE_BEFORE_IMPORT=true # Validate any bundle before import
      - CONNECTOR_SCOPE=application/pdf,text/plain,text/html
      - CONNECTOR_AUTO=true # Enable/disable auto-import of file
      - CONNECTOR_ONLY_CONTEXTUAL=false # Only extract data related to an entity (a report, a threat actor, etc.)
      - CONNECTOR_CONFIDENCE_LEVEL=15 # From 0 (Unknown) to 100 (Fully trusted)
      - CONNECTOR_LOG_LEVEL=info
      - IMPORT_DOCUMENT_CREATE_INDICATOR=true
    restart: always
    depends_on:
      opencti:
        condition: service_healthy
  connector-analysis:
    image: opencti/connector-import-document:6.5.6
    environment:
      - OPENCTI_URL=http://opencti:8080
      - OPENCTI_TOKEN=${OPENCTI_ADMIN_TOKEN}
      - CONNECTOR_ID=${CONNECTOR_ANALYSIS_ID} # Valid UUIDv4
      - CONNECTOR_TYPE=INTERNAL_ANALYSIS
      - CONNECTOR_NAME=ImportDocumentAnalysis
      - CONNECTOR_VALIDATE_BEFORE_IMPORT=false # Validate any bundle before import
      - CONNECTOR_SCOPE=application/pdf,text/plain,text/html
      - CONNECTOR_AUTO=true # Enable/disable auto-import of file
      - CONNECTOR_ONLY_CONTEXTUAL=false # Only extract data related to an entity (a report, a threat actor, etc.)
      - CONNECTOR_CONFIDENCE_LEVEL=15 # From 0 (Unknown) to 100 (Fully trusted)
      - CONNECTOR_LOG_LEVEL=info
    restart: always
    depends_on:
      opencti:
        condition: service_healthy

volumes:
  esdata:
  s3data:
  redisdata:
  amqpdata:
```

Now we go back to the github docker and paste the contents of the “.env.sample” file into a TXT to upload. There are places we need to edit here.

OPENCTI_ADMIN_PASSWORD
OPENCTI_ADMIN_TOKEN
MINIO_ROOT_PASSWORD
RABBITMQ_DEFAULT_PASS

we change these parts. You will use the UUIDv4 generator when receiving tokens.

```bash
OPENCTI_ADMIN_EMAIL=admin@opencti.io
OPENCTI_ADMIN_PASSWORD=changeme
OPENCTI_ADMIN_TOKEN=ChangeMe_UUIDv4
OPENCTI_BASE_URL=http://localhost:8080
MINIO_ROOT_USER=opencti
MINIO_ROOT_PASSWORD=changeme
RABBITMQ_DEFAULT_USER=opencti
RABBITMQ_DEFAULT_PASS=changeme
CONNECTOR_EXPORT_FILE_STIX_ID=dd817c8b-abae-460a-9ebc-97b1551e70e6
CONNECTOR_EXPORT_FILE_CSV_ID=7ba187fb-fde8–4063–92b5-c3da34060dd7
CONNECTOR_EXPORT_FILE_TXT_ID=ca715d9c-bd64–4351–91db-33a8d728a58b
CONNECTOR_IMPORT_FILE_STIX_ID=72327164–0b35–482b-b5d6-a5a3f76b845f
CONNECTOR_IMPORT_DOCUMENT_ID=c3970f8a-ce4b-4497-a381–20b7256f56f0
SMTP_HOSTNAME=localhost
ELASTIC_MEMORY_SIZE=4G
```

Now we need to load the created .env file. Go to Portainer. scroll down and in the environment variables section click on load variables from .env file.

Now in Portainer just click deploy the stack.

Wait for some time let the deployment take place, then go to the browser and navigate OpenCTI via

```bash
<LOCAL_IP>:8080
```

After some time OpenCTI login page will appear, enter your credentials to access OpenCTI.
