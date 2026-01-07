+++
draft = false
date = 2026-01-07T00:25:35+01:00
title = "Autopsie d'une sonnette connectée"
description = "Analyse et root d'une sonnette DiO Bell B01"
slug = "autopsie-sonnette-connectee"
authors = ["RedFish"]
tags = ["hardware", "hacking", "uart"]
categories = ["writeup"]
externalLink = ""
series = []
+++

### **Avant-propos**
Profitant de quelques jours de congés à Noël, j'ai décidé d'approfondir mes connaissances en _hardware hacking_ en m'attaquant à la sonnette **DiO Bell B01**. L'objectif était de comprendre son fonctionnement interne et de tenter d'obtenir un accès complet (root).

***Ce rapport est avant tout un journal d'expérimentation avec ses réussites et ses ratés. Il existe sûrement des méthodes plus optimisées pour arriver au même résultat, mais l'essentiel ici était d'apprendre.***


### Accès au port série (UART)
L'objectif est dans un premier temps d'analyser la sonnette et d'y trouver des ports de debug (uart) afin de s'y connecter.


L'UART est un protocole de communication très courant pour le débogage. Il est généralement composé de 3 ou 4 broches :

- **GND (Ground) :** La masse (référence 0V).
- **TX (Transmit) :** Sert à **envoyer** des données (depuis la sonnette).
- **RX (Receive) :** Sert à **recevoir** des données (vers la sonnette).
- **VCC :** L'alimentation


En observant la carte, on remarque 4 trous alignés qui semblent correspondre à cette interface. Nous devons maintenant identifier le rôle de chaque broche.

![](/images/sonnette/Pasted_image_20260105214928.png)

#### Identifier la masse (GND)

Pour cela, nous allons utiliser un multimètre en mode **"Continuité"**. Dans ce mode, le multimètre sonne lorsque la r"sistance entre les connecteurs est à 0.
- On place la sonde noire du multimètre sur une masse connue de l'appareil par exemple  le pôle négatif de l'alimentation).
- Avec la sonde rouge, on teste les 4 broches inconnues une par une.

Dès que le multimètre émet un bip, nous avons trouvé le GND.

#### Identifier TX et RX

Une fois le GND repéré, il faut différencier le TX du RX. Nous passons le multimètre en mode Voltmètre . Le principe est de tester les broches restantes **pendant la phase de boot** de la sonnette :

* **Identification du TX :** Au démarrage, la sonnette envoie des logs de debug. La tension sur la broche TX va donc fluctuer rapidement (oscillation entre 0 et 3.3V) durant les premières secondes. 
* **Identification du RX :** Cette broche, en attente de données, reste stable à une tension haute.

Cette méthode permet d'identifier formellement le : GND, RX et TX.

### Connexion via un Arduino UNO (Hack USB-TTL)

Pour communiquer avec l'interface UART de l'appareil, un adaptateur USB vers TTL est normalement requis. Ce module sert de pont entre le port USB de l'ordinateur et les broches série de la cible.

N'en ayant pas sous la main, je vais utiliser un Arduino UNO. Pour ce faire, il faut transformer l'Arduino en convertisseur "passif" sans que son propre processeur n'interfère avec les signaux. La manipulation consiste à relier la broche **RESET** de l'Arduino à la broche **GND**. Cela a pour effet de maintenir le microcontrôleur principal en état de redémarrage permanent, le rendant inactif.


On passe ensuite au câblage des  connecteurs. Les pins RX/TX sur le PCB de l'Arduino correspondent aux lignes du convertisseur USB. On branche donc : 
* La pin **RX** de la sonnette sur le **RX** de l'Arduino (Pin 0). 
* La pin **TX** de la sonnette sur le **TX** de l'Arduino (Pin 1).

Enfin, on relie le **GND** de l'Arduino au **GND** de la sonnette pour avoir une masse commune.

![](/images/sonnette/Pasted_image_20260105235435.png)


***Remarque: La sonnette étant déjà alimentée par sa propre source, il n'est pas nécessaire de relier la broche VCC***

![](/images/sonnette/PXL_20251226_171538196.jpg)

Sur le PC, on va utiliser l'outil **tio** pour lire et communiquer avec le protocole UART.

![](/images/sonnette/uart_1_2.png)


Quelques infos apparaissent ici, comme le nom de l'appareil (BELL11S), mais les infos affichées restent limitées. De plus, aucune interaction ne semble possible avec l'appareil : j'ai beau taper des commandes "help" ou faire des "retours chariot" (Entrée), le terminal semble figé...

