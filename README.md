# Guide de déploiement de la configuration Thunderbird par GPO

## Introduction

Ce guide détaille la procédure de déploiement automatisé de la configuration de Mozilla Thunderbird sur les postes clients via les stratégies de groupe (GPO) d'Active Directory. L'objectif est de simplifier la gestion des profils utilisateurs en préconfigurant les paramètres de messagerie, notamment les comptes IMAP/SMTP, ainsi que d'autres préférences générales de l'application. Nous aborderons deux méthodes complémentaires : l'utilisation du fichier `policies.json` pour les paramètres généraux et le mécanisme d'`autoconfig` pour la configuration des comptes de messagerie.

## Prérequis

Avant de commencer, assurez-vous de disposer des éléments suivants :

*   Un environnement Active Directory fonctionnel.
*   Un contrôleur de domaine avec les outils de gestion des stratégies de groupe (GPMC) installés.
*   Les fichiers d'installation de Mozilla Thunderbird (idéalement la version MSI pour un déploiement facile).
*   Les informations de configuration de vos serveurs de messagerie (IMAP, SMTP, ports, types de sécurité, etc.).
*   Un serveur web (IIS, Apache, Nginx) pour héberger le fichier `autoconfig` si vous choisissez cette méthode pour la configuration des comptes.

## Méthode 1 : Configuration des préférences générales avec `policies.json`

Le fichier `policies.json` permet de définir des politiques pour Thunderbird, similaires aux GPO pour d'autres applications Windows. Il est multiplateforme et est le moyen privilégié par Mozilla pour gérer les paramètres d'entreprise. [1]

### Création du fichier `policies.json`

Créez un fichier nommé `policies.json` avec le contenu JSON suivant. Cet exemple désactive la création de mots de passe maîtres, la révélation des mots de passe, la télémétrie et force les mises à jour manuelles.

```json
{
  "policies": {
    "DisableMasterPasswordCreation": true,
    "DisablePasswordReveal": true,
    "DisableTelemetry": true,
    "AppAutoUpdate": false,
    "ManualAppUpdateOnly": true
  }
}
```

### Déploiement du fichier `policies.json`

Le fichier `policies.json` doit être placé dans un dossier nommé `distribution` à l'intérieur du répertoire d'installation de Thunderbird sur chaque poste client. Par exemple, sur Windows, le chemin serait `C:\Program Files\Mozilla Thunderbird\distribution\policies.json`.

Pour déployer ce fichier via GPO :

1.  **Créez un partage réseau** accessible en lecture par tous les utilisateurs ou ordinateurs concernés. Par exemple, `\\votre_domaine\NETLOGON\ThunderbirdConfig`.
2.  **Copiez le fichier `policies.json`** dans ce partage.
3.  **Créez une nouvelle GPO** ou modifiez une GPO existante liée à l'unité d'organisation (OU) contenant les utilisateurs ou ordinateurs cibles.
4.  Dans l'éditeur de gestion des stratégies de groupe, naviguez vers :
    `Configuration ordinateur` ou `Configuration utilisateur` > `Préférences` > `Paramètres Windows` > `Fichiers`.
5.  **Créez un nouvel élément `Fichier`** :
    *   `Action` : `Mettre à jour`
    *   `Fichier source` : Le chemin UNC de votre fichier `policies.json` sur le partage réseau (ex: `\\votre_domaine\NETLOGON\ThunderbirdConfig\policies.json`).
    *   `Fichier de destination` : Le chemin local sur le poste client (ex: `C:\Program Files\Mozilla Thunderbird\distribution\policies.json`).
    *   Cochez `Exécuter dans le contexte de sécurité de l'utilisateur connecté` si vous déployez sous `Configuration utilisateur`.

Cette GPO s'assurera que le fichier `policies.json` est présent et à jour sur les postes clients, appliquant ainsi les préférences générales définies.

## Méthode 2 : Configuration des comptes de messagerie avec `autoconfig` (Mission Control Desktop - MCD)

