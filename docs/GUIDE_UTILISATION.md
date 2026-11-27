# Guide d'utilisation — SeedVR2 Studio

Version 2.0. L'interface est en français ou en anglais (sélecteur **FR / EN** en haut à droite). Les onglets
sont dans la barre du haut ; le numéro de version est affiché à côté du logo et dans **Paramètres**.

## Accueil
Accès direct aux pages Images, Vidéos et Modèles. Utilisation du GPU, VRAM (et pic mesuré par PyTorch), température, consommation, processeur, mémoire vive,
modèle chargé (bouton **Décharger le modèle** pour libérer la VRAM), file d'attente, traitement en cours avec
son rendu en direct, et **réglages recommandés pour votre GPU**. Tout est actualisé chaque seconde.

## Images
1. **Importez** par glisser-déposer, avec *Parcourir…*, ou en collant un chemin local.
2. **Choisissez le modèle** (groupes *SeedVR2* et *SeedVR2 Lite (GGUF)* ; la VRAM conseillée est indiquée).
3. **Taille de sortie** : facteur (×1,5 ×2 ×3 ×4) ou petit côté en pixels (720p … 2160p, jusqu'à 8192).
   Le ratio est toujours conservé. L'encadré affiche les résolutions d'origine et finale **exactes**,
   la durée estimée (mesurée après le premier traitement) et la VRAM estimée.
4. **Générer l'aperçu** : SeedVR2 calcule l'image entière avec exactement les réglages d'export.
   Comparez avec le **curseur**, *Côte à côte*, *Résultat seul* ou *Original seul* ; **molette** = zoom,
   **glisser** = déplacer, **double-clic** = 100 % (un pixel de l'écran = un pixel réel).
5. **Exporter cet aperçu** : écrit immédiatement le résultat déjà calculé, sans nouvelle génération.
   *Exporter la sélection / tout* place les images dans la file d'attente (traitement par lots) ; les images
   dont l'aperçu existe déjà ne sont pas recalculées.

Formats : PNG (8/16 bits), JPEG, WebP (avec ou sans perte), TIFF (8/16 bits), BMP, AVIF, OpenEXR
(linéaire, demi-flottant). Seuls les formats réellement disponibles sur votre machine sont proposés.
La transparence (canal alpha) et le profil couleur ICC d'origine sont conservés quand le format le permet.

## Vidéos
- Le **lecteur** permet de lire la vidéo, d'avancer image par image (◀ ▶) et de placer **Début ici** /
  **Fin ici** pour définir une séquence. Les zones violettes de la timeline sont **déjà calculées**.
- **Aperçu de cette image** : calcule le *segment* qui contient l'image (voir ci-dessous) et affiche le
  résultat dans le comparateur (onglet *Résultat de l'image*).
- **Aperçu de la séquence** (jusqu'à 600 images) : calcule la séquence, l'affiche en direct pendant le calcul,
  puis la propose dans le lecteur **avant/après** synchronisé (onglet *Comparaison avant/après*).
- **Exporter la vidéo** : conteneur, codec, préréglage (*Qualité maximale, Équilibré, Export rapide,
  Compression élevée, Sans perte, Personnalisé*), encodeur (logiciel ou NVENC si disponible), qualité CRF/CQ
  ou débit, 10 bits, profil ProRes, images/s, audio d'origine, dossier de destination. Les combinaisons
  conteneur/codec incompatibles ne sont jamais proposées. Option : exporter uniquement la séquence choisie.

### Rendu en direct pendant l'export
Dès le lancement, l'onglet **Rendu en direct** (aussi visible dans le tableau de bord et la file d'attente)
montre les images **réellement produites par SeedVR2**, dans l'ordre, au rythme du traitement : numéro
d'image, timecode, vitesse (images/s), temps écoulé et restant, double barre (affichage / calcul).
Chaque image calculée est envoyée en même temps à l'encodeur et à l'aperçu. **Pleine résolution** ouvre la
dernière image calculée en PNG sans perte. La taille et la qualité de l'aperçu en direct se règlent dans
*Paramètres* (« 0 = pleine résolution »).

### Pourquoi des « segments » ?
SeedVR2 traite les images par **lots** pour la cohérence temporelle : le rendu d'une image dépend de ses
voisines. La vidéo est donc découpée en segments fixes (par défaut ≈ 20 images, réglable). Un aperçu calcule
les segments entiers, avec les mêmes voisines, le même recouvrement et la même graine que l'export, puis les
conserve. À l'export, ces segments sont **réutilisés** (après vérification que les images sources sont
identiques) : ce que vous avez vu est exactement ce qui est encodé.

### Anti-scintillement
Le réglage **Anti-scintillement** (carte dans la colonne de droite de la page Vidéos) fonctionne avec tous les
modèles : SeedVR2, SeedVR2 Lite, SeedVR2 1.4B, upscalers WanGP et Lanczos.

| Niveau | Effet |
|---|---|
| Non | Chaque image est livrée telle que le modèle l'a calculée. |
| Faible | Teinte et luminosité légèrement lissées dans le temps ; un peu de lissage des détails dans les zones immobiles. |
| **Moyen** (par défaut) | Teinte, luminosité et détails des zones immobiles stabilisés ; raccords entre segments adoucis. |
| Fort | Teinte et luminosité calées sur la vidéo source ; détails des zones immobiles fortement lissés. Peut atténuer un modèle qui change volontairement les couleurs. |

Comment ça marche : la vidéo source ne scintille pas. Studio compare, à basse résolution, chaque image
calculée à l'image source correspondante, et lisse cet écart dans le temps. Dans les zones où la source ne
bouge pas, chaque image est mélangée avec la précédente ; là où la source bouge, rien n'est mélangé (pas de
traînées). Chaque segment démarre sur ses images de contexte (au moins 4), d'où des raccords plus doux.
L'aperçu reste identique à l'export. Limite : une déformation de forme (objet qui « ondule ») n'est pas
corrigée, et un détail qui se déplace lentement peut garder un léger fourmillement.

### Fidélité de l'export
L'image calculée est identique entre aperçu et export. Un codec **avec perte** (H.264, H.265, AV1, VP9)
modifie toutefois légèrement les pixels à la compression. Pour une correspondance **exacte** au pixel près,
choisissez **FFV1 dans MKV** (sans perte, RVB) ou le préréglage *Sans perte*. Le 10 bits active
automatiquement une chaîne 16 bits pour l'aperçu, afin qu'il reste identique à l'export.

## Upscalers WanGP et Lanczos
La liste *Modèle d'IA* contient, en plus de SeedVR2 et SeedVR2 Lite, les upscalers de WanGP
« System Video Postprocessing » et Lanczos. Seuls ceux qui conviennent au média sont actifs.

| Upscaler | Facteurs | Médias | Remarque |
|---|---|---|---|
| LTX 2.5 Upscale | ×2 | vidéo | modèle LTX 2.5 22B + IC-LoRA, préserve le mouvement |
| LTX 2.5 Detail Refiner | ×1, ×1,5, ×2 | vidéo | reconstruit textures et grain ; champ « style » facultatif |
| LTX 2.3 Upscale | ×2 | vidéo | modèle LTX 2.3 22B |
| RelayVSR Upscale | ×1 à ×4 (pas de 0,5) | vidéo, image | poids INT8 |
| Lanczos Upscale | ×1 à ×4 | vidéo, image | sans IA, sans GPU, sans WanGP ; identique au Lanczos de WanGP |
| FlashVSR One Pass Upscale | ×1 à ×4 | vidéo, image | rapide, en flux continu |
| FlashVSR Two Pass Upscale | ×1 à ×4 | vidéo, image | deux passes : plus de détails, ≈ 2× plus lent |
| DLSS 5 Spatial Upsampling | ×1 (DLAA), ×1,5, ×1,724, ×2, ×3 | vidéo, image | intensité réglable ; composants DLSS 5 requis |
| DLSS 5 Temporal Upsampling | ×2 à ×6 | vidéo | multiplie la **cadence**, pas la résolution |

- La taille de sortie des upscalers WanGP est arrondie par WanGP au multiple de 16 : l'estimation est
  marquée « ≈ » et la taille réelle est utilisée à l'export.
- **DLSS 5 Temporal Upsampling** se choisit à part (*Cadence*), en plus de n'importe quel upscaler spatial
  (SeedVR2 compris). Une vidéo de n images devient (n−1)×facteur+1 images, à une cadence multipliée ; l'audio
  est conservé. Il s'applique à l'aperçu de séquence et à l'export (l'aperçu d'une image montre le résultat
  spatial).