Le message "key up, to startup app" semble indiquer qu'il serait possible d'intercepter le démarrage en appuyant sur un bouton.

En regardant de plus près la carte électronique, un bouton "Reset" est accessible. En le maintenant appuyé lors du démarrage, le comportement change et les logs deviennent beaucoup plus verbeux.

![](/images/sonnette/Pasted_image_20260105235830.png)


Le message "key down, to startup uboot" s'affiche, confirmant que le bouton Reset correspond bien à la touche d'interruption attendue par le système.

Cependant, on ne semble pas pouvoir accéder à un shell directement. Un mot de passe est demandé par l'appareil lors du processus de boot: "please input password::".

![](/images/sonnette/Pasted_image_20260103085039.png)

J'ai testé plusieurs mots de passe par défaut couramment utilisés par les fabricants ("admin", "password", "root", "123456"...),  sans succès....


Les logs indiquent  que le démarrage est géré par **U-Boot**. Puisque l'accès console est verrouillé, nous allons devoir investiguer plus en profondeur pour contourner cette authentification.

### Tentative de bypass du mot de passe boot

La ligne suivante nous donne plus d'informations sur l'os et la puce de la sonnette:

![](/images/sonnette/Pasted_image_20260103090756.png)

Le processeur est identifié comme un **Hi3518EV300**. Il s'agit d'une puce fabriquée par **HiSilicon** (une filiale de Huawei), très courante dans le monde des caméras IP.

En  cherchant les mots de passe couramment utilisé par la marque HiSillicon, je suis tombé sur le github suivant: 

https://gist.github.com/gabonator/74cdd6ab4f733ff047356198c781f27d

Plus d'une cinquantaine de mots de passe y sont répertoriés pour les appareils HiSilicon. Malheureusement, après les avoir testés un par un, aucun ne fonctionne sur la sonnette.

 ### La méthode "Fault Injection" 

En cherchant d'autres vecteurs d'attaque  je suis tombé sur une vidéo de **Matt Brown**: https://www.youtube.com/watch?v=F-G-7-qo7Xg . Il y démontre comment contourner le prompt de mot de passe U-Boot via une attaque par **Fault Injection** (injection de fautes).

**Le principe :**
L'objectif est de corrompre la lecture des données pile au moment où U-Boot tente de charger le noyau Linux (Kernel) depuis la mémoire Flash vers la RAM.  Si U-boot n'arrive pas a lire le noyau correctement, il se met en mode sécurité et permet d'accéder à la console interactive que l'on cherche a atteindre.

**La pratique :**
Pour ce faire, il relie la pin Data Out (DO) de la puce mémoire flash au GND pile au moment du chargement du noyau.

Sur ma sonnette, la mémoire flash est la QH64A.
En regardant la datasheet, on identifie la pin Data Out (pin 2).

![](/images/sonnette/Pasted_image_20260103201857.png)


J’accroche directement un fil  sur la broche **DO (Pin 2)** de la mémoire Flash. L'autre extrémité prête à être mise en contact  avec une zone de masse (**GND**) de la carte.

![](/images/sonnette/PXL_20251227_142550521.jpg)

je  fais ensuite toucher le fil DO sur le GND  un peu avant qu'il me demande d'entrer mon mot de passe.

![](/images/sonnette/PXL_20251227_142210710.jpg)


En court-circuitant la mémoire Flash juste avant le chargement du noyau, des erreurs apparaissent bien dans la console. 

![](/images/sonnette/UART_test_bypass.png)

