Prometheus - Monitoring system & time series database


# intro

  * pull data from client server (maybe 60s)
  * push gateway for short lived job (push metrics at exit)

  * Pulling over HTTP offers a number of advantages:

       * You can run your monitoring on your laptop when developing changes.
       * You can more easily tell if a target is down.
       * You can manually go to a target and inspect its health with a web browser.
  *  works well for recording any purely numeric time series

  * not fit for If you need 100% accuracy

  * Prometheus is a system to collect and process metrics, not an event logging system. 
       * blog post [Logs and Metrics and Graphs, Oh My!](https://blog.raintank.io/logs-and-metrics-and-graphs-oh-my/) provides more details about the differences between logs and metrics.

  *  https://github.com/yunlzheng/prometheus-book

Metric names and labels
Every time series is uniquely identified by its metric name and optional key-value pairs called labels.




# local test shell


 * nohup  /Users/xxx/server/node_exporter-1.0.0/node_exporter   &


* nohup  /Users/xxx/server/prometheus-2.18.1/prometheus   --config.file=/Users/xxx/server/prometheus-2.18.1/prometheus.yml   &


* nohup  /Users/xxx/server/grafana-6.7.3/bin/grafana-server  --config /Users/xxx/server/grafana-6.7.3/conf/defaults.ini  --homepath /Users/xxx/server/grafana-6.7.3 &


Gauge    delta

Counter   increase


rate   =  increase/time