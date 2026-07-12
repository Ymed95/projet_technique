# Répartition du travail — ce soir, rendu 23h

Le dépôt de référence est déjà complet (app, 5 labs, policies Kyverno, pipeline CI, templates).
**On n'invente rien : on exécute, on personnalise (`Ymed95`), on prouve.** Les mêmes tickets
existent en Issues GitHub sur ce dépôt — cochez-les au fur et à mesure.

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

## Timeline suggérée (à adapter à l'heure qu'il vous reste)
| Bloc | Piste A | Piste B | Piste C |
|---|---|---|---|
| H+0 → H+1 | Lab 0 (build, push, digest) | Lab 3.1-3.2 (kind + Kyverno up, en parallèle, sans attendre le digest) | Active le workflow Lab 5, prépare threat model |
| H+1 → H+2 | Lab 1 (SBOM, scan, gate cassée) | Prépare/relit les 4 policies, attend `cosign.pub` + digest | Suit la CI, commence le rapport §1-2 |
| H+2 → H+3 | Lab 2 (signature + attestations) → livre digest + cosign.pub | Termine Lab 3 (déploiement accepté) | Bascule policy 03 en keyless, teste avec l'image CI |
| H+3 → H+4 | Aide Piste B/C, relit le rapport | Lab 4 (5 attaques + captures + vidéo) | Rédige rapport §3-5 avec les preuves qui arrivent |
| Dernière heure | Tout le monde | Tout le monde | Assemble rapport + threat model + checklist finale, commit, push |

**Règle d'or :** chaque membre **commit lui-même** ses preuves (la traçabilité Git est notée).
Ne travaillez pas dans un fichier partagé unique en fin de soirée — poussez au fil de l'eau sur
cette branche pour éviter un merge conflict à 22h55.
