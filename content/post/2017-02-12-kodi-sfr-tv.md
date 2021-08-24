---
categories:
- Software
date: "2017-02-12"
title: "Regarder SFR TV dans Kodi"
description: "Un guide pour ajouter les flux TV de SFR dans Kodi"
keywords: ["kodi", "sfr", "tv"]
aliases: ["/kodi-sfr-tv/"]
---

À partir du travail d'investigation de [Mohamed El Morabity](https://github.com/melmorabity) et de [Hugo (HugoPoi)](https://github.com/HugoPoi), il est maintenant possible de regarder les flux TV de SFR dans Kodi. Cela ne marche que pour le décodeur SFR évolution.

> **WIP:**
> Ce guide n'est fonctionnel que sous Linux pour le moment. J'ai prévu d'adapter la méthode à TvHeadend pour ajouter le timeshifting et le support de Windows.

## Prérequis
Pour suivre ce guide vous avez besoin :
* d'un abonnement TV chez SFR (logique),
* d'une box SFR Évolution : **notez le STBID et le SN écrit sur l'étiquette arrière de votre box**,
* d'un décodeur SFR Évolution,
* d'une installation de Kodi qui supporte les flux H264,
* d'un PC ou une machine virtuelle sous Linux (Ubuntu pris en exemple ici) avec les paquets **python3**, **libxslt** et **curl**.

Sous Ubuntu ces pré-requis peuvent être installés avec la commande

```bash
sudo apt install kodi python3 libxslt1.1 curl
```

## 1) Obtention de la liste des chaînes
Cette partie est basée sur [ce reverse](https://github.com/HugoPoi/9boxtv) d'HugoPoi.

D'abord il est question de récupérer la liste des chaînes du serveur de SFR :

* Placez vous avec un terminal dans un dossier de travail,
* Téléchargez **get_config.curl** et **request_config.xml** du [dépôt d'HugoPoi](https://github.com/HugoPoi/9boxtv),
  ```bash
  curl -O https://raw.githubusercontent.com/HugoPoi/9boxtv/master/request_config.xml
  curl -O https://raw.githubusercontent.com/HugoPoi/9boxtv/master/get_config.curl
  ```
* Il faut éditer l'entête de la requête pour que le serveur SFR réponde :
    * Editez **get_config.curl** et complétez la ligne 6 (cf prérequis),
    * Il n'est pas nécessaire d'éditer **request_config.xml** car le serveur l'ignore mais je conseille de mettre vos informations dedans.
* Envoyez la requête
  ```bash
  curl --config get_config.curl -o channels_list.xml
  ```

Maintenant il faut convertir ce fichier XML en une liste de lecture pour l'utiliser. Pour cela j'ai adapté la méthode d'HugoPoi :

* Téléchargez [generate_playlist.xslt](https://gist.github.com/erdnaxe/2ba72bb122bb19e056bc6aacf05fa0d0),
  ```bash
  curl -O https://gist.githubusercontent.com/erdnaxe/2ba72bb122bb19e056bc6aacf05fa0d0/raw/671d0bff5748c80da661fcba0af4fb9bed741ca2/generate_playlist.xslt
  ```
* Convertisez le XML en M3U avec *xsltproc*.
  ```bash
  xsltproc generate_playlist.xslt channels_list.xml > playlist_tv_sfr.m3u
  ```

Voilà ! **playlist_tv_sfr.m3u** contient la liste de toutes les chaînes de SFR. Pour tester vous pouvez l'ouvrir avec VLC.
Certaines chaînes sont chiffrées ce qui rendra la lecture impossible.

## 2) Obtention du programme TV
Cette partie est basée sur [ce programme](https://github.com/melmorabity/tv_grab_fr_sfr) de Mohamed El Morabity.

* Téléchargez [le script tv_grab_fr_sfr.py de Mohamed El Morabity](https://github.com/melmorabity/tv_grab_fr_sfr/blob/master/tv_grab_fr_sfr.py),
  ```bash
  curl -O https://raw.githubusercontent.com/melmorabity/tv_grab_fr_sfr/master/tv_grab_fr_sfr.py
  ```
* Lancez le script et redirigez la sortie dans un fichier XMLTV.
  ```bash
  python tv_grab_fr_sfr.py > epg_sfr.xmltv
  ```

Ainsi vous avez un fichier **epg_sfr.xmltv** contenant le programme TV de SFR.
Il faudra refaire la dernière manipulation quotidiennement.

## 3) Configuration de Kodi

* Vérifiez que Kodi st bien réglé sur le bon fuseau horaire.
* Activez l'extension **PVR IPTV Simple Client**. Elle est officielle donc souvent pré-installée avec Kodi.
* Configurez l'extension comme ci-dessous :

![Kodi PVR General Configuration](/assets/images/kodi-sfr-tv/kodi-pvr-gen.jpg)

![Kodi PVR EPG Configuration](/assets/images/kodi-sfr-tv/kodi-pvr-epg.jpg)

* Redémarrez Kodi et voilà ! Vous avez accès aux châines SFR dans Kodi.
