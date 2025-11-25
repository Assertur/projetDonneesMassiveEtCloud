# projet de Donnees Massive Et Cloud

### 1. Présentation

Ce projet est l'analyse d'un petit réseaux sociale, sur le plan de la scalabilité.
On peut retrouver l'application web à l'adresse : https://massivedataprojectuniv.ey.r.appspot.com
Elle est entièrement gérer par Google Cloud.

### 2. Analyse utilisateurs concurents

Cette première analyse a pour objectif de voir si l'application scale sur le nombre d'utilisateur connecté simultanément.
Afin que mon terminal Google Cloud puisse accéder à mon datastore et donc le peupler et le vider, j'ai créer une clé lié au datastore pour le terminal.
Pour cela j'ai d'abord peupler ma base en utilisant la commande :

```
python ./massive-gcp/seed.py --users 1000 --posts 50000 --follows-min 20 --follows-max 20
```

Une fois la base peuplé j'ai pu appliqué un petit script utilisant apach-bench, qui va envoyer une requête en concurrence 3 fois pour chaque nombre d'utilisateurs simultanés.
```
#!/bin/bash

URL="http://127.0.0.1:5000/api/timeline?user=user"

CONCURRENCIES=(1 10 20 50 100)
REPEATS=3

mkdir -p results

for c in "${CONCURRENCIES[@]}"; do
  for r in $(seq 1 $REPEATS); do
    echo "Running concurrency=$c, run=$r"

    # fichier pour stocker les temps par utilisateur
    OUTFILE="results/concurrency_${c}_run${r}.txt"
    > $OUTFILE 

    # lancer pour chaque utilisateur distinct simultanément
    for u in $(seq 1 $c); do
      # -n fixé à 100 par utilisateur (doit être >= -c)
      ab -n 100 -c $c "${URL}${u}" 2>&1 | grep "Time per request" >> $OUTFILE &
    done
    wait

    # calculer moyenne automatiquement
    awk '{ sum += $4 } END { print "Moyenne (ms):", sum/NR }' $OUTFILE
  done
done
```

Je n'ai pas fait l'occurence de 1000 car avec apach-bench on doit avoir un -n >= au -c et avoir un -n supérieur à 100 est fortement déconseillé.

Pour 1000 j'ai donc fait un autre script qui permet de lancer presque de façon simultané 1000 utilisateurs:

```
for g in {1..10}; do
  for u in $(seq $(( (g-1)*100 + 1 )) $(( g*100 )) ); do
    ab -n 10 -c 10 "http://127.0.0.1:5000/api/timeline?user=user$u" 2>&1 | grep "Time per request" >> results/concurrency_1000_run1.txt &
  done
  wait
done
```

Pour compter la moyenne, j'ai utilisé la commande suivante qui renvoie la moyenne :
```
grep "mean, across all concurrent requests" results/concurrency_1000_run1.txt | awk '{ sum += $4; n++ }
END { if (n>0) print sum/n; else print "0" }'
```

Les résultats obtenus ont montré qu'il n'y avait de différence que pour 1000 qui est vraiment plus long.

![Graphique Analyse Concurrent](./graphics/Temps%20par%20nombre%20d'utilisateur%20concurent.png)

### 3. Analyse nombre de post par utilisateur

Pour cette analyse il a fallu supprimer et recréer les données à chaque fois. Pour cela j'ai un script que je lance pour supprimer mes données depuis la console de Google Cloud.

```
from google.cloud import datastore

project_id = "massivedataprojectuniv"
client = datastore.Client(project=project_id)

# Liste des kinds
query = client.query(kind="__kind__")
kinds = [entity.key.id_or_name for entity in query.fetch()]

for kind in kinds:
    query = client.query(kind=kind)
    keys = [entity.key for entity in query.fetch()]
    if keys:
        client.delete_multi(keys)
        print(f"Supprimé {len(keys)} entités du kind {kind}")
```

J'ai repeuplé ensuite la base avec la même fonction en variant le nombre de post.
Pour 10 posts par utilisateur :

```
python ./massive-gcp/seed.py --users 100 --posts 1000 --follows-min 20 --follows-max 20
```

Pour 100 posts par utilisateur :

```
python ./massive-gcp/seed.py --users 100 --posts 10000 --follows-min 20 --follows-max 20
```

Pour 1000 posts par utilisateur :

```
python ./massive-gcp/seed.py --users 100 --posts 100000 --follows-min 20 --follows-max 20
```

Pour tester cette fois-ci je suis passé par curl avec la requête suivante, en changeant le nom du fichier avec les résultats à chaque fois :
```
for u in $(seq 1 50); do
  curl -s -o /dev/null -w "%{time_total}\n" \
  "https://massivedataprojectuniv.ey.r.appspot.com/api/timeline?user=user$u" >> results/post_10_run1.txt &
done
wait
```

J'ai ensuite calculé la moyenne avec la commande suivante : 
```
awk '{sum += $1} END {if (NR > 0) printf "%.2f ms\n", (sum/NR)*1000; else print "no data"}' results/post_10_run1.txt
```

J'ai fait le graphique suivant pour rendre les résultats parlant :
![Graphique Analyse Post](./graphics/Temps%20par%20nombre%20de%20post.png)

On peut voir qu'en moyenne ça ne change pas trop selon le nombre de post par user. Cependant le premier run est environ 2 fois plus long que les deux autres.

### 4. Analyse nombre de followee par utilisateur

Il a fallu à nouveau vider le datastore, avec la même fonction, et le repeupler.

Pour 10 followee par utilisateur :

```
python ./massive-gcp/seed.py --users 100 --posts 10000 --follows-min 10 --follows-max 10
```

Pour 50 followee par utilisateur :

```
python ./massive-gcp/seed.py --users 100 --posts 10000 --follows-min 50 --follows-max 50
```

Pour 100 followee par utilisateur :

```
python ./massive-gcp/seed.py --users 100 --posts 10000 --follows-min 100 --follows-max 100
```

J'ai ensuite lancer les requêtes via curl comme pour l'analyse par post, et calculé la moyenne de la même façon.

J'ai reporté les données pour faire un graphique.
![Graphique Analyse Followee](./graphics/Temps%20par%20nombre%20de%20followee.png)
On peut voir que le nombre de followee change radicalement le temps et que l'application ne scale donc pas dessus.

### 5. Conclusion

L'application ne scale donc pas sur le nombre d'utilisateur simultané à partir de 1000 et ne scale pas non plus sur le nombre de followee. Cependant elle n'a aucun problème à gérer le nombre de post.
