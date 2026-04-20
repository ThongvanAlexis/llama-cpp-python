# Plan : CI pour builder des wheels Windows CUDA de llama-cpp-python

## 1. Contexte et objectif

### Le problème

Depuis **novembre 2024** (version 0.3.4), le maintainer officiel abetlen a **arrete de publier les wheels Windows** sur son index `https://abetlen.github.io/llama-cpp-python/whl/cuXXX/`. Il ne publie plus que les wheels Linux et macOS Metal. Il n'a jamais publie de `0.3.5` Windows ni rien depuis.

L'index alternatif de la communaute (`jllllll.github.io/llama-cpp-python-cuBLAS-wheels`) est bloque a la version **0.2.26** (janvier 2024) et n'est plus maintenu.

Resultat : sur Windows, on ne peut plus utiliser llama-cpp-python avec support CUDA pour les modeles modernes :
- Pas de support pour **Qwen3** (avril 2025)
- Pas de support pour **Qwen3.5-35B-A3B** MoE hybride (fevrier 2026)
- Pas de support pour **Gemma 3** (mars 2025)
- Pas de support pour les versions recentes de llama.cpp (optimisations performance, flash attention etc.)

Sans construire manuellement un wheel sur chaque machine Windows (process long avec CMake + CUDA toolkit + Visual Studio Build Tools), on est bloque sur 0.3.4 max.

### L'objectif de cette CI

