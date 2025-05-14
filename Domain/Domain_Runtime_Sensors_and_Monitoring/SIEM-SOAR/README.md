# SIEM - SOAR

# Overview

The SIEM-SOAR collects data from probes as well as raw data. The SIEM then proceeds to read such data, store and process it, generating alerts if needed, according to a set correlation rule. Usually, data is taken from a Message Broker (i.e., Kafka). If the probe can't communicate directly with the message broker, a Filebeat instance could be attached to the probe. Its function is to collect the data from the log files of the probe and ship it to Logstash, which takes care of the parsing and formatting.
Please refer to the [Filebeat Reference](https://www.elastic.co/guide/en/beats/filebeat/current/index.html) for more information.

To simply and practically summarize:
```
If probe generates message in JSON
  send directly to KAFKA on custom Kafka topic
  create Logstash pipeline to forward message on SIEM from Kafka
If probe doesn't generate message in JSON
  create Logstash pipeline to send message on Kafka topic
  create Logstash pipeline to forward message on SIEM from Kafka
```
You can take as an example the pipeline just configured: those ending in _os are SIEM ones.

Once a probe and its communication with the message broker are set up, the next step is to create a correlation rule or use an already created one.
Both can be done using the Correlator, which is accessible on https://localhost:5000/app by default. 

Once the correlation rule is created, it needs to be started on the job manager dashboard, which is accessible on port 8081.

Lastly, depending on the probe and the attached correlation rule, a GUI dashboard needs to be setup on Grafana, which is accessible on port 3000. 


## Structure

- `src/` – Source code implementing the component
- `data/` – Input datasets or data processing scripts used or produced by this component
- `README.md` – This file


## Dependencies

The entire system only requires Docker Engine and the Docker Compose plugin installed on the host OS (Windows/Linux).

You can follow the following guide to install both on your host machine and check system requirements: https://docs.docker.com/get-docker/.

It's also recommended to follow [Linux / WSL2 post-installation steps](https://docs.docker.com/engine/install/linux-postinstall/) in order to follow the syntax used in this document for the CLI.

Please note that a Linux-based OS is recommended as Docker on Windows uses WSL2 and there are some problems with encryption and certificates.

## How to Run
To install the SIEM all you need is the folder containing the `docker.compose.yml`, the `.env` file, and the `conf` folder.

### .env file
The `.env` file should be located in the same folder of ***docker-compose.yml***. The provided one should serve as a default.

### Start the compose project
To install the SIEM move to the ***/siem*** folder then in the terminal run:

```bash
docker compose up
```

<sub>This is the syntax you can use after the post-installation steps linked above</sub>

You may need to add ```vm.max_map_count=262144``` in your ***/etc/sysctl.conf*** and run:

```bash
sudo sysctl -p
```

The first time the SIEM is set up, this will pull the necessary images, create the containers, volumes and the network and finally run them.

You can now access the web pages using the following links:
- JOBMANAGER-DASHBOARD : http://localhost:8081
- GRAFANA : http://localhost:3000
- CORRELATOR : https://localhost:5000/app
- OPENSEARCH : http://localhost:5601

<sub>Those links use the default .env file.</sub>

## Usage Example

To better understand the meaning of various fields in the pipeline, it's best to analyze it step by step.

![InputPipeline](assets/screenshots/inputpipeline.png)

These lines of code define the input for the pipeline. In this case, the input is chosen to be the Kafka topic from which messages are read.

![FilterPipeline](assets/screenshots/filterpipeline.png)

Here, a filter is defined to better parse the message. This filter is not mandatory, especially if the input message is a JSON. There are various types of filters, such as the dissect filter. To properly construct the filter, it is necessary to consult the documentation and the following website: https://dissect-tester.jorgelbg.me/ 

![OutputPipeline](assets/screenshots/outputpipeline.png)

Finally, in every Logstash pipeline, the output must be defined. In this case, two outputs are defined. One is to a specific Kafka topic, while the other is to a specific OpenSearch index. If OpenSearch is required, as shown in the example, it is important to correctly insert the user and password.


After this, you will then need to enable the pipeline in the `pipelines.yml` file stored in `probes/conf/logstash/settings/pipelines.yml` by adding:
```
- pipeline.id: alert_example
  path.config: "/usr/share/logstash/pipeline/alert_example.conf"
```

After this, restart logstash. You can do this in two ways:
```
1) docker compose down
   docker compose up

2) docker restart logstash
```


To verify that everything has worked correctly, go to OpenSearch -> Menù -> Management -> Index Management -> Indexes, and you should find the index with the name you configured in the Logstash pipeline.

On OpenSearch main page, after login, create the necessary to perform analysis with the alert:
```
Menu -> alerting -> create monitor:
    Nome: TestExample
    Index: example_alert_parsed2
    timefiled @timestamp
    query count of documents, 5 minutes

    Monitor defining method: Extraction query editor

    Add trigger:
        Name: AlertTrigger
        trigger condition:
            is above 0
        add action:
            channels: manage channels
                nome: TestChannel
                Channel type: custom webhook
                method : post
                webhook url: shuffle webhook url (you will have this after SOAR setup and configuration)
            message:
            
                {
                  "Monitor": "{{ctx.monitor.name}}",
                  "AlertStatus": "Entered",
                  "PeriodStart": "{{ctx.periodStart}}",
                  "PeriodEnd": "{{ctx.periodEnd}}"
                }
            

### ATTENTION
Regarding the webhook URL, you need to use the one provided by Shuffle, but it's important to replace "localhost" with the IP address of the host machine.
If you are using WSL, be careful when you run ifconfig to select the correct IP address. Always use the IP address of the host machine, not the one related to the WSL.

