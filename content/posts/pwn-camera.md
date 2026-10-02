+++
draft = false
date = 2026-10-02T00:00:00+02:00
title = "PWN d'une caméra connectée"
description = "Analyse et root d'une caméra IP low-cost"
slug = "pwn-camera-connectee"
authors = ["RedFish"]
tags = ["hardware", "hacking", "firmware", "squashfs"]
categories = ["writeup"]
externalLink = ""
series = []
+++

### **Avant-propos**
Après m'être attaqué à une sonnette connectée, j'ai décidé de récidiver sur une **caméra IP low-cost** achetée quelques euros. L'objectif était simple : obtenir un accès shell sur l'appareil.

***Ce rapport est avant tout un journal d'expérimentation avec ses réussites et ses ratés. Il existe sûrement des méthodes plus optimisées pour arriver au même résultat, mais l'essentiel ici était d'apprendre.***

### Recherche d'un port série (UART)
Comme pour la sonnette, le réflexe naturel est de chercher des pins d'interface série (UART) sur la carte mère de la caméra, afin d'obtenir un accès direct au shell.

Après inspection visuelle de la carte et tests au multimètre : rien. Aucun pin UART exploitable n'a été trouvé sur ce modèle...

Il va donc falloir trouver une autre porte d'entrée.

### Dump du firmware
Nouvelles approche : on va dumper le firmware de la caméra et essayer de le modifier pour y injecter un accès shell.

Pour ce faire, on se connecte à la puce mémoire flash à l'aide d'un programmateur **XGecu T56**, puis on utilise le logiciel **Xgpro** pour effectuer un "Read" complet de la puce. Cela permet de récupérer l'image binaire complète du firmware.

![](/images/camera/xgpro_read.png)

Ici, on arrive bien à dumper le firmware de la caméra. C'est un fichier binaire difficilement exploitable tel quel, mais on va pouvoir le manipuler.

#### Tentative 1 : modifier directement le firmware

Ma première idée a été de chercher directement dans le firmware un script qui serait lancé au démarrage et de l'éditer afin d'y ajouter un reverse shell.

Le problème majeur ici, c'est qu'il faut absolument éviter de modifier la taille du firmware, sous peine de casser sa structure interne ou les vérifications d'intégrité.

Pour contourner cette contrainte, il fallait trouver une partie du code existant que l'on pouvait remplacer par notre payload. C'est possible par exemple dans un script shell contenant des commentaires : on peut raccourcir un commentaire et insérer notre commande malveillante à la place.

En fouillant le dump, j'ai identifié un script manifestement exécuté au boot de la caméra : on y voit notamment le montage du "usr file system".

![](/images/camera/firmware_boot_script.png)

Ce script contient des commentaires, parfait pour y dissimuler le payload.

Le principe tient en une ligne : on remplace une partie d'un commentaire existant par notre commande, suivie d'un `#` qui re-commente la fin de la ligne. Le nombre de caractères reste identique, donc la taille du firmware aussi :

```bash
# AVANT — commentaire d'origine
# mount usr file system --------------------
/bin/mount -t squashfs /dev/mtdblock3 /mnt/usr

# APRÈS — même taille exactement, le payload se cache dans le commentaire
# mount usr file system echo hello#---------
/bin/mount -t squashfs /dev/mtdblock3 /mnt/usr
```

Pour valider l'idée, j'ai d'abord remplacé le début d'un commentaire par une simple commande `echo hello#` (le `#` permettant de commenter la suite de la ligne).

![](/images/camera/firmware_patch_hello.png)

Cependant, après réécriture du firmware sur la puce et redémarrage : la caméra refuse de booter.

Il y a donc visiblement une vérification d'intégrité (checksum, signature ou hash) qui empêche toute modification directe du firmware. Retour à la case départ...

#### Tentative 2 : modifier le SquashFS et reconstruire le firmware

Une autre solution consiste à extraire complètement le contenu du firmware, modifier les fichiers qui nous intéressent, puis **repacker** le tout avant de le réinjecter dans la puce.