- Les vidéos sont traitées par segments (97 images par défaut) précédés d'images de contexte (8 par défaut,
  16 pour DLSS 5, 1 pour DLSS Temporal) qui assurent la continuité ; comme pour SeedVR2, les segments calculés
  en aperçu sont réutilisés tels quels à l'export.
- Onglet **Modèles → Upscalers complémentaires** :
  - **Vérifier les fichiers** : WanGP établit la liste exacte des fichiers manquants de chaque upscaler, avec leur
    taille (lue sur Hugging Face) ; chaque carte affiche **Téléchargé** ou **À télécharger** (nombre de fichiers,
    taille, liste dépliable). Rien n'est téléchargé à cette étape.
  - **Télécharger** : télécharge uniquement les poids de l'upscaler choisi, avec le téléchargeur officiel de
    WanGP (progression, vitesse, temps restant, **Annuler**). Un transfert coupé par le réseau reprend là où il
    s'était arrêté ; un fichier annulé est supprimé et repart de zéro (comportement de WanGP). Pendant la mise
    en route, la carte affiche « Connexion à Hugging Face… N s » ; si aucune donnée n'arrive pendant une minute,
    elle le signale avec le dernier message de WanGP. Par défaut, Studio fait télécharger WanGP en HTTP
    classique plutôt qu'avec le protocole Xet de Hugging Face (option dans Paramètres → WanGP).
    Les téléchargements passent par la file des traitements : vous pouvez en lancer plusieurs.
    Les fichiers communs (FlashVSR One/Two Pass, LTX 2.5 Upscale/Detail Refiner) ne sont téléchargés qu'une fois.
  - **Tester** : lance un court traitement réel (et télécharge les poids s'ils manquent).
  - **Vérifier WanGP** (état réel de DLSS 5 et des fichiers), **Arrêter WanGP**.
  - **DLSS 5 Spatial — Intensité et Passes :** l'intensité va de 0 à 2 ; c'est un plafond imposé par WanGP et
    par le programme NVIDIA `nr-depth-worker.exe`, qui refuse toute autre valeur. Pour un effet plus marqué,
    le réglage **Passes DLSS 5** (1 à 3) refait passer l'image dans DLSS 5 à ×1 (affinage à la résolution
    native, même intensité) : l'effet se cumule, et chaque passe ajoute environ le temps de la première.
  - DLSS 5 Spatial ne télécharge ici que son guide de profondeur (Depth Anything) ; les composants NVIDIA
    s'installent avec `INSTALLER_DLSS5.bat`. DLSS 5 Temporal n'a pas de poids d'IA.
