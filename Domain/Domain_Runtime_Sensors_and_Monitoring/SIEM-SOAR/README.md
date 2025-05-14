# SIEM - SOAR

# How does it work

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


## Requirements

The entire system only requires Docker Engine and the Docker Compose plugin installed on the host OS (Windows/Linux).

You can follow the following guide to install both on your host machine and check system requirements: https://docs.docker.com/get-docker/.

It's also recommended to follow [Linux / WSL2 post-installation steps](https://docs.docker.com/engine/install/linux-postinstall/) in order to follow the syntax used in this document for the CLI.

Please note that a Linux-based OS is recommended as Docker on Windows uses WSL2 and there are some problems with encryption and certificates.

## SIEM installation
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


## SOAR installation
To install the SOAR all you need is the folder containing the `docker.compose.yml`, the `.env` file, and the `backend - frontend - functions` folders.

### .env file
The `.env` file should be located in the same folder of ***docker-compose.yml***. The provided one should serve as a default.

### Start the compose project
To install the SOAR move to the ***/soar*** folder then in the terminal run:

```bash
docker compose up
```

<sub>This is the syntax you can use after the post-installation steps linked above</sub>

The first time the SOAR is set up, this will pull the necessary images, create the containers, volumes and the network and finally run them.

You can now access the web pages using the following links:
- SHUFFLE-HTTPS : https://localhost:3443
- SHUFFLE-HTTP : http://localhost:3001

<sub>Those links use the default .env file.</sub>


## SIRP installation
To install the SIRP all you need is the folder containing the `docker.compose.yml`, the `.env` file, and the `thehive` folders.

### .env file
The `.env` file should be located in the same folder of ***docker-compose.yml***. The provided one should serve as a default.

### Start the compose project
To install the SIRP move to the ***/sirp*** folder then in the terminal run:

```bash
docker compose up
```

<sub>This is the syntax you can use after the post-installation steps linked above</sub>

The first time the SIRP is set up, this will pull the necessary images, create the containers, volumes and the network and finally run them.

You can now access the web pages using the following links:
- THEHIVE : http://localhost:9000
- CORTEX : http://localhost:9001
- MISP-HTTPS : https://localhost
- MISP-HTTP : http://localhost

<sub>Those links use the default .env file.</sub>

## Technologies
This list refers to the tools used by the system.
| Tool  |
| --- |
| Kafka |
| Opensearch |
| Flink |
| Grafana | 
| Shuffle |
| TheHive |
| Cortex |
| Misp |

In the Figure below it is represented the full architecture of the system with the technologies just listed.
![FullArchitecture](assets/FullArchitecture.jpg)

## Bootstrap the system

Below it will be provided a guide to reproduce a test scenario, the guide is procedural and has the sole purpose of presenting some of the functionality and potential of the entire system.

Please remember that the default users are:
| Tools | Username | Password |
| --- | --- | --- |
| Opensearch | admin | SxhTZycDGyTD8Fq |
| Grafana | admin | admin |
| Shuffle | admin | password |
| TheHive | admin@thehive.local | secret |
| Cortex | admin | admin |
| Misp | admin@admin.test | admin |

For Opensearch 2.12.0 and superior, a strong custom password is required. It can be changed in the `docker.compose.yml`.

## SIEM

The first step involves creating a Logstash pipeline that, in our case, will read from a specific Kafka topic, apply filtering (if needed), and write the data to another Kafka topic and to an OpenSearch index. In this example, we will use `alert_example.conf` contained in `probes/conf/logstash/pipeline/alert_example.conf`.

###### Example of pipeline

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
```

## SIRP

#### Misp initial configuration


```
To create auth keys:
  Go on Administration -> List Auth Keys -> Add authentication key. After this, substitute the existing one in [/thehive/application.conf] at line 50

Add events:
  Sync actions -> Feeds -> enable a feed -> Fetch and store all feed data


### ATTENTION (if you have some issues)
If the red brain icon appears in the bottom left corner of TheHive, you need to repeat the 'Create auth keys' step. You must change the key both in the [/thehive/application.conf] file at line 50 and in TheHive settings (by logging in with the admin account), setting the newly generated key.
```

#### Cortex initial configuration

If cortex doesn't work run the following commands in terminal:
```bash
#! to find index
curl -s http://localhost:9400/_all/_settings
#! to delete index
curl -X DELETE http://localhost:9400/cortex_{index}/
```
Then:
```
Add organization
  TestOrg
  Test Organization
