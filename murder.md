# Rapport d'enquête - SQL Murder Mystery

## Initialisation
J'ai commencé par ouvrir sqlite3 en tapant sqlite3 dans le cmd.
Puis j'ai ouvert le fichier de l'exercice en faisant ".open sql-murder-mystery.db".

J'ai tapé .tables pour voir où je pouvais chercher les informations necessaires à la résolution de l'énigme.
J'ai eu :
```text
crime_scene_report      get_fit_now_check_in    interview
drivers_license         get_fit_now_member      person
facebook_event_checkin  income                  solution
```

## Recherche d'indices
Je me suis dis que pour trouver les informations les plus interressantes il fallait que je regarde dans "crime_scene_report",j'ai tapé "SELECT * FROM crime_scene_report;" malheureusement il y avait beaucoup trop d'informations dedans.
Alors j'ai voulu prendre uniquement les informations qui étaient utiles en utilisant "SELECT 20180115, SQL City FROM crime_scene_report," mais ça n'a pas fonctionné.

Après des recherches j'ai vu qu'il fallait faire :
```SQL
SELECT * FROM crime_scene_report
WHERE city = 'SQL City'
AND type = 'murder'
AND date = '20180115';
```

La requête m'a donné cette information :
| Date | Type | Description | City |
| :--- | :--- | :--- | :--- |
| 20180115 | murder | Security footage shows that there were 2 witnesses. The first witness lives at the last house on "Northwestern Dr". The second witness, named Annabel, lives somewhere on "Franklin Ave". | SQL City |

Il faut donc que je cherche des informations de qui habite à "Northwestern Dr" et qui est "Annabel".

## Recherche des témoins
### 1. Mauvaise piste
J'ai voulu chercher dans "drivers_license" en tapant :
```SQL
SELECT * FROM drivers_license
WHERE city = 'Northwestern Dr';
```

Cela n'a pas marché, je me suis dis qu'il fallait peut etre que je regarde dans la class person mais je ne savais pas comment elle est structuré donc j'ai fais :

```SQL
.schema person 
```

Ce qui m'a donné :

| Colonne | Type | Spécificités / Clés |
| :--- | :--- | :--- |
| id | integer | PRIMARY KEY |
| name | text | |
| license_id | integer | Foreign Key (vers drivers_license) |
| address_number | integer | |
| address_street_name | text | |
| ssn | CHAR | References income (ssn) |

### 2. Recherche rue "Northwestern Dr"
J'ai tapé :
```SQL
SELECT * FROM person
WHERE address_street_name = 'Northwestern Dr'
```

Mais ça m'a donné une liste trop longue donc pour trouver la personne que je cherche j'ai fais :
```SQL
SELECT * FROM person
WHERE address_street_name = 'Northwestern Dr'
ORDER BY address_number DESC
LIMIT 1;
```

Qui m'a donné :
| ID | Name | License ID | Address Number | Street Name | SSN |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 14887 | Morty Schapiro | 118009 | 4919 | Northwestern Dr | 111564949 |

### 3. Annabel
J'ai ensuite fais mes recherches sur Annabel en faisant :
```SQL
SELECT * FROM person
WHERE address_street_name = 'Franklin Ave'
AND name LIKE '%Annabel%';
```

Qui m'a donné :
| ID | Name | License ID | Address Number | Street Name | SSN |
| :--- | :--- | :--- | :--- | :--- | :--- |
16371|Annabel Miller|490173|103|Franklin Ave|318771143


## Interrogatoires
J'ai regardé comment était constituée la table interview :
```SQL
.schéma interview
```
| Colonne | Type |
| :--- | :--- |
| person_id | integer |
| transcript | text |

### 1. Interview de Morty
Il faut maintenant que je trouve leurs dépositions, pour ça, j'ai tapé ça pour trouver celle de Morty:
```SQL
SELECT * FROM interview
WHERE person_id IN (14887);
```

Ce qui m'a donné :
| Person ID | Transcript |
| :--- | :--- |
| 14887 | I heard a gunshot and then saw a man run out. He had a "Get Fit Now Gym" bag. The membership number on the bag started with "48Z". Only gold members have those bags. The man got into a car with a plate that included "H42W". |

### 2. Interview de Annabel
Pour Annabel j'ai fais la même commande mais avec son id :
```SQL
SELECT * FROM interview
WHERE person_id IN (16371);
```

Ce qui m'a donné :
| Person ID | Transcript |
| :--- | :--- |
16371|I saw the murder happen, and I recognized the killer from my gym when I was working out last week on January the 9th.


## Identification de l'exécutant 

### 1. Recherche membres gold (Interview de Annabel)
Je dois maintenant cherhcer le tueur de la salle de sport avec le sac des membres gold.
J'ai tapé cette commande :
```SQL
SELECT * FROM get_fit_now_member
WHERE id == "48Z%" AND membership_status = "gold";
```