- Un seul moteur occupe le GPU à la fois : passer de SeedVR2 à WanGP (ou l'inverse) libère automatiquement
  la mémoire de l'autre. Le premier traitement WanGP prend quelques secondes de plus (chargement de WanGP).
- *Paramètres → WanGP* : décodeur FlashVSR (Tiny rapide / VAE Wan complet), garder WanGP chargé.

## Réglages avancés
| Réglage | Effet |
|---|---|
| Graine | Résultat reproductible ; changer la graine change le rendu. |
| Correction colorimétrique | `lab` (conseillé), `wavelet`, `wavelet_adaptive`, `hsv`, `adain`, `none`. |
| Images par lot (4n+1) | Vidéo : plus grand = meilleure cohérence temporelle, plus de VRAM. |
| Recouvrement temporel | Images partagées entre segments pour des transitions plus douces. |
| Images préliminaires | Réduit les artefacts des toutes premières images. |
| Taille des segments | 0 = automatique. |
| Bruit d'entrée / latent | Ajoute de la variation (0 par défaut). |

## Mémoire GPU
*Réglages → Mémoire GPU* : **BlockSwap** (blocs du modèle déchargés en RAM), déchargement du DiT / VAE en RAM,
**tuilage VAE** (encodage / décodage par tuiles). BlockSwap et le déchargement ne changent pas le rendu ;
le tuilage peut le modifier très légèrement. **Appliquer les réglages recommandés** règle tout selon votre VRAM.

En cas de mémoire insuffisante, le traitement **ne plante pas** : il libère la mémoire et réessaie avec des
réglages plus économes (déchargement, BlockSwap, puis — si autorisé dans *Paramètres* — tuilage et lots plus
petits). Les ajustements sont affichés sur la tâche et reportés dans vos réglages, pour que l'export
corresponde à l'aperçu. Si rien ne suffit, un message conseille un modèle plus léger (*SeedVR2 Lite*).

## File d'attente
Toutes les tâches (aperçus et exports) sont traitées une à une sur le GPU. Vous pouvez **mettre en pause**
(entre deux lots), **reprendre**, **annuler** (aucun fichier incomplet n'est laissé), réordonner les tâches en
attente et ouvrir le dossier d'un fichier exporté.

## Modèles
Liste de tous les modèles avec nom, version, taille, VRAM minimale et conseillée, résolution de sortie,
vitesse **mesurée** sur votre machine, compatibilité avec votre GPU, licence et état d'intégrité.
Actions : télécharger / reprendre / annuler (vitesse et temps restant affichés), **vérifier l'intégrité**
(SHA-256), supprimer, télécharger un ensemble, vérifier les mises à jour, **changer le dossier des modèles**
(avec déplacement automatique des fichiers).

## Paramètres
Langue, dossier de sortie, conserver le modèle en mémoire, comportement en cas de mémoire insuffisante,
GPU utilisé, aperçu en direct (activation, taille, qualité), taille maximale du cache des aperçus (les plus
anciens sont supprimés automatiquement), port, ouverture du navigateur, miroir Hugging Face, journal détaillé.

**Vitesse** (nouveau en 2.0) :
- **Connexions par téléchargement de modèle** (8 par défaut) : les modèles SeedVR2 et les poids WanGP sont
  téléchargés en plusieurs connexions parallèles, avec reprise et vérification SHA-256 calculée au fil du
  téléchargement. Mettez 1 si votre connexion ou votre pare-feu limite le nombre de connexions.
- **Encodeur matériel en priorité (NVENC)** : quand la carte le permet, l'export H.264 / H.265 / AV1 utilise
  l'encodeur de la carte graphique. Un encodeur choisi explicitement dans le formulaire d'export reste
  prioritaire. Dans tous les cas, l'encodage se fait pendant que le modèle calcule le segment suivant.