Create user
  login: testuser
  full name: TestUser
    organization TestOrg
    roles: read analyze orgadmin
    add password: secret
    create api key and substitute the existing one in [/thehive/application.conf] at line 35

    
After that, log in with the user you created earlier and proceed with the following instructions.
To enable analyzers go to organizations-analyzers
add misp_2_1
  url: https://{docker0_hostip} (ifconfig sulla macchina)
  key: MISP api key created above
  cert_check: false

### ATTENTION
If you are using WSL, be careful when you run ifconfig to select the correct IP address. Always use the IP address of the host machine, not the one related to the WSL.

### ATTENTION
You can use other analyzers instead of misp_2_1, because it could have some problems. For example, it is possible to use another analyzers such as 'Urlscan_io_Search_0_1_1' 
```



#### TheHive initial configuration
```
Create organization:
  TestOrg
  Test Organization

Add User:
  login: testuser@thehive.local
  name: TestUser
  org-admin of TestOrg

  preview: set a new password: secret
  create api key, copy and save it to later use on Shuffle if needed
```

After these steps rebuild and start SIRP containers:
```bash
docker compose up --build
```

## SOAR
On Shuffle main page (you need to use http):

```
Create Workflow:
    name: WorkflowExample
    description: Example
```
Then open Workflow created:
```
Add WebHook from triggers list:
    it should generate a webhook like this:
        http://localhost:3001/api/v1/hooks/webhook_ab2ca1df-d983-4703-8dc1-a8ca03904499

After doing this, we will need scripts to follow the workflow below:

1) Create the case on TheHive.
2) Create the observable.
3) Run the analyzer on the created observable using Cortex. (if you need.)
4) Check if the observable is already present on MISP by conducting a search. If it is found, print the response. Otherwise, export the case to MISP.

The scripts are located in /siem/scripts.
```

```
ESEMPIO PER MIGLIORARE

1) To create the case on TheHive, you need to use the 'createcase.py' script. In some scenario, it is necessary to produce an IDMEF message. So, if you need to do this, use 'createcase_with_idmef.py' script.

2) To create the observable based on the case ID and research the observables on MISP use 'createobs_plus_research.py' script.

```

```
To use the scripts, you need to select from the apps list 'Shuffle_Tools' and choose 'Execute Python' from the dropdown menu. Additionally, to ensure the correct flow, all the scripts and their respective nodes must be connected sequentially in the following order:
  Webhook
    Case
      Obs
```

### TRY IT
If you wanna try the workflow, there is a script you can use located in /siem/kafka_producer.py


Below are some demo images to show how the different sections should appear:
![demo1](assets/demo_June2023/demo1.jpg)
![demo2](assets/demo_June2023/demo2.jpg)
![demo3](assets/screenshots/shuffle.png)
![demo4](assets/screenshots/thehivecase.png)
![demo5](assets/screenshots/thehiveobs.png)
![demo6](assets/screenshots/misp.png)


## MICROSOFT TEAMS

It is possible to send a notification to someone through Microsoft Teams by using an application within Teams called Workflows. Before using Workflows, you need to create a Teams channel where the notification will be sent. Once the channel is set up, you can use the app to manage notifications.

Click on "New Flow" and choose from the available templates the one called "Post to a channel when a webhook request is received." Select the channel where the notification should be delivered, and then click "Create Flow."

Once clicked, a URL should appear, which will look like this, where it's possible to send the POST request: ``` https://prod-24.westeurope.logic.azure.com:443/workflows/ba069ba92734417398dae22f0ba73bad/triggers/manual/paths/invoke?api-version=2016-06-01&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=OxlyYaCB-hLOVII3vY3KJ_K3hfIFN9EUw2RGCSpDTls. ```

Now, select the flow that was created and click on "Edit" to set it up correctly, depending on the scenario and the message received.

Now, the workflow should be structured as follows:
![Workflowteams](assets/screenshots/workflowteams2.png)

At this point, it is necessary to add a new step called "Parse JSON". Select the content to be parsed, which typically corresponds to the body of the message. Next, it is important to provide the schema of the message that will arrive in Teams, so that Teams can understand the structure of the message to be parsed. Alternatively, you can press the "Generate from sample" button to automatically create the structure.

Finally, select "Card" as the next step and follow the instructions below: ([Click here if you have some issue](https://learn.microsoft.com/it-it/power-automate/create-adaptive-cards))

1) Post as: Flow bot  
2) Post in: Channel  
3) Schema: use a schema similar to the one found in the file **exampleofworkflow_teams.txt**. 


