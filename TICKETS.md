# Plan de projet & répartition du travail

Le dépôt de référence fournit l'application, les 5 labs, les politiques Kyverno, le pipeline CI
et les templates. Notre travail : **exécuter la chaîne, la personnaliser (`Ymed95`) et la prouver.**
Les mêmes tâches existent en Issues GitHub — cochez-les au fur et à mesure.

## Répartition (3 membres)

| Membre | Poste | Piste |
|---|---|---|
| Ymed | Windows/WSL2 | **A — build, SBOM, scan, signature, attestations** (#2) |
| Adelsino | Linux | **B — cluster Kyverno + démonstration attaque/défense** (#3) |
| Djamel | Windows/WSL2 | **C — CI de bout en bout + rapport + threat model** (#4) |

## Décisions d'équipe (stables)

- Identité GHCR unique : `ghcr.io/ymed95/scs-demo-app` (minuscules — GHCR l'exige).
- Les politiques `policies/kyverno/*.yaml` et `k8s/deployment.yaml` sont personnalisées `Ymed95` ;
  la clé publique cosign y est déjà intégrée.
- Un seul PAT GitHub `write:packages` suffit pour les push GHCR manuels (Labs 0-2). Jamais commité.
- Le pipeline CI (Lab 5) ne nécessite aucun secret : OIDC + `GITHUB_TOKEN`.
- On travaille **par digest**, jamais par tag mutable — c'est le cœur de la garantie d'intégrité.

---

## Mise en place de l'environnement

Les trois postes utilisent WSL2 Ubuntu (vrai Linux) → mêmes commandes pour tout le monde.

### Cloner le dépôt
```bash
git clone https://github.com/Ymed95/projet_technique.git
cd projet_technique
```

### Rendre Docker joignable dans WSL
- Docker Desktop → Settings → Resources → **WSL Integration** → activer pour la distro Ubuntu
  (`docker version` doit répondre dans le terminal Ubuntu), **ou**
- `sudo apt-get install -y docker.io && sudo service docker start && sudo usermod -aG docker $USER`.

### Installer les outils
```bash
# kind + kubectl + jq
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
sudo apt-get install -y jq
# syft / grype / cosign / buildx
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh  | sudo sh -s -- -b /usr/local/bin
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sudo sh -s -- -b /usr/local/bin
curl -sSfLo cosign https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
chmod +x cosign && sudo mv cosign /usr/local/bin/
mkdir -p ~/.docker/cli-plugins
BUILDX_VER=$(curl -fsSL https://api.github.com/repos/docker/buildx/releases/latest | jq -r .tag_name)
curl -fSL "https://github.com/docker/buildx/releases/download/${BUILDX_VER}/buildx-${BUILDX_VER}.linux-amd64" -o ~/.docker/cli-plugins/docker-buildx
chmod +x ~/.docker/cli-plugins/docker-buildx
```

### Vérifier + se connecter à GHCR
```bash
docker version && kind version && kubectl version --client && syft version && grype version && cosign version && jq --version
docker login ghcr.io -u Ymed95     # coller le PAT write:packages au prompt Password
```

---

## Piste A (Ymed) — Build & Preuves (Labs 0 → 1 → 2)
**Objectif :** une image signée, poussée sur GHCR, avec SBOM + scan + 2 attestations vérifiables.

- [x] Lab 0 : image buildée, poussée sur `ghcr.io/ymed95/scs-demo-app`, digest récupéré
- [x] Lab 1 : SBOM Syft (spdx-json), scan Grype avec `.grype.yaml`
- [x] Lab 1 : gate déclenchée sur CVE critiques corrigeables (preuve capturée)
- [x] Lab 2 : signature par clé + `cosign verify` réussi (`cosign.key` bien ignoré par git)
- [x] Lab 2 : attestation SBOM attachée + vérifiée
- [x] Lab 2 : attestation provenance attachée + vérifiée
- [x] `cosign tree` montre signature + 2 attestations
- [x] Digest + clé publique livrés à la Piste B (clé intégrée aux policies)
- [x] Sorties de commandes fournies à la Piste C (dossier `preuves/`, rapport §3)
- [ ] (Optionnel) Signature keyless (identité OIDC) pour enrichir le rapport

## Piste B (Adelsino) — Cluster qui refuse & attaque/défense (Labs 3 → 4)
**Objectif :** cluster kind + Kyverno en `Enforce`, image légitime acceptée, attaques bloquées et capturées.

- [ ] Lab 3 : `kind create cluster --config cluster/kind-config.yaml`, Kyverno `Ready`
- [ ] Lab 3 : namespace `app`, 4 `ClusterPolicy` appliquées (clé cosign déjà intégrée — `git pull`)
- [ ] Lab 3 : `k8s/deployment.yaml` avec le digest signé → `kubectl apply` → pod **Running**
- [ ] Lab 4 — Attaque 1 : image non signée → refusée + capture
- [ ] Lab 4 — Attaque 2 : image modifiée après signature (scénario SolarWinds) → refusée + capture
- [ ] Lab 4 — Attaque 3 : registry non autorisé → refusée + capture
- [ ] Lab 4 — Attaque 4 : tag `:latest` → refusée + capture
- [ ] Lab 4 — Attaque 5 (bonus) : signée sans provenance → refusée + capture
- [ ] Tableau de synthèse attaque → contrôle → menace rempli
- [ ] Captures livrées à la Piste C pour le rapport

## Piste C (Djamel) — CI, Rapport, Threat Model (Lab 5 + livrables)
**Objectif :** pipeline CI (bonus SLSA L2), rapport + threat model complets, dépôt propre.

- [ ] Lab 5 : activer le workflow `.github/workflows/supply-chain.yml`
- [ ] Lab 5 : vérifier build + SBOM + scan + push + sign (keyless) + attestations
- [ ] Lab 5 : variante **keyless** de `03-verify-signature.yaml` (bloc préparé) si CI verte
- [x] Rapport rédigé (`livrables/rapport.md`) — preuves P1-P5 intégrées
- [x] Threat model rédigé (`livrables/threat-model.md`)
- [ ] Renseigner les noms du groupe en tête des deux documents
- [ ] Intégrer les captures de la Piste B (section 4 du rapport)
- [ ] Repasser la checklist d'auto-évaluation de la consigne (section 8)

---

## Ordre de réalisation & dépendances

La signature **par clé** est faite en premier (déterministe, hors-ligne) ; keyless et CI sont des
compléments qui appuient la discussion SLSA L2.

**Dépendance principale :** la Piste A produit le **digest signé** + la **clé publique** (intégrée
aux policies). Sur cette base, la Piste B teste l'acceptation puis les attaques signature. Les
attaques *registry* et *:latest* (Piste B) ne dépendent pas de la signature et peuvent se faire en
parallèle. La Piste C (CI + rédaction) est largement indépendante et intègre les preuves des deux
autres au fil de l'eau.

**Règle d'équipe :** chaque membre **commit lui-même** ses contributions (la traçabilité Git fait
partie du sujet). On pousse au fil de l'eau sur la branche de travail pour éviter les conflits.
