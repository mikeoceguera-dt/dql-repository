# DQL Query to get Network usage per host 
---

This DQL Query was created to get the NIC network usage per host in a given time range. It will return the total bytes sent and received by a specific host.

```dql
timeseries by:{dt.entity.host},{
    bytes_rx = avg(dt.host.net.nic.bytes_rx),
    bytes_tx = avg(dt.host.net.nic.bytes_tx),
    filter: dt.entity.host == "host_name_here"
  }
  | fieldsAdd bytes_rx = bytes_rx[]+bytes_tx[]
  | sort arraySum(bytes_rx) desc
  | fieldsAdd hostName = entityName(dt.entity.host)
  | fieldsRemove dt.entity.host, bytes_tx
```
Feel free to use it in ur Dashboards :)



