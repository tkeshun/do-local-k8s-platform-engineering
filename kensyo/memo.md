ingress-nginxの動作検証


これで、control-planeに配置されるようになるらしい。
  tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/control-plane
    operator: Exists