En effet, sur ce type de caméra, l'intégralité du système Linux embarqué (scripts de démarrage, binaires, configurations, etc.) est stockée dans une partition compressée au format **SquashFS**. Ce SquashFS contient tout le root filesystem : on y retrouve par exemple `/etc`, `/bin`, `/usr`, ainsi que les scripts d'initialisation exécutés au démarrage.

Voici comment ça fonctionne concrètement :

**1️⃣ Extraction du SquashFS :**
On utilise l'outil `binwalk` pour analyser le firmware. Binwalk est capable de repérer les signatures connues (comme celles de SquashFS) et d'extraire automatiquement le contenu du système de fichiers compressé.

![](/images/camera/binwalk_squashfs_offset.png)

On observe que le firmware contient plusieurs partitions, dont un SquashFS à l'offset `2490368` (0x260000). C'est cette valeur qui nous servira pour la réinjection.

**2️⃣ Modification :**
Une fois le système de fichiers extrait, on peut naviguer dedans comme dans n'importe quel Linux : éditer des scripts (`/etc/init.d/` ou `/etc/rcS`, etc.), remplacer des binaires ou ajouter des fichiers (comme un script de reverse shell par exemple).

**3️⃣ Repack / reconstruction :**
Après modification, il faut reconstruire le SquashFS (en gardant le même format et la même taille globale si possible) avec `mksquashfs`. Ensuite, on réinjecte cette nouvelle image SquashFS exactement à l'offset d'origine dans le firmware (grâce à `dd`), pour reconstruire un firmware complet prêt à être flashé de nouveau sur la caméra.

![](/images/camera/firmware_repack.svg)

⚙️ Le script suivant permet d'automatiser toutes ces étapes (merci ChatGPT !) :

- `--extract` : extrait et décompresse le SquashFS
- `--rebuild` : reconstruit le SquashFS modifié et le replace dans le firmware

```bash
#!/bin/bash

FIRMWARE="W25Q64BV_test_reprise.BIN"
OUTPUT="new-firmware.bin"
SQUASHFS_OFFSET=2490368
EXTRACTED_DIR="_${FIRMWARE}.extracted"
SQUASHFS_IMG="new-squashfs.img"

check_deps() {
    for cmd in binwalk mksquashfs truncate stat dd; do
        command -v $cmd >/dev/null 2>&1 || {
            echo "Erreur : $cmd non installé. Installez-le avec 'sudo apt install $cmd'."
            exit 1
        }
    done
}

extract_firmware() {
    [ ! -f "$FIRMWARE" ] && { echo "Erreur : $FIRMWARE introuvable."; exit 1; }

    echo "📤 Extraction de $FIRMWARE..."
    binwalk -e --run-as=$USER "$FIRMWARE"
    [ ! -d "$EXTRACTED_DIR" ] && { echo "Erreur : Dossier $EXTRACTED_DIR non trouvé."; exit 1; }

    echo "✅ Extraction terminée. Modifiez les fichiers dans $EXTRACTED_DIR/squashfs-root"
}

rebuild_firmware() {
    ROOTFS_PATH="$EXTRACTED_DIR/squashfs-root"
    [ ! -d "$ROOTFS_PATH" ] && { echo "Erreur : $ROOTFS_PATH introuvable."; exit 1; }

    echo "🔧 Reconstruction SquashFS..."
    mksquashfs "$ROOTFS_PATH" "$SQUASHFS_IMG" -comp xz -noappend > /dev/null || {
        echo "Erreur lors de la reconstruction de SquashFS."
        exit 1
    }

    echo "📦 Reconstruction du firmware final dans le dossier courant..."
    cp "$FIRMWARE" "$OUTPUT" || {
        echo "Erreur : impossible de copier $FIRMWARE."
        exit 1
    }

    dd if="$SQUASHFS_IMG" of="$OUTPUT" bs=1 seek=$SQUASHFS_OFFSET conv=notrunc status=none || {
        echo "Erreur lors de l'intégration de SquashFS."
        exit 1
    }

    ORIGINAL_SIZE=$(stat -c%s "$FIRMWARE")
    FINAL_SIZE=$(stat -c%s "$OUTPUT")

    if [ "$FINAL_SIZE" -lt "$ORIGINAL_SIZE" ]; then
        echo "📏 Padding : Ajustement à $ORIGINAL_SIZE octets..."
        truncate -s "$ORIGINAL_SIZE" "$OUTPUT"
    elif [ "$FINAL_SIZE" -gt "$ORIGINAL_SIZE" ]; then
        echo "⚠️ Erreur : Le firmware reconstruit dépasse la taille originale."
        rm "$OUTPUT"
        exit 1
    fi

    echo "✅ Nouveau firmware généré : $OUTPUT (taille : $ORIGINAL_SIZE octets)"
}

usage() {
    echo "Usage : $0 [--extract | --rebuild]"
    exit 1
}

check_deps

case "$1" in
    --extract)
        extract_firmware
        ;;
    --rebuild)
        rebuild_firmware
        ;;
    *)
        usage
        ;;
esac
```

