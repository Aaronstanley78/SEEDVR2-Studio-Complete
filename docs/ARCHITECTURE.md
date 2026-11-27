# Architecture technique

```
SeedVR2_Studio/
├── INSTALLER.bat · LANCER.bat · METTRE_A_JOUR.bat · DESINSTALLER.bat   (Windows)
├── install.sh · launch.sh · update.sh · uninstall.sh                   (Linux / macOS)
├── installer/        install.py (installation, mise à jour) · uninstall.py · wangp_setup.py (WanGP)
├── INSTALLER_DLSS5.bat   composants DLSS 5 optionnels (script officiel de WanGP)
├── config/           default_config.json
├── engine/           seedvr2_upstream/ : moteur SeedVR2 de référence (Apache-2.0, non modifié)
├── backend/          serveur Python (FastAPI)
│   ├── requirements.txt
│   ├── studio/
│   │   ├── server.py         API REST + WebSocket, sert l'interface compilée
│   │   ├── engine/seedvr2.py intégration réelle du pipeline SeedVR2 (4 phases)
│   │   ├── engine/base.py    interface moteur, pause/annulation, erreurs
│   │   ├── engine/wangp.py   pont vers WanGP (processus relais, échanges sans perte)
│   │   ├── engine/wangp_worker.py  relais exécuté dans l'environnement de WanGP (API officielle)
│   │   ├── engine/lanczos.py Lanczos natif (identique à WanGP)
│   │   ├── upscalers.py      catalogue des upscalers WanGP (commit figé)
│   │   ├── pipelines.py      aperçus et exports images/vidéos, segments, récupération mémoire
│   │   ├── live_preview.py   diffusion des images calculées pendant l'export
│   │   ├── video_io.py       lecture/écriture vidéo par tubes FFmpeg, codecs, audio
│   │   ├── image_io.py       lecture/écriture d'images, master 16 bits, ICC, formats
│   │   ├── cache.py          cache des résultats réellement calculés
│   │   ├── jobs.py           file d'attente (un fil GPU), progression
│   │   ├── model_manager.py  modèles : état, téléchargement, SHA-256, suppression, dossier
│   │   ├── downloader.py     téléchargement avec reprise, vitesse, temps restant
│   │   ├── catalog.py        catalogue des modèles (issu du registre du moteur)
│   │   ├── hardware.py       GPU/VRAM/CPU/RAM/températures, recommandations
│   │   ├── ffmpeg_tools.py   localisation de FFmpeg, encodeurs réellement utilisables, ffprobe
│   │   ├── library.py        médias importés, miniatures, copies lisibles par le navigateur
│   │   ├── params.py         paramètres validés (Pydantic) et clé de cache
│   │   ├── resolution.py     calcul exact de la résolution de sortie
│   │   ├── settings.py · logs.py · events.py · perf.py · paths.py
│   └── tests/                tests automatisés (pytest)
├── frontend/         interface React + TypeScript + Tailwind (src/) et version compilée (dist/)
└── docs/             guides
```

Données créées à l'exécution : `data/` (réglages, médias importés, cache, journaux), `models/SEEDVR2/`
(modèles, dossier modifiable), `outputs/` (exports par défaut), `tools/ffmpeg/`, `.venv/`.

## Intégration de SeedVR2
`studio/engine/seedvr2.py` importe directement le pipeline du moteur de référence et enchaîne
`encode_all_batches → upscale_all_batches → decode_all_batches → postprocess_all_batches`, comme son CLI
(`_process_frames_core`) et son nœud (`video_upscaler.py`). Ajouts : pause/annulation entre deux lots
(crochet `interrupt_fn` appelé par le moteur), progression par phase, modèle conservé en mémoire entre deux
traitements (cache global du moteur, déchargé vers la RAM) et changement de modèle sans redémarrage,
déchargement explicite, détection des erreurs « out of memory », mesure du pic de VRAM et de la vitesse.
PyTorch n'est importé qu'au premier traitement : l'interface démarre instantanément.

