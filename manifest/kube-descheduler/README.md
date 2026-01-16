# Kube-Descheduler for Virtual Machine Load-Aware Workload Balancing

Load-aware workload balancing When used with the KubeVirtRelieveAndMigrate profile, the descheduler enables load-aware rebalancing of virtual machines by continuously evaluating node CPU utilization and pressure to identify nodes that are overutilized. When a node exceeds the desired deviation thresholds, the descheduler triggers live migration of eligible VMs.

By applying:

```yaml
apiVersion: operator.openshift.io/v1
kind: KubeDescheduler
metadata:
  name: cluster
  namespace: openshift-kube-descheduler-operator
spec:
  managementState: Managed
  deschedulingIntervalSeconds: 300
  mode: Automatic
  profiles:
    - KubeVirtRelieveAndMigrate
  profileCustomizations:
    devEnableSoftTainter: true
    devDeviationThresholds: AsymmetricMedium
    devActualUtilizationProfile: PrometheusCPUCombined
```

The descheduler will now automatically take action to proactively migrate workload away from nodes classified as overutilized. With the AsymmetricMedium profile, a node with utilization 20% or more above cluster average is considered overutilized. AsymmetricLow and AsymmetricHigh change that threshold to 10% and 30%, respectively. Use the option that most closely matches the needs and expectation for how quickly the descheduler will take action in the event of an imbalance to node utilization.