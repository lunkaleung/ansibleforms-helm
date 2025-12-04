# AnsibleForms Helm Chart

This Helm chart deploys the AnsibleForms application and its MySQL database on Kubernetes. It is designed for flexibility, security, and ease of use in both development and production environments.

## Features
- Deploys AnsibleForms and MySQL with configurable images and resources
- Handles all sensitive data (DB credentials, admin credentials, encryption secret) via Kubernetes Secrets
- All application environment variables are configurable via `values.yaml`
- Storage class and size for both server and MySQL are configurable
- Ingress is optional and highly customizable (hostname, TLS, annotations, etc.)
- Service type (ClusterIP, LoadBalancer, NodePort) is configurable, with support for static LoadBalancer IPs
- (Optional) Support for managing `forms.yaml` and forms/*.yaml definitions via ConfigMaps

## Usage

### 1. Clone the helm chart and values.yaml

```
git clone https://github.com/ansibleguy76/ansibleforms-helm
cp ./ansibleforms-helm/values.yaml my_values.yaml
```

### 2. Configure personal values

Update the my_values.yaml to your taste

#### minimalistic example without ingress:
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

#### Extended Example with ingress:
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



### 3. Using ConfigMaps for `forms.yaml` and form definitions (optional)

AnsibleForms uses a main configuration file (`forms.yaml`) and can also load additional form definitions from a `forms/` directory inside the persistent folder.

This chart provides optional values to mount those files from ConfigMaps:

```yaml
forms:
  configMap:
    enabled: true
    name: ansibleforms-forms                # ConfigMap containing the main forms.yaml
    key: forms.yaml                         # key in the ConfigMap
    mountPath: /app/dist/persistent/forms.yaml

  extraFormsConfigMap:
    enabled: true
    name: ansibleforms-forms-defs          # ConfigMap containing multiple form YAMLs
    mountPath: /app/dist/persistent/forms  # mounted as a directory
```

When `forms.configMap.enabled` is `false` (the default), the chart behaves as before and does not mount any extra ConfigMap. The same applies to `extraFormsConfigMap.enabled`.

#### Example: main forms configuration (`forms.yaml`)

The following ConfigMap provides the main `forms.yaml` file, which defines categories, roles, and constants:

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: ansibleforms-forms
  namespace: ansibleforms
data:
  forms.yaml: |
    categories:
      - name: Default
        icon: bars
      - name: Demo
        icon: heart
      - name: Maintenance
        icon: cogs
      - name: Internal
        icon: cogs
      - name: Vmware
        icon: cogs
    roles:
      - name: admin
        groups:
          - local/admins
          - ldap/k8admins
      - name: demo
        groups:
          - local/demo
      - name: public
        groups: []
      - name: internal
        groups:
          - ldap/internal
        users:
          - ldap/FRoca
    constants:
      AF_PLAYBOOKS: /app/dist/persistent/playbooks
```

With the `forms.configMap` values set as shown earlier, this `forms.yaml` will be mounted at `/app/dist/persistent/forms.yaml`, which is the default location used by AnsibleForms.

#### Example: additional form definitions (`forms/*.yaml`)

You can keep each form definition in a separate YAML file within a second ConfigMap. Each key under `data:` becomes a file inside `/app/dist/persistent/forms/`:

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: ansibleforms-forms-defs
  namespace: ansibleforms
data:
  awx-call-user_creation.yaml: |
    name: User Creation
    type: awx
    template: "jt-or_myawesomeor-create_user"
    roles:
      - internal
    categories:
      - Internal
    help: "Form to create FTP users"
    description: "Launch ftp user creation"
    fields:
      - name: survey_user
        label: Username
        type: text
        required: true
      - name: survey_password
        label: Password
        type: text
        required: true
      - name: survey_type
        label: Type
        type: enum
        values:
          - name: general
            value: general
          - name: av
            value: av
        required: true

  # more forms here, one per key:
  # otherform.yaml: |
  #   name: ...
  #   ...
```


### 3. Install the Chart

Depending on your environment, choose an existing namespace or choose to create a new one.

```sh
helm install ansibleforms ./ansibleforms-helm -f ./my_values.yaml -n ansibleforms --create-namespace
```

## Security Best Practices
- Use `--set` or `--set-file` to hide secrets

## Customization
- All environment variables can be set in `env:`.
- All resource limits, storage, and service types are configurable.
- Ingress can be enabled/disabled and fully customized.

## Notes
- Pods/services are named and discoverable by their service name within the namespace.

---
For a full list of environment variables and their meanings, see the comments in `values.yaml` or visit [ansibleforms.com](https://ansibleforms.com).
