# Randonneur8

PWA d'édition et de partage de randonnées à partir de fichiers **KMZ / KML / GPX**, avec tracé animé ("chenillard"), profil altimétrique, gestion de photos géolocalisées, et une fonction de **visite guidée** avec lecture audio des commentaires.

## Fonctionnalités

### Édition
- Chargement d'un fichier KMZ, KML ou GPX (import direct, partage depuis un gestionnaire de fichiers Android, ou ouverture via le menu "Ouvrir avec")
- **Mode d'import Remplacer / Ajouter** : bascule à côté de la zone de dépôt de fichier. "Remplacer" (comportement historique) efface les waypoints et le tracé actuels avant de charger le nouveau fichier. "Ajouter" conserve les waypoints déjà présents (y compris leurs commentaires et photos modifiés) et ajoute à la suite ceux du fichier importé, sans effacer la session en cours ; le tracé existant est conservé tel quel si un tracé est déjà chargé.
- Affichage du tracé sur fond OSM, Plan IGN V2, Ortho-photo IGN, **ou fichier MBtiles local**, avec animation "chenillard" (fourmis défilantes) cohérente sur tous les fonds de carte
- **Fond de carte MBtiles** : bouton "📦 MBtiles" pour ouvrir un fichier `.mbtiles` présent sur l'appareil (produit avec MOBAC ou avec la PWA [ign2mbt](https://bernardhoyez.github.io/PWA/ign2mbt/)). La base SQLite est lue directement dans le navigateur (librairie sql.js), sans conversion préalable ni envoi sur un serveur ; la carte se recadre automatiquement sur l'emprise du fichier. Ce fond reste disponible dans l'onglet Visite, utile hors couverture réseau. Il n'est en revanche pas repris dans les exports HTML/déploiement, qui restent basés sur les fonds WMTS en ligne.
- Ajout, édition et suppression de waypoints (nom, commentaire, photo)
- **Duplication de waypoint** : bouton "📄 Dupliquer" sur chaque waypoint. Le nouveau waypoint reprend toutes les propriétés de l'original (coordonnées, altitude, commentaire, photo) et est inséré juste après lui dans la liste ; son nom est suffixé `_bis`
- Profil altimétrique interactif avec calcul du dénivelé positif (D+) et négatif (D-), lissé pour limiter le bruit GPS
- Sauvegarde automatique de la session (texte en `localStorage`, photos en `IndexedDB`) : les modifications persistent après fermeture de l'app

### Visite guidée
- Navigation manuelle étape par étape (`« Précédent` / `Suivant »`) avec animation du marqueur le long du tracé entre deux waypoints
- Affichage de la photo associée à chaque étape
- Lecture du commentaire à la demande via le bouton **"Lire le commentaire"** (synthèse vocale, voix française si disponible)
- Export en **fichier HTML autonome** : diaporama exportable et partageable, photos encodées en base64, lisible hors-ligne dans n'importe quel navigateur, sans dépendance à l'application

### Déploiement
- Génération d'un package complet (KML + GPX + visionneuse HTML) prêt à déposer sur un hébergeur statique
- Information de déploiement affichée pour Vercel (`vercel --prod`)

## Installation sur smartphone (PWA)

1. Ouvrir l'URL de déploiement dans Chrome (Android)
2. Menu ⋮ → **Ajouter à l'écran d'accueil** / **Installer l'application**
3. Une fois installée, Randonneur8 apparaît dans le menu **Partager** des gestionnaires de fichiers pour les fichiers `.kmz`, `.kml`, `.gpx`

## Déploiement

### GitHub Pages (déploiement principal)
Le projet est préconfiguré pour être servi depuis un sous-chemin `/randonneur8/` (voir `manifest.json` et `sw.js`, variables `start_url`/`scope`/`SCOPE`).

```bash
git init
git add .
git commit -m "Randonneur8"
git remote add origin https://github.com/BernardHoyez/randonneur8.git
git push -u origin main
```
Puis activer GitHub Pages sur la branche `main` dans les paramètres du dépôt.
Déployé à terme sur : **https://BernardHoyez.github.io/randonneur8**

Le fichier `404.html` gère la redirection nécessaire au bon fonctionnement du Service Worker sur ce type d'hébergement.

### Vercel (optionnel)
```bash
cd randonneur8
vercel --prod
```
Le fichier `vercel.json` configure les en-têtes MIME nécessaires (`.kmz`, `.kml`, `.gpx`) et la route du `share-target`.

> Le **Partage de fichiers** (`share_target`) ne fonctionne que sur un hébergeur supportant les requêtes POST (Vercel). Sur GitHub Pages, l'application reste utilisable normalement mais ne pourra pas être proposée dans le menu "Partager" d'un gestionnaire de fichiers.

## Structure du projet

```
randonneur8/
├── index.html          application complète (HTML + CSS + JS, mono-fichier)
├── manifest.json        manifeste PWA (icônes, share_target, scope)
├── sw.js                 service worker "brise-cache" (cache, réception des fichiers partagés)
├── 404.html             redirection SPA pour GitHub Pages
├── vercel.json           configuration des en-têtes et routes Vercel
└── icons/
    ├── icon-192.png / icon-512.png                icônes standard
    └── icon-192-maskable.png / icon-512-maskable.png   icônes adaptatives Android
```

## Notes techniques

- **Aucune dépendance de build** : l'application est un unique fichier HTML, librement modifiable
- **Bibliothèques utilisées** (chargées depuis un CDN) : [Leaflet](https://leafletjs.com/) pour la carte, [JSZip](https://stuf.fr/jszip/) pour la lecture des KMZ
- **Persistance locale** : `localStorage` pour les métadonnées de session, `IndexedDB` pour les photos (les blobs ne sont pas sérialisables en `localStorage`)
- **Synthèse vocale** : Web Speech API (`SpeechSynthesis`), déclenchée uniquement sur action explicite de l'utilisateur — ce choix garantit la fiabilité sur mobile, où l'enchaînement automatique de lectures audio s'est révélé instable selon les navigateurs
- Le rendu du tracé animé utilise le renderer SVG par défaut de Leaflet avec une classe CSS (`stroke-dasharray` + `animation`), réappliquée automatiquement après chaque changement de fond de carte ou de zoom
- **Duplication de waypoint** : la copie partage temporairement le même blob photo que l'original (les blobs sont immuables), mais est enregistrée sous sa propre clé dans IndexedDB dès la sauvegarde de session ; modifier la photo de l'un n'affecte pas l'autre

## Historique des choix de conception

La fonction Visite a initialement été conçue avec un enchaînement automatique (lecture audio puis avance après une pause de quelques secondes). Ce mode s'est révélé instable sur certains navigateurs mobiles (blocages aléatoires de la synthèse vocale, désynchronisation de l'avance automatique). Le mode retenu — navigation et lecture déclenchées manuellement par l'utilisateur — s'est avéré nettement plus robuste sur l'ensemble des plateformes testées, sans perte d'usage significative.

## Évolutions par rapport à Randonneur6

- Ajout d'un bouton "📄 Dupliquer" sur chaque waypoint (carte de la liste des waypoints) : crée une copie complète du waypoint (nom suffixé `_bis`, mêmes coordonnées, altitude, commentaire et photo), insérée juste après l'original.
- Renommage complet du projet et des clés de stockage local (`randonneur8_db`, `randonneur8_voice`, `randonneur8_session`) pour ne pas interférer avec une session Randonneur6 existante sur le même appareil.
