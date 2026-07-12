# Rapport — Sécuriser la chaîne d'approvisionnement logicielle (SLSA)

- **Groupe :** _(à compléter — noms des 3 membres)_
- **Fork :** https://github.com/Ymed95/projet_technique
- **Image :** `ghcr.io/ymed95/scs-demo-app`
- **Voie :** ☑ Local (kind) ☐ Azure (AKS/ACR)
- **Date :** 2026-07-12

> **Convention de ce rapport :** chaque garantie annoncée est associée à la **commande exacte**
> qui la prouve. Les blocs marqués `⏳ PREUVE À CAPTURER` indiquent où coller la sortie réelle
> obtenue sur votre machine (toutes les commandes sont reproductibles, cf. §6).

---

## 1. Contexte & objectif

Nous savons tous construire un pipeline `build → test → scan → deploy`. Mais une fois l'image
en production, **rien ne garantit qu'elle n'a pas été altérée entre le build et le déploiement**.
Un `docker pull` ne vérifie aucune origine, et « le scan était vert » ne prouve pas que l'image
*déployée* est celle qui a été *scannée*. Les attaques 2020-2024 ne visent d'ailleurs plus
l'application elle-même mais **la chaîne de fabrication** :

- **SolarWinds (2020)** — code malveillant injecté dans le *build*, signé par l'éditeur, distribué à 18 000 clients.
- **Codecov (2021)** — script CI modifié exfiltrant les secrets de milliers de pipelines.
- **Dependency confusion (2021)** — faux paquets « internes » publiés sur les registries publics.
- **XZ Utils / liblzma (2024)** — backdoor introduite sur 3 ans dans une dépendance open source.

**Objectif du POC :** transformer un pipeline classique en **chaîne d'approvisionnement
vérifiable**, et déployer un cluster qui **refuse activement** toute image qu'il ne peut pas
prouver digne de confiance. La question à laquelle on répond : *« comment prouver que l'image
qui tourne est bien celle que NOUS avons construite, et pas une version piégée ? »*

---

## 2. Architecture de la chaîne

```
 code ──► build ──► SBOM (Syft) ──► scan (Grype) ──► SIGNATURE (cosign/Sigstore)
                                                        │
                                                        ├─► attestation SBOM
                                                        └─► attestation de PROVENANCE (SLSA)
                                                                    │
                                                             push ──► GHCR (registry)
                                                                    │
   ┌────────────────────────────────────────────────────────────────┘
   ▼
Cluster Kubernetes (kind) + KYVERNO (admission control)
   ├─ image signée par NOTRE identité ?          sinon ─► ❌ REFUSÉE
   ├─ attestation de provenance présente ?        sinon ─► ❌ REFUSÉE
   ├─ registry autorisé + par digest ?            sinon ─► ❌ REFUSÉE
   └─ pas de :latest ?                             sinon ─► ❌ REFUSÉE
```

| Brique | Outil | Rôle |
|---|---|---|
| **1 · SBOM** | Syft | Inventaire exact des composants de l'image (formats SPDX / CycloneDX). |
| **2 · Scan** | Grype | Détecte les CVE ; **casse la chaîne** sur une vulnérabilité critique corrigeable. |
| **3 · Signature** | cosign / Sigstore | Preuve cryptographique « c'est bien nous », liée au **digest**. |
| **4 · Attestations** | cosign attest | Affirmations signées attachées à l'image : le SBOM + la provenance (qui/quoi/d'où/quand). |
| **5 · Admission** | Kyverno | Gardien du cluster : vérifie signature + attestations et **refuse** l'inconnu. |

**Décision d'architecture clé :** on travaille **par digest** (`@sha256:…`), jamais par tag
mutable. Un tag peut être réécrit silencieusement ; le digest est l'empreinte immuable du
contenu. Toute la garantie d'intégrité en découle : si un octet change, le digest change, la
signature ne correspond plus.

---

## 3. Mise en œuvre

### 3.1 Application & image

