## Grafana monitor

1. deploy grafana, influxDB, Cadvisor with `docker stack deploy -c docker-compose-monitor.yml monitor`
2. create cadvisor database in influxDB container(`docker exec -it <influxDB container ID> bash`) with `influx -execute 'cadvisor'`
3. in browser access grafana dashboard with `ip:80`
4. login with `user: admin and password: admin` and change password after first login
5. add source: change Name is `influx`, Type is `influxDB`, Url is `http://influx:8086` and Database is `cadvisor` and then click `save and test` button
6. import dashboard in `Manage` button(in left panel) with `swarm-dashboard_rev1.json`
7. you will see a docker container dashboard

## Chonograf(influxDB dashboard)
port 9001

### Reference
`https://botleg.com/stories/monitoring-docker-swarm-with-cadvisor-influxdb-and-grafana/`
