
### **Building a SOC Lab: A Hands-On Guide to the ELK Detection Lab**

If you're looking to get hands-on with Security Operations Center (SOC) work, you need a lab. Theory is great, but there's no substitute for digging through real logs, building alerts, and investigating attack patterns.

This guide will walk you through setting up the **ELK Detection Lab**—a Docker-based environment packed with real-world attack data. It’s a complete virtual machine-like setup using the ELK Stack (Elasticsearch, Logstash, Kibana) with Filebeat, ready for you to practice threat detection and investigation.

---

### **First, What Is This Lab?**

At its core, this is a Docker project that sets up a full ELK stack. The magic is that it automatically downloads and imports real security datasets for you to play with, including:

- **EVTX-ATTACK-SAMPLES:** Windows event logs containing malicious activity.
- **Mordor Datasets:** Pre-recorded security incidents via the Mordor project.
- **Malware-Traffic-Analysis.net data:** Network traffic from real malware infections.

You get a ready-to-go lab filled with data to create alerts, run investigations, and simulate a real SOC environment.

---

### **Initial Setup & Crucial Fixes**

Before we start, there's an important step. **Do not run these commands from the `System32` directory** in Windows, as it has permissions restrictions. Use a different directory.

First, clone the lab repository:

```bash
git clone --recurse-submodules https://github.com/thomaspatzke/elk-detection-lab.git
cd elk-detection-lab
```

![[Pasted image 20251125181820.png]]


You'll also need **Docker**  installed on your machine.
https://www.docker.com/products/docker-desktop/



Now, let's fix a few things in the code to make sure everything runs smoothly.
#### **1. Fixing the Dockerfile**

We need to align the Elasticsearch client version in the `Dockerfile` to prevent compatibility issues. Update your `Dockerfile` to this:

```dockerfile
FROM python:3.7
RUN pip install elasticsearch==7.17.9 progressbar2 termcolor evtx tqdm urllib3

COPY install-datasets.sh /install-datasets.sh

# Fix line endings and permissions in the image
RUN apt update && apt install -y jq curl dos2unix \
    && dos2unix /install-datasets.sh \
    && chmod +x /install-datasets.sh

WORKDIR /datasets
ENTRYPOINT ["/install-datasets.sh"]
```

This ensures all components use the same Elasticsearch version (7.17.9) and that scripts have the correct line endings for Linux.

#### **2. Updating the Docker Compose File**

Security is enabled in this lab, so we need to configure passwords and encryption keys correctly in the `docker-compose.yml` file.

