# Répartition du travail — ce soir, rendu 23h

Le dépôt de référence est déjà complet (app, 5 labs, policies Kyverno, pipeline CI, templates).
**On n'invente rien : on exécute, on personnalise (`Ymed95`), on prouve.** Les mêmes tickets
existent en Issues GitHub sur ce dépôt — cochez-les au fur et à mesure.

## ⚡ SETUP EXPRESS (0 → 30 min) — À FAIRE MAINTENANT, EN PARALLÈLE

Équipe : **2 postes Windows + 1 poste Linux.**

**Affectation (choisie selon l'OS) :**
| Personne | OS | Piste | Pourquoi |
|---|---|---|---|
| 1 | **Linux** | **B — cluster Kyverno + attaque/défense** (#3) | `kind` tourne le plus proprement sur Linux |
| 2 | **Windows** | **A — build, SBOM, scan, signature** (#2) | a besoin de Docker + les CLI, poste de build principal |
| 3 | **Windows** | **C — CI + rapport + threat model** (#4) | le plus léger en local (la CI tourne sur GitHub) |

### 0. Cloner la branche de travail (les 3)
```bash
git clone https://github.com/Ymed95/projet_technique.git
cd projet_technique
git checkout claude/project-delivery-tonight-kt8qhz
```

### 1a. Installer les outils — Linux (personne 1)
```bash
# Docker (si absent) : suivez docs.docker.com/engine/install pour votre distro, puis :
sudo usermod -aG docker $USER   # puis re-login pour éviter sudo à chaque docker
# kind + kubectl + jq :
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
sudo apt-get install -y jq   # ou dnf/pacman selon la distro
# syft / grype / cosign :
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh  | sh -s -- -b /usr/local/bin
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin
curl -sSfLo cosign https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
chmod +x cosign && sudo mv cosign /usr/local/bin/
```

### 1b. Windows via WSL2 (personnes 2 & 3) — CHEMIN CHOISI ✅
**Bonne nouvelle : les 3 sont sous WSL2 Ubuntu → tout le monde utilise le bloc Linux 1a, dans le terminal Ubuntu.** Pas de winget, pas de binaires Windows.

**Le plus long : rendre Docker joignable DANS WSL. Faites-le en premier.**
- Option simple : **Docker Desktop** installé → Settings → Resources → **WSL Integration** → activer pour votre distro Ubuntu. Puis `docker version` doit répondre *dans le terminal Ubuntu*.
- Option sans Docker Desktop : `sudo apt-get install -y docker.io && sudo service docker start && sudo usermod -aG docker $USER` (puis rouvrir le terminal).

Ensuite, dans WSL2 Ubuntu, lancez le **bloc 1a** (kind, kubectl, jq, syft, grype, cosign).
> ⚠️ Travaillez **toujours dans le terminal WSL2 Ubuntu**, pas dans PowerShell. Clonez le repo côté Linux (`~/projet_technique`, pas `/mnt/c/...`) pour de meilleures perfs Docker.
> La personne 3 (Piste C) a surtout besoin de Docker + navigateur (la CI tourne sur GitHub).

### 2. Vérifier (chaque poste, selon sa piste)
```bash
docker version && kind version && kubectl version --client && syft version && grype version && cosign version && jq --version
```
Personne 3 (Piste C) : `docker version` + un compte GitHub suffisent pour démarrer.

### 3. PAT GitHub `write:packages` (personne 2 surtout — requise dès Lab 0)
1. GitHub → Settings → Developer settings → **Personal access tokens (classic)** → Generate → cocher **`write:packages`** (coche aussi `read:packages`). **Ne JAMAIS le commiter.**
2. Se connecter à GHCR :
```bash
echo "VOTRE_TOKEN" | docker login ghcr.io -u Ymed95 --password-stdin
```
> Si le PAT est créé sur un autre compte que `Ymed95`, connectez-vous avec CE compte-là et poussez sous `ghcr.io/CE-compte/...` — mais alors prévenez-moi pour que je réaligne les policies. **Le plus simple : PAT sur le compte `Ymed95`.**

**Quand `docker version` répond et que `docker login ghcr.io` réussit → chacun démarre sa piste (labs ci-dessous).**

---


## Décisions déjà prises (ne pas rediscuter)
- Identité GHCR unique pour toute l'équipe : `ghcr.io/Ymed95/scs-demo-app`.
- Toutes les policies `policies/kyverno/*.yaml` et `k8s/deployment.yaml` sont déjà personnalisées
  avec `Ymed95`. Il reste à y coller le `cosign.pub` (Piste A) et le vrai digest (Piste A → B).
- Une seule personne a besoin d'un **PAT GitHub `write:packages`** pour les push GHCR manuels
  (Labs 0-2 en local) : idéalement le compte `Ymed95`, sinon celui qui push crée le PAT et
  le partage en canal privé (jamais commité).
- Le pipeline CI (Lab 5) ne nécessite AUCUN secret : il utilise OIDC + `GITHUB_TOKEN` automatiquement.
  → **Activez-le tôt**, il peut tourner en fond pendant que vous faites le reste à la main.

## Piste A — Build & Preuves (Labs 0 → 1 → 2)
**Objectif :** une image signée, poussée sur GHCR, avec SBOM + scan + 2 attestations vérifiables.

- [ ] Lab 0 : outils installés, image buildée, poussée sur `ghcr.io/Ymed95/scs-demo-app`, digest récupéré
- [ ] Lab 1 : SBOM Syft (spdx-json), scan Grype avec `.grype.yaml` (`only-fixed: true`, `fail-on-severity: critical`)
- [ ] Lab 1 : démo gate cassée (downgrade Flask temporaire) + **capture** + rollback propre
- [ ] Lab 2 : signature par clé (`cosign generate-key-pair`, `cosign.key` bien dans `.gitignore` — déjà fait) + `cosign verify`
- [ ] Lab 2 : signature keyless + `cosign verify` (identité OIDC)
- [ ] Lab 2 : attestation SBOM attachée + vérifiée (`cosign verify-attestation --type spdxjson`)
- [ ] Lab 2 : attestation provenance attachée + vérifiée (`cosign verify-attestation --type slsaprovenance`)
- [ ] `cosign tree $DIGEST` montre signature + 2 attestations
- [ ] **Livrer à la Piste B :** le `$DIGEST`, le contenu de `cosign.pub`
- [ ] **Livrer à la Piste C :** toutes les sorties de commandes (pour le rapport §3)

## Piste B — Cluster qui refuse & démo attaque/défense (Labs 3 → 4)
**Objectif :** cluster kind + Kyverno en `Enforce`, image légitime acceptée, 5 attaques bloquées et capturées.

- [ ] Lab 3 : `kind create cluster --config cluster/kind-config.yaml`, Kyverno installé et `Ready`
- [ ] Lab 3 : namespace `app` créé, 4 `ClusterPolicy` appliquées (déjà personnalisées `Ymed95`)
- [ ] Lab 3 : coller le `cosign.pub` de la Piste A dans `03-verify-signature.yaml` et `04-require-provenance.yaml`
- [ ] Lab 3 : `k8s/deployment.yaml` mis à jour avec le vrai `$DIGEST` → `kubectl apply` → pod **Running**
- [ ] Lab 4 — Attaque 1 : image non signée → refusée + capture
- [ ] Lab 4 — Attaque 2 : image modifiée après signature (scénario SolarWinds) → refusée + capture
- [ ] Lab 4 — Attaque 3 : registry non autorisé (ex. nginx Docker Hub) → refusée + capture
- [ ] Lab 4 — Attaque 4 : tag `:latest` → refusée + capture
- [ ] Lab 4 — Attaque 5 (bonus) : signée sans provenance → refusée + capture
- [ ] **Vidéo** de la séquence complète (plan B soutenance)
- [ ] Tableau de synthèse attaque → contrôle → menace rempli
- [ ] **Livrer à la Piste C :** toutes les captures + la vidéo + le tableau

## Piste C — CI bout-en-bout, Rapport, Threat Model, intégration finale (Lab 5 + livrables)
**Objectif :** pipeline CI vert (bonus SLSA L2), rapport + threat model complets, dépôt prêt pour la soutenance.

- [ ] Lab 5 : activer le workflow `.github/workflows/supply-chain.yml` sur push vers `main`
- [ ] Lab 5 : vérifier qu'il build + SBOM + scan + push + sign (keyless) + 2 attestations, tout vert
- [ ] Lab 5 : basculer `policies/kyverno/03-verify-signature.yaml` en variante **keyless** (bloc déjà préparé, à décommenter) et re-tester l'admission avec l'image issue de la CI
- [ ] Rédiger `livrables/rapport.md` à partir de `livrables/TEMPLATE-rapport.md` (contexte, architecture, mise en œuvre + preuves, démo attaque/défense, positionnement SLSA, reproductibilité, bilan)
- [ ] Rédiger `livrables/threat-model.md` à partir de `livrables/TEMPLATE-threat-model.md` (actifs, menaces, table menace→contrôle→couverture, niveau SLSA visé vs atteint — **soyez honnêtes sur L2 vs L3**)
- [ ] Intégrer les captures/sorties de A et B dans le rapport
- [ ] Repasser la checklist d'auto-évaluation de la consigne (section 8) avant 23h
- [ ] Préparer les 12 min de démo live + anticiper les 5 min de questions

## Timeline verrouillée — 6 h, 3 personnes (rendu 23h)

**Décision qui dé-risque tout : on fait la signature PAR CLÉ en premier** (déterministe, hors-ligne,
pas d'auth navigateur), et c'est déjà le mode actif des policies fournies (`03`, `04` : bloc
`publicKeys`). Le **keyless** et la **CI** sont du **bonus** (ils débloquent la discussion SLSA L2) :
on ne les fait que si la démo par clé est verte et filmée. **Ne bloquez jamais le chemin critique
pour un bonus.**

**Chemin critique (ce qui doit absolument être fini) :**
`A: image signée par clé + attestations` → `B: policies avec cosign.pub + déploiement accepté`
→ `B: 4 attaques filmées` → `C: rapport + threat model assemblés`.
Le jalon le plus important : **A livre `$DIGEST` + `cosign.pub` à B au plus tard à H+3.**

| Bloc (2 h chacun) | Piste A — preuves | Piste B — cluster/démo | Piste C — CI + rédaction |
|---|---|---|---|
| **H+0 → H+2** | Lab 0 (outils, build, push GHCR, digest) **puis** Lab 1 (SBOM + scan + gate cassée + capture) | Lab 3.1-3.2 en parallèle : `kind` up, Kyverno `Ready`, ns `app`, applique les 4 policies (déjà `Ymed95`). Relis lab4 pour préparer les 5 attaques. | Active le workflow Lab 5 (aucun secret) → le laisser tourner. Démarre `threat-model.md` + rapport §1-2 (contexte, archi) — pas besoin des preuves pour ça. |
| **H+2 → H+4** | Lab 2 **par clé** : `generate-key-pair`, `sign`, `attest` SBOM + provenance, `cosign tree`. **→ livrer `$DIGEST` + `cosign.pub` à B (~H+3).** Puis keyless en bonus. | Colle `cosign.pub` dans policies `03` et `04`, mets le `$DIGEST` dans `k8s/deployment.yaml` → `kubectl apply` → **pod Running**. Enchaîne Lab 4 : les 5 attaques, **capture + vidéo** de chaque refus. | Suis la CI (verte ?). Rapport §3 (mise en œuvre) au fil des sorties que A pousse. Prépare le tableau attaque→contrôle→menace. |
| **H+4 → H+5** | Aide B à filmer / rejoue le cas nominal. Vérifie que `cosign.key` n'est PAS commité. | Termine captures + **vidéo complète** (plan B soutenance). Remplit le tableau de synthèse. | Rapport §4-5 (démo + SLSA, honnête sur L2/L3) avec les captures de B. Bonus si CI verte : bascule policy `03` en keyless et note-le. |
| **H+5 → H+6** | **Tous ensemble :** repasser la checklist consigne §8, assembler rapport + threat model, intégrer captures/vidéo, relire, commit + push final. Répéter la démo une fois pour la soutenance. |

**Si vous prenez du retard, coupez dans cet ordre (garde le noté) :** keyless → CI Lab 5 → attaque 5
(sans-provenance) → comparaison Trivy. **Ne coupez jamais :** SBOM+gate, signature par clé,
déploiement accepté, les 4 attaques filmées, rapport + threat model.

**Règle d'or :** chaque membre **commit lui-même** ses preuves (la traçabilité Git est notée).
Ne travaillez pas dans un fichier partagé unique en fin de soirée — poussez au fil de l'eau sur
cette branche pour éviter un merge conflict à 22h55.