L'offset `SQUASHFS_OFFSET` est à récupérer grâce à `binwalk` sur le firmware d'origine.

Une fois tout le firmware décompressé, on a donc bien accès au filesystem de notre caméra.

![](/images/camera/squashfs_extract_rcs.png)

Parmi les fichiers, on remarque que le script `init.d/rcS` comporte une ligne commentée :

```bash
#telnetd &
```

Cette ligne a sûrement été utilisée pour débugger la caméra en usine. On va essayer de décommenter cette ligne afin d'avoir un serveur telnet qui tourne au démarrage de la caméra :

```diff
-#telnetd &
+telnetd &
```

Une fois le firmware reconstruit et reflashé sur la puce, `telnetd` écoutera donc dès le boot : c'est notre **premier moyen** d'obtenir un shell sur la caméra.

***Remarque : la caméra ayant rendu l'âme avant la fin du write-up, il manque quelques photos de cette partie... Heureusement, la suite nous offrira une bien meilleure piste !***

### Moyen n°2 : get a shell from the sdcard

Le reflashage du firmware fonctionne, mais il faut démonter la caméra et la brancher au programmateur à chaque fois. Autant dire que ce n'est pas très pratique... Voyons donc un deuxième moyen, bien plus élégant : obtenir un shell **uniquement en mettant du contenu sur la carte SD**, sans interaction directe avec le système.

Pour ce faire, on regarde quelles applications lisent et utilisent le contenu de la carte SD ("sdcard"). Un simple `grep -r -a 'sdcard'` dans le firmware nous donne déjà de bonnes pistes.

![](/images/camera/grep_sdcard_prerun.png)

Parmi toutes les applications, le binaire `prerun` attire l'attention. Son nom semble indiquer qu'il doit être lancé au démarrage de la caméra.

Ni une, ni deux, on passe le binaire dans **Ghidra**, pour un coup de reverse.

### Analyse du code
Dans la fonction main, une partie du code saute aux yeux :

```c
system("/mnt/sdcard/auth_exshell.sh");
```

![](/images/camera/ghidra_auth_exshell.png)

Cela veut dire qu'en passant certaines conditions, il est possible d'exécuter automatiquement un script stocké sur la carte SD !

Les conditions pour arriver au `system("/mnt/sdcard/auth_exshell.sh");` sont les suivantes :

```c
undefined4 main(void)
{
    ...
	local_14 = access("/mnt/sdcard/local_update.conf",0);
    if (local_14 == 0)
    {
      local_c = was_update_patch_had_already_used(0);
      if (local_c == 0)
      {
        local_14 = local_updater(1);
        if (local_14 != 10)
        {
          ...
          local_c = was_update_patch_had_already_used(1);
          ...
        }
      }
      else
      {
        local_14 = access("/mnt/sdcard/auth_exshell.sh",0);
        if (local_14 == 0)
        {
          system("/mnt/sdcard/auth_exshell.sh");  // <-- objectif final
        }
      }
    }
...
}
```

