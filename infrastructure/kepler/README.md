# Kepler pilot (homelab only)

## Feasibility and scope

The homelab has four amd64 Talos Linux v1.13.2 nodes (`master01`, `worker01`–`worker03`), running kernel 6.18.29. Talos v1.13 includes the Powercap/Intel RAPL kernel support, and the nodes carry the Intel microcode extension label. This is a good candidate for a hardware-meter pilot, but Kubernetes node metadata does not prove that each node exposes a readable RAPL or hwmon energy sensor.

The edge-vps cluster is intentionally excluded: it is a VPS/virtualized environment and does not provide trustworthy physical host energy counters. Kepler v0.12's current design reads hardware meters; no trained VM-estimator path is included in this deployment.

The Grafana k8s-monitoring chart's bundled Kepler dependency is pinned upstream at chart 0.6.1 (Kepler app release-0.8.0). This configuration disables that bundled copy and deploys the standalone Kepler OCI chart/image v0.12.0 instead, which is the post-eBPF re-architecture.

## Staged rollout

1. Preflight `worker01` from a trusted Talos admin workstation. Check whether the host exposes a readable and increasing RAPL counter, for example:

   `talosctl -n worker01 read /sys/class/powercap/intel-rapl/intel-rapl:0/energy_uj`

   Repeat after several seconds. If this path is absent, check whether an hwmon power/energy sensor exists. Do not enable the fake meter in the real monitoring path.

2. The HelmRelease initially schedules Kepler only on `worker01`. It uses RAPL first and hwmon as fallback, exports only node and pod metrics, and leaves the edge-vps collector unchanged. The pod uses hostPID plus host `/proc` and `/sys` read-only mounts; the upstream chart defaults its container to privileged, so this is a high-trust node agent even though the pilot is limited to one worker.

3. After the HelmRelease is reconciled, verify the pod is Ready and inspect its startup logs for the selected meter. In VictoriaMetrics/Grafana Explore, check:

   `kepler_node_cpu_watts{cluster="homelab",zone="package"}`

   `kepler_node_cpu_joules_total{cluster="homelab",zone="package"}`

   `kepler_pod_cpu_watts{cluster="homelab",zone="package"}`

   Confirm the node series names `worker01`, the counters increase, and the power values move plausibly during a controlled CPU load. The support kubeconfig in this environment cannot read Pods/DaemonSets, so runtime validation must be done by an operator with the needed read access.

4. Only after the pilot works on Talos should you remove `kubernetes.io/hostname: worker01` from the HelmRelease nodeSelector to expand to all homelab nodes. Keep Kepler disabled on edge-vps.

5. If the pilot reports no usable power meter or crashes, do not expand it. Revert the Kepler HelmRelease and homelab monitoring patch through GitOps.

## Metric interpretation

- `kepler_node_cpu_watts` and `kepler_node_cpu_joules_total` with `zone="package"` measure CPU package power/energy. They are not whole-server/chassis power; a BMC/Redfish or external PDU is needed for that.
- Pod CPU power/energy is attributed from node measurements and workload CPU activity; it is an estimate, not a physical per-Pod meter. Use it for relative trends and experiments, not billing, chargeback, regulatory reporting, or financial accounting without local calibration.
- Kepler remains a privileged host-level collector in this chart. Do not install this pilot into a banking production cluster without a separate security review and a tested least-privilege alternative.

## GitOps behavior

The homelab infrastructure Kustomization owns the HelmRelease and the hostMetrics scrape patch. Flux pruning the HelmRelease uninstalls Kepler. No direct `kubectl apply`, commit, or push was performed.
