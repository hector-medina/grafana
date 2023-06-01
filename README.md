# grafana

## Instalación

Hay que hacerse una cuenta en https://grafana.com/ 

Añadir un `stack` si no lo tiene creado. La versión gratis solo puede tener uno.

![Captura de Pantalla 2023-06-01 a las 20 47 56](https://github.com/hector-medina/grafana/assets/76477956/315aba33-aec8-410d-87a7-0fd6bea0297e)

Entrar al entorno de grafana para ese stack, dando click en `Launch`.

![Captura de Pantalla 2023-06-01 a las 20 48 47](https://github.com/hector-medina/grafana/assets/76477956/77d31293-dd4b-4107-8fe7-d93f0f0fdf55)

En el menú lateral ir a `Connections` y buscar `Linux Server`.

El primer es instalar el agente de grafana, para ello le damos al botón `Run the Grafana Agent`. Nos pedirá el sistema operativo que usamos y crear una API key (o usar una creada anteriormente). Creamos la API key dandole un nombre descriptivo. Es importante guardar la API key, ya que es nuestra contraseña.

![Captura de Pantalla 2023-06-01 a las 20 51 23](https://github.com/hector-medina/grafana/assets/76477956/d7213d5b-0bce-4a11-a984-d19ab05f1bb3)

Una vez finalizado, nos dará un comando que copiaremos y pegaremos en el servidor. Instalará el agente de grafana, iniciará un servicio de systemd y añadirá una configuración básica. 

Por defecto el fichero de configuración de grafana está en `/etc/gtafana-agent.yml`. Como es un fichero yml, es muy importante que esté bien formateado (es como python, cada indentación tiene que ser de dos espacios)

Hay que añadirle la información del paso 2 de la integración de grafana indicando el nombre del host. Ese nombre se usará para seleccionar luego de qué servidor quieres ver las métricas. El fichero tiene que quedar algo así:

`````
integrations:
  node_exporter:
    enabled: true
    # disable unused collectors
    disable_collectors:
      - ipvs #high cardinality on kubelet
      - btrfs
      - infiniband
      - xfs
      - zfs
    # exclude dynamic interfaces
    netclass_ignored_devices: "^(veth.*|cali.*|[a-f0-9]{15})$"
    netdev_device_exclude: "^(veth.*|cali.*|[a-f0-9]{15})$"
    # disable tmpfs
    filesystem_fs_types_exclude: "^(autofs|binfmt_misc|bpf|cgroup2?|configfs|debugfs|devpts|devtmpfs|tmpfs|fusectl|hugetlbfs|iso9660|mqueue|nsfs|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|selinuxfs|squashfs|sysfs|tracefs)$"
    # drop extensive scrape statistics
    metric_relabel_configs:
    - action: drop
      regex: node_scrape_collector_.+
      source_labels: [__name__]
    relabel_configs:
    - replacement: grafana
      target_label: instance
  agent:
    enabled: true
    relabel_configs:
    - action: replace
      source_labels:
      - agent_hostname
      target_label: instance
    - action: replace
      target_label: job
      replacement: "integrations/agent-check"
    metric_relabel_configs:
    - action: keep
      regex: (prometheus_target_.*|prometheus_sd_discovered_targets|agent_build.*|agent_wal_samples_appended_total|process_start_time_seconds)
      source_labels:
      - __name__
  prometheus_remote_write:
  - basic_auth:
      password: glc_eyJvIjoiODEwNDM0IiwibiI6InNlcnZlci1kZS1wcnVlYmFzLTIiLCJrIjoiZTkwWGE0cTBwMUQ1bzB5WWVTNE0wNkFoIiwibSI6eyJyIjoicHJvZC1ldS13ZXN0LTMifX0=
      username: 811005
    url: https://prometheus-prod-22-prod-eu-west-3.grafana.net/api/prom/push
logs:
  configs:
  - clients:
    - basic_auth:
        password: glc_eyJvIjoiODEwNDM0IiwibiI6InNlcnZlci1kZS1wcnVlYmFzLTIiLCJrIjoiZTkwWGE0cTBwMUQ1bzB5WWVTNE0wNkFoIiwibSI6eyJyIjoicHJvZC1ldS13ZXN0LTMifX0=
        username: 404471
      url: https://logs-prod-013.grafana.net/loki/api/v1/push
    name: integrations
    positions:
      filename: /tmp/positions.yaml
    scrape_configs:
    - job_name: integrations/node_exporter_journal_scrape
      journal:
        max_age: 24h
        labels:
          instance: grafana
          job: integrations/node_exporter
      relabel_configs:
      - source_labels: ['__journal__systemd_unit']
        target_label: 'unit'
      - source_labels: ['__journal__boot_id']
        target_label: 'boot_id'
      - source_labels: ['__journal__transport']
        target_label: 'transport'
      - source_labels: ['__journal_priority_keyword']
        target_label: 'level'
    - job_name: integrations/node_exporter_direct_scrape
      static_configs:
      - targets:
        - localhost
        labels:
          instance: grafana
          __path__: /var/log/{syslog,messages,*.log}
          job: integrations/node_exporter

metrics:
  configs:
  - name: integrations
    remote_write:
    - basic_auth:
        password: glc_eyJvIjoiODEwNDM0IiwibiI6InNlcnZlci1kZS1wcnVlYmFzLTIiLCJrIjoiZTkwWGE0cTBwMUQ1bzB5WWVTNE0wNkFoIiwibSI6eyJyIjoicHJvZC1ldS13ZXN0LTMifX0=
        username: 811005
      url: https://prometheus-prod-22-prod-eu-west-3.grafana.net/api/prom/push
    scrape_configs:
  global:
    scrape_interval: 60s
  wal_directory: /tmp/grafana-agent-wal
`````

En mi caso la instancia se llama `grafana`, pero puedes ponerle `servidor-de-multas` o `nginx` o `ocs_inventory_alzira`. Lo que quieras. Debe ser único, sino luego no se podrá filtrar bien. 

Por último, cada vez que modifiques el fichero `/etc/grafana-agent.yml` tienes que reiniciar el servicio de grafana con el comando `systemctl restart grafana-agent`. El agente ya estaría enviando logs y métricas a la nube.

## Uso.

Ahora para usarlo tienes que crear un dashboard. La integración te facilita todo esto, si le das al botón `Install dashboard and alerts` lo instalará. Para ver los dashboards tienes que usar el menú lateral `Dashboards` y seleccionar el que quieras. Por defecto se organizan en carpetas, pero puedes crear tus propias carpetas y tus propios dashboards con la información que te guste tener.

![Captura de Pantalla 2023-06-01 a las 21 09 04](https://github.com/hector-medina/grafana/assets/76477956/35cf2512-fcad-4701-90a0-eeafb46968b6)

## Alertas.

Las alertas no las tengo muy claras aún, sorry. 