Donc, pour exécuter notre script `auth_exshell.sh`, il faut que :

- Le fichier `/mnt/sdcard/local_update.conf` existe
- La fonction `was_update_patch_had_already_used(0)` retourne **1**
- Et bien sûr, que le script `auth_exshell.sh` soit présent sur la carte SD

La fonction à regarder en détail est `was_update_patch_had_already_used`. Dans le main, cette fonction est appelée à deux endroits différents avec deux paramètres différents : 0 et 1.

Cette fonction a **deux comportements distincts selon le paramètre `mode`** :

- Si `mode == 0` :

```c
undefined4 was_update_patch_had_already_used(int param_1)
{
	if (param_1 == 0)
	{
		// Lit la valeur de last_id depuis la carte SD
		IFCReadStringOnce("/mnt/sdcard/update_record", "[PATCH]", "last_id", &local_50, "");
		...
		// Lit le DEV_ID interne depuis le système
		IFCReadStringOnce("/mnt/mtd/network_Info.ini", "[NETINFO]", "DEV_ID", &local_28, "");

		...
		 // Compare les deux
		iVar1 = strcmp((char *)&local_28, local_48);

		if (iVar1 == 0)
		{
		    return 1;  // match OK
		}
		return 0;  // mismatch

	}
}
```

On remarque ici que la fonction compare la valeur de `last_id` stockée sur la carte SD avec celle de `DEV_ID` stockée dans `/mnt/mtd/network_Info.ini`. Si les deux valeurs sont identiques, la fonction retourne 1.

- Si `mode == 1` :

```c
undefined4 was_update_patch_had_already_used(int param_1)
{
	if (param_1 != 0)
	{
		....
		// Lit le DEV_ID stocké sur l'appareil
		IFCReadStringOnce("/mnt/mtd/network_Info.ini", "[NETINFO]", "DEV_ID", &local_28, "");
		...
		// Écrit le DEV_ID dans le fichier de la carte SD
		IFCWriteStringOnce("/mnt/sdcard/update_record", "[PATCH]", "last_id", (char *)&local_28);
		....
	}
}
```

Ici la fonction lit la valeur de `DEV_ID` stockée dans `/mnt/mtd/network_Info.ini` puis l'écrit dans le champ "last_id" de `/mnt/sdcard/update_record`.

#### La faille
C'est ici que réside la vulnérabilité : le firmware **fournit lui-même la valeur qu'il exige pour autoriser l'exécution de notre script**.

Il suffit de forcer un appel à `was_update_patch_had_already_used(1)` pour que le bon `DEV_ID` soit écrit sur la carte SD.

### Exploitation
Il n'est donc pas possible d'exécuter directement le contenu du script de `system("/mnt/sdcard/auth_exshell.sh")` puisque l'on ne connaît pas la valeur de `DEV_ID`.

Mais si `was_update_patch_had_already_used(1)` est appelé, `DEV_ID` sera écrit dans le champ "last_id" de `/mnt/sdcard/update_record`.

Reprenons le main en l'annotant :

```c
undefined4 main(void)
{
    ...
    local_14 = access("/mnt/sdcard/local_update.conf", 0);
    // Fichier déclencheur de la mise à jour
    if (local_14 == 0)
    {

        local_c = was_update_patch_had_already_used(0);
		// Vérifie si l'update a déjà été faite, en comparant last_id de "/mnt/sdcard/update_record"
        // avec DEV_ID stocké sur l'appareil
        // --> retourne 1 si [PATCH]::last_id == [NETINFO]::DEV_ID
        // --> retourne 0 sinon

        if (local_c == 0)
        {
            // On entre ici puisque qu'on veut forcer le firmware à écrire DEV_ID dedans.

            local_14 = local_updater(1);
            /*
             * local_updater(1) va :
             *  - lire /mnt/mtd/mvconf/patchmanage.conf
             *  - lancer une tentative de mise à jour depuis la carte SD
             *  - supprimer /mnt/sdcard/local_update.conf si /mnt/sdcard/patch_reuse n'existe pas
             */

            if (local_14 != 10) // != 10 → le code a exécuté une tentative d'update
            {
                ...

                local_c = was_update_patch_had_already_used(1);
                // Force l'écriture du DEV_ID dans le champ "last_id" de "/mnt/sdcard/update_record"
                ...
            }
        }
        else
        {
            local_14 = access("/mnt/sdcard/auth_exshell.sh", 0);
            if (local_14 == 0)
            {

                system("/mnt/sdcard/auth_exshell.sh");// <-- objectif final
            }
        }
    }
    ...
}
```

