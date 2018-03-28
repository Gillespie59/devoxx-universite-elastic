# Step 0

In order to launch the demo app, you should run the following command

```shell
docker-compose up
```

We have many routes :

* GET http://localhost:8080/rest/products
* GET http://localhost:8080/fake/url returing 404
* GET http://localhost:8080/long/task returing 200 after 5s
* GET http://localhost:8080/rest/weather calling an external service

# Step 1

In order to setup Elasticsearch and Kibana in our project, you need to add these extra lines in your docker-compose file

```yml
elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.2.2
    ports:
        - 9200:9200
        - 9300:9300
kibana:
    image: docker.elastic.co/kibana/kibana:6.2.2
    depends_on:
        - elasticsearch
    ports:
        - 5601:5601
```

And run the following command

```shell
docker-compose up
```

# Step 2

We can explain first the content of the packetbeat.yml file

In order to start you should executed the following commands :

```shell
docker-compose up
sudo chown root config/packetbeat/packetbeat.yml
sudo ./packetbeat -e -c config/packetbeat/packetbeat.yml
```

# Step 3

We will now enable Filebeat and index and visualize the logs generated by NGINX. These logs will be indexed wihout any modifications

In order to start you should executed the following commands :

```shell
docker-compose up
sudo chown root config/filebeat/filebeat.yml
sudo ./filebeat -e -c config/filebeat/filebeat.yml
```

# Step 4

We will now add Logstash in order to ngrok the log indexeed by filebeat

In Filebeat, the only thing we need to change is the output. We will send the document to logstash instead of elasticsearch cluster

```shell
docker-compose up
sudo chown root config/filebeat/filebeat.yml
sudo ./filebeat -e -c config/filebeat/filebeat.yml
./logstash -f config/logstash/logstash.conf
```

We will start with an easy logstash configuration file with only the beats input and elasticsearch and stdout outputs

Then we will add filters :

* grok

```shell
filter {
    grok {
        match => { "message" => "%{IPORHOST:[nginx][access][remote_ip]} - %{DATA:[nginx][access][user_name]} \[%{HTTPDATE:[nginx][access][time]}\] \"%{WORD:[nginx][access][method]} %{DATA:[nginx][access][url]} HTTP/%{NUMBER:[nginx][access][http_version]}\" %{NUMBER:[nginx][access][response_code]} %{NUMBER:[nginx][access][body_sent][bytes]} \"%{DATA:[nginx][access][referrer]}\" \"%{DATA:[nginx][access][agent]}\""}
    }
}
```

* date

```shell
date {
    match => [ "[nginx][access][time]", "dd/MMM/YYYY:H:m:s Z" ]
}  
```

We will finally add a mutate filter in order to delete the original message

```shell
mutate {
    remove_field => [ "message" ]
}
```

# Step 5

In this section we will reimplement the previous logstash configuration but with the built-in ingest feature. 

We will first use the simulate API. 

- Ingest a message without any processors

'''shell
POST http://localhost:9201/_ingest/pipeline/_simulate
{
  "pipeline" : {
  	"processors": []
  },
  "docs" : [
    { "_source": { "message": "172.18.0.1 - - [25/Mar/2018:17:07:43 +0000] \"GET / HTTP/1.1\" 304 0 \"-\" \"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36\" \"-\"\r\n"} }
  ]
}
'''
- Add the grok processor

'''shell
POST http://localhost:9201/_ingest/pipeline/_simulate
{
  "pipeline" : {
  	"processors": [
  		{
	    	"grok": {
	        	"field": "message",
	        	"patterns": ["%{IPORHOST:[nginx][access][remote_ip]} - %{DATA:[nginx][access][user_name]} \\[%{HTTPDATE:[nginx][access][time]}\\] \"%{WORD:[nginx][access][method]} %{DATA:[nginx][access][url]} HTTP/%{NUMBER:[nginx][access][http_version]}\" %{NUMBER:[nginx][access][response_code]} %{NUMBER:[nginx][access][body_sent][bytes]} \"%{DATA:[nginx][access][referrer]}\" \"%{DATA:[nginx][access][agent]}\""]
    		}
    	}
    ]
  },
  "docs" : [
    { "_source": { "message": "172.18.0.1 - - [25/Mar/2018:17:07:43 +0000] \"GET / HTTP/1.1\" 304 0 \"-\" \"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36\" \"-\"\r\n"} }
  ]
}
'''

- Add the remove processor

'''shell
POST http://localhost:9201/_ingest/pipeline/_simulate
{
  "pipeline" : {
  	"processors": [
  		{
	    	"grok": {
	        	"field": "message",
	        	"patterns": ["%{IPORHOST:[nginx][access][remote_ip]} - %{DATA:[nginx][access][user_name]} \\[%{HTTPDATE:[nginx][access][time]}\\] \"%{WORD:[nginx][access][method]} %{DATA:[nginx][access][url]} HTTP/%{NUMBER:[nginx][access][http_version]}\" %{NUMBER:[nginx][access][response_code]} %{NUMBER:[nginx][access][body_sent][bytes]} \"%{DATA:[nginx][access][referrer]}\" \"%{DATA:[nginx][access][agent]}\""]
    		},
    		"remove": {
			    "field": "message"
			  }
    	}
    ]
  },
  "docs" : [
    { "_source": { "message": "172.18.0.1 - - [25/Mar/2018:17:07:43 +0000] \"GET / HTTP/1.1\" 304 0 \"-\" \"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36\" \"-\"\r\n"} }
  ]
}
'''

- Store the pipeline in ES

'''shell
PUT http://localhost:9201/_ingest/pipeline/nginx
{
  	"processors": [
  		{
	    	"grok": {
	        	"field": "message",
	        	"patterns": ["%{IPORHOST:[nginx][access][remote_ip]} - %{DATA:[nginx][access][user_name]} \\[%{HTTPDATE:[nginx][access][time]}\\] \"%{WORD:[nginx][access][method]} %{DATA:[nginx][access][url]} HTTP/%{NUMBER:[nginx][access][http_version]}\" %{NUMBER:[nginx][access][response_code]} %{NUMBER:[nginx][access][body_sent][bytes]} \"%{DATA:[nginx][access][referrer]}\" \"%{DATA:[nginx][access][agent]}\""]
    		},
    		"remove": {
			    "field": "message"
			  }
    	}
    ]
  }
'''

- Use the previous stored pipeline when indexing a new document

'''shell
PUT http://localhost:9201/my-index/_doc/my-id?pipeline=nginx
{ "message": "172.18.0.1 - - [25/Mar/2018:17:07:43 +0000] \"GET / HTTP/1.1\" 304 0 \"-\" \"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36\" \"-\"\r\n"}
```

```shell
GET http://localhost:9201/my-index/_doc/my-id
```

# Step 6

We will add now the last beat of the day : Metricbeat for Docker container

```shell
docker-compose up
sudo chown root config/metricbeat/metricbeat.yml
sudo ./metricbeat -e -c config/metricbeat/metricbeat.yml
```

With the following configuration

```yml
metricbeat.modules:
- module: docker
  metricsets: ["container", "cpu", "diskio", "healthcheck", "info", "memory", "network"]
  hosts: ["unix:///var/run/docker.sock"]
  period: 10s
```