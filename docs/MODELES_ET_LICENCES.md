# Modèles, sources et licences

## Ce qui a été vérifié
- **SeedVR2** est publié par ByteDance Seed (article arXiv 2506.05301, ICLR 2026), code :
  https://github.com/ByteDance-Seed/SeedVR — poids officiels : https://huggingface.co/ByteDance-Seed/SeedVR2-3B
  et https://huggingface.co/ByteDance-Seed/SeedVR2-7B — licence **Apache-2.0**. Deux tailles existent : 3B et 7B.
- Les poids officiels sont au format `.pth`. SeedVR2 Studio utilise les **mêmes poids convertis** (safetensors
  FP16/FP8 et GGUF) publiés pour le moteur d'inférence de référence hors ComfyUI,
  **ComfyUI-SeedVR2_VideoUpscaler** (https://github.com/numz/ComfyUI-SeedVR2_VideoUpscaler, Apache-2.0),
  depuis les dépôts Hugging Face `numz/SeedVR2_comfyUI` et `AInVFX/SeedVR2_comfyUI`.
- Les noms de fichiers, dépôts et empreintes SHA-256 du catalogue proviennent **mot pour mot** du registre
  de ce moteur (`engine/seedvr2_upstream/src/utils/model_registry.py`, v2.5.24). Un test automatisé vérifie
  la concordance : aucun modèle ni aucune URL n'a été inventé.

## À propos de « SeedVR2 Lite »
**Aucun modèle n'est publié par ByteDance sous le nom « SeedVR2 Lite ».** Le nom désigne, sur GitHub,
un projet communautaire d'interface (ReSerendipity/SeedVR2-Lite, Apache-2.0) qui utilise les mêmes poids
SeedVR2 3B/7B. Dans SeedVR2 Studio, la famille **« SeedVR2 Lite »** regroupe donc les **variantes légères
réelles et compatibles** : les modèles SeedVR2 quantifiés au format **GGUF** (4 bits Q4_K_M, 8 bits Q8_0),
qui réduisent fortement la VRAM nécessaire.

## Catalogue
| Fichier | Famille | Précision | Taille ≈ | VRAM min / conseillée* |
|---|---|---|---|---|
| ema_vae_fp16.safetensors | VAE (obligatoire) | FP16 | 0,5 Go | — |
| seedvr2_ema_3b_fp16.safetensors | SeedVR2 3B | FP16 | 6,8 Go | 12 / 16 Go |
| seedvr2_ema_3b_fp8_e4m3fn.safetensors | SeedVR2 3B | FP8 | 3,4 Go | 8 / 12 Go |
| seedvr2_ema_7b_fp16.safetensors | SeedVR2 7B | FP16 | 16,5 Go | 20 / 24 Go |
| seedvr2_ema_7b_fp8_e4m3fn_mixed_block35_fp16.safetensors | SeedVR2 7B | FP8 mixte | 8,5 Go | 12 / 16 Go |
| seedvr2_ema_7b_sharp_fp16.safetensors | SeedVR2 7B Sharp | FP16 | 16,5 Go | 20 / 24 Go |
| seedvr2_ema_7b_sharp_fp8_e4m3fn_mixed_block35_fp16.safetensors | SeedVR2 7B Sharp | FP8 mixte | 8,5 Go | 12 / 16 Go |
| seedvr2_ema_3b-Q4_K_M.gguf | **SeedVR2 Lite** 3B | GGUF 4 bits | 1,9 Go | 6 / 8 Go |
| seedvr2_ema_3b-Q8_0.gguf | **SeedVR2 Lite** 3B | GGUF 8 bits | 3,7 Go | 8 / 10 Go |
| seedvr2_ema_7b-Q4_K_M.gguf | **SeedVR2 Lite** 7B | GGUF 4 bits | 4,8 Go | 8 / 12 Go |
| seedvr2_ema_7b_sharp-Q4_K_M.gguf | **SeedVR2 Lite** 7B Sharp | GGUF 4 bits | 4,8 Go | 8 / 12 Go |

\* Estimations : « min » avec BlockSwap et tuilage VAE, « conseillée » sans optimisation, pour des sorties
autour de 1080p. La consommation réelle dépend de la résolution et de la taille des lots ; elle est mesurée
et affichée (pic VRAM) après chaque traitement. Les tailles exactes sont lues sur le serveur au téléchargement.

## Modèle communautaire : SeedVR2-1.4B
| Fichier | Modèle | Précision | Taille | VRAM (min. / conseillée, estimation) |
|---|---|---|---|---|
| seedvr2_distill_6L_1.4B_sharp_fp16.safetensors | **SeedVR2 1.4B Sharp (distillé)** | FP16 | 2,7 Go | 4 / 6 Go |

- **Origine :** distillation communautaire de SeedVR2-7B Sharp par *lvladikov*
  ([lvladikov/SeedVR2-1.4B](https://huggingface.co/lvladikov/SeedVR2-1.4B)). **Ce n'est pas un modèle ByteDance.**
- **Architecture :** celle du 7B avec 6 blocs de transformeur au lieu de 36 (1,44 milliard de paramètres) ;
  largeur, têtes et VAE inchangés. Le moteur intégré choisit sa configuration d'après le nom du fichier : Studio
  fournit donc la sienne (`backend/studio/engine/seedvr2_configs/1_4b/main.yaml`, configuration 7B avec
  `num_layers: 6`). Vérification automatisée : le modèle ainsi construit compte exactement les 198 tenseurs annoncés.
- **Téléchargement :** à la révision épinglée `7694e0f361dde8521668e9f8e1d242a1ee90035a`, contrôlé par SHA-256
  (`58cd0f8e…e33a`, 2 885 236 568 octets). Le VAE publié avec le modèle est le VAE SeedVR2 standard (même empreinte
  `20678548…`) : rien de plus à télécharger.
- **Qualité :** inférieure au 3B selon son auteur, mais nettement plus rapide et plus léger ; des jointures de tuiles
  peuvent apparaître au-delà d'environ 8 mégapixels (moins de blocs à fenêtre décalée).
- **Licence :** Apache-2.0 indiquée par l'auteur ; modèle dérivé de SeedVR2-7B, dont la licence et les conditions
  s'appliquent aussi.
- **Non vérifié ici :** la qualité d'image réelle (pas de GPU ni d'accès à Hugging Face dans l'environnement de
  construction).

## Upscalers WanGP « System Video Postprocessing »
Vérifiés dans le code source de WanGP (deepbeepmeep/Wan2GP, commit `38758d3c73366422efcf609abb63cab8385f3552`
du 13 novembre 2026, dossier `postprocessing/`) : les neuf upscalers demandés existent sous ces noms de
méthode, avec ces facteurs (un test automatisé le vérifie quand les sources sont présentes).

| Upscaler | Méthode WanGP | Modèle / technologie | Poids (téléchargés par WanGP) |
|---|---|---|---|
| LTX 2.5 Upscale | `ltx25*2` | LTX 2.5 22B distillé + IC-LoRA pixel spatial upscaler | modèle LTX 2.5 (dizaines de Go) |
| LTX 2.5 Detail Refiner | `ltx25_refine*1/1.5/2` | LTX 2.5 22B + IC-LoRA « Refine Details » | idem |
| LTX 2.3 Upscale | `ltx23*2` | LTX 2.3 22B + LoRA distillé + IC-LoRA | modèle LTX 2.3 (dizaines de Go) |
| RelayVSR Upscale | `relayvsr*1…4` | RelayVSR (×2 natif / ×4), INT8 | `DeepBeepMeep/Wan2.1`, dossier RelayVSR |
| Lanczos Upscale | `lanczos*1…4` | PIL Lanczos, multiple de 16 | aucun (natif dans Studio) |
| FlashVSR One Pass | `flashvsr*1…4` | FlashVSR v1.1 | `DeepBeepMeep/Wan2.1`, dossier FlashVSR + VAE Wan 2.1 |
| FlashVSR Two Pass | `flashvsr2pass*1…4` | FlashVSR v1.1, deux passes | idem |
| DLSS 5 Spatial Upsampling | `dlss5*1/1.5/1.724/2/3` | NVIDIA DLSS 5 Neural Rendering | guide Depth Anything + composants NVIDIA tiers |
| DLSS 5 Temporal Upsampling | `dlssg*2…6` | NVIDIA DLSS Frame Generation | aucun poids d'IA + composants NVIDIA tiers |

Les besoins exacts en VRAM dépendent des profils mémoire de WanGP ; les indications affichées sont des
ordres de grandeur, et la vitesse réelle est mesurée puis affichée après chaque traitement.

**DLSS 5** : les bibliothèques NVIDIA (`nvngx_dlss*.dll`), ReShade et le module RenoDX ne sont ni inclus ni
téléchargés automatiquement par Studio. `INSTALLER_DLSS5.bat` exécute le script officiel de WanGP, qui les
récupère depuis leurs sources tierces après acceptation explicite (« I ACCEPT ») ; leurs licences propres
s'appliquent. La documentation de WanGP (`docs/DLSS5.md`) détaille ces composants et leurs limites.

## Licences
- SeedVR2 (modèles et code d'origine) : Apache-2.0 — © ByteDance Ltd.
- ComfyUI-SeedVR2_VideoUpscaler (copie non modifiée dans `engine/seedvr2_upstream`) : Apache-2.0
  (voir `engine/seedvr2_upstream/LICENSE`).
- FFmpeg (téléchargé à l'installation, build BtbN « gpl ») : GPL — https://ffmpeg.org/legal.html
- PyTorch : BSD — React, Vite, Tailwind CSS, lucide-react : MIT — FastAPI, Uvicorn : MIT/BSD.
- **WanGP** (non inclus dans l'archive ; téléchargé à l'installation) : **WanGP Community License 2.0**,
  © 2024-2026 DeepBeepMeep — usage personnel et gratuit autorisé, revente et services payants interdits sans
  accord de l'auteur. SeedVR2 Studio l'utilise sans le modifier, via son API officielle, et le mentionne dans
  son interface et sa documentation comme l'exige l'API de WanGP. Studio n'est ni affilié à WanGP ni approuvé
  par son auteur. Conséquence : un logiciel qui intègre WanGP ne doit pas être vendu.
- Le relais `backend/studio/engine/wangp_worker.py` et le Lanczos natif sont du code original de Studio.
- Code de SeedVR2 Studio : Apache-2.0.