L'exploitation est donc en deux étapes :

![](/images/camera/exploit_sdcard.svg)

**1) Forcer le firmware à appeler `was_update_patch_had_already_used(1)`**

On crée les fichiers `/mnt/sdcard/update_record`, `/mnt/sdcard/local_update.conf` et `/mnt/sdcard/patch_reuse` vides, ainsi que notre script `/mnt/sdcard/auth_exshell.sh`.

La présence de `local_update.conf` va déclencher le "mode mise à jour".
La présence de `/mnt/sdcard/patch_reuse` va empêcher la suppression de `/mnt/sdcard/local_update.conf`.

Ensuite, `was_update_patch_had_already_used(0)` va retourner 0 puisque l'on n'a écrit aucun champ "last_id" dans `update_record`.

Enfin, `was_update_patch_had_already_used(1)` est appelé. Le firmware écrit le `DEV_ID` dans le champ "last_id" de `update_record` sur la carte SD.

Le fichier `update_record` devient donc :

```
[PATCH]
last_id=ABCDEF123456
```

**2) Exécution du payload**

Après un reboot de la caméra :

- `was_update_patch_had_already_used(0)` lit `last_id=ABCDEF123456` dans le fichier `update_record`
- `DEV_ID` est toujours égal à `ABCDEF123456`
- `strcmp()` retourne 0 → **la fonction retourne 1**

On entre donc dans le else :

```c
...
        local_c = was_update_patch_had_already_used(0);
        if (local_c == 0)

		else
        { // On entre ici
            local_14 = access("/mnt/sdcard/auth_exshell.sh", 0);
            if (local_14 == 0)
            {
                system("/mnt/sdcard/auth_exshell.sh");// <-- objectif final
            }
        }
    }
    ...
}
```

Le script `/mnt/sdcard/auth_exshell.sh` est ensuite exécuté !!!

#### POC

On met les fichiers nécessaires sur la carte SD. Pour le test, notre script se contente d'écrire un fichier témoin : `echo PWNED > /tmp/PWNED`.

![](/images/camera/sdcard_files.png)

On insère la carte SD dans la caméra. On attend qu'elle s'initialise, puis on la reboot.

Au second boot, la magie opère : le fichier `PWNED` est bien présent dans `/tmp`, accessible via telnet !

![](/images/camera/telnet_pwned.png)

On a donc bien un accès shell sur la caméra, obtenu uniquement en insérant une carte SD. 🎉

### La caméra parle derrière notre dos

Maintenant qu'on a un accès complet à la caméra (et au réseau qu'elle utilise), autant en profiter pour observer à qui elle cause.

Un petit passage à la **Wireshark** nous apprend que la caméra résout le domaine `ak802.av380.net`, qui répond l'adresse `47.91.75.122`.

![](/images/camera/wireshark_dns_ak802.png)

On la voit ensuite échanger des paquets UDP avec ce serveur, mais également avec une autre adresse : `143.42.204.40`.

![](/images/camera/wireshark_servers.png)

De quoi s'agit-il ? Télémétrie ? Serveur de mise à jour ? Canal C2 dans le cas où l'appareil serait compromis ? C'est une piste que je garde pour un prochain article...

A suivre...