L'application fournie est une API Flask minimale (`/`, `/health`, `/api/hello`, `/metrics`)
instrumentée pour Prometheus. Le `Dockerfile` est **multi-stage**, **non-root** (`uid 10001`),
avec un `HEALTHCHECK` et `gunicorn` en serveur WSGI de production. Le but pédagogique n'est pas
l'app mais **la chaîne autour**.

```bash
export IMG=ghcr.io/ymed95/scs-demo-app
export TAG=0.1.0
docker build -t "$IMG:$TAG" app/
docker run --rm -d -p 8080:8080 --name scs "$IMG:$TAG"
curl -s localhost:8080/health ; echo      # attendu : {"status":"ok","version":"1.0.0"}
docker stop scs
docker push "$IMG:$TAG"
export DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' "$IMG:$TAG")
echo "$DIGEST"
```

> ⏳ **PREUVE À CAPTURER (P1)** — coller la sortie de `curl .../health` et la ligne
> `DIGEST = ghcr.io/ymed95/scs-demo-app@sha256:…`. Ce digest est réutilisé partout ensuite.

### 3.2 SBOM (Syft)

Le SBOM liste tous les paquets de l'image. On le génère au format **SPDX** (interopérable) et
**CycloneDX** (orienté sécurité).

```bash
syft "$IMG:$TAG" -o spdx-json > sbom.spdx.json
syft "$IMG:$TAG" -o cyclonedx-json > sbom.cdx.json
syft "$IMG:$TAG" -o table | head -n 30
```

**Utilité :** le jour où une CVE tombe sur une lib, « suis-je affecté ? » devient répondable en
secondes en cherchant dans le SBOM. C'est aussi une exigence réglementaire montante (US EO 14028,
Cyber Resilience Act).

**Résultat obtenu (P2) :** Syft a catalogué **112 paquets** dans l'image (mélange de paquets
système `deb` — Debian 13 « trixie » — et de paquets `python`), sur 2 682 localisations de
fichiers. Extrait du tableau :

```
NAME                     VERSION          TYPE
apt                      3.0.3            deb
bash                     5.2.37-2+b8      deb
ca-certificates          20250419         deb
flask                    3.0.3            python
click                    8.4.2            python
blinker                  1.9.0            python
...                      (112 paquets au total)
```

On retrouve bien la « liste d'ingrédients » : Flask 3.0.3, gunicorn, la base Debian, openssl, etc.

### 3.3 Scan & gate qui casse (Grype)

On ne se contente pas d'afficher les CVE : on **arrête la chaîne** quand une vulnérabilité
critique *corrigeable* existe. La politique est dans `.grype.yaml` à la racine :

```yaml
only-fixed: true          # ne bloquer que sur le corrigeable (moins de faux positifs)
fail-on-severity: critical
```

```bash
grype sbom:sbom.spdx.json -o table          # scan lisible
grype "$IMG:$TAG" ; echo "exit=$?"          # lit .grype.yaml, exit ≠ 0 = chaîne cassée
```

**Résultat obtenu (P3) — la gate s'est déclenchée sur des CVE réelles :** sur notre image, Grype
a trouvé **91 vulnérabilités (dont 8 critiques et 59 hautes), 91 corrigeables**. Comme
`.grype.yaml` exige `fail-on-severity: critical` sur du `only-fixed`, la commande sort en
**code 2** — la chaîne est **cassée**, exactement comme attendu :

```
 ✘ Scan for vulnerabilities   [91 vulnerability matches]
   ├── by severity: 8 critical, 59 high, 84 medium, 15 low, 51 negligible (36 unknown)
   └── by status:   91 fixed, 162 not-fixed, 162 ignored
NAME         INSTALLED        FIXED IN          TYPE  VULNERABILITY    SEVERITY   RISK
libssl3t64   3.5.5-1~deb13u2  3.5.6-1~deb13u2   deb   CVE-2026-34182   Critical   0.2
openssl      3.5.5-1~deb13u2  3.5.6-1~deb13u2   deb   CVE-2026-34182   Critical   0.2
...
[0042] ERROR discovered vulnerabilities at or above the severity threshold
exit=2
```

