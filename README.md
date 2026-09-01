# OpenShift Blue-Green Deployment

Blue-Green deployment strategy for `httpd-app`, using two independent Deployments (blue and green) and a switchable "live" Service.

## Architecture

- **app-blue** — Deployment running the current live version (v1)
- **app-green** — Deployment running the new version (v2)
- **app-blue-svc** / **app-green-svc** — internal Services for testing each color independently, without affecting live traffic
- **app-live-svc** — the Service the Route actually points to; its selector is flipped between `blue` and `green` to switch live traffic
- **myapp-route** — Route exposing the app externally, always pointing to `app-live-svc`; it never changes during a cutover

## Repository structure
├── base/
│ ├── live-service.yaml # Service the Route points to (selector flips blue/green)
│ ├── route.yaml # Route exposing the app externally
│ └── kustomization.yaml
├── overlays/
│ ├── blue/
│ │ ├── blue-deployment.yaml
│ │ ├── blue-service.yaml
│ │ └── kustomization.yaml
│ └── green/
│ ├── green-deployment.yaml
│ ├── green-service.yaml
│ └── kustomization.yaml
└── README.md

`overlays/blue` and `overlays/green` each pull in the shared `base/` (live Service + Route) and add only their own color's Deployment and Service.

## Deploy

Deploy only blue:
```bash
oc new-project bluegreen-demo
oc apply -k overlays/blue
```

Deploy only green:
```bash
oc apply -k overlays/green
```

Deploy both (for side-by-side testing before cutover):
```bash
oc apply -k overlays/blue
oc apply -k overlays/green
```

## Test the new (green) version before cutover

```bash
oc port-forward svc/app-green-svc 8081:8080
curl localhost:8081
```

## Cut traffic over to green

```bash
oc patch svc app-live-svc -p '{"spec":{"selector":{"app":"myapp","version":"green"}}}'
```

## Rollback to blue

```bash
oc patch svc app-live-svc -p '{"spec":{"selector":{"app":"myapp","version":"blue"}}}'
```

## Retire old version

Once green is confirmed stable:
```bash
oc delete deployment app-blue
```
