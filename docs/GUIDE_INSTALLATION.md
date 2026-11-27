# Guide d'installation — SeedVR2 Studio

## 1. Avant de commencer

- **Carte graphique NVIDIA** recommandée. Mettez le pilote à jour : https://www.nvidia.com/drivers
  (version 570 ou plus conseillée ; obligatoire pour les RTX série 50).
- **Python 3.10 à 3.13** (3.12 conseillé). S'il est absent, `INSTALLER.bat` propose de l'installer
  automatiquement avec *winget* ; sinon téléchargez-le sur https://www.python.org/downloads/ en cochant
  **« Add python.exe to PATH »**.
- **Connexion internet** : PyTorch (≈ 2,5 Go), FFmpeg (≈ 150 Mo) et les modèles (2,4 à ≈ 76 Go).
- **Chemin court** : extrayez l'archive par exemple dans `D:\SeedVR2_Studio` (évitez les chemins très longs).

## 2. Installation (Windows)

Double-cliquez sur **`INSTALLER.bat`**. L'installateur affiche chaque étape :

| Étape | Ce qui est fait |
|---|---|
| 1. Matériel | Détection de la carte, de la VRAM, du pilote, de la RAM et de l'espace disque |
| 2. PyTorch | Installation de la version CUDA adaptée au pilote (CUDA 12.8 / 12.6 / 11.8, ou processeur) |
| 3. Dépendances | Bibliothèques du moteur SeedVR2 et de l'application (la version CUDA de PyTorch est verrouillée) |
| 4. Vérification | Test de CUDA et chargement effectif du moteur SeedVR2 |
| 5. FFmpeg | Téléchargement du build officiel (BtbN), empreinte SHA-256 vérifiée, installé dans `tools\ffmpeg` |
| 6. Modèles | Choix de l'ensemble, téléchargement avec barre de progression, taille, vitesse, temps restant |
| 7. WanGP (optionnel) | Code WanGP au commit figé, installateur officiel de WanGP en mode automatique (Python 3.11, PyTorch, Triton, noyaux), préchargement facultatif des poids des upscalers |
| 8. Contrôle final | Encodeurs vidéo réellement utilisables, modèles prêts, interface présente |

### Choix des modèles

| Ensemble | Contenu | Taille |
|---|---|---|
| **1 · Recommandé** (défaut) | VAE + SeedVR2 3B FP16 + 3B FP8 + 7B FP8 + **Lite** 3B Q4 + 3B Q8 + 7B Q4 | ≈ 30 Go |
| 2 · Complet | les 12 fichiers (dont 7B FP16, les variantes 7B Sharp et SeedVR2-1.4B communautaire) | ≈ 79 Go |
| 3 · Minimal | VAE + Lite 3B Q4 | ≈ 2,4 Go |
| 4 · Aucun | à télécharger plus tard dans l'onglet *Modèles* | — |

Les téléchargements **reprennent automatiquement** là où ils s'étaient arrêtés (coupure, fermeture de la
fenêtre…) : il suffit de relancer `INSTALLER.bat`. Chaque fichier est contrôlé par son empreinte SHA-256
officielle ; un fichier corrompu est rejeté.

Options avancées (en ligne de commande) : `INSTALLER.bat --models complet --yes`, `--cpu` (sans CUDA),
variable `SEEDVR2_TORCH_INDEX=cu126` pour forcer une version CUDA.

### Upscalers WanGP (LTX, FlashVSR, RelayVSR, DLSS 5)

