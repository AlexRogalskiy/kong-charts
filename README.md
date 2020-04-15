This repository contains **demo** [Helm](https://helm.sh/) charts for deploying Kong on Kubernetes backed by Cassandra running within DataStax Astra.

GitHub Actions handle building out the packages and publishing them to Github Pages. Work is underway to upstream the changes to Kong and the charts in to the official [Kong](https://github.com/kong/kong) repositories.

## Installation

Using these charts is as simple as adding this repository to your Helm installation. After that is complete spin up a database in [DataStax Astra](https://www.datastax.com/cloud/datastax-astra) and collect its credentials. From there run the `helm install` command and the appropriate components will be pushed out to your Kubernetes cluster.

```bash
# Set environment variables to be used during helm install
ASTRA_PROXY_URL="CLUSTERID-REGION.db.astra.datastax.com" # hostname from cqlshrc in secure-connect bundle
ASTRA_PORT="3xxxx" # port from cqlshrc in secure-connect bundle
ASTRA_KEYSPACE="your_keyspace"
ASTRA_USERNAME="your_username"
ASTRA_PASSWORD="your_password"

# Create a namespace and config map with files from the secure-connect directory
kubectl create namespace kong
kubectl create configmap -n kong kong-cassandra-cm --from-file secure-connect/

helm repo add datastax-examples-kong https://datastax-examples.github.io/kong-charts/
helm install datastax-examples-kong/kong -n kong \
  --namespace kong \
  --set env.database=cassandra \
  --set env.cassandra_contact_points=$ASTRA_PROXY_URL \
  --set env.cassandra_use_proxy=true \
  --set env.cassandra_cert=/etc/nginx/secure-connect/cert \
  --set env.cassandra_cafile=/etc/nginx/secure-connect/ca.crt \
  --set env.cassandra_port=$ASTRA_PORT \
  --set env.cassandra_ssl=true \
  --set env.cassandra_ssl_verify=true \
  --set env.cassandra_keyspace=$ASTRA_KEYSPACE \
  --set env.cassandra_username=$ASTRA_USERNAME \
  --set env.cassandra_password=$ASTRA_PASSWORD \
  --set env.cassandra_consistency=LOCAL_QUORUM \
  --set image.repository=datastaxlabs/astra-kong \
  --set image.tag=v2.0.2 \
  --set admin.enabled=true \
  --set admin.http.enabled=true
```

### Validate Installation

```bash
# Verify pods, note press Ctrl+C to exit
watch kubectl get pods -n kong

# Kong Admin test
kubectl port-forward -n kong svc/kong-kong-admin 8001:8001 & # Wait a second while the port forwarding comes online
curl  http://localhost:8001/

## Kong Behavior test
curl -i -X POST \
  --url http://localhost:8001/services/ \
  --data 'name=example-service' \
  --data 'url=http://mockbin.org'

curl -i -X POST \
  --url http://localhost:8001/services/example-service/routes \
  --data 'hosts[]=example.com'

curl -i -X GET \
  --url http://`kubectl get svc kong-kong-proxy -n kong | tail -n 1 | awk '{print $4}'`/bin/4bdd1415-1e7e-4713-8f5c-272a9af4858b \
  --header 'Host: example.com' \
  --header 'Accept: application/json'
```

## Support

The code, examples, and snippets provided in this repository are not intended for production use and are not "Supported Software" under any DataStax subscriptions or other agreements.

## License

Refer to the [DataStax Labs Terms](https://www.datastax.com/terms/datastax-labs-terms) for full details on the license and terms related to the use of the software described here.
