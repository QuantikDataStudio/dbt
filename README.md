# Le jeu de données Airbnb
## Source: 
https://insideairbnb.com/get-the-data/

## Traitement
1. Division du fichier `listings.csv.gz` en 2 fichiers:
   1. [listings](dataset/listings.csv) avec un nombre de colonnes réduit et qui ne contient que les donneées qui 
   touchent directement au listing (i.e. on a enlevé les données de l'hôte et sur les revues) 
   2. [hosts](dataset/hosts.csv) ce fichier, extrait du fichier `listings.csv.gz`, ne contient que les infos concernant 
   l'hoôte. Ici aussi, nous avons limité le nombre de colonnes par rapport à toutes les infos qu'on avait
2. [reviews](dataset/reviews.csv) ce fichier provient du fichier `listings.csv.gz` et la seule modification qu'on y a 
apportée c'est de limiter le nombre de caractères dans le texte du commentaire pour limiter la taille du fichier