Je me suis trompé de syntaxe donc ça m'a mis une erreur.
J'ai donc tapé cette commande :
```SQL
SELECT * FROM get_fit_now_member
WHERE id LIKE '48Z%'
AND membership_status = 'gold';
```

Deux résultats sont apparus :
| id | person_id | name | membership_start_date | membership_status |
| :--- | :--- | :--- | :--- | :--- |
| 48Z7A | 28819 | Joe Germuska | 20160305 | gold |
| 48Z55 | 67318 | Jeremy Bowers | 20160101 | gold |

Pour trouver qui est le meutrier entre ces deux personnes, il faut que je sache le quel des deux était à la salle de gym le 9 janvier 2018.
Je fais donc cette commande :
```SQL
SELECT * FROM get_fit_now_check_in
WHERE check_in_date = "09/01/2018"
AND membership_id LIKE '48Z%';
```

Mais je n'avais pas marqué la date correctement, j'ai donc corrigé et tapé :
```SQL
SELECT * FROM get_fit_now_check_in
WHERE check_in_date = '20180109'
AND membership_id LIKE '48Z%';
```

Ce qui m'a donné :
| membership_id | check_in_date | check_in_time | check_out_time |
| :--- | :--- | :--- | :--- |
| 48Z7A | 20180109 | 1600 | 1730 |
| 48Z55 | 20180109 | 1530 | 1700 |

Donc malheureusement ces deux personnes y étaient ce jour là.

### 2. La plaque d'imatriculation (Interview de Morty)
Je m'interresse du coup à l'interview de Morty qui donnait une plaque d'imatriculation, donc je fais cette commande :
```SQL
SELECT * FROM get_fit_now_member
WHERE id IN ('48Z7A', '48Z55')
AND plate_number LIKE '%H42W%';
```

Il y avait une erreur de syntaxe donc j'ai essayé :
```SQL
SELECT p.name, dl.plate_number
FROM get_fit_now_member m
JOIN person p ON m.person_id = p.id
JOIN drivers_license dl ON p.license_id = dl.id
WHERE m.id IN ('48Z7A', '48Z55')
AND dl.plate_number LIKE '%H42W%';
```

Qui m'a donné :
| Name | Plate Number |
| :--- | :--- |
| Jeremy Bowers | 0H42W2 |

Le meurtrier était donc Jeremy Bowers.

## Recherche du commanditaire 

### 1. Interview de Jeremy

J'ai donc regardé son interview avec son id que j'avais trouvé avant (67318) et j'ai trouvé des informations interressantes :
```SQL
SELECT * FROM interview
WHERE person_id = 67318;
```

Il a dit :
```text
I was hired by a woman with a lot of money. I don't know her name but I know she's around 5'5" (65") or 5'7" (67"). She has red hair and she drives a Tesla Model S. I know that she attended the SQL Symphony Concert 3 times in December 2017.
```
### 2. Recherche de la femme rousse
Je dois donc trouver une femme, rousse, qui conduit une Tesla Model S, et qui est allée au "SQL Symphony Concert" 3 fois en décembre 2017.

J'ai d'abord regardé comment etait constituée la table "facebook_event_checkin" :

```SQL
.schema facebook_event_checkin
```

| Colonne | Type |
| :--- | :--- |
| person_id | integer |
| event_id | integer |
| event_name | text |
| date | integer |

J'ai essayé de faire une grosse requête pour tout trouver d'un coup (Nom + Voiture + Concert) :
```SQL
SELECT p.name
FROM person p
JOIN drivers_license dl ON p.license_id = dl.id
JOIN facebook_event_checkin fb ON p.id = fb.person_id
WHERE dl.hair_color = 'red'
AND dl.car_make = 'Tesla'
AND fb.event_name = 'SQL Symphony Concert'
AND fb.date == "201712%";
```

Mais ça n'a pas marché car j'ai utilisé == avec un %, alors qu'il fallait utiliser LIKE.
J'ai donc corrigé ma requête pour trouver la personne qui correspond à tous les critères :
```SQL
SELECT p.name, fb.event_name, fb.date
FROM person p
JOIN drivers_license dl ON p.license_id = dl.id
JOIN facebook_event_checkin fb ON p.id = fb.person_id
WHERE dl.hair_color = 'red'
AND dl.car_make = 'Tesla'
AND dl.car_model = 'Model S'
AND dl.gender = 'female'
AND fb.event_name = 'SQL Symphony Concert'
AND fb.date LIKE '201712%';
```

Le résultat m'a affiché une seule personne qui apparaissait 3 fois pour ce concert en décembre :
| Name | Event Name | Date |
| :--- | :--- | :--- |
| Miranda Priestly | SQL Symphony Concert | 20171206 |
| Miranda Priestly | SQL Symphony Concert | 20171212 |
| Miranda Priestly | SQL Symphony Concert | 20171229 |

C'est donc Miranda Priestly qui a engagé le tueur.

## Résolution de l'affaire
### Le meutrier était donc Jeremy Bowers et son commanditaire était Miranda Priestly.