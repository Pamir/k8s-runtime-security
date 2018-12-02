```bash
gcloud beta container clusters create \
    --zone us-central1-a \
    twistlock-cluster

gcloud container clusters get-credentials \
    --zone us-central1-a \
     twistlock-cluster
wget -O twistlock.tar.gz https://cdn.twistlock.com/releases/fc650abd/twistlock_18_11_96.tar.gz
tar -zxf twistlock.tar.gz

./twistcli console export kubernetes --service-type LoadBalancer


```
Output of the command will be like the text below

```text
Enter Twistlock access token (required for pulling the Console image): 
neither storage class nor persistent volume labels were provided, using cluster default behavior
Saving output file to /Users/x/dev/tools_data/twistlock/osx/twistlock_console.yaml
```

Install console
```bash
kubectl apply -f twistlock_console.yaml
kubens twistlock
kubectl get svc 

```

