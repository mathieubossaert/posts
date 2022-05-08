### Contexte

Comme pour l'article de [Florian May](https://forum.getodk.org/u/florian_may/summary) publié sur le [forum d'ODK](https://forum.getodk.org/t/turtle-monitoring-in-western-australia/37565), il est question ici de suivi de tortues, 
mais en France et sur une espèce d'eau douce, la Cistude d'Europe, [_Emys Orbicularis_](https://inpn.mnhn.fr/espece/cd_nom/77381/tab/fiche).
Même si la tortue Caouanne fait son retour sur nos plages de méditerranée, un suivi tel que celui mis en place en Australie n'est pas encore nécessaire ;-)

Pour suivre le populations de cistude, les collègues posent les pièges le premier jours de la semaine, qu'ils relèvent les 3 jours suivants.
Les individus capturés sont marqués (s'ils ne le sont pas déjà) et mesurés. L'ensemble des information est actuellement noté sur une fiche papier.

Au cours du printemps 2022, Vivian Millet, en stage de formation Idgéo a developpé un formulaire dédié à ce protocole.

Au cours des discussions avec les collègues en charge du terrain, ils nous ont expliqué que les pièges changent de place toutes les semaines, et qu'un suivi plus fin de leur emplacement serait utile.
Jusqu'à maintenant, le numéro du piège était reporté sur la fiche de capture. Et les pièges localisés à la main sur une carte.

Nous avons proposé de gérer ce référentiel de pièges dans ODK, et de mettre à jour la liste des pièges mobilisables dans le formualire de capture quand ils sont déplacés.
Nous avons donc développé un formualire dédié à la pose des pièges.
Une fois ce formualire fonctionnel, nous avons voulu adapter [ce test](https://forum.getodk.org/t/updating-external-media-files-for-select-questions-from-another-form-using-centrals-api/37295/4) et le rendre aussi automatique que possible.

L'idée générale est la suivante :

* si les pièges sont déplacés (= si on utilise le formulaire de pose de pièges)
* alors on met à jour le référnetiel des pièges utilisés dans le formulaire de capture (= on génère ce référentiel avec les nouvelles données et on l'envoi à ODK Central)
* et on met à jour le formulaire (= on publie une nouvelle version)

### Mise ne oeuvre

C'est parti !
Nous avons donc développé deux formulaires :

* _**CMR_Cistude_pose_pieges**_ utilisé pour la pose des pièges 
* et _**CMR_Cistude_captures**_ utilsisé pour le suivi des individus capturés

Toutes les requêtes SQL présentées ci-dessous sont executées dans notre base de données "métier". Rien n'est exécuté dans la BDD de Central à laquelle nous ne touchons jamais.

Nous utilisons une tâche cron (tous les jours à 22h) qui exécute les requêtes suivantes :

Nous utilisons pour cela les fonctions de [central2pg](https://github.com/mathieubossaert/central2pg/tree/master), placées dans le schéma _data_from_central_ schema pour récupéréer les données relatives aux pièges :

```sql
SELECT data_from_central.data_from_central_to_pg(
    'my_login','my_password','my.central.fdqn',5,
    'CMR_Cistude_pose_pieges', -- form ID
    'data_from_central', -- schema where to create tables and store data
    'point,point_auto' -- columns to ignore in json transformation to database attributes (geojson fields of GeoWidgets)
);
```

Nous avons créé une vue appelée _cmr_cistude_pose_pieges_session_courante_ qui retourne seulement les pièges posés lors de la dernière sessions (max(date)).
Le resultat de cette vue est copié dans un fichier geojson, dans un endroit accessible en écriture à l'utilisateur postgres. (commande COPY TO...)
Pour le moment, cette requête est executée même s'il n'y a pas de nouveau piège.

Ce geojson est ensuite utilisé par le formualire de capture pour proposer à la personne qui capture les cistudes de choisir le piège visité sur une carte :
-> https://docs.getodk.org/form-question-types/#select-one-from-map-widget

```sql
COPY (
    WITH places AS (
        SELECT id_piege, label, value, geometry
    FROM data_from_central.cmr_cistude_pose_pieges_session_courante
    )
SELECT
  json_build_object(
    'type', 'FeatureCollection',
    'features', json_agg(ST_AsGeoJSON(t.*)::json)
  )
FROM places AS t
      ) to '/home/postgres/medias_odk/pieges.geojson';
```
Nous utilisons un numéro de version de formulaire que nous avons souhaité "porteur d'information".
Il est composé de l'année sur 4 caractères à laquelle nous concaténons le numéro du jour dans l'année (_DOY_) sur 3 caractères (complété par des 0 à gauche).

> concat(extract(YEAR FROM date_max),lpad(extract(DOY FROM date_max)::text,3,'0'))

Nous comparons ensuite la version courante du formulaire avec son "hypothétique" nouvelle version.
Si cette dernière est plus grande que l'actuelle, nous créons un brouillon de formulaire.

Pour cela et pour les autres étapes, nous avons ajouté de nouvelles fonctions à la bibliothèque "central2pg"..

```sql
SELECT data_from_central.create_draft('my_login','my_password','my.central.fdqn',5,'CMR_Cistude_captures')
FROM data_from_central.cmr_cistude_pose_pieges_session_courante
WHERE concat(extract(YEAR FROM date_max),lpad(extract(doy FROM date_max)::text,3,'0')) > data_from_central.get_form_version('my_login','my_password','my.central.fdqn',5,'CMR_Cistude_captures')
LIMIT 1;
```

Une fois le brouillon créé, nous poussons le geojson comme pièce jointe à ce brouillon.


```sql
SELECT data_from_central.push_media_to_central('my_login','my_password','my.central.fdqn',5,'CMR_Cistude_captures', '/home/postgres/medias_odk', 'pieges.geojson')
FROM data_from_central.cmr_cistude_pose_pieges_session_courante
WHERE concat(extract(YEAR FROM date_max),lpad(extract(doy FROM date_max)::text,3,'0'))::integer > data_from_central.get_form_version('my_login','my_password','my.central.fdqn',5,'CMR_Cistude_captures')::integer
LIMIT 1;
```
Et nous publions la nouvelle version ;-)

```sql
SELECT data_from_central.publish_form_version('my_login','my_password','my.central.fdqn',5,
    'CMR_Cistude_captures', concat(extract(YEAR FROM date_max),lpad(extract(doy FROM date_max)::text,3,'0'))::integer                            )
FROM data_from_central.cmr_cistude_pose_pieges_session_courante
WHERE concat(extract(YEAR FROM date_max),lpad(extract(doy FROM date_max)::text,3,'0')) > data_from_central.get_form_version('my_login','my_password','my.central.fdqn',5,'CMR_Cistude_captures')
LIMIT 1;
```