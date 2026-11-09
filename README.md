# BIRD : images et sémantique visuelle

Le programme parcourt les **359 œuvres**, sans phase pilote ni limite à 20.
Il utilise l'API OpenAI (clé API et crédits API requis), indépendamment de
l'abonnement ChatGPT/Codex. Il ne lance pas Codex ni une session de navigateur.

## Lancement Windows

1. Cloner le dépôt.
2. Copier **`.env.example`** en **`.env`** (à côté de `analyse_images.py`), puis remplacer
   uniquement `COLLE_TA_CLE_ICI` par la clé API. `.env` est ignoré par git : ne jamais le commiter.
   Les clés se gèrent sur https://platform.openai.com/api-keys.
3. Dans PowerShell, ouvert dans ce dossier :

```powershell
py -m pip install -r requirements.txt
py analyse_images.py
```

macOS/Linux : remplacer `py` par `python3`.
La commande parcourt toute la collection automatiquement, œuvre après œuvre.
Les limites de recherche ci-dessous concernent les candidats **par œuvre**,
jamais le nombre d'œuvres du dataset.

## Reprise

Relancer `py analyse_images.py` : les résultats enregistrés ne sont pas refaits.
Pour réessayer les erreurs et images non trouvées :

```powershell
py analyse_images.py --retry-failed
```

Les appels API déjà terminés restent en cache même avec cette option. Pour
forcer une nouvelle recherche/annotation pour un identifiant, supprimer son
fichier `outputs/resultats/ID.json` et son dossier `outputs/traces/ID/`, puis
relancer. Les autres œuvres restent en cache. Ne pas lancer deux instances
sur le même dossier outputs. Fermer le CSV dans Excel pendant l'exécution.

## Fonctionnement

1. Lire les métadonnées BIRD : titre, artiste, date, inventaire et sources.
2. Chercher des reproductions sur `NMFA Source` puis `External source`.
3. Si aucune correspondance n'est retenue, rechercher des notices avec
   l'outil web_search de l'API OpenAI (jusqu'à six pages issues des sources).
4. Extraire les URL d'images du HTML : OpenGraph, JSON-LD, liens, img/srcset.
   Télécharger les candidats, écarter les trop petites images et logos connus.
5. Faire contrôler par le modèle le lien entre chaque reproduction, sa notice
   et l'œuvre demandée. Au plus huit candidats évalués par phase, soit seize
   au total par œuvre. Une correspondance incertaine reste à vérifier.
6. Sauvegarder une reproduction retenue dans `images/ID_BIRD.jpg`.
7. Envoyer **l'image seule** dans un nouvel appel indépendant pour décrire son
   contenu, son sujet, ses caractéristiques visuelles et son style probable.
8. Comparer le sujet visuel à la classification existante et sauvegarder.

Le sujet existant et la description BIRD ne sont pas donnés au modèle lors de
l'annotation visuelle. Ils peuvent servir au contrôle d'identité précédent.
Les images du catalogue sont des copies JPEG réencodées à la résolution
récupérée. L'analyse utilise une copie limitée à 1600 pixels de côté.

## Résultats

- `outputs/oeuvres_semantique.csv` : **une ligne par œuvre**, y compris celles
  sans image ou encore en attente. Mis à jour après chaque œuvre.
- `outputs/images/ID.jpg` : reproductions retenues par le contrôle automatique.
- `outputs/resultats/ID.json` : résultat individuel et point de reprise.
- `outputs/traces/ID/` : candidats, sources, justifications et réponses API.
- `outputs/usage.jsonl` : consommation en tokens et identifiants des appels.
- `outputs/bilan.json` : effectifs par statut.

Champs principaux : image_id, titre, artiste, source_page, source_image,
sujet_existant, sujet_principal, sujets_secondaires, desaccord_sujet,
description_visuelle, style_visuel_probable, justification_style, figuration,
couleurs_dominantes, elements_visibles, composition, limites_visuelles.

Statuts :
- `ok` : image retenue et annotée par le modèle ; pas de validation humaine.
- `a_verifier` : des candidats ont été évalués sans correspondance suffisante.
- `image_introuvable` : aucun candidat exploitable récupéré dans cette recherche.
- `erreur` : traitement échoué (type/motif enregistré).
- `en_attente` : œuvre non encore traitée.

Les erreurs d'accès à un site sont conservées dans les JSON. Elles ne prouvent
pas que l'image n'existe pas. Un blocage, une galerie chargée uniquement en
JavaScript ou un serveur IIIF non exposé dans les balises exploitées peut
nécessiter une récupération complémentaire. Le script n'invente pas d'image
pour atteindre artificiellement 359 succès.

## Interprétation

Les neuf sujets sont conservés, avec `NA` autorisé. Le style visuel est une
hypothèse descriptive, pas une attribution historique documentée à un mouvement.
Les sujets religieux ou historiques peuvent nécessiter un contexte absent de
l'image. Un désaccord image/texte signale un cas à examiner, pas une erreur
certaine de la première classification. Une identité « retenue par IA » n'est
pas une garantie d'exactitude. La pipeline conserve les URL pour l'audit et
l'examen des conditions de réutilisation avant diffusion des reproductions.

## Coût et configuration

Chaque contrôle d'image, annotation et recherche utilise l'API payante.
Le coût dépend des candidats et des recherches nécessaires : aucun total n'est
annoncé avant traitement. La reprise et le cache évitent les appels terminés
inutiles. Modèles configurables dans `.env` ; ils doivent prendre en charge
Responses, les images et JSON Schema, et web_search pour le modèle de recherche.
Les crédits API sont distincts de l'abonnement ChatGPT.

## Validation du programme fourni

`py analyse_images.py --check-input` vérifie les colonnes, les identifiants et
le nombre de lignes sans clé ni appel API. Le programme a été vérifié hors
ligne ; le parcours réel des sites et les appels payants nécessitent ta clé.

Documentation technique :
- https://developers.openai.com/api/docs/guides/images-vision
- https://developers.openai.com/api/docs/guides/tools-web-search
- https://developers.openai.com/api/docs/guides/structured-outputs

## Licence et attribution

- **Code** : licence MIT, voir [`LICENSE`](LICENSE).
- **Données BIRD originales** : fournies par le projet MBANv2 / musée des Beaux-Arts de Nancy,
  sous licence **CC BY-NC-SA 4.0** (https://mbanv2.loria.fr/). Les fichiers de données dérivés
  de BIRD présents dans ce dépôt restent soumis à cette licence (pas d'usage commercial,
  partage dans les mêmes conditions).
- Si vous utilisez ces données, citez l'article original :
  > Alexanne Worm et al., *BIRD: A Museum Open Dataset Combining Behavior Patterns and
  > Identity Types to Better Model Visitors' Experience*. DOI : 10.1145/3708319.3733686