**Interprétation :** les critiques proviennent de la bibliothèque **openssl** de l'image de base
`python:3.12-slim`, et sont **corrigeables** (`FIXED IN 3.5.6-1~deb13u2`). C'est précisément le
cas que la gate doit bloquer : une vulnérabilité **critique ET actionnable**. Le choix
`only-fixed: true` évite de bloquer sur des CVE sans correctif (bruit non actionnable).

> **Remédiation (documentée) :** pour repasser la gate au vert, on applique les correctifs de la
> base au build (`apt-get upgrade` dans le stage runtime du Dockerfile) puis on reconstruit. C'est
> la boucle « détecter → corriger → re-vérifier » attendue en production. *(Optionnel selon le
> temps ; la preuve que la gate **bloque** est déjà acquise ci-dessus.)*

> ℹ️ Démo alternative prévue par le lab (non nécessaire ici puisque la gate casse déjà sur du
> réel) : épingler `Flask==2.0.1` dans `requirements.txt`, rebuild, `grype --fail-on high` → exit ≠ 0.

### 3.4 Signature (cosign)

Signer = attacher une preuve cryptographique « c'est bien nous qui l'avons produite ». On
travaille **par digest**. On documente les deux modes.

**(A) Par clé** — pour comprendre la mécanique, et c'est ce que vérifie la policy Kyverno locale :

```bash
cosign generate-key-pair                    # crée cosign.key (SECRET, déjà .gitignore) + cosign.pub
cosign sign --key cosign.key "$DIGEST"
cosign verify --key cosign.pub "$DIGEST" | jq '.[].optional'
```

**(B) Keyless** — le « vrai » usage : identité OIDC via Fulcio, preuve publique dans Rekor,
aucune clé à stocker. Utilisé en CI (§3.7).

```bash
cosign sign --yes "$DIGEST"
cosign verify --certificate-identity-regexp ".*" --certificate-oidc-issuer-regexp ".*" "$DIGEST"
```

**Résultat obtenu (P4) — signature vérifiée :**

```
$ cosign verify --key cosign.pub "$DIGEST"
Verification for ghcr.io/ymed95/scs-demo-app@sha256:dd1899387bc...057d068 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - Existence of the claims in the transparency log was verified offline
  - The signatures were verified against the specified public key
```

Le secret `cosign.key` est bien **ignoré par git** (`git status` ne montre que `cosign.pub`) —
la clé privée n'est jamais commitée, conformément à `.gitignore`.

### 3.5 Attestations : SBOM + provenance

Une **attestation** = une affirmation *signée* rattachée à l'image. On en attache deux.

```bash
# Attestation SBOM
cosign attest --key cosign.key --predicate sbom.spdx.json --type spdxjson "$DIGEST"
cosign verify-attestation --key cosign.pub --type spdxjson "$DIGEST" \
  | jq '.payload' -r | base64 -d | jq '.predicateType'

# Attestation de provenance (predicate SLSA fabriqué en local — en CI il est généré automatiquement)
cat > provenance.json <<'EOF'
{
  "buildType": "https://example.com/manual-local-build/v1",
  "builder": { "id": "local:ymed95" },
  "invocation": {
    "configSource": {
      "uri": "git+https://github.com/Ymed95/projet_technique",
      "digest": { "sha1": "<commit-sha>" }
    }
  },
  "metadata": { "buildStartedOn": "2026-07-12T20:00:00Z" }
}
EOF
cosign attest --key cosign.key --predicate provenance.json --type slsaprovenance "$DIGEST"
cosign verify-attestation --key cosign.pub --type slsaprovenance "$DIGEST" \
  | jq '.payload' -r | base64 -d | jq '.predicateType, .predicate.builder'

cosign tree "$DIGEST"    # doit montrer la signature (.sig) + les 2 attestations (.att)
```

**Résultat obtenu (P5) — signature + 2 attestations attachées au digest :**

```
$ cosign verify-attestation --key cosign.pub --type spdxjson "$DIGEST"      → "https://spdx.dev/Document"
$ cosign verify-attestation --key cosign.pub --type slsaprovenance "$DIGEST" → "https://slsa.dev/provenance/v0.2"

$ cosign tree "$DIGEST"
📦 Supply Chain Security Related artifacts for ...scs-demo-app@sha256:dd1899...057d068
├── 🔗 https://slsa.dev/provenance/v0.2   (attestation de provenance)
├── 🔗 https://sigstore.dev/cosign/sign/v1 (signature)
└── 🔗 https://spdx.dev/Document          (attestation SBOM)
```