La configuration des comptes de messagerie via `policies.json` n'est pas directement supportée. La méthode recommandée par Mozilla pour la configuration centralisée des comptes est l'utilisation du mécanisme `autoconfig`, également connu sous le nom de Mission Control Desktop (MCD). Cette méthode utilise des fichiers JavaScript (`.js` et `.cfg`) pour définir les paramètres des comptes. [2]

### Fonctionnement de l'`autoconfig`

Thunderbird recherche deux fichiers pour l'autoconfiguration :

1.  **`autoconf.js`** : Un fichier JavaScript qui appelle le fichier de configuration principal. Il doit être placé dans le répertoire `defaults\pref` de l'installation de Thunderbird.
2.  **`thunderbird.cfg`** : Le fichier de configuration principal, également un fichier JavaScript, qui contient les paramètres des comptes. Il doit être placé à la racine du répertoire d'installation de Thunderbird.

Ces fichiers peuvent utiliser des variables d'environnement (comme `USER`, `MAIL`, `CN`) pour personnaliser la configuration pour chaque utilisateur.

### Création des fichiers `autoconfig`

#### `autoconf.js`

Créez un fichier nommé `autoconf.js` avec le contenu suivant :

```javascript
pref("general.config.filename", "thunderbird.cfg");
pref("general.config.obscure_value", 0); // 0 = pas d'encodage (texte clair)
```

#### `thunderbird.cfg`

Créez un fichier nommé `thunderbird.cfg` avec le contenu suivant. **N'oubliez pas de remplacer les placeholders (`<serveur_imap>`, `<serveur_smtp>`, `<votre_domaine>`) par vos propres informations.**

```javascript
//
// Fichier de configuration pour Thunderbird
//

// Ne pas modifier la première ligne
//

// Configuration des comptes de messagerie
try {

  // Définir les préférences par défaut
  pref("mail.rights.version", 1);
  pref("mail.server.default.hostname", "<serveur_imap>");
  pref("mail.server.default.port", 993);
  pref("mail.server.default.socket_type", "SSL");
  pref("mail.server.default.authentication", "password-cleartext");

  // Configurer le premier compte de messagerie
  var email = getenv("MAIL");
  if (email == "") {
    email = getenv("USER") + "@<votre_domaine>";
  }

  var realname = getenv("CN");
  if (realname == "") {
    realname = getenv("USER");
  }

  var username = getenv("USER");

  // Serveur entrant (IMAP)
  pref("mail.server.account1.hostname", "<serveur_imap>");
  pref("mail.server.account1.port", 993);
  pref("mail.server.account1.socket_type", "SSL");
  pref("mail.server.account1.authentication", "password-cleartext");
  pref("mail.server.account1.name", email);
  pref("mail.server.account1.userName", username);
  pref("mail.server.account1.type", "imap");

  // Serveur sortant (SMTP)
  pref("mail.smtpserver.smtp1.hostname", "<serveur_smtp>");
  pref("mail.smtpserver.smtp1.port", 587);
  pref("mail.smtpserver.smtp1.socket_type", "STARTTLS");
  pref("mail.smtpserver.smtp1.authentication", "password-cleartext");
  pref("mail.smtpserver.smtp1.username", username);
  pref("mail.smtpserver.smtp1.description", "Serveur SMTP");

  // Identité
  pref("mail.identity.id1.fullName", realname);
  pref("mail.identity.id1.useremail", email);
  pref("mail.identity.id1.valid", true);
  pref("mail.identity.id1.smtpServer", "smtp1");

  // Association du compte et de l'identité
  pref("mail.account.account1.identities", "id1");
  pref("mail.account.account1.server", "server1");

  // Définir le compte par défaut
  pref("mail.accountmanager.defaultaccount", "account1");

} catch (e) {
  displayError("catch", e);
}
```

### Déploiement des fichiers `autoconfig` via GPO

Le déploiement de ces fichiers est similaire à celui de `policies.json`, mais les chemins de destination sont différents.

