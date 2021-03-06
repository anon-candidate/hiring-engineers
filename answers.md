Your answers to the questions go here.

## Prerequisites - Setup the environment

> You can utilize any OS/host that you would like to complete this exercise. However, we recommend one of the following approaches:
>
> * You can spin up a fresh linux VM via Vagrant or other tools so that you don’t run into any OS or dependency issues. [Here are instructions](https://github.com/DataDog/hiring-engineers/blob/solutions-engineer/README.md#vagrant) for setting up a Vagrant Ubuntu VM. We strongly recommend using minimum `v. 16.04` to avoid dependency issues.
> * You can utilize a Containerized approach with Docker for Linux and our dockerized Datadog Agent image.

As my host OS is Windows 10 x64, I opted for VMware Workstation 15 to set up my guest VM (Ubuntu 18.04). VMWare makes it easy and convenient to run VMs on Windows - it provides excellent tools for host/guest communication and sharing, as well as arguably the best support for HDPI displays. 

> Then, sign up for Datadog (use “Datadog Recruiting Candidate” in the “Company” field), get the Agent reporting metrics from your local machine.

This was a very straightforward process. It was simply a matter of signing up for Datadog and executing the provided installation script for Ubuntu:
```bash
$ DD_API_KEY=********* bash -c "$(curl -L https://raw.githubusercontent.com/DataDog/datadog-agent/master/cmd/agent/install_script.sh)"
```
Immediately after installation, incoming metrics could be seen in the “System – Metrics” dashboard:

![Alt text](img/1-System-Metrics-Datadog.png?raw=true "Datadog System Metrics Dashboard")

## Collecting Metrics:

> * Add tags in the Agent config file and show us a screenshot of your host and its tags on the Host Map page in Datadog.

```bash
$ sudo vi /etc/datadog-agent/datadog.yaml
```

Set arbitrary tags: 
```bash
# Set the host's tags (optional)
tags:
   - env:dev
   - tier:webserver 
```

Restart agent:
```bash
$ sudo systemctl restart datadog-agent
```

View of Host Map page:
![Alt text](img/2-Host-Map_Datadog.png?raw=true "Datadog Host Map page")

>
> * Install a database on your machine (MongoDB, MySQL, or PostgreSQL) and then install the respective Datadog integration for that database.

Install MySQL:
```bash
$ sudo apt update
$ sudo apt install mysql-server
$ sudo mysql_secure_installation
$ sudo mysql

mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '*******';
mysql> FLUSH PRIVILEGES;
mysql> exit;
```

Follow integration setup instructions here:
https://docs.datadoghq.com/integrations/mysql/
```bash
$ mysql -u root -p

mysql> CREATE USER 'datadog'@'localhost' IDENTIFIED WITH mysql_native_password by '********';

```

Grant privileges to Agent to collect DB metrics:
```bash
mysql> GRANT REPLICATION CLIENT ON *.* TO 'datadog'@'localhost' WITH MAX_USER_CONNECTIONS 5;
mysql> GRANT PROCESS ON *.* TO 'datadog'@'localhost';
mysql> GRANT SELECT ON performance_schema.* TO 'datadog'@'localhost';
```

Create and populate Agent MySQL config as follows:
```bash
$ sudo vi /etc/datadog-agent/conf.d/mysql.d/conf.yaml

  init_config:

  instances:
    - server: 127.0.0.1
      user: datadog
      pass: '**********'
      port: 3306
      options:
          replication: 0
          galera_cluster: true
          extra_status_metrics: true
          extra_innodb_metrics: true
          extra_performance_metrics: true
          schema_size_metrics: false
          disable_innodb_metrics: false

$ sudo systemctl restart datadog-agent
```

Note MySQL app listed on Infrstructure List page:
![Alt text](img/3a-Infrastructure_List_Datadog.png?raw=true "Infrastructure List")


View of newly created "MySql Overview" dashboard:
![Alt text](img/3b-MySQL-Overview_Datadog.png?raw=true "MySql Overview")


> * Create a custom Agent check that submits a metric named my_metric with a random value between 0 and 1000.

Create the new agent Check:
```bash
$ sudo vi /etc/datadog-agent/checks.d/custom-my-new-check.py
```

```python
import random

# the following try/except block will make the custom check compatible with any Agent version
try:
    # first, try to import the base class from old versions of the Agent...
    from checks import AgentCheck
except ImportError:
    # ...if the above failed, the check is running in Agent version 6 or later
    from datadog_checks.checks import AgentCheck

# content of the special variable __version__ will be shown in the Agent status page
__version__ = "1.0.0"


class MyNewCheck(AgentCheck):
    def check(self, instance):
        self.gauge('my_metric', random.randint(0,1000), tags=['some_key:some_value'])
```

Create the Check YAML config (must have same name as Check Python file itself):
```bash
$ sudo vi /etc/datadog-agent/conf.d/custom-my-new-check.yaml
```

Set initial contents of config as follows:
```yaml
instances: [{}]
```

Ensure permissions are correct and restart the agent:
```bash
$ sudo chown dd-agent:dd-agent /etc/datadog-agent/checks.d/custom-my-new-check.py
$ sudo chown dd-agent:dd-agent /etc/datadog-agent/conf.d/custom-my-new-check.yaml
$ sudo systemctl restart datadog-agent
```
Verify Check is running:
```bash
$ sudo -u dd-agent -- datadog-agent check custom-my-new-check
=== Series ===
{
  "series": [
    {
      "metric": "my_metric",
      "points": [
        [
          1560933040,
          1
        ]
      ],
      "tags": [
        "some_key:some_value"
      ],
      "host": "ubuntu",
      "type": "gauge",
      "interval": 0,
      "source_type_name": "System"
    }
  ]
}
=========
Collector
=========

  Running Checks
  ==============
    
    custom-my-new-check (1.0.0)
    ---------------------------
      Instance ID: custom-my-new-check:d884b5186b651429 [OK]
      Total Runs: 1
      Metric Samples: Last Run: 1, Total: 1
      Events: Last Run: 0, Total: 0
      Service Checks: Last Run: 0, Total: 0
      Average Execution Time : 0s
      

Check has run only once, if some metrics are missing you can try again with --check-rate to see any other metric if available.
```


> * Change your check's collection interval so that it only submits the metric once every 45 seconds.

Change contents of conf.d/custom-my-new-check.yml to the following:

```yaml
init_config:

instances:
  - min_collection_interval: 45
```

Verify Check is running at expected interval through Metric Explorer:

![Alt text](img/4-Metric-Explorer_Datadog.png?raw=true "Datadog Metric Explorer")



> * **Bonus Question** Can you change the collection interval without modifying the Python check file you created?

Yes, change the interval in the Check's YAML configuration file, as described above.


## Visualizing Data:

> Utilize the Datadog API to create a Timeboard that contains:

> * Your custom metric scoped over your host.
> * Any metric from the Integration on your Database with the anomaly function applied.
> * Your custom metric with the rollup function applied to sum up all the points for the past hour into one bucket

I added "MySQL CPU Time" to the dashboard as it provided data I could visualise immediately. 

I chose to make the API call via Curl:

```bash
curl  -X POST -H \"Content-type: application/json\" \
-d "{
      \"graphs\" : [
		{
          \"title\": \"My Custom Check Metric\",
          \"definition\": {
              \"events\": [],
              \"requests\": [
                  {\"q\": \"my_metric{host:ubuntu}\"}
              ],
              \"viz\": \"timeseries\"
          }
		},
		{
          \"title\": \"MySQL CPU Time (Anomaly Detection)\",
          \"definition\": {
              \"events\": [],
              \"requests\": [
                  {\"q\": \"anomalies(avg:mysql.performance.user_time{*}, 'basic', 4, direction='above', alert_window='last_1m')\"}
              ],
              \"viz\": \"timeseries\"
          }
		},
		{
          \"title\": \"My Custom Check Metric (Summed)\",
          \"definition\": {
              \"events\": [],
              \"requests\": [
                  {\"q\": \"my_metric{host:ubuntu}.rollup(sum, 3600)\"}
              ],
              \"viz\": \"timeseries\"
          }
		}		
	  ],
      \"title\" : \"My Awesome Dashboard\",
      \"description\" : \"A dashboard containing my custom metric + database stuff.\",
      \"template_variables\": [{
          \"name\": \"ubuntu\",
          \"prefix\": \"host\"
      }],
      \"read_only\": \"True\"
}" \
"https://api.datadoghq.com/api/v1/dash?api_key=${api_key}&application_key=${app_key}"
```

API response:
```bash
{"dash":{"read_only":true,"graphs":[{"definition":{"viz":"timeseries","requests":[{"q":"my_metric{host:ubuntu}"}],"events":[]},"title":"My Custom Check Metric"},{"definition":{"viz":"timeseries","requests":[{"q":"anomalies(avg:mysql.performance.user_time{*}, 'basic', 4, direction='above', alert_window='last_1m')"}],"events":[]},"title":"MySQL CPU Time (Anomaly Detection)"},{"definition":{"viz":"timeseries","requests":[{"q":"my_metric{host:ubuntu}.rollup(sum, 3600)"}],"events":[]},"title":"My Custom Check Metric (Summed)"}],"template_variables":[{"prefix":"host","name":"ubuntu"}],"description":"A dashboard containing my custom metric + database stuff.","title":"My Awesome Dashboard","created":"2019-06-19T11:32:09.463544+00:00","new_id":"day-53d-x2v","id":1250296,"created_by":{"disabled":false,"handle":"feeney.john@gmail.com","name":"**************","is_admin":true,"role":null,"access_role":"adm","verified":true,"email":"***********@gmail.com","icon":"https://secure.gravatar.com/avatar/b7e70f2a4e9e07cb81008531290985a0?s=48&d=retro"},"modified":"2019-06-19T11:32:09.463544+00:00"},"url":"/dash/1250296/my-awesome-dashboard","resource":"/api/v1/dash/1250296"}
```

This is what the graphs look like in my new Timeboard:

![Alt text](img/5-My-Awesome-Dashboard_Datadog-v2.png?raw=true "New Timeboard")


> Please be sure, when submitting your hiring challenge, to include the script that you've used to create this Timeboard.

> Once this is created, access the Dashboard from your Dashboard List in the UI:

> * Set the Timeboard's timeframe to the past 5 minutes

Easiest way to do this I found was to use the Timeboard keyboard shortcuts (Alt + \[ and Alt + \]):

![Alt text](img/6-My-Awesome-Dashboard-Timeboard_Tools.png?raw=true)

Which allows for a 5 minute timeseries:

![Alt text](img/6-My-Awesome-Dashboard-Timeframe_Datadog.png?raw=true)


> * Take a snapshot of this graph and use the @ notation to send it to yourself.

Annotating a graph results in the generaton of a snapshot of the graph during that given timeframe.
  
![Alt text](img/7-My-Awesome-Dashboard_Annotation.png?raw=true)

Below are a couple of snapshots I received in the Datadog application using this mechanism:

![Alt text](img/8-Snapshots_Datadog.png?raw=true)

And here is one of the emails I received:
![Alt text](img/8b-anomaly-email.png?raw=true)


Very nice and intuitive feature!


> * **Bonus Question**: What is the Anomaly graph displaying?


The anomaly graph highlights deviations from past behaviour (or algorithmically deduced trends). These deviations are marked in red. Recurring or expected behaviour is bounded by a grey area in the chart.  

![Alt text](img/9-anomaly-graph.png?raw=true)


## Monitoring Data

> Since you’ve already caught your test metric going above 800 once, you don’t want to have to continually watch this dashboard to be alerted when it goes above 800 again. So let’s make life easier by creating a monitor.
>
> Create a new Metric Monitor that watches the average of your custom metric (my_metric) and will alert if it’s above the following values over the past 5 minutes:
> 
> * Warning threshold of 500
> * Alerting threshold of 800
> * And also ensure that it will notify you if there is No Data for this query over the past 10m.

The monitor thresholds were configured through the UI as follows:
![Alt text](img/10-Metric-Monitor-Setup_Datadog.png?raw=true)

>
> Please configure the monitor’s message so that it will:
>
> * Send you an email whenever the monitor triggers.
> * Create different messages based on whether the monitor is in an Alert, Warning, or No Data state.
> * Include the metric value that caused the monitor to trigger and host ip when the Monitor triggers an Alert state.
>
The following is the monitor's message configuration:
 
```bash
{{#is_warning}}
Custom metric has exceeded the Warning threshold!
{{/is_warning}}

{{#is_alert}}
Custom metric has exceeded the Alert threshold!
Host IP: {{host.ip}}
Value: {{value}}
{{/is_alert}}

{{#is_no_data}}
No data for custom metric has been transmitted during the last 10 minutes!
{{/is_no_data}}  

Notify: @*********@gmail.com
```

> * When this monitor sends you an email notification, take a screenshot of the email that it sends you.
> 

The Alert email, once received, looks like this:
![Alt text](img/11-Alert-email.png?raw=true)


> * **Bonus Question**: Since this monitor is going to alert pretty often, you don’t want to be alerted when you are out of the office. Set up two scheduled downtimes for this monitor:

Setting up a scheduled downtime is really easy: Monitors > Manage Downtime > Scheduled Downtime
> 
>  * One that silences it from 7pm to 9am daily on M-F,

![Alt text](img/12-Schedule_Downtime_M-F-v2.png?raw=true)


>  * And one that silences it all day on Sat-Sun.

![Alt text](img/12-Schedule_Downtime_S-S.png?raw=true)

>  * Make sure that your email is notified when you schedule the downtime and take a screenshot of that notification.

Set up a one-time downtime schedule for the purpose of receiving an email:

![Alt text](img/13-one-time-downtime.png?raw=true)

The email appears as follows:

![Alt text](img/14-Downtime-email.png?raw=true)

## Collecting APM Data:

> Given the following Flask app (or any Python/Ruby/Go app of your choice) instrument this using Datadog’s APM solution:

> ```python
> from flask import Flask
> import logging
> import sys
>
> # Have flask use stdout as the logger
> main_logger = logging.getLogger()
> main_logger.setLevel(logging.DEBUG)
> c = logging.StreamHandler(sys.stdout)
> formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
> c.setFormatter(formatter)
> main_logger.addHandler(c)
> 
> app = Flask(__name__)
> 
> @app.route('/')
> def api_entry():
>     return 'Entrypoint to the Application'
>
> @app.route('/api/apm')
> def apm_endpoint():
>     return 'Getting APM Started'
> 
> @app.route('/api/trace')
> def trace_endpoint():
>    return 'Posting Traces'
>
> if __name__ == '__main__':
>     app.run(host='0.0.0.0', port='5050')
```
```
> * **Note**: Using both ddtrace-run and manually inserting the Middleware has been known to cause issues. Please only use one or the other.

Install prerequisites:
```bash
$ sudo apt install python-pip
$ sudo pip install flask
```

Install the Datadog tracing library:
```bash
$ sudo pip install ddtrace
```

Output:
```bash
Successfully installed ddtrace-0.26.0 psutil-5.6.3
```

Create a new Flask file - with the above code - called "_instrumentation-example.py_" and launch with the Datadog tracing library:

```bash
$ ddtrace-run python instrumentation-example.py 
 * Serving Flask app "instrumentation-example" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
2019-06-19 13:56:22,362 INFO [werkzeug] [_internal.py:122] -  * Running on http://0.0.0.0:5050/ (Press CTRL+C to quit)
2019-06-19 13:56:22,362 - werkzeug - INFO -  * Running on http://0.0.0.0:5050/ (Press CTRL+C to quit)

```

Load test the application with Apache Jmeter. 200 users with a ramp-up time of 60 seconds and an infinite loop.

![Alt text](img/16-Jmeter-Load-Test.png?raw=true)


Observed the presence of a new service:
![Alt text](img/15-service.png?raw=true)

 
The service dashboard provides an overview of the following metrics:
![Alt text](img/17-APM-Dashboard1.png?raw=true)
![Alt text](img/17-APM-Dashboard2.png?raw=true)




> * **Bonus Question**: What is the difference between a Service and a Resource?

In Datadog terminology, a Service is a logical collection of operations. Examples of services include a database or a web application. 

Resources, on the other hand, are processes or actions within these services. So, taking a web application for example, a resource might be an MVC controller or URL endpoint. In a database service, SQL queries are considered resources.   

> Provide a link and a screenshot of a Dashboard with both APM and Infrastructure Metrics.

Link to APM and Infrastructure Screenboard [can be found here](https://p.datadoghq.com/sb/cxt6eo591kjs99jg-917c139b3eec03957759c5d6e6bacdaf):


Screenshot of Screenboard:
![Alt text](img/19-My-Screenboard.png?raw=true)

>
> Please include your fully instrumented app in your submission, as well.

App uses the code from the exercise above.

## Final Question:

> Datadog has been used in a lot of creative ways in the past. We’ve written some blog posts about using Datadog to monitor the NYC Subway System, Pokemon Go, and even office restroom availability!
> 
> Is there anything creative you would use Datadog for?

My Wireless ISP provides a patchy service from time to time. Download speeds can suffer in the evenings due to high contention. I often complain, but it would be good to back up my griping with data. 

I have a satellite box that runs on Linux and is wired to my router. I think a nifty idea might be to write a node.js or python app that periodically records ping times to google.com and the results of speed tests using the [speedtest.net API](https://github.com/ddsol/speedtest.net). These metrics would then be transmitted to Datadog and, through various chart types, I could visualise just how crappy a service I am getting and paying for!

