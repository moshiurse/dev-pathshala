# 🔗 Supply Chain Security — সফটওয়্যার সাপ্লাই চেইন সিকিউরিটি

## 📖 ভূমিকা

Modern application code-এর প্রায় ৮০-৯০% থাকে third-party dependency (npm, Composer, Maven, PyPI)। আপনার কোড নিরাপদ হলেও, একটি compromised dependency পুরো production-কে danger-এ ফেলতে পারে। **Supply Chain Security** = কোড-এর source থেকে production deployment পর্যন্ত পুরো pipeline-এ tamper, malicious injection ও compromise রোধ।

```
   ┌─────────┐   ┌────────┐   ┌──────────┐   ┌───────┐   ┌──────────┐   ┌───────┐
   │Developer│──►│ VCS    │──►│CI Build  │──►│Artifact│──►│Registry │──►│Deploy │
   │ machine │   │(GitHub)│   │ pipeline │   │ store  │   │(npm/    │   │ Prod  │
   │         │   │        │   │          │   │        │   │ Composer)│  │       │
   └─────────┘   └────────┘   └──────────┘   └───────┘   └──────────┘   └───────┘
       ▲             ▲             ▲             ▲             ▲             ▲
   Compromised   Token leak   Malicious      Tampered    Typosquat,    Untrusted
   IDE plugin   PR injection   action       binary      malicious     image
                                            substitute   maintainer
```

প্রতিটি stage-এ আক্রমণ-vector আছে। SolarWinds, Codecov, Log4Shell, xz-utils backdoor — সবগুলো supply chain attack।

---

## 🎯 প্রধান হুমকি (Threats)

### 1. Dependency Confusion
Internal package (e.g., `@daraz/auth-utils`) public registry-এ same name-এ malicious version publish করে attacker। Developer-এর `npm install` public version pull করে যদি registry priority ভুল হয়। ২০২১-এ Alex Birsan এই কৌশলে Apple, Microsoft, PayPal এর internal system-এ ঢুকেছেন।

### 2. Typosquatting
`lodash` এর বদলে `lodahs`, `request` এর বদলে `requst` — typo করে install করালে malicious package। PyPI-তে `python-sqlite` (real `sqlite3`) crypto stealer; npm-এ `crossenv` (real `cross-env`) — credential exfiltration।

### 3. Malicious Package / Maintainer Compromise

**প্রসিদ্ধ ঘটনা:**
- **`event-stream` (npm, ২০১৮):** Maintainer ownership transfer; নতুন maintainer `flatmap-stream` সাব-dependency-এ Bitcoin wallet stealer যোগ করে। 8M+ download আগে ধরা পড়ে।
- **`ua-parser-js` (২০২১):** Maintainer-এর npm account hijack; তিনটি version-এ crypto miner + password stealer publish।
- **`node-ipc` (২০২২):** Maintainer protestware হিসেবে রাশিয়া/বেলারুশ IP-তে file destroy করে।
- **`xz-utils` (CVE-2024-3094):** ২+ বছর ধরে maintainer trust gain করে SSH backdoor inject — কাছাকাছি disaster।
- **PyPI `colorama` typosquat (২০২৪):** Browser/Telegram credentials চুরি।
- **Codecov bash uploader (২০২১):** CI script-এ environment variable exfiltration।

### 4. Compromised Build Tools
GitHub Action (`tj-actions/changed-files` ২০২৫-এ compromise হয়েছিল) — pipeline-এর secret token leak।

### 5. Lockfile Attacks
- `package-lock.json` integrity হ্যাশ tamper
- Indirect dependency upgrade (lockfile poisoning) — direct dep same রেখেও transitive malicious version।

### 6. Source Code Repository Attacks
- Stolen GitHub PAT/SSH key
- Force-push to default branch
- Malicious PR auto-merge
- Branch protection bypass

### 7. Container/Image Tampering
Public Docker Hub image-এ crypto miner; Alpine-এর fake mirror; `latest` tag swap।

---

## 🛡️ Defense — Layered Approach

