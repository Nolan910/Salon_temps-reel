# Salon temps réel

## Choix technologiques — Séance 1

| Besoin | Sens des échanges | Fréquence | Latence tolérée | Perte tolérée | Données |
|---|---|---|---|---|---|
| **1. Messages et accusés** | Bidirectionnel : le client envoie, le serveur diffuse au salon et répond à l'expéditeur | Moyenne, irrégulière, avec des rafales | < 1 s | Nulle : ni perte, ni doublon, ordre garanti | Texte (JSON, 1 à 2 000 caractères) |
| **2a. Présence** | Bidirectionnel : le client signale, le serveur relaie aux autres | Faible pour présence, élevée pour saisie (chaque frappe) | Quelques secondes | Faible pour présence, élevée pour saisie | Texte (liste de noms) |
| **3. Note partagée** | Bidirectionnel, plusieurs émetteurs simultanés | Faible à moyenne (envoi à la validation ou après une pause) | ~1 s, quelques secondes acceptables | Nulle pour une modification acceptée, ordre significatif | Texte (note entière + numéro de version) |
| **5. Reprise après coupure** | Serveur → client pour le rattrapage, après que le client a indiqué son dernier identifiant reçu | Rare (à chaque reconnexion), mais en rafale | Quelques secondes | **Nulle** dans la limite de l'historique, dans l'ordre et sans doublon | Texte (liste de messages) |