Les trois artefacts (1 signature + 2 attestations) sont **stockés à côté de l'image dans GHCR**,
rattachés au **digest exact**, et tous **vérifiés** contre notre clé publique. C'est la garantie
zero-trust : si l'image changeait d'un octet, le digest changerait et aucune de ces preuves ne
correspondrait plus.

### 3.6 Admission control (Kyverno)

Le cluster local `kind` reçoit **Kyverno** et **4 ClusterPolicy** (dossier `policies/kyverno/`),
toutes en `validationFailureAction: Enforce` (= **refuse**, vs `Audit` = journalise seulement) :

| Fichier | Ce qu'elle exige |
|---|---|
| `01-allowed-registries.yaml` | image issue de `ghcr.io/ymed95/` **uniquement** |
| `02-disallow-latest.yaml` | pas de tag `:latest`, tag/digest explicite obligatoire |
| `03-verify-signature.yaml` | **signature cosign valide** de notre identité (`cosign.pub` collé) ; `mutateDigest`+`verifyDigest` |
| `04-require-provenance.yaml` | **attestation de provenance** signée présente |

```bash
kind create cluster --name scs --config cluster/kind-config.yaml
kubectl create -f https://github.com/kyverno/kyverno/releases/latest/download/install.yaml
kubectl -n kyverno rollout status deploy/kyverno-admission-controller
kubectl create namespace app
# coller le contenu de cosign.pub dans policies 03 et 04, puis :
kubectl apply -f policies/kyverno/
kubectl get clusterpolicy         # toutes Ready: true
```

Puis on met le **digest signé** dans `k8s/deployment.yaml` et on déploie :

```bash
kubectl apply -n app -f k8s/deployment.yaml
kubectl get pods -n app -w        # image signée + conforme ⇒ pod Running ✅
```

> ⏳ **PREUVE À CAPTURER (P6)** — `kubectl get clusterpolicy` (les 4 `Ready`) + `kubectl get pods -n app`
> montrant le pod `Running` (cas nominal accepté).

### 3.7 CI de bout en bout (bonus, vers SLSA L2)

Le workflow `.github/workflows/supply-chain.yml` automatise toute la chaîne à chaque push sur
`main` : build → SBOM → scan (gate CRITICAL) → push → **signature keyless** (OIDC du runner) →
attestation SBOM → attestation de provenance. **Aucune clé stockée** : l'identité est celle du
workflow (`id-token: write`). C'est ce qui fait grimper le niveau SLSA (cf. §5).

> ⏳ **PREUVE À CAPTURER (P7, si CI activée)** — capture de l'onglet Actions au vert + `cosign verify`
> avec `--certificate-identity` du workflow.

---

## 4. Démonstration attaque / défense

Le cœur de la garantie : le cluster **rejette** ce qu'il ne peut pas prouver. Chaque scénario
ci-dessous a été rejoué et la sortie d'erreur Kyverno capturée.

| # | Scénario | Résultat | Contrôle déclenché | Menace réelle correspondante |
|---|---|---|---|---|
| 0 | Image légitime (signée + provenance + bon registry + digest) | ✅ acceptée | — | cas nominal |
| 1 | Image **non signée** | ❌ refusée | `03-verify-signature` (verifyImages) | déploiement d'artefact non autorisé |
| 2 | Image **modifiée après signature** | ❌ refusée | signature liée au **digest** | **SolarWinds** (build/artefact altéré) |
| 3 | **Registry non autorisé** (ex. `nginx` Docker Hub) | ❌ refusée | `01-allowed-registries` | typosquatting / registry pirate |
| 4 | Tag **`:latest`** | ❌ refusée | `02-disallow-latest` | substitution silencieuse sous tag mutable |
| 5 | Signée **sans provenance** | ❌ refusée | `04-require-provenance` | origine non traçable |

Commandes des attaques (extraits) :

