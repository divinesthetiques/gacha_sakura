# Grimoire des Cartes — jeu gacha pour Twitch

Un jeu où des cartes apparaissent aléatoirement sur ton stream ; tes viewers
les attrapent avec une récompense de points de chaîne, et peuvent consulter
leur bibliothèque personnelle via une commande de chat.

Tout tient dans **un seul fichier** (`index.html`), avec trois "vues" :

| Vue | URL | Usage |
|---|---|---|
| Overlay | `index.html?vue=overlay` | À mettre dans OBS (Browser Source) |
| Admin | `index.html?vue=admin` | Pour toi, gérer les cartes et réglages |
| Bibliothèque | `index.html?vue=bibliotheque&joueur=pseudo` | Envoyée automatiquement en chat |

Comme il n'y a pas de serveur à coder, trois services gratuits font le
travail : **Firebase** (stocke les cartes et les collections, partagé entre
toutes les vues), **Twitch** (points de chaîne + chat), et **Discord**
(annonces, facultatif).

---

## 1. Créer le projet Firebase (gratuit)

1. Va sur [console.firebase.google.com](https://console.firebase.google.com), crée un projet.
2. Dans **Build > Firestore Database**, clique sur "Créer une base de données", mode **production**.
3. Dans **Règles**, remplace par :
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /{document=**} {
         allow read: if true;
         allow write: if true;
       }
     }
   }
   ```
   *(Ces règles sont ouvertes en écriture pour rester simple — correct pour un
   projet perso à petite échelle. Si ça t'inquiète, on peut les restreindre
   par la suite.)*
4. Dans **Paramètres du projet > Général**, ajoute une "application Web",
   copie l'objet `firebaseConfig` affiché, et colle ses valeurs dans le bloc
   `CONFIG.firebase` en haut de `index.html`.

## 2. Créer l'application Twitch

1. Va sur [dev.twitch.tv/console/apps](https://dev.twitch.tv/console/apps) → **Enregistrer votre appli**.
2. Catégorie : "Application Web". Type de client : **Public**.
3. Dans **URL de redirection OAuth**, ajoute (adapte à ton URL d'hébergement, voir étape 4) :
   - `https://TON-SITE/index.html?vue=admin`
   - `https://TON-SITE/index.html?vue=overlay`
4. Copie le **Client ID** dans `CONFIG.twitch.clientId`, et mets ton pseudo Twitch (en minuscules) dans `CONFIG.twitch.channel`.
5. Sur ton tableau de bord créateur Twitch, crée la récompense de points de
   chaîne que tu veux (ex. "Attraper la carte") — tu la sélectionneras
   ensuite dans l'onglet Twitch du panneau admin.

## 3. Discord (facultatif)

Dans les paramètres d'un salon Discord → **Intégrations > Webhooks > Nouveau
webhook**, copie l'URL et colle-la dans `CONFIG.discordWebhookUrl`. Laisse
vide pour désactiver les annonces.

## 4. Héberger le fichier

Il faut une vraie URL en `https://` (Twitch l'exige pour l'OAuth). Le plus
simple et gratuit : **GitHub Pages**.

1. Crée un dépôt GitHub, dépose `index.html` dedans.
2. **Settings > Pages** → active la publication depuis la branche principale.
3. Ton site sera à une adresse du type `https://tonpseudo.github.io/gacha-sakura/index.html`.
4. Reviens compléter `CONFIG.siteUrl` avec cette adresse exacte, et
   mets à jour les URLs de redirection Twitch (étape 2) si besoin.

## 5. Ajouter tes cartes

Ouvre `index.html?vue=admin` dans ton navigateur habituel, onglet **Cartes** :
donne un nom, une rareté et upload ton image pour chaque carte.

Onglet **Twitch** : connecte-toi, puis choisis dans la liste la récompense de
points créée à l'étape 2.

Onglet **Réglages** : ajuste la fréquence d'apparition et les chances de
capture par rareté.

## 6. Brancher l'overlay dans OBS

