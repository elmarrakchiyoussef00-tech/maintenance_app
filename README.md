# Application de gestion de maintenance — équipements médicaux

Application Flutter mobile pour la gestion de maintenance d'équipements
médicaux, avec 4 profils : **client**, **coordinatrice**, **ingénieur**,
**chef d'équipe**. Le parcours de navigation reproduit exactement l'arborescence
définie dans le cahier des charges.

## 1. Lancer l'application

### 1.1 Démarrer le backend (authentification réelle)

```bash
cd backend
node scripts/seed.js
node src/server.js
```

Voir `backend/README.md` pour le détail. Aucune installation (`npm install`)
n'est nécessaire — le backend n'utilise que des modules natifs de Node.js.

### 1.2 Lancer l'app Flutter

```bash
flutter pub get
flutter run
```

L'écran de connexion appelle le backend réel : le mot de passe est vérifié
côté serveur (haché, jamais en clair), un jeton de session (JWT) est renvoyé
et stocké de façon chiffrée sur l'appareil (`flutter_secure_storage`). Au
prochain lancement, l'app revalide ce jeton auprès du serveur et rouvre
directement le bon dashboard (session persistante), sans repasser par
l'écran de connexion.

**Toutes les données affichées dans l'app (clients, équipements,
interventions, notifications, demandes...) proviennent désormais du backend
réel** — il n'y a plus de mode hors-ligne ni de données simulées côté
Flutter. Si l'API n'est pas démarrée, chaque écran affiche un message clair
avec un bouton "Réessayer" plutôt que de planter.

Configuration de l'URL de l'API : `lib/services/api_config.dart`
(par défaut : `localhost` sur iOS/web/bureau, `10.0.2.2` sur émulateur Android
— à adapter pour un appareil physique, voir les commentaires du fichier).

Comptes de démonstration (voir `backend/README.md` pour le détail) :

| Rôle | Email | Mot de passe |
|---|---|---|
| Client | client@demo.ma | Client123! |
| Coordinatrice | coordinatrice@demo.ma | Coord123! |
| Ingénieur | ingenieur@demo.ma | Ingenieur123! |
| Chef d'équipe | chef@demo.ma | Chef123! |

## 2. Structure du projet

```
backend/                         API (Node.js natif, zéro dépendance)
  src/server.js                  serveur HTTP + routage
  src/routes/auth.js             login / me / logout
  src/routes/clients.js          clients, leurs équipements, leurs documents
  src/routes/equipements.js      fiche équipement + historique d'interventions
  src/routes/interventions.js    liste/fiche/mise à jour, missions, historique ingénieurs
  src/routes/signalements.js     déclarations de panne (lecture + création)
  src/routes/notifications.js    notifications (coordination ou client)
  src/routes/dashboard.js        statistiques agrégées en direct
  src/lib/password.js            hachage scrypt salé
  src/lib/jwt.js                 signature/vérification JWT (HS256) maison
  src/lib/store.js               accès générique aux fichiers JSON (à remplacer par PostgreSQL)
  src/lib/router.js              routeur minimal avec paramètres d'URL
  src/data/*.json                données de démonstration (une "table" par fichier)
  scripts/seed.js                génère les comptes de démonstration

lib/
  main.dart                      point d'entrée + AuthGate (session persistante)
  theme/app_theme.dart           couleurs et thème Material
  models/                        User, Client, Equipment, Intervention (+ parsing JSON)
  services/                      AuthService, DataService (appels API réels), config API
  widgets/                       InfoCard, NavRow, MetricCard, StatusBadge, VideoBackground,
                                  LogoutAction, DataLoader (loading/erreur/retry générique)
  screens/
    login/                       écran de connexion (réel, sans mode hors-ligne)
    coordinator/                 Dashboard, Clients, Fiche client, Équipements,
                                  Fiche équipement, Historique ingénieurs, Intervention
    engineer/                    Dashboard missions, Fiche mission, Fiche équipement,
                                  Formulaire d'intervention (persiste réellement)
    client/                      Dashboard, Mes équipements, Fiche équipement,
                                  Déclarer une panne (persiste réellement), Mes demandes
    team_lead/                   Dashboard avec statistiques
    shared/                      Notifications, Paramètres, Documents, Interventions (vue globale)

database/
  schema.sql                     schéma PostgreSQL complet (tables, contraintes, vues, données de test)
```

