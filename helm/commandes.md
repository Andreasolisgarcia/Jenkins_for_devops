# Useful commands

## Helm
helm template movie-service ./charts \
  -f ./values-movies.yaml \
  -f ./values-secret.yaml

helm install movie-service ./charts \
  -f ./values-movies.yaml \
  -f ./values-secret.yaml

## Debug
kubectl exec -it movie-service-db-0 -n dev -- env | grep POSTGRES

## Utils
echo "bW92aWVfZGJfcGFzc3dvcmQ=" | base64 --decode
