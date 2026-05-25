# LAB-15-Analyse-Dynamique-Android-Inspection-TLS-HTTPS-et-Gestion-du-SSL-Pinning
# Étape 1 — Installation de Frida et démarrage de frida-server

## 1.1 Installation de Frida sur le poste de travail

La première étape consiste à installer les outils Frida sur la machine Windows utilisée pour l’analyse dynamique Android.

Frida permet l’instrumentation dynamique des applications mobiles pendant leur exécution afin d’intercepter ou modifier certains comportements internes.

Exemple d’installation :

```bash id="p8u2fd"
pip install frida-tools
```

Une fois installé, Frida devient accessible depuis PowerShell ou l’invite de commande.

**Capture associée :**

<img width="744" height="157" alt="image" src="https://github.com/user-attachments/assets/9e7ea138-7b1d-44da-b2c8-9cf6f1951e14" />


---

## 1.2 Préparation de l’environnement ADB et de l’appareil Android

Avant d’utiliser Frida, il est nécessaire de vérifier la communication entre le poste de travail et l’émulateur Android.

Commande utilisée :

```bash id="x4n7ka"
adb devices
```

Cette commande affiche les appareils Android détectés via ADB.

La présence de l’émulateur dans la liste confirme que la communication est correctement établie.

**Capture associée :**

<img width="610" height="108" alt="image" src="https://github.com/user-attachments/assets/c2f1bfc8-051b-4196-b26c-0a02dbe2f50e" />

---

## 1.3 Déploiement et démarrage de `frida-server`

Après la connexion ADB, le binaire `frida-server` est transféré vers l’émulateur Android.

Les opérations réalisées incluent généralement :

* transfert du fichier ;
* ajout des permissions d’exécution ;
* lancement du service Frida.

Exemple :

```bash id="r9m5hv"
adb push frida-server /data/local/tmp/
```

```bash id="j3q8tw"
adb shell chmod 755 /data/local/tmp/frida-server
```

```bash id="z6u1pc"
adb shell /data/local/tmp/frida-server &
```

Le démarrage de `frida-server` permet à Frida d’interagir avec les processus Android actifs.

**Captures associées :**

<img width="1099" height="759" alt="image" src="https://github.com/user-attachments/assets/ad20f6f5-8807-4f36-b2f5-24f5363b7990" />


---

# Étape 2 — Lancement de l’application cible sous Frida

Une fois `frida-server` opérationnel, l’application Android peut être lancée sous instrumentation dynamique.

Exemple :

```bash id="h1v9de"
frida -U -f com.example.app
```

### Explication

| Paramètre         | Fonction                        |
| ----------------- | ------------------------------- |
| `-U`              | Connexion à l’émulateur Android |
| `-f`              | Lance l’application cible       |
| `com.example.app` | Package Android analysé         |

Cette étape prépare l’environnement pour l’injection des hooks Java et des scripts de bypass.

**Capture associée :**

<img width="537" height="277" alt="image" src="https://github.com/user-attachments/assets/354ef3d0-66ae-4e22-ac25-84a051c506ae" />

---

# Étape 3 — Script Java « universel » de bypass SSL Pinning

Un script Frida Java peut être utilisé afin d’intercepter certains mécanismes de validation TLS/SSL présents dans l’application Android.

Le principe consiste à :

* neutraliser les vérifications de certificats ;
* contourner le SSL pinning ;
* autoriser les connexions HTTPS interceptées.

Ces hooks sont injectés dynamiquement dans l’application via Frida pendant son exécution.

L’objectif principal est d’analyser le trafic HTTPS dans un environnement pédagogique contrôlé.

**Capture associée :**

<img width="730" height="231" alt="image" src="https://github.com/user-attachments/assets/d73360dc-fc64-4f25-a29c-435da8ae1ae5" />

---

# Étape 4 — Variantes et protections spécifiques

Certaines applications Android utilisent plusieurs couches de protection TLS/SSL simultanément.

Dans ce cas, les bypass génériques deviennent insuffisants et nécessitent une approche ciblée.

**Captures associées :**

<img width="730" height="231" alt="image" src="https://github.com/user-attachments/assets/42d2d0e3-228d-4d2c-91be-b86ea1e51842" />

---

# 5.1 Identification des bibliothèques utilisées