## 3. Base de données (`database/schema.sql`)

Le schéma couvre toutes les entités du parcours :

- **clients** — cliniques/hôpitaux
- **contrats** — contrats de maintenance liés à un client
- **utilisateurs** — les 4 rôles, avec `client_id` renseigné pour les comptes clients
- **equipements** — rattachés à un client et à un contrat
- **signalements** + **signalement_medias** — déclarations de panne faites par un client (photos/audio)
- **interventions** — missions des ingénieurs, liées à un équipement et éventuellement à un signalement
- **pieces_remplacees**, **intervention_medias** — détail d'une intervention (pièces, photos, PDF)
- **documents**, **notifications**
- deux vues : `vue_interventions_par_ingenieur` et `vue_dashboard_stats` (statistiques du dashboard)

Pour l'installer :

```bash
createdb maintenance_db
psql maintenance_db -f database/schema.sql
```

## 4. Données métier : réellement servies par le backend

Clients, équipements, interventions, signalements et notifications sont
désormais servis par le backend (voir `backend/README.md` pour le détail de
chaque route et des règles d'accès par rôle). Deux actions écrivent
réellement des données, testées de bout en bout :

- **L'ingénieur termine une intervention** (formulaire → `PATCH /api/interventions/:id`) :
  le changement est persisté et immédiatement visible par la coordinatrice.
- **Le client déclare une panne** (`POST /api/signalements`) : crée un
  signalement et une notification pour la coordination.

Pour passer des fichiers JSON de démonstration à PostgreSQL
(`database/schema.sql`), voir la section 6 de `backend/README.md` —
seul `src/lib/store.js` a besoin d'être remplacé par de vraies requêtes SQL,
les routes n'en dépendent que via une interface `readAll()`/`writeAll()`.

## 5. Identité visuelle

- **Écran de connexion** : vidéo de présentation T2S Group en arrière-plan
  (`assets/videos/intro_background.mp4`, en boucle, muette), avec le logo
  **T2S Solutions** en haut et le logo **ZEISS** ("Équipements maintenus : ")
  en bas de la carte de connexion.
- Utilise le package `video_player` (ajouté à `pubspec.yaml`).
- Les fichiers vidéo étant embarqués dans les assets, la taille de l'APK/IPA
  augmente d'environ 2 Mo — à remplacer par un flux distant (URL) si besoin
  de réduire la taille du binaire (`VideoPlayerController.networkUrl`).

## 6. Ce qui est démonstratif vs fonctionnel

- **Authentification : réelle** (mot de passe vérifié côté serveur, jeton
  JWT, session persistante, déconnexion).
- **Données métier : réelles**, servies par le backend (voir section 4) —
  clients, équipements, interventions, historique, signalements,
  notifications, statistiques du dashboard (calculées en direct, plus de
  chiffres fixes).
- **Écrit réellement en base** : mise à jour d'une intervention par
  l'ingénieur (PATCH), déclaration de panne par un client (POST + notification).
- Photos et PDF (formulaire d'intervention, déclaration de panne) restent
  des boutons décoratifs côté Flutter : aucun endpoint de téléversement de
  fichiers n'a été implémenté.
- Pièces remplacées / Rapports PDF au niveau d'un équipement restent des
  lignes de menu statiques (non détaillées dans le cahier des charges initial).
- Le stockage JSON du backend n'est pas conçu pour des accès concurrents à
  grande échelle — voir `backend/README.md §7` pour les limites connues.