L'installateur propose ensuite d'installer **WanGP** (≈ 10 Go hors poids). Il :
1. récupère le code de WanGP au commit testé avec cette version de Studio (avec `git` s'il est présent,
   sinon par l'archive GitHub) dans `engines\wangp` ;
2. lance **l'installateur officiel de WanGP** en mode automatique (`setup.py install --auto --env uv`), qui
   choisit Python, PyTorch, Triton et les noyaux selon votre carte et crée `engines\wangp\env_uv` ;
3. propose de **précharger les poids** : *Léger* (FlashVSR One/Two Pass + RelayVSR, quelques Go),
   *Complet* (+ LTX 2.3 / 2.5 Upscale et Detail Refiner, plusieurs dizaines de Go) ou *Plus tard* (au premier
   usage). Le préchargement lance un court traitement réel par upscaler : il vérifie donc aussi qu'il fonctionne.

WanGP est indépendant de SeedVR2 : si son installation échoue, SeedVR2 et Lanczos restent utilisables.
Options : `INSTALLER.bat --wangp oui|non --wangp-prefetch leger|complet|aucun`.

### DLSS 5 (facultatif)

**DLSS 5 Spatial Upsampling** et **DLSS 5 Temporal Upsampling** utilisent des bibliothèques NVIDIA DLSS,
ReShade et le module RenoDX, qui ne sont fournis ni par Studio ni par WanGP. Lancez **`INSTALLER_DLSS5.bat`** :
il exécute le script officiel de WanGP, qui télécharge ces composants depuis leurs sources, vérifie leurs
empreintes SHA-256 et les signatures NVIDIA, et **vous demande de taper « I ACCEPT »** après lecture de leurs
conditions. Prérequis : Windows 11, carte RTX 30+ (RTX 40+ et planification GPU matérielle « HAGS » pour le
Temporal Upsampling). Ensuite : onglet *Modèles* → **Vérifier WanGP** pour voir l'état réel de DLSS 5.

## 3. Lancement

Double-cliquez sur **`LANCER.bat`**. Le navigateur s'ouvre sur `http://127.0.0.1:7860`
(un autre port est choisi automatiquement s'il est occupé). Gardez la fenêtre noire ouverte :
la fermer arrête le logiciel. L'interface n'est accessible que depuis votre ordinateur.

## 4. Mise à jour

**`METTRE_A_JOUR.bat`** : met à jour PyTorch et les dépendances compatibles, réinstalle le dernier FFmpeg,
vérifie l'intégrité des modèles et signale ceux dont une nouvelle version est publiée. WanGP est ramené au
commit validé pour cette version de Studio (et ses dépendances mises à jour si ce commit a changé). Pour mettre à jour
le code de l'application, extrayez la nouvelle archive par-dessus le dossier (modèles, réglages et médias
sont conservés), puis lancez `METTRE_A_JOUR.bat`.

## 5. Désinstallation

**`DESINSTALLER.bat`** supprime l'environnement Python, FFmpeg, le cache et les journaux. Il demande une
confirmation **séparée** avant de supprimer : les modèles, WanGP (avec son environnement et ses poids), vos
données (médias importés, réglages) et vos fichiers exportés.

## 6. Linux / macOS

```bash
./install.sh        # mêmes étapes que INSTALLER.bat
./launch.sh         # lancement
./update.sh         # mise à jour
./uninstall.sh      # désinstallation
```
Sous macOS, installez FFmpeg avec `brew install ffmpeg`. Apple Silicon fonctionne (MPS), plus lentement.

## 7. Problèmes fréquents

| Message | Solution |
|---|---|
| « Python 3.10 à 3.13 introuvable » | Installez Python 3.12 en cochant « Add to PATH », relancez. |
| « fonctionne SANS accélération GPU » | Mettez à jour le pilote NVIDIA, puis `METTRE_A_JOUR.bat`. |
| Téléchargement interrompu | Relancez `INSTALLER.bat` : la reprise est automatique. |
| « Espace disque insuffisant » | Libérez de l'espace ou choisissez un autre dossier de modèles (onglet Modèles). |
| Hugging Face inaccessible | Paramètres → « Serveur des modèles » : indiquez un miroir (ex. `https://hf-mirror.com`). |
| Mémoire GPU insuffisante | Voir le guide d'utilisation, section « Mémoire ». |
| « WanGP n'est pas installé » | Relancez `INSTALLER.bat` et répondez Oui à WanGP. |
| « No automatic profile » (WanGP) | Carte non reconnue par l'installateur de WanGP : voir la doc de WanGP (`docs/INSTALLATION.md`). |
| DLSS 5 « missing dxgi.dll / nvngx… » | Lancez `INSTALLER_DLSS5.bat`, puis *Vérifier WanGP*. |

Journal détaillé : `data\logs\seedvr2_studio.log` (et onglet *Journal* de l'interface).