L’analyse des classes Java chargées en mémoire a permis d’identifier plusieurs composants liés à la validation TLS/SSL et aux mécanismes de confiance réseau.

Les éléments détectés incluent :

| Composant                        | Fonction                                       |
| -------------------------------- | ---------------------------------------------- |
| `com.android.okhttp`             | Bibliothèque réseau HTTP/HTTPS                 |
| `CertificatePinner`              | Implémentation du SSL Pinning dans OkHttp      |
| `javax.net.ssl.X509TrustManager` | Validation des certificats X.509               |
| `TrustManagerImpl` (Conscrypt)   | Gestionnaire interne de validation TLS Android |

Ces résultats montrent que l’application utilise plusieurs mécanismes de validation combinés afin de renforcer la sécurité des communications réseau.

Cette architecture augmente considérablement la résistance aux attaques de type MITM (*Man-in-the-Middle*) et complique les opérations de bypass via instrumentation dynamique.

---

# 5.2 Classification du mécanisme de protection

L’analyse permet de classifier le système de protection comme un mécanisme de :

## Double SSL Pinning (OkHttp + TrustManager Android)

Cette architecture applique plusieurs couches indépendantes de validation TLS.

---

## Couche 1 — `OkHttp CertificatePinner`

Le composant `CertificatePinner` effectue une vérification explicite des empreintes de certificats serveur.

Même si le certificat est accepté par Android, la connexion est rejetée si l’empreinte ne correspond pas à celle attendue par l’application.

---

## Couche 2 — TrustManager Android (Conscrypt)

Le `TrustManager` Android intervient au niveau système afin de :

* valider la chaîne de certification ;
* vérifier l’autorité de certification (CA) ;
* contrôler l’intégrité des certificats TLS.

Cette combinaison crée une protection réseau renforcée contre les interceptions HTTPS.

---

# 5.3 Stratégie de contournement adaptée

Face à plusieurs couches de validation TLS, une approche modulaire a été utilisée afin de limiter les effets secondaires et préserver la stabilité de l’application.

---

## Étape 1 — Bypass du `CertificatePinner` OkHttp

La première phase cible :

```text id="n5r2mw"
CertificatePinner.check()
```

Le hook intercepte cette méthode afin de neutraliser la vérification des empreintes de certificats.

Les connexions HTTPS sont alors acceptées sans appliquer le SSL pinning défini par l’application.

---

## Étape 2 — Neutralisation du `TrustManager` Android

La seconde phase cible les méthodes :

```text id="c4k8zt"
checkServerTrusted()
```

```text id="v7u9xd"
checkClientTrusted()
```

Ces hooks permettent d’accepter dynamiquement tous les certificats présentés pendant l’échange TLS.

L’objectif est de contourner les validations système indépendamment de l’autorité de certification utilisée.

Cette approche progressive améliore la stabilité de l’application et réduit les risques de détection.

---

# 5.4 Importance d’une approche ciblée

Les scripts universels de bypass SSL peuvent provoquer plusieurs problèmes lorsqu’ils modifient simultanément un grand nombre de classes sensibles.

Les risques observés incluent :

* crash immédiat de l’application ;
* détection de Frida ;
* déclenchement de protections natives `.so` ;
* fermeture volontaire du processus ;
* blocage des fonctionnalités réseau.

Dans cette application, la présence probable de mécanismes anti-instrumentation impose donc une approche sélective centrée uniquement sur les composants réellement utilisés.

Cette méthodologie améliore :

* la stabilité ;
* la discrétion ;
* la fiabilité de l’analyse dynamique.

---

# 5.5 Conclusion de l’étape

Cette phase démontre que le succès d’un bypass SSL/TLS dépend principalement de la compréhension de l’architecture de sécurité de l’application analysée.

Les résultats mettent en évidence plusieurs points essentiels :

* l’identification préalable des bibliothèques utilisées est indispensable ;
* chaque mécanisme TLS doit être traité individuellement ;
* les hooks universels peuvent provoquer des instabilités importantes ;
* une approche modulaire améliore fortement la stabilité du bypass.

Ainsi, l’analyse dynamique avancée d’applications Android protégées nécessite une adaptation continue des techniques de hooking selon les mécanismes observés pendant l’exécution.