### Layer 1: Source Code Integrity

#### Branch protection (GitHub)
```yaml
# Required reviews, signed commits, no force push
- Require pull request reviews (>=2)
- Require signed commits
- Require status checks (CI green)
- Restrict push to main
- Require linear history
- No bypass for admins
```

#### GPG/Sigstore signed commits
```bash
# Setup GPG signing
git config --global commit.gpgsign true
git config --global user.signingkey ABC123

# Or Sigstore gitsign (keyless)
brew install sigstore/tap/gitsign
git config --global gpg.format x509
git config --global gpg.x509.program gitsign
```

### Layer 2: Dependency Hygiene

#### Lockfiles ALWAYS committed
- `package-lock.json` (npm), `pnpm-lock.yaml`, `yarn.lock`, `composer.lock`, `Pipfile.lock`, `go.sum`
- `npm ci` (not `npm install`) in CI — fails if lockfile mismatch।

#### Integrity hashes
`package-lock.json` এ প্রতিটা package-এর SRI hash (`sha512-...`):
```json
{
  "name": "axios",
  "version": "1.6.0",
  "resolved": "https://registry.npmjs.org/axios/-/axios-1.6.0.tgz",
  "integrity": "sha512-..."
}
```
npm এই hash verify করে — tampered tarball reject।

#### Audit tools
```bash
# npm
npm audit --omit=dev
npm audit fix

# Composer
composer audit
composer audit --format=json | jq

# Yarn
yarn npm audit

# pnpm
pnpm audit --prod

# Python
pip install pip-audit
pip-audit

# Go
govulncheck ./...

# OSS-Index, Snyk, Socket.dev — third-party scanners
```

#### Dependabot / Renovate (auto PR for upgrades)

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule: { interval: "weekly" }
    open-pull-requests-limit: 10
    groups:
      minor-patch:
        update-types: ["minor", "patch"]
    ignore:
      - dependency-name: "*"
        update-types: ["version-update:semver-major"]  # manual review for major
  
  - package-ecosystem: "composer"
    directory: "/"
    schedule: { interval: "weekly" }

  - package-ecosystem: "docker"
    directory: "/"
    schedule: { interval: "weekly" }

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule: { interval: "weekly" }
```

#### Dependency review (GitHub native)
PR-এ যোগ হওয়া dep-এ vulnerability থাকলে block।

```yaml
# .github/workflows/dep-review.yml
name: Dependency Review
on: [pull_request]
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: high
          deny-licenses: GPL-3.0, AGPL-3.0
```

### Layer 3: Dependency Confusion Mitigation

#### Scoped packages
Internal package always `@daraz/...`, `@bkash/...` — scope-এ public registry-তে hijack ঠেকান (scope register করুন)।

#### `.npmrc` per scope
```
@daraz:registry=https://npm.daraz-internal.com
//npm.daraz-internal.com/:_authToken=${INTERNAL_TOKEN}
registry=https://registry.npmjs.org/
```

#### Composer
```json
{
  "repositories": [
    { "type": "composer", "url": "https://composer.bkash-internal.com" },
    { "type": "composer", "url": "https://packagist.org" }
  ],
  "config": {
    "secure-http": true
  }
}
```

#### Vendoring (extreme)
Critical dep code repo-এ commit। Trade-off: upgrade manual, কিন্তু supply chain pin।

---

## Layer 4: SBOM (Software Bill of Materials)

প্রতিটি build-এ machine-readable inventory যা list করে কোন version কোন dep-এ আছে।

### Standards
- **CycloneDX** (OWASP, JSON/XML, security-focused)
- **SPDX** (Linux Foundation, license-focused)

### Generate

```bash
# CycloneDX for Node.js
npm install -g @cyclonedx/cyclonedx-npm
cyclonedx-npm --output-file sbom.json

# CycloneDX for PHP
composer global require cyclonedx/cyclonedx-php-composer
composer CycloneDX:make-sbom --output-file=sbom.xml

