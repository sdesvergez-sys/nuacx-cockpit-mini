# NuaCX Cockpit — intégration embarquée (mode mini)

Ce dépôt sert à **tester et documenter** l'intégration du Cockpit NuaCX (l'interface agent) dans
une iframe étroite, façon panneau latéral de CRM — ce qu'on appelle le **mode mini**. Il ne
contient qu'une seule page HTML autonome (aucune dépendance, aucun backend à installer) pour
essayer concrètement le mécanisme avant de l'intégrer dans votre propre application.

> Ce dépôt est volontairement minimal : il couvre ce dont une intégration a besoin au quotidien,
> pas la documentation technique complète de NuaCX (sécurité, architecture interne, limites
> détaillées...), qui reste réservée aux équipes NuaCX.

## Prérequis

- Une instance NuaCX accessible depuis votre navigateur (une URL — en local, typiquement
  `http://localhost:8080`).
- Un compte **agent** sur cette instance, avec le droit d'accès au Cockpit.
- Un navigateur récent (Chrome, Edge ou Firefox).
- De quoi servir un fichier HTML en local (`python3`, `npx serve`, ou équivalent) — voir pourquoi
  ci-dessous.

## Démarrage rapide

1. Clonez ce dépôt, puis servez-le avec un petit serveur HTTP local plutôt que d'ouvrir le fichier
   directement (double-clic/`file://`) :
   ```bash
   cd nuacx-cockpit-mini
   python3 -m http.server 9000
   ```
   Le navigateur restreint certaines permissions (notamment le micro) pour une page ouverte en
   `file://` — un vrai serveur local, même minimal, évite ce piège.
2. Ouvrez `http://localhost:9000/console-embed-nuacx.html`.
3. Dans la section **Connexion**, renseignez l'URL de votre instance NuaCX (ex.
   `http://localhost:8080`), laissez **Mode compact** coché, puis cliquez **Charger**.
4. Le panneau affiche le Cockpit intégré avec un écran **« Se connecter »** — aucun jeton à copier
   à la main. Cliquez le bouton : une fenêtre pop-up s'ouvre, vous authentifie (identifiants NuaCX
   habituels), puis se referme toute seule. Le Cockpit se charge alors automatiquement.
5. Une fois le softphone enregistré (statut visible en haut du panneau), testez le clic-to-call
   depuis la section **Actions** de la console : il compose le numéro dans le Cockpit embarqué,
   exactement comme si l'agent l'avait tapé lui-même.

Si la pop-up est bloquée par votre navigateur, autorisez les pop-ups pour le domaine de votre
instance NuaCX et recliquez sur le bouton de connexion du Cockpit.

## Ce que vous testez

### Le mode compact (`?mode=mini`)

Ajouté à l'URL du Cockpit, ce paramètre adapte l'affichage à un panneau étroit : barres latérales
repliées par défaut, boutons d'appel compactés, fenêtres de dialogue centrées plutôt qu'ancrées.
Gabarit minimal recommandé : **400 × 650 px**. Décochez la case dans la console pour comparer avec
le rendu standard (plein écran) dans le même panneau.

### Le clic-to-call

Un hôte (votre CRM, votre portail) peut demander au Cockpit embarqué de composer un numéro, comme
si l'agent l'avait saisi lui-même. Dans la console de test, le champ **Clic-to-Call** envoie ce
signal à l'iframe. Il faut que l'agent soit déjà connecté (softphone enregistré) pour que l'appel
parte réellement.

### Le screen-pop et la journalisation

Le Cockpit informe la page hôte de deux moments clés d'un appel voix :

- à la **présentation d'un appel entrant** (pour afficher la fiche du contact correspondant côté
  CRM avant même que l'agent décroche) ;
- à la **clôture de l'appel** (code de conclusion choisi par l'agent, durée, correspondance
  annuaire éventuelle — pour journaliser l'interaction côté CRM).

Pour les observer dans la console de test, passez un vrai appel voix pendant que le Cockpit est
chargé : les cartes **Événements reçus** se remplissent automatiquement.

## Intégrer le Cockpit dans votre propre page

Le strict minimum :

```html
<iframe src="https://votre-instance-nuacx/workspace?mode=mini"
        allow="microphone"
        style="width:420px; height:680px; border:0"></iframe>
```

Points importants :

- **`allow="microphone"`** est indispensable — sans lui, le softphone ne peut jamais s'enregistrer
  (délégation de permission par origine, pas par simple présence d'une iframe).
- **Aucun jeton à gérer côté hôte.** Le Cockpit détecte lui-même qu'il est chargé sans
  authentification et affiche son propre écran de connexion (voir le parcours décrit plus haut) —
  votre application n'a rien à faire de particulier, juste embarquer l'iframe normalement.
- **Taille minimale recommandée : 400 × 650 px.** En dessous, le contenu reste utilisable mais un
  ascenseur peut apparaître.

## Le pont d'intégration (`postMessage`)

Une fois chargé, le Cockpit communique avec la page hôte via `window.postMessage`. Les messages
sont des **objets JavaScript natifs** (pas des chaînes JSON) : lisez `event.data` directement,
sans `JSON.parse`.

**Émis par le Cockpit** (`window.parent.postMessage({ type, data }, '*')`) :

| Type | Quand | Contenu (`data`) |
|---|---|---|
| `nuacx:ready` | Le Cockpit a fini de charger. | `{}` |
| `nuacx:screenPop` | Un appel entrant se présente à l'agent. | `{ call_record_id, caller_id_number, caller_id_name, linked_entry }` |
| `nuacx:interactionLogged` | L'agent a conclu un appel (code de conclusion soumis). | `{ call_record_id, caller_id_number, caller_id_name, linked_entry, duration_seconds, wrapup_code, comment }` |

**Reçu par le Cockpit** :

| Type | Effet | Contenu (`data`) |
|---|---|---|
| `nuacx:clickToCall` | Compose le numéro fourni, comme si l'agent l'avait tapé. | `{ number }` |

Exemple côté page hôte :

```js
// Écouter les événements du Cockpit
window.addEventListener('message', (event) => {
  const msg = event.data;
  if (!msg || typeof msg !== 'object') return;
  if (msg.type === 'nuacx:screenPop') {
    console.log('Appel entrant :', msg.data.caller_id_number, msg.data.caller_id_name);
  }
});

// Déclencher un clic-to-call
document.querySelector('iframe').contentWindow.postMessage(
  { type: 'nuacx:clickToCall', data: { number: '+33612345678' } },
  '*',
);
```

### À savoir côté sécurité

Les messages ne vérifient pas l'origine de la page hôte (`targetOrigin: "*"`) : n'importe quel
script exécuté dans votre page peut envoyer un `nuacx:clickToCall` à l'iframe, ou lire les
événements qu'elle émet. Gardez votre page hôte sous votre contrôle (pas de script tiers non
fiable dessus) — c'est la même prudence qu'avec n'importe quel widget embarqué sensible.

## Structure de ce dépôt

- **`console-embed-nuacx.html`** — la console de test elle-même. Aucune dépendance, aucun
  backend : ouvrez-la via un serveur HTTP local et pointez-la vers votre instance NuaCX.

## Pour aller plus loin

Ce dépôt couvre l'essentiel pour démarrer une intégration. La documentation technique complète
(arbitrages de sécurité, limites connues, détails d'implémentation) reste disponible dans la
documentation interne NuaCX — rapprochez-vous de votre contact NuaCX si vous en avez besoin.
