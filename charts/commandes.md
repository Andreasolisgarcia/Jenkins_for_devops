kubectl create configmap db-configmap \
  --from-literal=POSTGRES_USER=movie_db_username \
  --from-literal=POSTGRES_DB=movie_db_dev \
  --dry-run=client -o yaml > configmap.yaml


kubectl create secret generic db-secret \
  --from-literal=POSTGRES_PASSWORD=movie_db_password \
  --dry-run=client -o yaml > secret.yaml

  echo "bW92aWVfZGJfcGFzc3dvcmQ=" | base64 --decode

  kubectl exec -it movie-service-db-0 -n dev -- env | grep POSTGRES

  kubectl exec        # exécuter une commande dans un pod
-it                 # interactif + terminal (comme docker exec)
movie-service-db-0  # nom du pod
-n dev              # dans le namespace dev
--                  # séparateur : ce qui suit c'est la commande à exécuter
env                 # commande Linux qui affiche toutes les variables d'environnement
| grep POSTGRES     # filtre uniquement les lignes contenant "POSTGRES"

helm template movie-service ./charts -f ./charts/values-movies.yaml --debug

##format PG Uri de la Base de données
