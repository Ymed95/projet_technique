# Threat Model — Chaîne d'approvisionnement logicielle

- **Groupe :** Ymed, Djamel, Adelsino · **Date :** 2026-07-12
- **Périmètre :** POC `ghcr.io/ymed95/scs-demo-app` — chaîne build → registry → cluster (kind + Kyverno).

> Objectif : montrer qu'on **raisonne menaces → contrôles → couverture**, pas seulement
> « on a installé des outils ». Méthode : identification de l'actif, de la surface d'attaque,
> puis table de couverture et risque résiduel assumé.

---

## 1. Actif à protéger

**L'artefact (l'image conteneur) qui tourne en production doit être *exactement* celui produit
à partir du code revu, par notre chaîne, sans altération.**

Propriétés de sécurité visées :
- **Intégrité** — le contenu déployé = le contenu signé (aucune modification post-build).
- **Authenticité** — l'image vient bien de *nous* (identité vérifiable).
- **Traçabilité / provenance** — on sait *qui* a construit *quoi*, *depuis où*, *quand*.

Le principe directeur : **le cluster ne fait jamais confiance à un tag** ; il exige une preuve
cryptographique rattachée au **digest**. Si un octet change, le digest change, la signature ne
correspond plus, l'admission refuse.

---

## 2. Surface & acteurs de menace

```
[Dépendances amont] ─► [Build (poste dev / runner CI)] ─► [Registry GHCR] ─► [Cluster K8s]
        T3                    T1, T7                          T1, T6            T2, T5
```

- **Dépendances tierces (amont)** — bibliothèques compromises (ex. backdoor XZ Utils).
- **Étape de build / runner CI compromis** — injection dans le build (SolarWinds), vol de
  secrets (Codecov).
- **Registry compromis / substitution d'image** — remplacement du contenu sous un tag.
- **Accès cluster non autorisé** — déploiement direct d'une image pirate.
- **Développeur négligent** — tag `:latest`, image non signée, image externe non auditée.

---

## 3. Table menaces → contrôles → couverture

| # | Menace | Vecteur | Contrôle mis en place | Couverture | Risque résiduel |
|---|---|---|---|---|---|
| **T1** | Artefact **altéré après build** | substitution dans le registry | signature cosign liée au **digest** + Kyverno `verifyImages` (`03`) | **Forte** | le build lui-même (voir T7) |
| **T2** | **Déploiement non autorisé** dans le cluster | accès kubectl / RBAC faible | admission Kyverno en `Enforce` : signature requise | **Forte** | RBAC du cluster à durcir en parallèle |
| **T3** | **Dépendance vulnérable** | CVE dans une lib amont | SBOM (Syft) + gate Grype (`fail-on: critical`, `only-fixed`) | **Moyenne** | 0-day / CVE sans correctif (choix `only-fixed`) |
| **T4** | **Origine inconnue** de l'image | pas de traçabilité | attestation de **provenance** SLSA exigée (`04`) | **Forte** | provenance falsifiable si build non isolé (< L3) |
| **T5** | **Substitution silencieuse** | tag mutable `:latest` | interdiction `:latest` (`02`) + déploiement par digest | **Forte** | — |
| **T6** | **Registry pirate / typosquat** | image tirée d'une source externe | politique registres autorisés `ghcr.io/ymed95/` (`01`) | **Forte** | — |
| **T7** | **Build compromis** (runner/poste) | injection à la compilation | CI keyless hébergée (identité OIDC du workflow) → vers SLSA L2 | **Partielle** | mainteneur malveillant, build non isolé (L3) |

**Lecture :** chaque contrôle du cluster (les 4 policies Kyverno) répond à une menace précise, et
la combinaison SBOM+scan couvre l'amont. Le maillon le plus dur à couvrir reste **T7 (le build
lui-même)** — c'est précisément ce que SLSA L3 adresse et que notre POC n'atteint pas.

---

## 4. Ce qui reste hors périmètre / non couvert

- **Compromission du build lui-même** (viser SLSA L3 : build isolé, éphémère, non contournable).
- **Sécurité du poste développeur** et des **secrets en amont** (ex. notre token GHCR est stocké
  en base64 dans `~/.docker/config.json` — limite connue, à remplacer par un credential helper).
- **Vulnérabilités 0-day** ou sans correctif disponible (non bloquées par `only-fixed: true`,
  choix assumé pour garder la gate actionnable).
- **RBAC / sécurité réseau du cluster** — Kyverno protège l'admission des images, pas l'ensemble
  de la posture Kubernetes.

---

## 5. Niveau SLSA visé vs atteint

| | Visé | Atteint | Justification |
|---|---|---|---|
| Provenance existe (**L1**) | ✅ | ✅ | attestation `slsaprovenance` attachée et vérifiable (`cosign verify-attestation`) |
| Build hébergé + provenance signée (**L2**) | ✅ | ✅ **atteint** | pipeline CI vert, signature **keyless** par l'OIDC du runner GitHub, vérifiée avec l'identité exacte du workflow (preuve P7 du rapport, Rekor logIndex 2154146393) |
| Build isolé infalsifiable (**L3**) | — | ❌ | hors périmètre : exigerait un générateur isolé + séparation stricte des droits |

**Conclusion honnête :** le POC atteint **SLSA L1** en local (signature par clé, poste non isolé)
et **SLSA L2** via la CI keyless (activée et vérifiée, cf. rapport §3.7). Il reste contournable par quiconque contrôle le
poste de build (local) ou dispose des droits de modification du workflow (CI) — c'est la frontière
L2 → L3 que nous documentons en toute transparence.