## Intégration de WanGP
- WanGP est installé par son installateur officiel dans `engines/wangp` (son propre environnement Python 3.11,
  PyTorch, Triton). Studio ne l'importe jamais dans son propre processus : il lance
  `engine/wangp_worker.py` avec le Python de WanGP. Ce relais utilise uniquement l'API officielle
  (`shared.api.init`, `WanGPSession.submit_media_postprocessing`, `job.cancel()`).
- Protocole : commandes JSON sur l'entrée standard, événements JSON sur une copie du descripteur de sortie
  (WanGP redirige `sys.stdout` pendant ses traitements). Les rappels sont protégés : une exception dans un
  rappel ferait échouer le traitement WanGP (constaté pendant le développement).
- Échanges **sans perte** : Studio envoie chaque segment en FFV1 RVB (ou une image en PNG) et configure WanGP
  pour répondre en FFV1 RVB (`video_output_codec = libx264_lossless` + `mkv`) et en PNG (WanGP utilise du JPEG
  95 % par défaut). La configuration de WanGP (installée ou par défaut) est conservée ; seuls ces réglages et
  l'activation de FlashVSR (désactivé par défaut) sont appliqués en mémoire au démarrage du relais.
- Segments vidéo : 97 images + contexte (retiré de la sortie), complétés à 8k+1 images pour les fenêtres
  temporelles de WanGP (images de bourrage retirées). Même cache et même vérification d'empreinte que SeedVR2.
- **DLSS 5 Temporal Upsampling** : étape séparée appliquée au résultat spatial de chaque segment, précédée de
  la dernière image (agrandie) du segment précédent. n images → (n−1)×f+1 sans contexte, n×f avec contexte :
  l'enchaînement des segments donne exactement le même résultat que la vidéo d'un bloc (testé). La cadence de
  sortie est multipliée ; l'audio est conservé.
- Un seul moteur GPU actif : passer de SeedVR2 à WanGP décharge SeedVR2, et inversement (arrêt du relais).

## Fidélité aperçu ⇄ export
1. **Clé de cache** = empreinte des seuls paramètres qui changent les pixels (modèle, résolution, graine,
   correction colorimétrique, bruit, lots, recouvrement, tuilage…). Les réglages purement mémoire (BlockSwap,
   déchargement) n'en font pas partie : on peut les modifier sans invalider un aperçu.
2. **Images** : l'aperçu calcule l'image entière ; le résultat est conservé en PNG 16 bits sans perte (master).
   L'affichage et les exports 8 bits passent par la même fonction d'arrondi : identiques au pixel près (testé).
3. **Vidéos** : grille de segments fixe ancrée sur l'image 0, contexte de recouvrement lu dans les images
   sources précédentes, images préliminaires uniquement sur le premier segment, même graine. Un aperçu calcule
   des segments complets et les conserve avec l'empreinte des images d'entrée. L'export relit la vidéo de façon
   continue, recalcule l'empreinte de chaque segment et réutilise le segment en cache s'il correspond (sinon il
   le calcule). Test automatisé : image n° 23 de l'export FFV1 = image n° 23 de l'aperçu, au pixel près, et
   seuls les segments jamais prévisualisés sont calculés.
4. **Couleurs** : décodage avec la matrice YUV→RVB correcte de la source (BT.709/601/2020, plage
   automatique), encodage en BT.709 explicite avec étiquetage du fichier. Les images restent en RVB pleine
   plage entre les deux. Profils ICC des images conservés.
5. **Codecs avec perte** : l'interface le signale et propose FFV1 (RVB sans perte) pour une correspondance
   exacte ; l'export 10 bits active une chaîne 16 bits pour l'aperçu comme pour l'export.

## Aperçu en direct pendant l'encodage
Dans `handle_video_export`, chaque segment calculé est écrit dans le tube de l'encodeur FFmpeg **puis**
transmis à `LivePreview`, qui tourne dans un fil séparé (redimensionnement + JPEG hors du fil de calcul) et
publie les images sur le WebSocket dans l'ordre chronologique, réparties sur la durée réelle de calcul du
segment (défilement continu au rythme réel, sans prétendre au temps réel ; images sautées si l'affichage prend
du retard, jamais dans le désordre). La dernière image est disponible en PNG pleine résolution
(`/api/jobs/{id}/live-frame`).

