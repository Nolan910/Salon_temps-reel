# Salon temps réel

## Choix technologiques — Séance 1

### Étape 1 — Caractérisation des besoins

| Besoin | Sens des échanges | Fréquence | Latence tolérée | Perte tolérée | Données |
|---|---|---|---|---|---|
| **1. Messages et accusés** | Bidirectionnel : le client envoie, le serveur diffuse au salon et répond à l'expéditeur | Moyenne, irrégulière, avec des rafales | < 1 s | Nulle : ni perte, ni doublon, ordre garanti | Texte (JSON, 1 à 2 000 caractères) |
| **2. Présence et saisie** | Bidirectionnel : le client signale, le serveur relaie aux autres | Faible pour la présence, élevée pour la saisie (limitée volontairement) | Quelques secondes pour la présence, < 1 s pour la saisie | Faible pour la présence, élevée pour la saisie | Texte (liste de noms, signal minimal) |
| **3. Note partagée** | Bidirectionnel, avec plusieurs émetteurs simultanés | Faible à moyenne (envoi à la validation ou après une pause, pas à chaque frappe) | ~1 s, quelques secondes acceptables | Nulle pour une modification acceptée, et ordre significatif | Texte (la note entière + numéro de version) |
| **5. Reprise après coupure** | Serveur → client pour le rattrapage, le client indiquant d'abord son dernier identifiant reçu | Rare (à chaque reconnexion), mais en rafale | Quelques secondes (le temps de se reconnecter) | Nulle dans la limite de l'historique, dans l'ordre et sans doublon | Texte (liste de messages) |

#### Justifications

- **1. Messages et accusés** : profil « chat » du cours. Le sens est bidirectionnel à cause de l'accusé de réception, adressé spécifiquement à l'expéditeur. L'identifiant fourni par le client évite les doublons lors d'un renvoi.
- **2. Présence et saisie** : deux informations de nature différente. La présence change rarement (arrivées et départs) et un événement perdu se corrige en renvoyant la liste complète. La saisie produit beaucoup de petits événements éphémères : le signal est limité en fréquence et expire côté client pour que l'indicateur disparaisse seul.
- **3. Note partagée** : plusieurs émetteurs, donc l'ordre compte. Le numéro de version permet au client d'ignorer un état plus ancien que celui affiché, et au serveur de refuser une modification fondée sur une version périmée (conflit).
- **5. Reprise après coupure** : le client indique son dernier identifiant reçu. Le serveur conserve un historique borné par salon, avec des identifiants strictement croissants. Si l'identifiant du client est trop ancien, l'historique est rechargé et l'utilisateur prévenu. Le client dédoublonne par identifiant.

### Étape 2 — Choix d'une technologie par besoin

| Besoin | Technologie | Justification | Principale limite |
|---|---|---|---|
| **1. Messages et accusés** | SSE (réception) + POST (envoi) | La diffusion va du serveur vers le client, à fréquence moyenne : c'est le cas d'usage de SSE. La réponse au POST sert directement d'accusé : succès avec le numéro attribué, ou erreur avec un motif. | Deux canaux : un message peut arriver par SSE avant la réponse du POST, donc on dédoublonne avec l'identifiant fourni par le client. |
| **2. Présence et saisie** | SSE (diffusion) + POST limité (signal de saisie) | Pour la présence, l'ouverture et la fermeture du stream SSE marquent l'arrivée et le départ, sans requête supplémentaire. Le signal de saisie reste rare si on le limite à un POST toutes les 2 à 3 s. | Chaque signal de saisie coûte une requête HTTP complète, avec ses en-têtes, pour une information qui peut se perdre sans conséquence. |
| **3. Note partagée** | SSE (diffusion) + POST/PUT versionné | HTTP gère bien le contrôle de version : une modification sur une version périmée reçoit un 409 avec la version courante. Les modifications acceptées sont diffusées à tous par SSE. | La note entière est renvoyée à chaque modification, et l'ordre entre la réponse HTTP et l'événement SSE est à gérer avec le numéro de version. |
| **5. Reprise après coupure** | SSE | La reconnexion et la reprise sont natives : le navigateur renvoie le dernier identifiant reçu, et le serveur rejoue la suite. | Le serveur doit conserver un historique. Comme il n'y a qu'un `Last-Event-ID` par stream, les identifiants doivent être globaux, et non par salon, si un seul stream sert plusieurs salons. |

#### Vérifications

- **Le client peut-il émettre ?** Pas par SSE, qui est unidirectionnel. L'utilisateur émet par des requêtes HTTP ordinaires (POST/PUT), dont la réponse sert d'accusé.
- **Une seule technologie pour tous les besoins ?** Oui, la même combinaison SSE + POST. Elle se justifie besoin par besoin : tous les flux descendants passent par un unique stream SSE, et les émissions du client restent peu fréquentes (messages, note) ou volontairement limitées (saisie).
- **Critères utilisés** : sens des échanges (diffusion majoritairement serveur → client), fréquence d'émission du client (faible à moyenne), perte tolérée (nulle pour messages, note et reprise, d'où l'intérêt de la reprise native de SSE).

### Étape 3 — Contraintes des autres besoins

- **4. Authentification, validation, débit** : Comment authentifier le stream SSE alors qu'`EventSource` ne permet pas d'en-tête `Authorization`, et comment garantir que le stream et les POST appartiennent bien au même utilisateur ?
- **6. Plusieurs instances** : Si le POST d'un message arrive sur l'instance B alors que le stream SSE d'un destinataire est ouvert sur l'instance A, comment A sait-elle qu'elle doit le diffuser ? 
- **7. Tests, charge, métriques** | Comment vérifier dans un test automatisé que deux clients SSE reçoivent le message posté par un troisième, alors que la réponse du stream ne se termine jamais ?