# Syft (multi-language, container-aware)
syft packages dir:. -o cyclonedx-json > sbom.json
syft packages docker:daraz-app:1.0 -o spdx-json > sbom-image.json

# Grype to scan SBOM for CVEs
grype sbom:sbom.json
```

### SBOM-এ থাকে
- Component name, version, supplier
- Hash (SHA256, integrity)
- License
- Dependency relationships
- PURL (Package URL) — `pkg:npm/lodash@4.17.21`
- CPE — `cpe:2.3:a:lodash:lodash:4.17.21:*:*:*:*:*:*:*`

### Use cases
- CVE alert এলে `grep` করে দেখা কোন service affected
- Customer-কে compliance dispatch (US Executive Order 14028)
- Audit trail

---

## Layer 5: Build Provenance & Signing (Sigstore/cosign, SLSA)

### SLSA Framework (Supply-chain Levels for Software Artifacts)

| Level | Requirement |
|-------|-------------|
| **L0** | কোনো guarantee নেই |
| **L1** | Build process documented; provenance generate হয় |
| **L2** | Provenance authenticated (signed); hosted build platform |
| **L3** | Build platform tamper-resistant; non-falsifiable provenance |
| **L4** | Two-person review; hermetic builds; reproducible |

GitHub Actions + `slsa-github-generator` দিয়ে SLSA L3 achievable।

### Sigstore / cosign — keyless signing

```bash
# Sign artifact
cosign sign-blob --bundle app.tar.bundle app.tar.gz
# OIDC pop-up authenticates, signs, logs to Rekor (immutable transparency log)

# Verify
cosign verify-blob \
  --bundle app.tar.bundle \
  --certificate-identity "https://github.com/daraz/payment-service/.github/workflows/release.yml@refs/tags/v1.0.0" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  app.tar.gz
```

### Container image signing
```bash
cosign sign daraz/payment-service:v1.0.0
cosign verify daraz/payment-service:v1.0.0 --certificate-identity ...
```

Kubernetes admission controller (Kyverno, sigstore-policy-controller) verify না হলে pod না চালু।

### Build provenance attestation

```yaml
# GitHub Actions — generate SLSA provenance
jobs:
  build:
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - id: hash
        run: echo "hash=$(sha256sum dist/*.tgz | base64 -w0)" >> "$GITHUB_OUTPUT"

  provenance:
    needs: build
    permissions:
      actions: read
      id-token: write
      contents: write
    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v2.0.0
    with:
      base64-subjects: "${{ needs.build.outputs.hash }}"
```

---

## Layer 6: Publishing Security

### npm publish with 2FA
```bash
npm profile enable-2fa auth-and-writes
# Now publish requires OTP
```

### npm OIDC / Trusted Publishing (২০২৫+)
Token-less publish from CI:
```yaml
# .github/workflows/publish.yml
permissions:
  id-token: write
  contents: read
steps:
  - uses: actions/setup-node@v4
    with:
      registry-url: 'https://registry.npmjs.org'
  - run: npm publish --provenance --access public
```
`--provenance` flag npm-এ build provenance attach — npmjs.com-এ green badge।

### Composer (Packagist) signed commits
GitHub-এ tag GPG sign; Packagist tag-based release pull।

### Restrict who can publish
- npm package-এ "publishers" team scope
- Packagist "maintainers" allow-list
- Multiple maintainer (single point of failure থেকে রক্ষা)

---

## Layer 7: Runtime / Container

### Distroless / minimal images
```dockerfile
FROM node:20-alpine AS build
COPY . .
RUN npm ci && npm run build

FROM gcr.io/distroless/nodejs20-debian12
COPY --from=build /app/dist /app
USER nonroot
CMD ["/app/server.js"]
```

কম package = কম attack surface।

### Image scanning (Trivy, Grype, Snyk)

```yaml
- name: Trivy scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'daraz/payment:${{ github.sha }}'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'
```

### Pin base image by digest, not tag
```dockerfile
FROM node:20-alpine@sha256:abc123...   # immutable
# নয়
FROM node:20-alpine                     # mutable, can swap
```

### Pin GitHub Actions by SHA, not tag
```yaml
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11  # v4.1.1
# নয়
- uses: actions/checkout@v4   # tag, can be retagged
```

---

## 💻 PHP/Composer Hardened CI (Bangladesh — bKash example)

```yaml
# .github/workflows/ci.yml
name: Hardened CI
on: [pull_request, push]