## Anti-scintillement (`backend/studio/deflicker.py`, 2.0)
Appliqué dans `ChunkProcessor.process` sur la sortie du modèle, contexte compris, avant la conversion en
8/16 bits ; commun à tous les moteurs. Basses fréquences : écart sortie − source sur une grille de 32 px de petit
côté, moyenne glissante (Faible α = 0,5, Moyen α = 0,2) ou suppression (Fort). Hautes fréquences : filtre
récursif pondéré par l'immobilité de la source (écart moyen entre images sources, seuil 0,035), poids maximal
0,35 / 0,6 / 0,8. Calcul image par image avec PyTorch (GPU si disponible). `effective_context()` garantit au
moins 4 images de contexte ; le niveau et le contexte font partie de la clé de cache vidéo.

## Téléchargement parallèle (`downloader.py`, 2.0)
Sondage `Range: bytes=0-0` (URL finale du CDN, taille, prise en charge des plages), puis blocs de 8 Mio
récupérés par un pool de connexions (sessions `requests` par fil) avec une fenêtre de 2 × connexions ; un seul
écrivain ajoute les blocs dans l'ordre et met à jour le SHA-256. Liens 401/403/410 renouvelés, nouvelles
tentatives par bloc, arrêt propre des autres connexions en cas d'erreur ou d'annulation. Les poids WanGP :
l'inventaire du relais (`wangp_worker.missing_files`) fournit le chemin final (calculé avec
`files_locator.get_smart_download_root`, comme `process_files_def`) et l'URL ; `_wangp_fast_download`
les télécharge avant d'appeler le téléchargeur de WanGP, qui ne fait plus que constater leur présence.

## Encodage en tâche de fond (`video_io.py`, 2.0)
`VideoWriter.write()` dépose les images dans une file bornée (1,5 Gio) ; un fil les envoie à FFmpeg. Une
erreur de FFmpeg est signalée au `write()`/`close()` suivant. `PrefetchReader` décode le bloc source suivant
pendant le calcul. `choose_encoder(prefer_hardware=True)` place NVENC en premier.

## Gestion de la mémoire et des erreurs
- Paliers automatiques en cas d'OOM : déchargement → BlockSwap ½ → BlockSwap max + E/S → (mode « auto »)
  tuilage VAE 1024 → 512 → lots plus petits. Les réglages finalement utilisés sont renvoyés à l'interface.
- Segments vidéo bornés en mémoire (longues vidéos), fichiers temporaires supprimés en fin de tâche ou
  à l'annulation, cache des aperçus plafonné (suppression des plus anciens).
- Erreurs utilisateur explicites (modèle manquant, combinaison codec/conteneur, FFmpeg absent…).

## Tests
`cd backend && python -m pytest -q` — 35 tests (+ 1 test contre une vraie installation de WanGP, lancé si
`SEEDVR2_TEST_WANGP_ROOT` et `SEEDVR2_TEST_WANGP_PYTHON` sont fournis).

### Exécutés et réussis dans l'environnement de développement (Linux, sans GPU)
- Catalogue identique au registre du moteur (noms, dépôts, SHA-256) ; ensembles de modèles.
- Calcul de la résolution identique à celui du moteur (68 cas comparés au code du moteur).
- **Contrat d'API du moteur** : chaque fonction du pipeline SeedVR2 appelée existe avec les paramètres utilisés.
- Import réel du moteur SeedVR2 avec PyTorch.
- Images : master 16 bits sans perte, aperçu = export au pixel près, tous les formats, alpha, ICC.
- Téléchargeur : téléchargement, **reprise** d'un fichier partiel, annulation, rejet d'une empreinte fausse.
- Vidéo : lecture avec recherche = lecture continue (image par image), aller-retour FFV1 sans perte,
  détection des encodeurs réellement fonctionnels (NVENC écarté sans carte NVIDIA).
