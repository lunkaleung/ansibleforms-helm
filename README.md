# AnsibleForms Helm Chart

This Helm chart deploys the AnsibleForms application and its MySQL database on Kubernetes. It is designed for flexibility, security, and ease of use in both development and production environments.

## Features
- Deploys AnsibleForms and MySQL with configurable images and resources
- Handles all sensitive data (DB credentials, admin credentials, encryption secret) via Kubernetes Secrets
- All application environment variables are configurable via `values.yaml`
- Storage class and size for both server and MySQL are configurable
- Ingress is optional and highly customizable (hostname, TLS, annotations, etc.)
- Service type (ClusterIP, LoadBalancer, NodePort) is configurable, with support for static LoadBalancer IPs

## Usage

### 1. Configure personal values
Create a project folder and create a `my_values.yaml` file.
Edit your `my_values.yaml` to set your desired configuration. For sensitive values, create a private file (e.g. `my-secrets.yaml`) and override them at install time.

#### minimalistic example `values.yaml` without ingress:
```yaml
application:
  server:
    env:
      HTTPS: 1 # auto sets port to 443
      ENCRYPTION_SECRET: "Abc123Abc123Abc123Abc123Abc123Abc1" # optional but recommended, just a 32 char ramdom string

# storage: # not provided -> default storageclass

# container: # not provided -> latest v6 beta

service:
  server:
    type: LoadBalancer
    loadbalancer:
      ip: 10.0.0.1      # the ip to serve ansibleforms https://10.0.0.1
```

#### Extended Example `values.yaml` with ingress:
```yaml
application:
  server:
    env:
      HTTPS: 0 # auto sets ports on 80 or 443
      # ...other env vars (see values.yaml for all options)
      ENCRYPTION_SECRET: "Abc123Abc123Abc123Abc123Abc123Abc1"
      ADMIN_USERNAME: admin
      ADMIN_PASSWORD: MyAppPassword1!!
  mysql:
    user: root
    password: MyDbPassword1!!

storage:
  server:
    className: nfs-csi       # just an example storage provider
    size: 1Gi
  mysql:
    className: nfs-csi-nomap # just an example storage provider
    size: 1Gi

container:
  server:
    image: ansibleguy/ansibleforms:betav6
    resources:
      limits:
        cpu: "0.5"
        memory: 512Mi
      requests:
        cpu: "0.25"
        memory: 256Mi
  mysql:
    image: mysql:latest
    resources:
      limits:
        cpu: "0.5"
        memory: 512Mi
      requests:
        cpu: "0.25"
        memory: 256Mi

service:
  server:
    type: ClusterIP # service is exposed with ingress
  mysql:
    type: ClusterIP

ingress:
  enabled: true # enable ingress
  className: nginx
  hostname: ansibleforms.example.com
  path: /
  pathType: Prefix
  tls:
    enabled: true
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
  issuer: letsencrypt-prod
  extraHosts: [] # ["other.example.com"]
  extraPaths: [] # [{ path: /api, serviceName: server, servicePort: 80 }]
  extraAnnotations: {}  
```

### 3. Install the Chart
```sh
helm install ansibleforms ./ansibleforms-helm -f my_values.yaml -n ansibleforms
```

## Security Best Practices
- Never commit real secrets to `values.yaml` or your repo.
- Use a private override file for secrets and add it to `.gitignore`.
- Use `--set` or `--set-file` for secrets in CI/CD.

## Customization
- All environment variables can be set in `env:`.
- All resource limits, storage, and service types are configurable.
- Ingress can be enabled/disabled and fully customized.

## Notes
- Pods/services are named and discoverable by their service name within the namespace.

---
For a full list of environment variables and their meanings, see the comments in `values.yaml` or visit [ansibleforms.com](https://ansibleforms.com).