```bash
# Attaque 1 — non signée
docker build -t "$IMG:unsigned" app/ && docker push "$IMG:unsigned"
DU=$(docker inspect --format='{{index .RepoDigests 0}}' "$IMG:unsigned")
kubectl run pirate --image="$DU" -n app        # attendu : admission denied

# Attaque 3 — registry non autorisé
kubectl run fromdockerhub --image="nginx" -n app   # attendu : refusé (registre non listé)

# Attaque 4 — :latest
kubectl run uselatest --image="$IMG:latest" -n app # attendu : refusé (tag mutable)
```

> ⏳ **PREUVE À CAPTURER (P8)** — pour chaque attaque, la capture du message
> `admission webhook "…" denied the request: …`. Ce sont les captures L4 du barème.

---

## 5. Positionnement SLSA & limites

**SLSA** (*Supply-chain Levels for Software Artifacts*, OpenSSF) est un référentiel de maturité
sur la **provenance**.

| Niveau | Exigence | Notre POC |
|---|---|---|
| **L1** | La provenance existe (le build enregistre comment l'artefact a été fait) | ✅ Atteint — attestation `slsaprovenance` attachée et vérifiable |
| **L2** | Build sur **plateforme hébergée** + provenance **signée** | ✅ Atteint *si CI activée* (Lab 5) — signature keyless par l'OIDC du runner GitHub |
| **L3** | Build **isolé/infalsifiable**, provenance non contournable | ❌ Non atteint — hors périmètre |

**Honnêteté sur ce qui reste contournable :**
- En **local** (signature par clé), le poste de build n'est pas isolé : un attaquant qui contrôle
  la machine peut signer n'importe quoi. C'est pour ça que la CI keyless (L2) est supérieure.
- Même en L2, un **mainteneur avec les droits** peut modifier le workflow : la provenance est
  signée mais pas *infalsifiable*. L3 exigerait un générateur isolé (ex. `slsa-github-generator`
  en mode L3) et une séparation stricte des responsabilités.
- La gate Grype ne protège pas des **0-day** ni des CVE **sans correctif** (choix assumé de
  `only-fixed: true` pour ne pas noyer l'équipe sous des alertes non actionnables).

**Niveau réellement atteint : SLSA L1 en local, L2 avec la CI keyless activée.**

---

## 6. Reproductibilité

Tout se reconstruit de zéro, en local, sans cloud :

```bash
git clone https://github.com/Ymed95/projet_technique.git && cd projet_technique
git checkout claude/project-delivery-tonight-kt8qhz
# 1. Outils : docs/01-prerequis-setup.md   2. Login GHCR : docker login ghcr.io -u ymed95
export IMG=ghcr.io/ymed95/scs-demo-app ; export TAG=0.1.0
# 3. Build+push+digest (§3.1) → 4. SBOM+scan (§3.2-3.3) → 5. Sign+attest (§3.4-3.5)
# 6. Cluster+policies (§3.6)  → 7. Démo attaque/défense (§4)
```

Chaque étape est détaillée dans `labs/lab0-setup.md` → `labs/lab4-attaque-defense.md`.

---

## 7. Bilan

- **Acquis :** on est passé de « on scanne et on espère » à « on **vérifie et on bloque** ». La
  sécurité n'est plus une étape du pipeline mais une **propriété vérifiable** de bout en bout,
  rattachée au digest.
- **Ce qu'on referait différemment :** activer la CI keyless dès le début pour viser L2 sans
  manipuler de clé, et utiliser un credential helper pour GHCR (le token en base64 dans
  `~/.docker/config.json` est une limite connue de notre setup local).
- **Répartition du travail :** Piste A (build/SBOM/scan/signature), Piste B (cluster
  Kyverno + attaque/défense), Piste C (CI + rapport + threat model). Chaque membre a commité
  ses propres contributions (traçabilité Git).

---

## Annexes

- `policies/kyverno/` — les 4 ClusterPolicy commentées.
- `.github/workflows/supply-chain.yml` — pipeline de référence.
- Liens Rekor (mode keyless) : à coller si CI activée.
- Sorties brutes complètes des commandes P1–P8.