- API de bout en bout : import, aperçu image puis export depuis le cache sans relancer le moteur,
  aperçu d'image vidéo, aperçu de séquence avec réutilisation des segments, export FFV1 identique à l'aperçu,
  export H.264 d'une plage avec audio, images en direct reçues dans l'ordre, pause / reprise / annulation sans
  fichier incomplet, récupération automatique après manque de mémoire simulé, rejet des combinaisons
  incompatibles.
- Installateur exécuté de bout en bout (Linux) : dépendances, chargement du moteur, téléchargement réel de
  FFmpeg avec SHA-256, gestion propre d'un échec de téléchargement des modèles ; lancement réel du serveur ;
  désinstallation.
- Interface : compilation TypeScript stricte, vérification visuelle de toutes les pages dans un navigateur.

Ces tests de bout en bout utilisent `backend/tests/fake_engine.py`, un **moteur factice sans IA** activé
uniquement par `SEEDVR2_STUDIO_TEST_ENGINE=1` (bandeau rouge permanent dans l'interface) pour valider tout le
reste du logiciel sans GPU. Il n'est jamais utilisé par les scripts d'installation ou de lancement.

### Upscalers WanGP — exécuté et réussi dans l'environnement de développement (sans GPU)
- Catalogue comparé aux sources de WanGP (méthodes, facteurs) ; libellés des neuf upscalers demandés.
- **Vraie API WanGP** (WanGP v19.292, commit figé, dépendances réelles installées) : le relais démarre WanGP,
  traite une vidéo et une image sans perte, remonte la détection DLSS 5 réelle (« missing dxgi.dll » ici), et
  signale proprement l'échec de téléchargement des poids FlashVSR (Hugging Face inaccessible ici). Le Lanczos
  de WanGP via le pont est **identique au pixel près** au Lanczos natif de Studio. Réserve : WanGP refusant de
  démarrer sans carte NVIDIA, ces essais ont utilisé une simulation des fonctions CUDA **limitée aux tests**
  (aucun calcul GPU n'a eu lieu).
- Circuit complet avec un faux WanGP au même contrat (`tests/fake_wangp`) : aperçus et exports identiques au
  pixel près, segments + contexte + bourrage sans perte d'image, DLSS Temporal enchaîné sans couture (y compris
  sur une plage), aperçu de séquence temporel, annulation, erreur mémoire, préchargement, refus des
  combinaisons impossibles (LTX sur image, temporel sur image).
- Récupération du code WanGP au commit figé avec git (idempotente) ; l'installateur officiel de WanGP démarre
  (il refuse ici faute de carte NVIDIA, ce que Studio signale sans bloquer SeedVR2).
- Interface : nouveaux écrans vérifiés dans un navigateur (sélection des upscalers, DLSS Temporal, rendu en
  direct des images interpolées, section WanGP de l'onglet Modèles).

### À valider sur un ordinateur équipé d'un GPU compatible
- Upscalers WanGP réels (LTX, FlashVSR, RelayVSR, DLSS 5) : qualité, VRAM, vitesse, téléchargement des poids ;
  installation automatique de WanGP sous Windows ; `INSTALLER_DLSS5.bat`.
- Inférence SeedVR2 réelle (qualité, VRAM, vitesse) : les poids ne pouvaient pas être téléchargés depuis
  l'environnement de développement (Hugging Face inaccessible) et il ne disposait pas de GPU.
- Téléchargement réel des modèles depuis Hugging Face (le mécanisme est testé avec un serveur local).
- Installation de PyTorch CUDA sous Windows et scripts `.bat` (écrits pour Windows 10/11, non exécutés ici).
- Encodeurs NVENC, et le gain de l'encodage en tâche de fond sur une vraie carte.
- Anti-scintillement sur de vraies sorties SeedVR2 / WanGP (testé sur des vidéos synthétiques qui scintillent).
- Téléchargement parallèle depuis le vrai CDN de Hugging Face (testé avec un serveur local qui bride chaque
  connexion).