1. Ajoute une source **Navigateur** dans OBS, URL = `.../index.html?vue=overlay`.
2. La première fois, **clique droit sur la source > Interagir**, puis clique
   sur la petite roue ⚙ en haut à droite et connecte-toi à Twitch depuis
   cette fenêtre (le jeton reste stocké dans ce navigateur OBS uniquement —
   c'est normal, à refaire une seule fois).
3. Une fois connecté, les points verts "Twitch" et "Chat" s'allument : c'est
   prêt, les cartes commenceront à apparaître selon tes réglages.

## 7. La commande de chat

Un viewer tape `!macollection` (modifiable dans `CONFIG.chatCommand`) → le
jeu lui répond avec le lien vers sa bibliothèque personnelle, qui affiche
toutes les cartes du jeu, celles obtenues en couleur, les autres en grisé.

---

## 8. Protéger le panneau admin (obligatoire pour la sécurité)

Le panneau admin demande maintenant un e-mail et un mot de passe avant de
s'afficher. Deux choses à faire une seule fois :

1. Dans la console Firebase, va dans **Build > Authentication**, clique
   "Get started", puis active la méthode **E-mail/Mot de passe**.
2. Toujours dans Authentication, onglet **Users**, clique "Add user" et
   crée ton propre compte (l'e-mail et le mot de passe que tu utiliseras
   pour te connecter à `?vue=admin`).
3. Retourne dans **Firestore Database > Règles** et remplace-les par
   celles-ci (elles interdisent désormais la modification des cartes et
   réglages à qui n'est pas connecté, tout en laissant l'overlay et la
   bibliothèque fonctionner normalement) :
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /cartes/{doc} {
         allow read: if true;
         allow write: if request.auth != null;
       }
       match /collections/{doc} {
         allow read: if true;
         allow write: if request.auth != null;
       }
       match /config/{doc} {
         allow read: if true;
         allow write: if request.auth != null;
       }
       match /captures/{doc} {
         allow read: if true;
         allow write: if true;
       }
     }
   }
   ```

*(Limite à connaître : les rédemptions de cartes restent ouvertes en
écriture côté base de données, car c'est l'overlay — pas un compte connecté —
qui les enregistre. Ce n'est pas gênant en pratique : ça demanderait des
connaissances techniques pour en abuser, mais ce n'est pas une protection
absolue.)*

## 9. Bonus de chance pour les abonnés

Comme la permission nécessaire (`channel:read:subscriptions`) a été ajoutée
après ta première connexion, il faut te reconnecter une fois :

1. Va dans `?vue=admin` > onglet **Connexion Twitch** > "Se déconnecter".
2. Reconnecte-toi — Twitch te demandera d'autoriser la nouvelle permission.
3. Dans l'onglet **Réglages**, règle le bonus (en points de %) ajouté à la
   chance de capture pour les abonnés.
4. Refais cette même reconnexion dans le navigateur d'OBS (clic droit sur
   la source > Interagir) puisque c'est lui qui gère les captures en direct.

## 10. Taille des cartes à l'écran

Toujours dans l'onglet **Réglages** du panneau admin, un menu "Taille des
cartes à l'écran" propose Petite / Moyenne / Grande. Par défaut le jeu
utilise désormais un format plus discret (Moyenne) qu'au premier essai —
ajuste selon la résolution de ton stream et l'espace que tu veux lui laisser
à l'écran.

### Notes importantes

- **Un seul appareil doit garder l'overlay ouvert** avec la connexion Twitch
  active (celui d'OBS) : c'est lui qui écoute les points de chaîne et
  répond dans le chat. Le panneau admin sert uniquement à la configuration.
- Les cartes utilisent tes propres images — pense à rester sur un usage fan
  non commercial, dans l'esprit d'un jeu communautaire pour ton stream.
- Si tu vois des messages d'erreur liés au CSP ou à Firebase au premier
  chargement, vérifie que toutes les valeurs de `CONFIG` sont bien remplies.