Le système affiche les valeurs des registres (signe d'un crash), mais au lieu de me donner la main sur un shell, la puce finit systématiquement par redémarrer le CPU.

Malgré de multiples tentatives en variant le timing, aucun shell n'a pu être obtenu. La méthode de l'injection de fautes ne semble donc pas fonctionner ici....


### Cette fois-ci c'est la bonne !
Les tentatives précédentes pour contourner le mot de passe ayant échoué, nous passons à la méthode forte : l'extraction directe du firmware pour y dénicher le sésame.

Pour ce faire, j'utilise un programmateur universel **T48**. C'est un outil matériel qui se branche en USB et permet de lire et écrire le contenu brut ("Dump") de milliers de puces mémoire.

Afin d'interagir avec la mémoire sans avoir à la dessouder, j'utilise une pince de test, qui vient se fixer directement sur les broches du composant. La pince est reliée au T48, ce qui permet d'interfacer la puce avec le logiciel de gestion **Xgpro**.

Je branche donc la pince sur la puce mémoire de l'appareil.

![](/images/sonnette/PXL_20251229_182745034.jpg)


Dans le logiciel, je sélectionne la référence exacte de la mémoire Flash identifiée sur la carte : la **XM25QH64A**.
![](/images/sonnette/XPRO2.png)

Je peux finalement lire le contenu de la puce:

![](/images/sonnette/XPRO3.png)

J'en extrais le firmware.  C'est un  fichier binaire difficilement exploitable tel quel.

En exécutant un binwalk sur notre fichier, il analyse les signatures hexadécimales pour identifier les différentes parties du firmware. 


![](/images/sonnette/Pasted_image_20260103095509.png)

On observe que le firmware est composé de 3 grandes sections : 
* **u-boot.bin** : Le bootloader. 
* **app.img** : Le noyau ou l'application principale. 
* **jffs2** : Un système de fichiers compressé.


Binwalk permet également d'extraire automatiquement ces partitions pour nous permettre de les analyser individuellement via la commande : `binwalk -e XM25QH64A@SOIC8.BIN`

Une fois l'extraction terminée, nous obtenons un dossier `_XM25QH64A@SOIC8.BIN.extracted` contenant les différentes partitions. 

![](/images/sonnette/Pasted_image_20260103095926.png)


La partition qui nous intéresse en priorité est **u-boot.bin**. C''est le bootloader,qui nous demande le mot de passe.

Pour l'analyser, j'utilise l'outil **Ghidra**

Avant toute chose, il est important de voir **l'importance de la Base Address** ! 

Pour que Ghidra puisse comprendre les sauts  et les références aux variables, il est crucial de définir la "Base Address" (l'adresse virtuelle de départ). 
Sans cela, Ghidra charge le binaire à l'adresse `0x0000`, alors que dans la réalité, le processeur exécute ce code à une adresse bien spécifique en RAM. Si la base Adresse n'est pas bonne, tous les pointeurs (ex: "va chercher le texte à l'adresse 0x40847D50") pointeront au mauvais endroit ou dans le vide . 
Pour trouver cette adresse, j'analyse le fichier brut à la recherche  de la commande de démarrage (bootcmd). 


![](/images/sonnette/Pasted_image_20260103102202.png)

On voit ici que l'adresse de départ du noyau est 0x40000000.
Le système charge donc le noyau à l'adresse 0x40000000, c'est donc cette valeur que nous allons utiliser comme point de départ pour aligner le code.

Dans Ghidra, à l'importation du fichier, je configure donc : 
* **Language :** ARM:LE:32:Cortex:default (Le Hi3518EV300 est un Cortex A7 en **Little Endian**). 
* **Base Address :** 0x40000000.

![](/images/sonnette/ghidra2_1.png)

### Décompilation
La méthode la plus simple pour trouver une fonction d'authentification est de chercher les chaînes de caractères (strings) affichées lors du login. Je lance une recherche sur le mot "password".

![](/images/sonnette/Pasted_image_20260103111036.png)

Deux résultats intéressants ressortent :
1. `please input password::` (Le prompt)
2. `password err` (Le message d'erreur) 

Je décide de me concentrer sur **"password err"**. Si je trouve l'endroit où ce message est utilisé, je trouverai forcément la condition qui a échoué juste avant (la comparaison du mot de passe).

En demandant à Ghidra les références croisées (XREFS) vers cette string, je tombe sur une fonction unique.

![](/images/sonnette/Pasted_image_20260103111121.png)

En double-cliquant dessus, j'atterris dans le désassembleur, et Ghidra me génère une vue décompilée en C

![](/images/sonnette/Pasted_image_20260103115520.png)

La chaine semble être dans un printf.
On suppose donc que la comparaison avec l'entrée utilisateur et le mot de passe s’effectue juste avant.


Après un peu de nettoyage et de renommage des variables pour rendre le code lisible, la logique de la fonction apparaît clairement :

![](/images/sonnette/Pasted_image_20260103115815.png)

La ligne 36:

```c
uVar5 = FUN_40831f40((int)&stack0x00000004,DAT_40814a5c);
```
Semble être un strcmp qui compare l'entrée utilisateur (le pointeur à stack0x00000004 ) et  le password (DAT_40814a5c).


On aurait donc le code suivant simplifié et annoté:

```c
printf("please input password::");
// ... lecture de l'entrée utilisateur ...

// Comparaison de l'entrée utilisateur avec le vrai mot de passe
is_valid = check_password((int)&user_input, encrypted_password_ptr);

if (is_valid != 0) {
    printf("password err\n");
....
```

en double cliquant sur la variable DAT_40814a5c, on arrive l'adresse physique `0x40847D50` où est stocké le mot de passe de référence.

![](/images/sonnette/Pasted_image_20260103120122.png)


####  Le secret "chiffré"
En se rendant à l'adresse 0x40847D50 on s'attend à voir le mot de passe en clair (ASCII). 
Mais surprise :

![](/images/sonnette/Pasted_image_20260103120257.png)

Les valeurs (`8F 8F 8C...`) ne correspondent à aucun texte lisible. Les octets sont trop élevés (supérieurs à `0x7F`) pour de l'ASCII standard. Cela peut indiquer une **obfuscation**.


Face à ces données brutes, j'ai demandé de l'aide à ChatGPT pour identifier le motif. L'IA a rapidement mis en évidence que la chaîne est simplement "inversée" (Bitwise NOT). 
Pour déchiffrer le mot de passe, il suffit d'effectuer l'opération mathématique : `0xFF - Valeur`.

* Exemple : `0xFF - 0x8F = 0x70` ('p') 
 
En appliquant cette logique à toute la suite d'octets,on trouve le mot de passe final: ***pps_password***.

### Dive into UART

De retour sur la console UART, je redémarre la sonnette. Au prompt du mot de passe, je saisis : `pps_password`.

**Succès !** L'accès au shell est déverrouillé. Le prompt change pour `pps #`.

![](/images/sonnette/Pasted_image_20260106001548.png)


Cependant, en essayant des commandes classiques (`ls`, `pwd`), je me rends vite compte qu'il ne s'agit pas d'un shell Linux standard, mais du shell interactif de **U-Boot**


Une méthode pour obtenir un accès shell root sur un système embarqué consiste à modifier les variables d'environnement au démarrage afin de forcer l'exécution d'un shell (`/bin/sh`) à la place du programme initial.

Je commence par afficher la configuration actuelle via la commande `printenv` :

![](/images/sonnette/Pasted_image_20260105234627.png)

L'analyse des variables `bootargs` nous montre les paramètres passés au noyau. Je tente alors de les modifier pour injecter l'init manuellement :

![](/images/sonnette/Pasted_image_20260105234712.png)

Malheureusement, la sonnette freeze au démarrage, ce qui laisse supposer que le système d'exploitation n'est pas un Linux standard ou qu'il ne possède pas de binaire `/bin/sh` accessible.

Il va falloir trouver une autre porte d'entrée.

### Avoir un Shell
On a donc toujours pas de réel shell sur l'appareil! Il faut aller plus loin!

![](/images/sonnette/agl7ze.jpg)

Je branche ma sonnette connecté et je la configure sur mon réseau WIFI.
Une fois connectée à mon point d'accès Wi-Fi, je récupère son adresse IP et lance un scan de ports complet.

![](/images/sonnette/Pasted_image_20260105222315.png)


Un port  est ouvert dessus : le  **8090**. Dans la documentation rien n'y fait mention. 
En tentant d'y accéder via un navigateur, je reçois une erreur générique, mais le serveur répond.

![](/images/sonnette/Pasted_image_20260105222610.png)


J'utilise ensuite l'outil `gobuster` pour énumérer les URLs accessibles sur ce serveur web.

![](/images/sonnette/Pasted_image_20260105222834.png)

Le serveur HTTP de la sonnette gère mal les connexions multiples et renvoie une réponse inattendue, provoquant une erreur d'affichage dans Gobuster :

```html
Unsolicited response received on idle HTTP channel starting with "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\nConnection: close\r\n\r\n<html><body><h>Enjoy your smart home camera!</h><br/><br/><ul style=\"list-style-type:circle\"><li><a href=\"/flash/upgrade/release_package\"> upgrading from upgrade.bin </a></li><li><a href=\"/flash/upgrade/percent\"> view upgrading percent </a></li><li><a href=\"/devices/deviceinfo\"> view device information </a></li><li><a href=\"/sys/reboot\"> reboot device </a></li><li><a href=\"/sys/sleep\"> sleep device</a></li><li><a href=\"/sys/active\"> active device</a></li><li><a href=\"/sys/console\"> ....
```

Dans le bloc de code HTML brut recraché par l'erreur, le serveur fait fuiter la liste complète des actions disponibles sous forme de liens :
- `/flash/upgrade/release_package`
- `/sys/reboot`
- `/log/upload`
- `/sys/console`

L'URL `http://192.168.1.139:8090/sys/console` semble prometteuse! 
Cependant, en tentant d'y accéder, le navigateur demande une authentification **HTTP Basic Auth**.

![](/images/sonnette/Pasted_image_20260105223104.png)

Les identifiants `admin:admin`, `root:password`...  échouent. Il va falloir retourner dans le firmware pour trouver ces accès.
#### Reverse Engineering de la partition `app.img`

Nous savons que `u-boot.bin` contenait le mot de passe du bootloader. Les identifiants de l'application Web doivent logiquement se trouver dans la partition système : **`app.img`**.

J'extrais donc cette partition et je l'analyse avec Ghidra. Mon objectif est de trouver où le serveur Web vérifie l'en-tête "Authorization".


Plutôt que de chercher manuellement dans des milliers de lignes de code assembleur, je vais plutôt utiliser  l'analyse assistée par IA via le **Model Context Protocol (MCP)**.

Le **Model Context Protocol (MCP)** est un standard ouvert qui permet de connecter des assistants d'intelligence artificielle (comme Claude) à des outils locaux et des contextes de données. Concrètement, cela permet à l'IA ne pas se contenter de "chat", mais de lire directement le code décompilé dans Ghidra et d'y exécuter des recherches.

#### **Recherche des identifiants assistée par IA (MCP)**

Pour réaliser cette interface entre l'IA et Ghidra, j'utilise donc deux briques  :

 1. **L'extension GhidraMCP :** `https://github.com/LaurieWired/GhidraMCP` 
	 Ce plugin transforme l'instance de Ghidra en un serveur local. L'IA s'y connecte et peut ainsi invoquer directement les fonctions de l'API de Ghidra pour effectuer des tâches complexes, comme rechercher des strings, analyser des références ou décompiler une fonction, le tout sans intervention manuelle.
    
2. **Le Client Claude Desktop (Linux) :** L'application officielle Claude n'étant pas nativement disponible sous Linux, j'utilise: `https://github.com/aaddrick/claude-desktop-debian`


Plugin GhidraMCP sur Ghidra:

![](/images/sonnette/Pasted_image_20260106001939.png)


Le prompt que j'ai ensuite utilisé sur Claude est le suivant: 

```html
Agis comme un expert en rétro-ingénierie. Je suspecte que des identifiants "Basic Auth" (HTTP) sont codés en dur dans ce binaire. Utilise les outils Ghidra pour mener l'enquête en suivant ces étapes :

1. Recherche de chaînes suspectes : Cherche toutes les chaînes de caractères (strings) contenant "Authorization", "Basic ", "user", "password", "login" ou "auth".
2. Analyse des références (XREFs) : Pour chaque chaîne pertinente trouvée, identifie la fonction qui l'utilise.
3. Décompilation : Décompile la fonction (ou les fonctions) qui manipulent ces chaînes pour comprendre le contexte. Regarde comment la chaîne est construite.
4. Détection Base64 : Le Basic Auth étant souvent sous la forme `base64(user:pass)`, cherche si des chaînes étranges (alphanumériques longues terminant par `=` ou `==`) sont passées à une fonction de décodage ou concaténées après "Basic ". Résume tes trouvailles : indique le nom de la fonction, l'adresse mémoire, et les valeurs potentielles du user/pass si elles sont lisibles.
```

Claude prend alors le contrôle de l'API Ghidra, scanne la mémoire à la recherche des chaînes spécifiées et analyse le graphe d'appel des fonctions pour comprendre comment ces chaînes sont utilisées.

![](/images/sonnette/Pasted_image_20260105224843.png)

Finalement l'IA identifie trois paires d'identifiants potentiels codés en dur dans le binaire :

![](/images/sonnette/Pasted_image_20260105225011.png)

Je teste ces identifiants sur la page de login du port 8090 (`/sys/console`). Les deux premiers ne fonctionnent pas. Cependant, le troisième couple fonctionne :
`WeEyE / &$ChuTian_91`

![](/images/sonnette/Pasted_image_20260105225218.png)

La page web affiche alors un message  : `=====device console success=====`. 

Je m'attendais à voir apparaître un terminal interactif directement dans le navigateur, mais rien ne se passe.


Je tente alors de me reconnecter à l'interface UART.

Et là, à l'instant où la connexion série s'établit, je tombe directement sur un shell actif!

![](/images/sonnette/Pasted_image_20260105225747.png)


Il semble donc que l'authentification réussie sur l'endpoint `/sys/console` ait agi comme un interrupteur, activant l'accès shell sur le port série qui était jusqu'alors bridé.


A suivre...