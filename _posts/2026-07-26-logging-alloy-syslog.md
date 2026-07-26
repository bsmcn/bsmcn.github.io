---
title: "Syslog with Grafana Alloy"
last_modified_at: 2026-07-26T19:00:00+02:00
categories:
  - Logging
tags:
  - kubernetes
  - k8s
  - alloy
  - loki
  - syslog
  - cilium
---

## Introduction
Grafana Alloy is available on popular platforms, but there is still one area it hasn't reached yet: network devices. These still cling to the traditional syslog. In this post, you'll see how I expose the syslog service[^1] with the help of Grafana Alloy running on a Kubernetes cluster. Logs are then forwarded to Loki and are browsable with the modern k8s observability stack.

### Note on exposing the service
Syslog is a TCP/UDP service. The setup was created back in the days when the UDPRoute and TCPRoute[^2] were not supported by the Cilium Gateway API. I used NodePort to circumvent that.

## Usage
Configure your network devices to use `<k8s-node-IP>:30514` as the external syslog server.

## Config

The GitHub issue[^3] reports that the common syslog fields are not inserted as labels out of the box, even though the documentation[^1] states otherwise. The relabeling rules insert them back.

The processing rules are used to drop a few unwanted message patterns.

`values.yaml` snippet for the `grafana/alloy`[^4] chart:
```yaml
alloy:
  extraPorts: # exposes on the container and with a service
    - name: syslog
      port: 1514
      targetPort: 1514
      protocol: TCP

  configMap:
    content: |
      loki.source.syslog "syslog_input" {
        listener {
          address = "0.0.0.0:1514"
          protocol = "tcp"
          syslog_format = "rfc3164"
          labels = {
            job = "syslog",
          }
        }

        forward_to = [loki.process.syslog.receiver]
        relabel_rules = loki.relabel.syslog.rules
      }

      loki.relabel "syslog" {
        forward_to = []

        rule {
          source_labels = ["__syslog_message_hostname"]
          target_label  = "hostname"
        }

        rule {
          source_labels = ["__syslog_message_facility"]
          target_label  = "facility"
        }

        rule {
          source_labels = ["__syslog_message_severity"]
          target_label  = "severity"
        }

        rule {
          source_labels = ["__syslog_message_app_name"]
          target_label  = "appname"
        }
      }

      loki.process "syslog" {
        stage.drop {
            expression  = ".*Agent forwarding disabled\\."
        }

        stage.drop {
            expression  = ".*Port forwarding disabled\\."
        }

        stage.drop {
            expression  = ".*Pty allocation disabled\\."
        }

        forward_to = [loki.write.default.receiver]
      }

      loki.write "default" {
        endpoint {
          url = "http://loki-gateway.platform-logging.svc.cluster.local/loki/api/v1/push"
        }
      }
```

NodePort:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: alloy-syslog
  namespace: platform-logging
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: alloy
  ports:
    - port: 1514
      nodePort: 30514
```

## References
[^1]: [loki.source.syslog alloy component](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.syslog/)
[^2]: [Cilium Gateway API: Add TCPRoute and UDPRoute support](https://github.com/cilium/cilium/pull/46184)
[^3]: [loki.source.syslog - no internal labels like __syslog available](https://github.com/grafana/alloy/issues/2266)
[^4]: [Grafana Alloy Helm chart](https://github.com/grafana/alloy/blob/main/operations/helm/charts/alloy/values.yaml)