permissions:
  contents: read
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Harden runner
        uses: step-security/harden-runner@v2
        with:
          egress-policy: block
          allowed-endpoints: >
            github.com:443
            packagist.org:443
            repo.packagist.org:443
            api.github.com:443

      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
        with:
          persist-credentials: false

      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'

      # Verify lockfile integrity
      - run: composer validate --strict
      - run: composer install --no-dev --prefer-dist --no-progress --no-interaction
      - run: composer audit

      # SBOM
      - run: composer global require cyclonedx/cyclonedx-php-composer
      - run: composer CycloneDX:make-sbom --output-file=sbom.xml

      # Static analysis
      - run: vendor/bin/psalm --taint-analysis
      - run: vendor/bin/phpstan analyse

      # Tests
      - run: vendor/bin/phpunit

      # Sign artifact
      - run: tar czf app.tar.gz app/ vendor/ public/
      - uses: sigstore/cosign-installer@v3
      - run: cosign sign-blob --yes --bundle app.bundle app.tar.gz

      - uses: actions/upload-artifact@v4
        with:
          name: build
          path: |
            app.tar.gz
            app.bundle
            sbom.xml
```

---

## 💻 Node.js Hardened CI (Daraz example)

```yaml
name: Daraz Build & Sign
on:
  push:
    tags: ['v*']

permissions:
  contents: read
  id-token: write
  attestations: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: step-security/harden-runner@v2
        with:
          egress-policy: audit

      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
        with:
          persist-credentials: false

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://registry.npmjs.org'
          cache: 'npm'

      # Strict reproducible install
      - run: npm ci --ignore-scripts            # 🔥 disable post-install scripts
      - run: npm audit signatures               # verify package signatures
      - run: npm audit --omit=dev --audit-level=high
      - run: npx better-npm-audit audit

      # SBOM
      - run: npx @cyclonedx/cyclonedx-npm --output-file sbom.json

      # Build
      - run: npm run build

      # Container build & sign
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/daraz/web:${{ github.ref_name }}
          provenance: true
          sbom: true

      - uses: actions/attest-build-provenance@v1
        with:
          subject-name: ghcr.io/daraz/web
          subject-digest: ${{ steps.build.outputs.digest }}
          push-to-registry: true

      # Publish to npm with provenance
      - run: npm publish --provenance --access restricted
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

---

## ✅ Operational Checklist (BD context)

**Daily:**
- [ ] `npm audit` / `composer audit` CI gate
- [ ] Dependabot PR review

**Weekly:**
- [ ] Dependency review meeting (security champion)
- [ ] SBOM diff vs last week
- [ ] Container image rescan

**On release:**
- [ ] SBOM generated and stored
- [ ] Artifact signed (cosign / npm provenance)
- [ ] Provenance attestation attached
- [ ] Release notes-এ CVE-fix changelog

**Quarterly:**
- [ ] Penetration test on CI/CD pipeline
- [ ] Token rotation (npm, Packagist, GitHub PAT)
- [ ] Access review (publish permissions)
- [ ] Incident response drill ("What if event-stream-2 happens?")

---

## 🇧🇩 Bangladesh — Real-world Practices

