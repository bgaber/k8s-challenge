The Helm Chart structure for this project was created by running the following command:

```
helm create k8s-challenge
```

Then the Chart.yaml, values.yaml and all manifest files in the /templates directory were modified for this project.

The k8s manifests files in the /templates directory were validated using this command:

```
helm template k8s-challenge --debug
```

The application was installed with this Helm Chart command:

```
helm install k8s-challenge k8s-challenge
```

The application was uninstalled with this Helm Chart command:

```
helm install k8s-challenge
```