Here’s the updated structure (I've fixed the YAML formatting and passwords for consistency):

```yaml
version: "3.7"

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.9
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=true
      - xpack.monitoring.collection.enabled=true
      - ELASTIC_PASSWORD=HA1dEHuJFVU0vdzzVgeu
    ports:
      - "9200:9200"
    networks:
      - elastic

  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.9
    ports:
      - "5601:5601"
    networks:
      - elastic
    depends_on:
      - elasticsearch
    environment:
      - ELASTICSEARCH_USERNAME=elastic
      - ELASTICSEARCH_PASSWORD=HA1dEHuJFVU0vdzzVgeu
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - XPACK_ENCRYPTEDSAVEDOBJECTS_ENCRYPTIONKEY=a7a6311933d3503b89bc2dbc36572c33a6c10925682e591bffcab6911c06786d
      - XPACK_REPORTING_ENCRYPTIONKEY=b8b7H22a4Hc4f03c99dc2ef12e45b16ea9c5d2e34f8a7b1c2d3e4f5a6b7c8d9e
      - XPACK_SECURITY_ENCRYPTIONKEY=c9c8533b55d5e0+d00ed3fg23f56c27fb0d6e3f45g9b8c2d3e4f5a6b7c8d9e0f

  filebeat:
    image: docker.elastic.co/beats/filebeat:7.17.9
    command: filebeat -e --strict.perms=false
    volumes:
      - ./malware-traffic-analysis.net:/logs:ro
      - ./filebeat-config/suricata.yml:/usr/share/filebeat/modules.d/suricata.yml
    networks:
      - elastic
    depends_on:
      - elasticsearch
      - kibana
    environment:
      - ELASTICSEARCH_USERNAME=elastic
      - ELASTICSEARCH_PASSWORD=HA1dEHuJFVU0vdzzVgeu
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200

  loader:
    build: .
    volumes:
      - ../datasets
    networks:
      - elastic
    depends_on:
      - elasticsearch
    environment:
      - ELASTICSEARCH_USERNAME=elastic
      - ELASTICSEARCH_PASSWORD=HA1dEHuJFVU0vdzzVgeu
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200

networks:
  elastic:
```

**Why these changes matter:**

- **Part 1 (Elasticsearch):** Enables security and sets the password. This is essential for accessing all Elasticsearch features.
- **Part 2 (Kibana):** Configures encryption keys, allowing you to work in a secure environment and make changes safely.
- **Part 3 (Filebeat):** The `--strict.perms=false` command tells Filebeat to ignore strict file permission checks on its configuration files, which helps it start reading logs without issues.
- **Part 4 (Loader & Filebeat):** These sections provide the login credentials so both services can communicate with the secured Elasticsearch instance and upload logs.



### **The Data Installation Script**

The `install-datasets.sh` file is the heart of the lab—it’s responsible for populating your Elasticsearch with all the security data. Since the GitHub repo doesn’t come with the data pre-loaded, this script downloads and imports everything automatically.

Because we enabled security, the script must use the credentials we set earlier. Here’s a snippet of the crucial part:

```bash
ES_URL="http://elasticsearch:9200"
ES_USER="elastic"
ES_PASS="HA1dEHuJFVU0vdzzVgeu"

echo "Waiting for Elasticsearch..."
until curl -s elasticsearch:9200
do
  sleep 1
done

# ... (script waits for Elasticsearch to be ready)

# Import Mordor datasets
mordor/scripts/es-import.py --elasticsearch "http://$ES_USER:$ES_PASS@elasticsearch:9200" -r mordor

# Import EVTX attack samples
evtx2es/src/evtx2es/evtx2es.py --host elasticsearch --index winlogbeat-evtx EVTX-ATTACK-SAMPLES

# Import Kibana saved objects (dashboards, visualizations)
curl -s -u "$ES_USER:$ES_PASS" -X POST -H "kbn-xsrf: true" --form file=@kibana-settings.ndjson "kibana:5601/api/saved_objects/_import?overwrite=true"
```

---

### **Preparing the Environment**

Before starting the lab, we need to ensure our scripts will run correctly in the Linux-based Docker environment.

Run these commands to convert Windows-style line endings to Unix-style and make the scripts executable:

```bash
# Convert Python scripts to Unix format
dos2unix mordor/scripts/es-import.py
dos2unix evtx2es/src/evtx2es/evtx2es.py

# Make them executable
chmod +x mordor/scripts/es-import.py
chmod +x evtx2es/src/evtx2es/evtx2es.py
```

---

### **Launching the Lab**

Now for the exciting part! Open your terminal (Git Bash or Command Prompt) in the `elk-detection-lab` directory and run:

```bash
./elk-detection-lab.sh init
```

This command starts the installation, applies all our configurations, and begins running the Docker containers. It will take a few minutes to download everything and import the datasets.

Once it's done, you'll have a running stack of containers. You can think of this like a small apartment building where each container has a specific job:

![[Pasted image 20251125185401.png]]


| Container     | Job Description                                                                 | Port    |
|---------------|-------------------------------------------------------------------------------|---------|
| **elasticsearch** | The database. Stores and indexes all the log data for fast searching.         | 9200    |
| **kibana-1**      | The user interface. Visualizes the data with graphs, charts, and search bars. | 5601    |
| **filebeat-1**    | The log shipper. Collects and sends log files to Elasticsearch.               | -       |
| **loader-1**      | The data installer. Imports the pre-built log datasets into the system.       | -       |

**To access your lab:** Open your web browser and go to 
`http://localhost:9200`. Log in with the username `elastic` and the password `HA1dEHuJFVU0vdzzVgeu`.
and 
`http://localhost:5601`. Log in with the username `elastic` and the password `HA1dEHuJFVU0vdzzVgeu`.


### **Handy Management Commands**

Here are some useful commands for managing your lab:

- **Stop the lab:**
  ```bash
  docker-compose down
  ```

- **Rebuild without reinstalling data (quick restart):**
  ```bash
  docker-compose up -d
  ```

- **Rebuild from scratch (if something breaks):**
  ```bash
  docker-compose build --no-cache
  ```

- **Check running containers for debugging:**
  ```bash
  docker-compose ps
  ```



### **You're Ready to Investigate!**

Your SOC lab is now live. You can start exploring the imported datasets in Kibana, build detection rules, and investigate the attack scenarios present in the data. This is your sandbox—break things, test ideas, and build your cybersecurity skills.

Happy hunting!