- **bKash:** Hardened CI with `step-security/harden-runner`, egress allowlist, signed artifacts; private Composer mirror (`composer.bkash.internal`)। Publish-এ 2FA + OIDC।
- **Daraz BD:** Internal npm registry (Verdaccio) + Snyk Enterprise scan; Trivy gate Kubernetes admission।
- **City Bank/EBL** (Internet banking): SBOM mandated by Bangladesh Bank cybersecurity guideline (BB CSG-2023); Sigstore-signed images।
- **Pathao:** Renovate bot every dep upgrade, manual review for tier-1 services (payment, ride matching)।
- **Tiger IT (govt vendor):** SBOM submission required for NID/election system contracts; SLSA L3 target।
- **Grameenphone:** Container image signing, Cosign verification at K8s admission।
- **Robi Axiata:** Internal artifact registry (JFrog Artifactory), proxy for npm/Composer; vulnerability gate।
- **Local startups (often weak):** plain `npm install` without lock-check, public Docker Hub `latest`, no audit। SME ও govt portal-এ supply chain risk বেশি।

---

## ⚠️ Common Pitfalls

1. **Lockfile commit না করা** → প্রতিবার fresh install, version drift, integrity check uselessly।
2. **`npm install` instead of `npm ci`** in CI — lockfile silently update।
3. **Post-install script run** by default — malicious package execute on install। `--ignore-scripts` consider।
4. **Tag-pinned actions/images** — supply chain attack surface।
5. **Public + Private mixed scope** — confusion attack possible।
6. **No 2FA on registry account** — single password leak = whole package family compromise।
7. **`latest` tag in production** — image swap risk।
8. **No SBOM generated** — incident-এ blast radius বের করা impossible।
9. **Long-lived registry tokens** — leak হলে damage long। Short-lived OIDC use করুন।
10. **Trust transitive dep blindly** — direct dep audit করছেন কিন্তু `npm ls` দিয়ে indirect দেখছেন না।
11. **Sole maintainer dep** — bus factor 1; project archive হলে patch নাই।
12. **No build reproducibility** — debug impossible "যা CI-এ ছিল তা reproduce করা যাচ্ছে না"।

---

## 📊 Tooling Quick Reference

| Need | Tool |
|------|------|
| SBOM generate | Syft, CycloneDX-npm, CycloneDX-php, sbom-tool |
| Vulnerability scan | Grype, Trivy, Snyk, OSV-Scanner, Dependabot |
| Container scan | Trivy, Clair, Anchore, Snyk Container |
| Signing | Sigstore cosign, GPG, Notary v2 |
| Provenance | SLSA generator, GitHub attestations |
| Registry | Verdaccio (npm), Satis/Packeton (Composer), Harbor, Artifactory |
| Auto-update | Dependabot, Renovate |
| Secret scan | gitleaks, TruffleHog, GitHub secret scanning |
| Static analysis | Semgrep, CodeQL, Psalm, PHPStan, ESLint |
| License compliance | FOSSA, Snyk License, license-checker |
| Admission | Kyverno, Gatekeeper, sigstore-policy-controller |
| Runner hardening | step-security/harden-runner |

---

## 📝 সারসংক্ষেপ

- **Supply chain attack = ৮০% modern code-এ third-party dep এর কারণে major risk।** event-stream, ua-parser-js, xz-utils, SolarWinds — কেস studies।
- **Threats:** dependency confusion, typosquat, malicious maintainer, lockfile poison, build tool compromise, container tamper।
- **Defense layered:**
  1. Source integrity (signed commits, branch protection)
  2. Lockfile + integrity hash + audit
  3. Scoped packages, internal registry (dependency confusion মুক্ত)
  4. SBOM (CycloneDX/SPDX) প্রতিটি build-এ
  5. Sigstore/cosign signing + SLSA provenance
  6. Publishing 2FA + OIDC trusted publishing
  7. Hardened CI runner (egress block), pinned actions/images by SHA
- **SLSA framework L0-L4** maturity level; L3 reasonable target।
- **Tooling:** Syft, Trivy, Grype, Dependabot, Renovate, cosign, harden-runner।
- **Bangladesh-এ** Bangladesh Bank CSG mandate, bKash/Daraz hardened pipeline; SME/govt-এ এখনো gap।
- **Operational:** SBOM diff, audit gate, token rotation, incident drill — সিনিয়র ইঞ্জিনিয়ারের responsibility।
