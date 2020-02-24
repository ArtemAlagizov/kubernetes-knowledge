# some tips motha fucka

## important key indicators to monitor for in a cluster:
  * latency / uptime for your endpoints
  * latency / health checks for your app
  * latency / health checks for your dbs
  * latency / health checks for your nodes
  * we had a bunch of queues, pub sub systems so healthchecks for those
  * we had autoscaling based on queue counts so alerts and charts for that
  * we had slack alerts for increasing and decreasing nodes (make sure you have node anti affinity to make sure pods are spread out so if one node is drained an endpoint doesn’t go down)
  * we had alerts for cpu, ram, disks hitting at 60 and 80%
  * we used stackdriver / bluemedora (works on aws) for alerts / health checks / root cause analysis