1.  **Créez un partage réseau** accessible en lecture par tous les utilisateurs ou ordinateurs concernés. Par exemple, `\\votre_domaine\NETLOGON\ThunderbirdAutoConfig`.
2.  **Copiez les fichiers `autoconf.js` et `thunderbird.cfg`** dans ce partage.
3.  **Créez ou modifiez une GPO** liée à l'OU appropriée.
4.  Dans l'éditeur de gestion des stratégies de groupe, naviguez vers :
    `Configuration ordinateur` ou `Configuration utilisateur` > `Préférences` > `Paramètres Windows` > `Fichiers`.
5.  **Créez un nouvel élément `Fichier` pour `autoconf.js`** :
    *   `Action` : `Mettre à jour`
    *   `Fichier source` : `\\vtre_domaine\NETLOGON\ThunderbirdAutoConfig\autoconf.js`
    *   `Fichier de destination` : `C:\Program Files\Mozilla Thunderbird\defaults\pref\autoconf.js`
6.  **Créez un nouvel élément `Fichier` pour `thunderbird.cfg`** :
    *   `Action` : `Mettre à jour`
    *   `Fichier source` : `\\votre_domaine\NETLOGON\ThunderbirdAutoConfig\thunderbird.cfg`
    *   `Fichier de destination` : `C:\Program Files\Mozilla Thunderbird\thunderbird.cfg`

Assurez-vous que les permissions sur le partage réseau permettent aux utilisateurs ou aux ordinateurs de lire ces fichiers.

## Recommandations et Bonnes Pratiques

*   **Testez toujours** les configurations sur un petit groupe d'utilisateurs avant un déploiement à grande échelle.
*   **Utilisez des variables d'environnement** (`%USERDOMAIN%`, `%USERNAME%`, etc.) dans votre `thunderbird.cfg` pour rendre la configuration dynamique et adaptée à chaque utilisateur.
*   **Considérez l'hébergement du fichier `autoconfig` sur un serveur web** si vous avez de nombreux domaines de messagerie ou si vous souhaitez une configuration plus flexible. Thunderbird peut rechercher un fichier `config-v1.1.xml` à des emplacements spécifiques (par exemple, `http://autoconfig.votre_domaine.com/mail/config-v1.1.xml`). [3]
*   **Documentation** : Maintenez une documentation à jour de vos configurations GPO et des fichiers `autoconfig`.
*   **Sécurité** : Pour les mots de passe, il est fortement recommandé de ne pas les inclure en clair dans le fichier `thunderbird.cfg`. Thunderbird gérera la demande de mot de passe au premier lancement. Pour une sécurité accrue, explorez des solutions d'authentification unique (SSO) si votre environnement le permet.

## Références

