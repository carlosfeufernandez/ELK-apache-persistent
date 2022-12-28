# Logs y Métricas de aplicaciones en entorno docker

Integración de Logs y Metricas de aplicaciones en entorno Docker.

Caso de uso diferente al de:
- Monitorización del sistema como tal.
- Envío de logs de todos los dockers.

## Requerimientos:
- Elastic Stack instalado en network "elastic"

```
docker network create elastic
docker volume create es01-data

# Arrancamos cluster de un único nodo con la opción `discovery.type: single-node` a través de variable de entorno.
docker run -d --name es01-test --net elastic -v es01-data:/usr/share/elasticsearch/data -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.16.3

# Arrancamos Kibana apuntando a `es01-test` a través de variable de entorno.
docker run -d --name kib01-test --net elastic -p 5601:5601 -e "ELASTICSEARCH_HOSTS=http://es01-test:9200" docker.elastic.co/kibana/kibana:7.16.3
```

- Posteriormente podemos destruir todos los contenedores y al volverlos al crear tendríamos los datos y el estado anterior.


- Elasticsearch con nombre `es01-test`.

## Instrucciones:

- Copia el fichero [metricbeat.docker.yml](metricbeat.docker.yml) a tu directorio de trabajo.

- Copia el fichero [filebeat.docker.yml](filebeat.docker.yml) a tu directorio de trabajo.

- Lanzamos metricbeat en modo integración con demonio docker:

```
docker run -d \
  --name=metricbeat-docker \
  --network=elastic \
  --user=root \
  --volume="$(pwd)/metricbeat.docker.yml:/usr/share/metricbeat/metricbeat.yml:ro" \
  --volume="/var/run/docker.sock:/var/run/docker.sock:ro" \
  --volume="/sys/fs/cgroup:/hostfs/sys/fs/cgroup:ro" \
  --volume="/proc:/hostfs/proc:ro" \
  --volume="/:/hostfs:ro" \
  docker.elastic.co/beats/metricbeat:7.16.3 metricbeat -e -E output.elasticsearch.hosts=["es01-test:9200"]
```

- Lanzamos filebeat en modo integración con demonio docker:

```
docker run -d --rm \
  --name=filebeat \
  --user=root \
  --network=elastic \
  -v $(pwd)/filebeat.docker.yml:/usr/share/filebeat/filebeat.yml:ro \
  --volume="/var/lib/docker/containers:/var/lib/docker/containers:ro" \
  --volume="/var/run/docker.sock:/var/run/docker.sock:ro" \
  -e ELASTICSEARCH_HOSTS=es01-test:9200 \
  docker.elastic.co/beats/filebeat:7.16.3 filebeat -e -strict.perms=false
```

- Lanzamos un apache activando los módulos de metricbeat y filebeat correspondientes:

```
docker run \
  --network elastic \
  --label co.elastic.logs/module=apache2 \
  --label co.elastic.logs/fileset.stdout=access \
  --label co.elastic.logs/fileset.stderr=error \
  --label co.elastic.metrics/module=apache \
  --label co.elastic.metrics/metricsets=status \
  --label co.elastic.metrics/hosts='${data.host}:${data.port}' \
  --detach=true \
  --name my-apache-app \
  -p 8080:80 \
  httpd:2.4
```

- Lanzamos el SETUP de los dashboards de filebeat y metricbeat:

```
docker run --network elastic \
docker.elastic.co/beats/filebeat:7.16.3 \
setup -E setup.kibana.host=kib01-test:5601 \
-E output.elasticsearch.hosts=["es01-test:9200"]
docker run --network elastic \
docker.elastic.co/beats/metricbeat:7.16.3 \
setup -E setup.kibana.host=kib01-test:5601 \
-E output.elasticsearch.hosts=["es01-test:9200"]
```

El comando anterior lanza un docker de apache2 con los labels necesarios para que Metricbeat y Filebeat sean capaces de sacar métricas y procesar los logs.
