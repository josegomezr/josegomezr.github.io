---
title: Composing helm deployments
layout: post
tags: [blog, helm, k8s]
categories: [blog]
lang: 
---

Finally understood how to make a helm deployment use another chart and configure it! I'll show you how

<!-- more -->

Your main chart in the `Chart.yaml` file needs to specify dependencies in the `dependencies` block like so:

```yaml
# Chart.yaml
dependencies: # A list of the chart requirements (optional)
  - name: prometheus-pushgateway # <- important!
    version: ^2.13.0
    repository: https://prometheus-community.github.io/helm-charts
    tags: # (optional)
      - pushgateway
```

⚠ Remember to `helm dependency build`!

Then in your values.yaml, a root-level key with the same name as your dependency (see the `# <- important!` remark) will provide values to the referred chart.

```yaml
# values.yaml
prometheus-pushgateway:
  serviceAccount:
    create: no
```

And that will be passed down to the `prometheus-pushgateway` chart effectivelty not rendering the service account yaml file.

You can always validate it with `helm template`

```bash
helm template mychart -s charts/prometheus-pushgateway/templates/serviceaccount.yaml helm-chart/
Error: could not find template charts/prometheus-pushgateway/templates/serviceaccount.yaml in chart
```