[1] Customizing Thunderbird using policies.json. Disponible sur : [https://enterprise.thunderbird.net/manage-updates-policies-and-customization/managing-thunderbird-policies/customizing-thunderbird-using-policies.json](https://enterprise.thunderbird.net/manage-updates-policies-and-customization/managing-thunderbird-policies/customizing-thunderbird-using-policies.json)
[2] Mission Control Desktop (MCD) - Thunderbird AutoConfig. Disponible sur : [https://enterprise.thunderbird.net/deploy/mcd-thunderbird-autoconfig](https://enterprise.thunderbird.net/deploy/mcd-thunderbird-autoconfig)
[3] Thunderbird Autoconfiguration - Config file format. Disponible sur : [https://www.bucksch.org/1/projects/thunderbird/autoconfiguration/config-file-format.html](https://www.bucksch.org/1/projects/thunderbird/autoconfiguration/config-file-format.html)




## Configuration pour les comptes Office 365 (Authentification OAuth2)

Pour les comptes Office 365, Thunderbird utilise l'authentification OAuth2, qui est une méthode d'authentification moderne et sécurisée. La configuration via `autoconfig` doit être adaptée pour spécifier cette méthode.

### Adaptation du fichier `thunderbird.cfg` pour Office 365

Le fichier `thunderbird.cfg` doit être modifié pour inclure les paramètres spécifiques à Office 365, notamment les noms de serveurs et la méthode d'authentification OAuth2. Voici un exemple de `thunderbird.cfg` adapté :

```javascript
//
// Fichier de configuration pour Thunderbird
//

// Ne pas modifier la première ligne
//

// Configuration des comptes de messagerie pour Office 365
try {

  // Définir les préférences par défaut
  // REMPLACEZ <votre_domaine> par votre nom de domaine

  pref("mail.rights.version", 1);

  // Configurer le premier compte de messagerie
  var email = getenv("MAIL");
  if (email == "") {
    email = getenv("USER") + "@<votre_domaine>";
  }

  var realname = getenv("CN");
  if (realname == "") {
    realname = getenv("USER");
  }

  var username = getenv("USER");

  // Serveur entrant (IMAP) pour Office 365
  pref("mail.server.account1.hostname", "outlook.office365.com");
  pref("mail.server.account1.port", 993);
  pref("mail.server.account1.socket_type", "SSL");
  pref("mail.server.account1.authentication", "OAuth2");
  pref("mail.server.account1.name", email);
  pref("mail.server.account1.userName", email); // Pour Office 365, le nom d'utilisateur est généralement l'adresse e-mail complète
  pref("mail.server.account1.type", "imap");

  // Serveur sortant (SMTP) pour Office 365
  pref("mail.smtpserver.smtp1.hostname", "smtp.office365.com");
  pref("mail.smtpserver.smtp1.port", 587);
  pref("mail.smtpserver.smtp1.socket_type", "STARTTLS");
  pref("mail.smtpserver.smtp1.authentication", "OAuth2");
  pref("mail.smtpserver.smtp1.username", email); // Pour Office 365, le nom d'utilisateur est généralement l'adresse e-mail complète
  pref("mail.smtpserver.smtp1.description", "Serveur SMTP Office 365");

  // Identité
  pref("mail.identity.id1.fullName", realname);
  pref("mail.identity.id1.useremail", email);
  pref("mail.identity.id1.valid", true);
  pref("mail.identity.id1.smtpServer", "smtp1");

  // Association du compte et de l'identité
  pref("mail.account.account1.identities", "id1");
  pref("mail.account.account1.server", "server1");

  // Définir le compte par défaut
  pref("mail.accountmanager.defaultaccount", "account1");

} catch (e) {
  displayError("catch", e);
}
```

### Points importants pour Office 365

*   **Nom d'utilisateur** : Pour Office 365, le nom d'utilisateur est généralement l'adresse e-mail complète.
*   **Serveurs** : Utilisez `outlook.office365.com` pour IMAP (port 993, SSL/TLS) et `smtp.office365.com` pour SMTP (port 587, STARTTLS).
*   **Authentification** : La méthode d'authentification doit être `OAuth2`. Thunderbird gérera la fenêtre d'authentification Microsoft au premier lancement.
*   **Variables d'environnement** : Assurez-vous que les variables d'environnement `MAIL`, `USER`, et `CN` sont correctement définies dans votre environnement Active Directory pour que le script puisse récupérer les informations de l'utilisateur.

Le déploiement de ces fichiers (`autoconf.js` et le `thunderbird.cfg` modifié) via GPO reste le même que décrit précédemment dans la section "Déploiement des fichiers `autoconfig` via GPO".



## Gestion de plusieurs boîtes mails

Pour configurer plusieurs boîtes mails pour un même utilisateur, le script `thunderbird.cfg` doit être adapté pour définir chaque compte de messagerie, son serveur entrant (IMAP) et son serveur sortant (SMTP), ainsi que son identité associée.

### Adaptation du fichier `thunderbird.cfg` pour plusieurs boîtes mails

Le fichier `thunderbird.cfg` a été mis à jour pour inclure des sections pour trois comptes de messagerie distincts. Chaque compte (`account1`, `account2`, `account3`) a ses propres paramètres de serveur IMAP et SMTP, ainsi que son identité (`id1`, `id2`, `id3`).

**Points clés à noter :**

*   **Variables d'environnement ou valeurs statiques** : Pour le premier compte (principal), le script tente de récupérer l'adresse e-mail, le nom complet et le nom d'utilisateur à partir des variables d'environnement (`MAIL`, `CN`, `USER`). Pour les comptes supplémentaires (`account2`, `account3`), vous devrez **remplacer les placeholders** (`<adresse_email_compte2>`, `<nom_complet_compte2>`, etc.) par les informations spécifiques à chaque boîte mail.
*   **Numérotation des comptes** : Chaque compte, serveur et identité est numéroté de manière séquentielle (ex: `account1`, `server1`, `id1`, puis `account2`, `server2`, `id2`, etc.). Cette numérotation est cruciale pour que Thunderbird puisse identifier et gérer correctement chaque boîte mail.
*   **Liste des comptes** : La ligne `pref("mail.accountmanager.accounts", "account1,account2,account3");` liste tous les comptes qui doivent être configurés. Assurez-vous que tous les comptes que vous souhaitez déployer sont inclus dans cette liste.
*   **Compte par défaut** : La ligne `pref("mail.accountmanager.defaultaccount", "account1");` définit le compte qui sera affiché par défaut au lancement de Thunderbird. Vous pouvez modifier `account1` par `account2` ou `account3` si vous souhaitez qu'un autre compte soit le compte par défaut.

### Exemple de `thunderbird.cfg` pour plusieurs boîtes mails (Office 365)

```javascript
//
// Fichier de configuration pour Thunderbird
//

// Ne pas modifier la première ligne
//

// Configuration des comptes de messagerie pour Office 365
try {

  // Définir les préférences par défaut
  // REMPLACEZ <votre_domaine_principal> par votre nom de domaine principal
  // REMPLACEZ <adresse_email_compte2>, <nom_complet_compte2> par les informations du compte 2
  // REMPLACEZ <adresse_email_compte3>, <nom_complet_compte3> par les informations du compte 3

  pref("mail.rights.version", 1);

  // Configurer le premier compte de messagerie (principal)
  var email1 = getenv("MAIL");
  if (email1 == "") {
    email1 = getenv("USER") + "@<votre_domaine_principal>";
  }

  var realname1 = getenv("CN");
  if (realname1 == "") {
    realname1 = getenv("USER");
  }

  var username1 = email1; // Pour Office 365, le nom d'utilisateur est généralement l'adresse e-mail complète

  // Serveur entrant (IMAP) pour Office 365 - Compte 1
  pref("mail.server.server1.hostname", "outlook.office365.com");
  pref("mail.server.server1.port", 993);
  pref("mail.server.server1.socket_type", "SSL");
  pref("mail.server.server1.authentication", "OAuth2");
  pref("mail.server.server1.name", email1);
  pref("mail.server.server1.userName", username1);
  pref("mail.server.server1.type", "imap");

  // Serveur sortant (SMTP) pour Office 365 - Compte 1
  pref("mail.smtpserver.smtp1.hostname", "smtp.office365.com");
  pref("mail.smtpserver.smtp1.port", 587);
  pref("mail.smtpserver.smtp1.socket_type", "STARTTLS");
  pref("mail.smtpserver.smtp1.authentication", "OAuth2");
  pref("mail.smtpserver.smtp1.username", username1);
  pref("mail.smtpserver.smtp1.description", "Serveur SMTP Office 365 - Compte Principal");

  // Identité - Compte 1
  pref("mail.identity.id1.fullName", realname1);
  pref("mail.identity.id1.useremail", email1);
  pref("mail.identity.id1.valid", true);
  pref("mail.identity.id1.smtpServer", "smtp1");

  // Association du compte et de l'identité - Compte 1
  pref("mail.account.account1.identities", "id1");
  pref("mail.account.account1.server", "server1");

  // Configurer le deuxième compte de messagerie
  var email2 = "<adresse_email_compte2>"; // REMPLACEZ par l'adresse e-mail du deuxième compte
  var realname2 = "<nom_complet_compte2>"; // REMPLACEZ par le nom complet du deuxième compte
  var username2 = email2;

  // Serveur entrant (IMAP) pour Office 365 - Compte 2
  pref("mail.server.server2.hostname", "outlook.office365.com");
  pref("mail.server.server2.port", 993);
  pref("mail.server.server2.socket_type", "SSL");
  pref("mail.server.server2.authentication", "OAuth2");
  pref("mail.server.server2.name", email2);
  pref("mail.server.server2.userName", username2);
  pref("mail.server.server2.type", "imap");

  // Serveur sortant (SMTP) pour Office 365 - Compte 2
  pref("mail.smtpserver.smtp2.hostname", "smtp.office365.com");
  pref("mail.smtpserver.smtp2.port", 587);
  pref("mail.smtpserver.smtp2.socket_type", "STARTTLS");
  pref("mail.smtpserver.smtp2.authentication", "OAuth2");
  pref("mail.smtpserver.smtp2.username", username2);
  pref("mail.smtpserver.smtp2.description", "Serveur SMTP Office 365 - Compte 2");

  // Identité - Compte 2
  pref("mail.identity.id2.fullName", realname2);
  pref("mail.identity.id2.useremail", email2);
  pref("mail.identity.id2.valid", true);
  pref("mail.identity.id2.smtpServer", "smtp2");

  // Association du compte et de l'identité - Compte 2
  pref("mail.account.account2.identities", "id2");
  pref("mail.account.account2.server", "server2");

  // Configurer le troisième compte de messagerie
  var email3 = "<adresse_email_compte3>"; // REMPLACEZ par l'adresse e-mail du troisième compte
  var realname3 = "<nom_complet_compte3>"; // REMPLACEZ par le nom complet du troisième compte
  var username3 = email3;

  // Serveur entrant (IMAP) pour Office 365 - Compte 3
  pref("mail.server.server3.hostname", "outlook.office365.com");
  pref("mail.server.server3.port", 993);
  pref("mail.server.server3.socket_type", "SSL");
  pref("mail.server.server3.authentication", "OAuth2");
  pref("mail.server.server3.name", email3);
  pref("mail.server.server3.userName", username3);
  pref("mail.server.server3.type", "imap");

  // Serveur sortant (SMTP) pour Office 365 - Compte 3
  pref("mail.smtpserver.smtp3.hostname", "smtp.office365.com");
  pref("mail.smtpserver.smtp3.port", 587);
  pref("mail.smtpserver.smtp3.socket_type", "STARTTLS");
  pref("mail.smtpserver.smtp3.authentication", "OAuth2");
  pref("mail.smtpserver.smtp3.username", username3);
  pref("mail.smtpserver.smtp3.description", "Serveur SMTP Office 365 - Compte 3");

  // Identité - Compte 3
  pref("mail.identity.id3.fullName", realname3);
  pref("mail.identity.id3.useremail", email3);
  pref("mail.identity.id3.valid", true);
  pref("mail.identity.id3.smtpServer", "smtp3");

  // Association du compte et de l'identité - Compte 3
  pref("mail.account.account3.identities", "id3");
  pref("mail.account.account3.server", "server3");

  // Lister tous les comptes configurés
  pref("mail.accountmanager.accounts", "account1,account2,account3");

  // Définir le compte par défaut (peut être account1, account2 ou account3)
  pref("mail.accountmanager.defaultaccount", "account1");

} catch (e) {
  displayError("catch", e);
}
```

### Déploiement via GPO

Le déploiement des fichiers `autoconf.js` et `thunderbird.cfg` via GPO reste le même que décrit précédemment dans la section "Déploiement des fichiers `autoconfig` via GPO". Assurez-vous simplement que le fichier `thunderbird.cfg` mis à jour est bien celui qui est copié sur les postes clients.

