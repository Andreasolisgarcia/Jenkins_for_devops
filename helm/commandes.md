# Useful commands

## Helm
helm template movie-service ./charts \
  -f ./values-movie.yaml \
  -f ./values-movie-secret.yaml

  helm template cast-service ./charts \
  -f ./values-cast.yaml \
  -f ./values-cast-secret.yaml

helm install movie-service ./charts \
  -f ./values-movie.yaml \
  -f ./values-movie-secret.yaml

## Debug
kubectl exec -it movie-service-db-0 -n dev -- env | grep POSTGRES

## Utils
echo "bW92aWVfZGJfcGFzc3dvcmQ=" | base64 --decode
