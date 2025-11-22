# exec to halyard pod 
kubectl -n opsmx-oss exec -it oss-spin-spinnaker-halyard-0 -- bash

# inside the pod:
mkdir -p /home/spinnaker/.hal/default/profiles
cat >/home/spinnaker/.hal/default/profiles/echo-local.yml <<'YAML'
rest:
  enabled: true
  endpoints:
    - url: http://fluentbit-http.collection.svc.cluster.local:8080/
      wrap: false
YAML

# still inside the pod
hal deploy apply

# tail echo logs, watch for POST updates
kubectl -n opsmx-oss logs deploy/spin-echo -f | grep -i rest