Fournir des wheels Windows CUDA pour llama-cpp-python, disponibles a la demande, **pour la version de notre choix** (typiquement la derniere release amont) ciblant :
- **Python 3.11** (cible prioritaire, celle qu'on utilise)
- **Windows x64**
- **CUDA 12.4** (compatible avec driver CUDA 13.x et toute RTX 30xx/40xx)

daire un truc generique qui pourra builder pour d'autre version de python et cuda, on cible uniquement windows, le repo officiel llama-cpp build deja pour linux et autre

Publier les wheels comme **artefacts GitHub Releases** du fork, ou comme **index pip compatible** via GitHub Pages (comme fait abetlen pour Linux).

---

## 2. Finding important : le workflow EXISTE DEJA dans le repo

Le fichier `.github/workflows/build-wheels-cuda.yaml` du repo amont **contient deja tout le code pour builder Windows + CUDA**. Le support a juste ete **desactive par un simple commentaire**.

### Preuve dans le code

Ligne 22-27 du workflow actuel :

```powershell
$matrix = @{
    'os' = @('ubuntu-22.04') #, 'windows-2022')   <-- WINDOWS COMMENTE ICI
    'pyver' = @("3.9", "3.10", "3.11", "3.12")
    'cuda' = @("12.1.1", "12.2.2", "12.3.2", "12.4.1") #, "12.5.1", "12.6.1")
    'releasetag' = @("basic")
}
```

`windows-2022` est **commente**. Si on le decommente, la matrice genere aussi des jobs Windows.

### Steps Windows-specific deja ecrits

Le workflow contient deja tous les steps necessaires pour Windows, conditionnes par `if: runner.os == 'Windows'` :

1. **Ligne 46-50** : Install MSBuild (`microsoft/setup-msbuild@v2`)
2. **Ligne 70-76** : Cache VS Integration pour CUDA (deja en place, clef `cuda-${{ matrix.cuda }}-vs-integration`)
3. **Ligne 78-86** : Telecharge le zip CUDA depuis `Jimver/cuda-toolkit` et extrait les MSBuildExtensions
4. **Ligne 88-94** : Copie les MSBuildExtensions dans Visual Studio 2019 Enterprise
5. **Ligne 96-113** : Install du CUDA toolkit via mamba (works sur Windows)
6. **Ligne 115-169** : Build du wheel, avec la logique `$IsLinux` vs Windows deja differenciee

En gros : **80% du travail est deja fait.**

### Ce qu'il faut changer (minimal)

1. **Decommenter `'windows-2022'`** dans la matrice des OS
2. **Reduire la matrice** pour aller plus vite : on veut juste `pyver = "3.11"` et `cuda = "12.4.1"` pour nos besoins (la matrice complete ferait ~4*4 = 16 jobs avec Linux + Windows, on peut se contenter de 1-2 jobs Windows)
3. **Ajouter un declenchement manuel** (`workflow_dispatch` est deja la, ligne 3 : `on: workflow_dispatch`) ou sur tag de release
4. **Publier les wheels** : le step `softprops/action-gh-release@v2` ligne 173 fonctionne deja sur un push de tag ; si on veut publier ailleurs que dans Releases, il faudra modifier

---

## 3. Reimplementer le mecanisme de cache de keepassxc

Le workflow actuel de llama-cpp-python a **seulement un cache** pour les VS MSBuildExtensions CUDA. C'est pas suffisant pour rebuild rapidement si on touche un peu le code.

Keepassxc a un systeme de cache plus complet dans `C:\claude_checkouts\keepassxc\.github\workflows\build.yml`. Les elements a reprendre :

Attention a bien sauvgarder les caches en cas d'erreur, on aura surement beaucoup de fail de build lors du dev, on veut gagner du temps avec lors de la phase de test/fix

### A. sccache (compiler-level cache)

```yaml
- name: Set up sccache
  id: sccache-windows
  uses: hendrikmuhs/ccache-action@v1.2
  with:
    variant: sccache
    key: windows-llama-cpp-cuda
    append-timestamp: false

- name: Compiler cache status
  run: |
    echo "Exact hit: ${{ steps.sccache-windows.outputs.cache-hit }}"
    echo "Restored: ${{ steps.sccache-windows.outputs.cache-matched-key }}"
```

Puis dans le step Build Wheel, cabler sccache via les flags CMake (a ajouter aux CMAKE_ARGS existants) :

```
-DCMAKE_C_COMPILER_LAUNCHER=sccache
-DCMAKE_CXX_COMPILER_LAUNCHER=sccache
-DCMAKE_CUDA_COMPILER_LAUNCHER=sccache
```

**Gain attendu** : sur un rebuild avec les memes sources, 80-95% des objets sont restitues depuis le cache. Passerait de ~30-40 min a ~5 min pour un rebuild incremental.

### B. Cache du CUDA toolkit installer

Le CUDA toolkit installer fait ~3 GB. Keepassxc ne le fait pas car il utilise vcpkg, mais pour nous c'est le principal time-waster.

```yaml
- name: Cache CUDA installer zip
  id: cuda-installer-cache
  if: runner.os == 'Windows'
  uses: actions/cache@v4
  with:
    path: cudainstaller.zip
    key: cuda-installer-${{ matrix.cuda }}-windows

- name: Download CUDA installer
  if: runner.os == 'Windows' && steps.cuda-installer-cache.outputs.cache-hit != 'true'
  run: |
    # le bloc download existant (Invoke-RestMethod ...)
```

**Gain attendu** : evite ~5-10 min de telechargement par job.

### C. Cache des packages mamba/conda

Analogue au vcpkg binary cache de keepassxc. mamba a sa propre logique, mais on peut cacher `~/.conda/pkgs` ou le dossier `CONDA_PKGS_DIRS`.

```yaml
- name: Cache mamba packages
  uses: actions/cache@v4
  with:
    path: |
      ~/conda_pkgs_dir
      ~/.conda/pkgs
    key: mamba-cuda-${{ matrix.cuda }}-${{ runner.os }}
    restore-keys: |
      mamba-cuda-${{ matrix.cuda }}-
      mamba-cuda-
```

**Gain attendu** : reduction du `mamba install cuda-toolkit` de ~10-15 min a ~2-3 min sur cache hit.

### D. Cache pip (deja partiellement en place)

Le workflow a deja `cache: 'pip'` dans `actions/setup-python@v5` (ligne 59). C'est bon, on touche pas.

---

## 4. Matrice cible proposee (minimale pour nos besoins)

Pour aller vite lors du build initial (quelques jobs seulement) :

```powershell
$matrix = @{
    'os' = @('windows-2022')
    'pyver' = @("3.11")
    'cuda' = @("12.4.1")
    'releasetag' = @("basic")
}
```

1 seul job, ~30-45 min au premier run, ~5-10 min aux rebuilds grace au cache.

Si on veut elargir plus tard (autres Python, autres CUDA), il suffit d'ajouter dans la matrice.

---

## 5. Notes de compatibilite CUDA

- Le runner `windows-2022` a **Visual Studio 2022 Enterprise** installe de base. Le workflow amont pointe ligne 92 sur `Microsoft Visual Studio\2019\Enterprise` - il faudra probablement **ajuster le chemin** vers `2022\Enterprise` ou detecter automatiquement la version.
- Le flag `vs-version: '[16.11,16.12)'` ligne 50 correspond a VS 2019 (v16). A adapter a VS 2022 (v17) : `vs-version: '[17.0,18.0)'`.
- CUDA 12.4.1 est supporte par VS 2022 depuis la version 17.9+.

---

## 6. Etapes de mise en oeuvre (ordre conseille)

1. **Fork** du repo `abetlen/llama-cpp-python` (deja fait, on est dans `C:\claude_checkouts\llama-cpp-python\`)
2. Creer une branche `feature/windows-cuda-ci` dans le fork
3. Copier `build-wheels-cuda.yaml` vers `build-wheels-cuda-windows.yaml` (nouveau fichier, eviter les conflits de merge amont)
4. Dans ce nouveau fichier :
   - Reduire la matrice a la combinaison ciblee (windows-2022, 3.11, 12.4.1)
   - Garder uniquement les steps Windows-specific (supprimer les branches $IsLinux)
   - Ajuster les chemins VS 2019 -> VS 2022
   - Ajouter les 3 caches additionnels (sccache, CUDA installer zip, mamba pkgs)
5. Trigger manuel (`workflow_dispatch`) pour le premier run
6. Debug si erreurs (les issues habituelles : version VS, chemin CUDA_PATH, flag MSVC, etc.)
7. Une fois qu'un wheel est produit avec succes :
   - **Option A** : Publier en `Release` GitHub (mecanisme softprops deja en place)
   - **Option B** : Publier sur GitHub Pages du fork, format pip-compatible (plus bas)
8. Tester le wheel en local : `pip install llama_cpp_python-0.3.20-cp311-cp311-win_amd64.whl`
9. Si OK, le projet APHP bascule sur ce wheel.

### Option B : pip-compatible index sur GitHub Pages

Pour un index au format de `https://abetlen.github.io/llama-cpp-python/whl/cu124/` :
- Branche `gh-pages` dans le fork
- Structure : `whl/cu124/llama-cpp-python/index.html` avec des liens `<a href="llama_cpp_python-0.3.20-cp311-cp311-win_amd64.whl">`
- Uploader le .whl dans ce meme dossier (ou sur un release asset)
- Activer GitHub Pages pointant sur la branche `gh-pages`

Utilisation finale :
```
pip install llama_cpp_python==0.3.20 --extra-index-url https://<username>.github.io/llama-cpp-python/whl/cu124
```

---

## 7. Risques et pieges connus

- **Runner timeout** : les jobs GitHub Actions gratuits ont un timeout de 6h. Un build complet de llama.cpp avec CUDA pour 1 target Python + 1 CUDA version tient largement dans ce budget (~30-45 min), mais ne pas mettre une matrice trop large.
- **Taille du wheel** : llama-cpp-python CUDA peut depasser 200 MB. OK pour GitHub Release (limite 2 GB), OK pour GitHub Pages.
- **Compatibilite ABI** : un wheel `cp311` ne marche QUE sur Python 3.11. Pour couvrir 3.10/3.11/3.12/3.13 il faut un job par version.
- **Driver vs Runtime CUDA** : le wheel compile avec CUDA 12.4 marche sur tout driver >= 525.x (forward compatibility). Donc CUDA 13.2 driver de l'utilisateur APHP fera tourner un wheel cu124 sans souci.
- **VS Integration cache obsolete** : si on change la version CUDA, purger le cache `cuda-X.Y.Z-vs-integration`.

---

## 8. Intention derriere la desactivation par l'auteur

**A lire imperativement avant de decommenter `windows-2022`.** Le but de cette section est d'eviter qu'on se dise "il suffit de retirer le `#`" et qu'on perde une journee a comprendre pourquoi ca casse pareil.

### Le commit reel

Contrairement a ce que la section 1 sous-entendait (arret en novembre 2024 avec la 0.3.4), la desactivation effective du CI Windows date du **6 juillet 2025**, commit `98fda8c "fix(ci): Temporarily disable windows cuda wheels"`, par abetlen lui-meme. Le commit a ete pousse **directement sur master**, sans PR, sans issue liee, sans description plus longue que le titre. Le diff est exactement les deux commentaires qu'on voit aujourd'hui : commenter `windows-2022` dans la matrice et figer le runner Linux a `ubuntu-22.04`.

(Le fait que la 0.3.4 de novembre 2024 soit la derniere version avec wheels Windows publies est une consequence du calendrier de release, pas du commit. Entre 0.3.4 et juillet 2025 le CI Windows etait deja boiteux, voir plus bas.)

### Pas de decision technique : abandon par attrition

Il n'existe **aucune justification ecrite** de cette desactivation, ni dans le commit, ni en issue, ni en discussion, ni sur Reddit/HN. En revanche, le contexte des commits voisins est tres parlant - c'est une serie de pansements en urgence sur toute la CI :

```
cce4887 fix(ci): Fix macos cpu builds
98fda8c fix(ci): Temporarily disable windows cuda wheels    <-- ici
7011bc1 fix(ci): Update docker runner
82ad829 fix(ci): update runners for cpu builds
11d28df fix(ci): Remove macos-13 builds to fix cross compilation error
ae54cde fix(ci): Update cuda build action to use ubuntu 22.04
```

Lecture : abetlen eteignait des feux sur toute la CI ce week-end-la (les images runner GitHub avaient bouge). Windows CUDA a ete sacrifie comme macos-13 a ete sacrifie, pour redevenir vert le plus vite possible. Le mot "Temporarily" dans le titre du commit etait sincere - mais 9 mois plus tard, rien n'a bouge.

### Pourquoi specifiquement Windows + CUDA est un puits sans fond

L'historique des issues montre que ce build casse en boucle depuis debut 2024, toujours pour les memes raisons :

1. **Incompatibilite Visual Studio / nvcc**. Les runners GitHub mettent a jour leur image VS 2022 (17.x), nvcc verifie strictement la version et refuse avec :
   > `unsupported Microsoft Visual Studio version! Only the versions between 2017 and 2022 (inclusive) are supported!`

   Voir issues #1543, #1551, #1838, #1894. C'est aussi la raison du `#, "12.5.1", "12.6.1"` commente a cote de `windows-2022` (commit `4f17ae5 "Remove cuda version 12.5.0 incompatibility with VS"`).

2. **Le contournement `-allow-unsupported-compiler` produit des wheels qui crashent au runtime** (confirme sur #1543). Donc on ne peut pas juste passer outre - il faut vraiment matcher VS et CUDA.

3. **Le workflow est un fork de celui de jllllll**, lui-meme abandonne depuis janvier 2024. Abetlen n'est manifestement pas un dev Windows ; il a herite d'un truc qu'il ne maitrise pas, et a chaque rotation des images runner ca repete.

### Signaux que le projet upstream est en pause

- Issue #2068 (sept 2025) "Where can I download wheel for Cuda 12.8?" : aucune reponse maintainer.
- Issue #2127 (fev 2026) : l'index `whl/cu125` renvoie 404, aucune reponse.
- Issue #2136 : **Quansight propose explicitement du financement d'ingenierie** pour reparer la matrice de wheels (3.13 / 3.14 / free-threaded), et attend toujours un signal du maintainer plusieurs semaines plus tard.

Bref : ce n'est pas un choix architectural a contourner, c'est un manque de mainteneur Windows cote upstream. C'est aussi pour ca que notre fork a un sens : on ne va pas convaincre abetlen de rebrancher Windows, on doit juste le faire dans notre coin.

### Implications operationnelles pour notre reactivation

Decommenter `windows-2022` **NE SUFFIRA PAS**. Le CI casse pour de vraies raisons, qu'il faut traiter explicitement dans notre nouveau workflow :

1. **Pinner la version de Visual Studio** sur le runner. `windows-2022` arrive avec une VS 17.x recente que nvcc 12.4 peut refuser. Soit on degrade VS (`vs-version: '[17.4,17.10)'` par exemple, a tester), soit on bascule sur le runner `windows-2025` si dispo, soit on installe une `BuildTools` specifique via `microsoft/setup-msbuild`. La section 5 du present plan mentionne deja le sujet en passant - le rendre prioritaire.

2. **Verifier la compatibilite CUDA 12.4.1 / VS 17.x du jour**. La matrice de support NVIDIA bouge ; verifier sur les release notes CUDA que la combinaison ciblee est supportee officiellement, sinon ajuster.

3. **Tester le wheel produit avant publication**, pas juste verifier qu'il compile. Le piege historique : `-allow-unsupported-compiler` produit des `.whl` qui s'installent puis segfault au premier `Llama(...)`. Il faut un step de smoke test (`python -c "from llama_cpp import Llama; Llama(...)"` sur un mini-modele) sur le runner Windows avant le publish.

4. **Ne pas s'attendre a un merge upstream**. Notre workflow vit dans notre fork. Si on veut un jour proposer le retour Windows en upstream, il faudra un effort de test/maintenance autonome (cf. l'offre Quansight qui pourrit dans #2136 : abetlen ne reagit pas meme a du financement gratuit).

### Resume en une phrase

Le `# windows-2022` n'est pas un toggle a flipper, c'est une **cicatrice de bataille perdue**. Notre plan doit prevoir le combat VS/nvcc, pas juste le decommentaire.

## 9. le test du wheel buildé

dans llm_for_ci_test on a tiny-llama.gguf (27Kb)
c'est un mini gguf justement pour tester.

faudrais rajouter une etape qui verifie que le wheel fonctionne puisque un des probleme est que le wheel generé peu avoir l'air bon mais pete lors d'un vrai appel

proposition de test : 

```
- name: Smoke test wheel
  run: |
    pip install llama_cpp_python-*.whl
    python -c "
    from llama_cpp import Llama
    llm = Llama(model_path='tests/fixtures/tinyllama.gguf', n_ctx=64)
    output = llm('hello', max_tokens=2)
    print(output)
    print('Smoke test passed')
    "
```

Pas de GPU nécessaire pour ce test — ça tourne en CPU, et ça vérifie que le wheel charge, instancie un modèle, et fait de l'inférence sans segfault. Le vrai test CUDA tu le fais en local de toute façon.