# HomepageSC

HomepageSC provides Kubernetes Ingress support to [Homepage by Ben Phelps](https://github.com/benphelps/homepage).

Running as a side car container to your HomePage container, this side car enumerates all Ingresses in your Kubernetes cluster, building a [service.yaml](https://gethomepage.dev/en/configs/services/) fully populated with all your endpoints.

## Usage

Deploy Homepage and HomepageSC in a single deployment with a shared volume for them to exchange their configuration.

```yaml
## Deploy Homepage and HomepageSC with a shared config volume
apiVersion: apps/v1
kind: Deployment
metadata:
  name: homepage
  namespace: homepage
  labels:
    app: homepage
spec:
  selector:
    matchLabels:
      app: homepage
  template:
    metadata:
      labels:
        app: homepage
    spec:
      volumes:
        - name: shared-data
          emptyDir: {}
      serviceAccountName: homepage-discovery
      containers:
        - name: homepage
          image: 'ghcr.io/benphelps/homepage:latest'
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 3000
              protocol: TCP
          volumeMounts:
            - mountPath: "/app/config" # Mount the shared config directory
              name: shared-data
        - name: homepagesc
          image: 'ghcr.io/uatec/homepagesc:latest'
          volumeMounts:
            - mountPath: "/app/config" # Mount the shared config directory
              name: shared-data
          env:
            - name: OUTPUTLOCATION # Tell HomepageSC to output services in to the shared directory
              value: "/app/config/services.yaml"
            - name: INCLUSTER # Use kubeconfig provided by the cluster itself
              value: "true"


# Grant HomepageSC permission to discover ingresses and widget secrets
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: homepage-discovery
  namespace: homepage
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: homepage-discovery
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: homepage-discovery
  namespace: homepage
```

Configure how your ingress appears with annotations to give Homepage further information about them.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ghost
  namespace: publishing
  annotations:
    homepagesc.io/appName: Awesome Blog
    homepagesc.io/group: Publishing
    homepagesc.io/healthCheck: http://ghost.publish.svc.cluster.local:2368/ghost/api/v3/admin/site
    homepagesc.io/icon: https://github.com/walkxcode/dashboard-icons/raw/main/png/ghost.png
spec:
  rules:
  - host: www.awesomeblog.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              name: http
```

### Annotations

HomepageSC exposes extra Homepage configuration options using ingress annotations. These annotations broadly mirror the [service configuration](https://gethomepage.dev/en/configs/services/) of Homepage.

If there is only one host/path in the ingress all the annotations below can be used as is. If there are *multiple host/paths* then you will need to set the annotations per host/path. The way this works is by altering the annotations following this format: `hompagesc.io/<service_name>_<service_port>_<annotation_label>`
For example, if your path forwards traffic to a service called `my-service` on port number `1234` you would update the annotations like to something like this `homepagesc.io/my_service_1234_enable`

*NOTE: the service name will be normalised to use underscores to replace anything that isn't an alphanumeric character. This is to ensure it's valid in an annotation key.*

| Annotation                                   | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
|----------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `homepagesc.io/enable`              | Override the default enabling behaviour by setting this to `true` to show your service or `false` to hide it.                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `homepagesc.io/appName`             | A custom name for your application. Use if you don't want to use the name of the ingress.                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| `homepagesc.io/group`               | A custom group name. Use if you want the application to show in a different group than the namespace it is running in.                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `homepagesc.io/icon`                | Services may have an icon attached to them, you can use icons from [Dashboard Icons](https://github.com/walkxcode/dashboard-icons) automatically, by passing the name of the icon, with, or without `.png`. You can also specify icons from [material design icons](https://materialdesignicons.com) with `mdi-XX`. If you would like to load a remote icon, you may pass the URL to it. If you would like to load a local icon, first create a Docker mount to `/app/public/icons` and then reference your icon as `/icons/myicon.png`. |
| `homepagesc.io/description`         | Use to provide a subtitle or other additional info to the service tile.                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| `homepagesc.io/url_path`            | Manually set the url path for the service. This can be required if the path contains regex making it unusable as a straight url or if the path is not where you want to link to from homepage.                                                                                                                                                                                                                                                                                                                                           | 
| `homepagesc.io/target`              | Changes the behaviour of links on the homepage. Possible options include `_blank`, `_self`, and `_top`. Use _blank to open links in a new tab, _self to open links in the same tab, and _top to open links in a new window.                                                                                                                                                                                                                                                                                                              | 
| `homepagesc.io/healthCheck`         | Services may have an optional ping property that allows you to monitor the availability of an endpoint you chose and have the response time displayed. You do not need to set your ping URL equal to your href URL.                                                                                                                                                                                                                                                                                                                      |
| `homepagesc.io/widget_type`         | Use to specify the type of widget to be rendered for this service. Widget's are documented on the [service widget](https://gethomepage.dev/en/configs/service-widgets/) page.                                                                                                                                                                                                                                                                                                                                                            |
| `homepagesc.io/widget_secret`       | Specify a widget secret to be added to the `key` field of the widget config. The secret locator format is `{namespace}/{secretName}/{secretField}`.                                                                                                                                                                                                                                                                                                                                                                                      |
| `homepagesc.io/widget_url`          | Specify a widget url if it's different from the ingress host/path.                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `homepagesc.io/widget_username_and_password_secret`          | Specify a widget secret that contains `username` and `password` keys for widgets that require username and password. format: `{namespace}/{secretName}`.                                                                                                                                                                                                                                                                                                                                                                                      |

### Settings

HomepageSC or global service settings can be configured using environment variables or command line parameters.

| Env Var            | Command Line                                              | Default       | Description                                                                                                                                                                                                |
|--------------------|-----------------------------------------------------------|---------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `INCLUSTER`        | `--InCluster` or `--InCluster=[true,false]`               | `FALSE`       | By default HomepageSC runs in development mode, using your configured kubeconfig. Set this to tell HomepageSC to retrieve config from the cluster it is running in.                                        |
| `INCLUDEBYDEFAULT` | `--IncludeByDefault` or `--IncludeByDefault=[true,false]` | `true`        | Defines the behaviour for ingresses which do not have the `homepagesc.io/enable` annotation. Set to false to require services to be explicitly enabled.                                           |
| `OUTPUTLOCATION`   | `--OutputLocation=/app/config/service.yaml`               | `undefined`   | Instruct HomepageSC to write configuration to a location on disk. This should be `service.yaml` within the HomePage configuration directory. Default behaviour is to write config to stdout to validation. |
| `DEFAULTTARGET`    | `--DefaultTarget=[_blank,_self,_top]`                     | `undefined`   | Change the behaviour of links for all services (which do not specify their own `homepagesc.io/target` annotation).                                                                                |


## Contributing

Pull requests are welcome. For major changes, please open an issue first
to discuss what you would like to change.

Please make sure to update tests as appropriate.

## License

[MIT](https://choosealicense.com/licenses/mit/)

## Roadmap
- [x] swappable kubeconfig (commmand line flag)
- [x] configurable output directory
- [ ] include manual config
- [x] ping
  - [x] fully qualify back end name
- [x] icon
  - [x] test image config
- [ ] homepage to support HTTP 204 as success
- [x] enable services
  - [x] from annotation
  - [x] default from config
- [x] targets
  - [x] from annotations
  - [x] default from config
- [ ] widgets
  - [x] basic
  - [x] secrets
    - [x] key
    - [ ] multi field
- [x] https ingresses
- [x] move to own namespaced attributes