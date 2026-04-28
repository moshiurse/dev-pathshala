# 📘 GitOps

> **GitOps — Git রিপোজিটরিকে infrastructure ও application deployment-এর "single source of truth" হিসেবে ব্যবহার করার পদ্ধতি।**

---

## 📖 সংজ্ঞা ও মূল ধারণা

GitOps হলো একটি **operational framework** যেখানে পুরো সিস্টেমের desired state Git রিপোজিটরিতে **declaratively** সংরক্ষিত থাকে। একটি automated agent (যেমন ArgoCD, Flux) ক্রমাগত Git-এর state আর cluster-এর actual state তুলনা করে এবং **drift** থাকলে স্বয়ংক্রিয়ভাবে **reconcile** করে।

### GitOps-এর ৪টি মূলনীতি

1. **Declarative Configuration**: পুরো সিস্টেম YAML/JSON ফাইলে ঘোষণামূলকভাবে সংজ্ঞায়িত
2. **Version Controlled**: সকল কনফিগারেশন Git-এ versioned এবং auditable
3. **Automated Delivery**: অনুমোদিত পরিবর্তন স্বয়ংক্রিয়ভাবে সিস্টেমে প্রয়োগ হয়
4. **Continuous Reconciliation**: Agent ক্রমাগত desired state ও actual state মিলিয়ে দেখে

### Push vs Pull Model

```
Traditional CI/CD (Push Model):
Developer ──► Git Push ──► CI Pipeline ──► kubectl apply ──► Cluster
                           CI-তে cluster credentials থাকে (নিরাপত্তা ঝুঁকি)

GitOps (Pull Model):
Developer ──► Git Push ──► Git Repo (desired state)
                               ▲
                               │ watch & sync
                               │
                          ┌────┴─────┐
                          │ ArgoCD / │ ──► Cluster (actual state)
                          │ Flux     │     reconcile করে
                          └──────────┘
                          Cluster-এর ভেতরে চলে (নিরাপদ)
```

### জনপ্রিয় GitOps টুলস

| টুল | ধরন | বৈশিষ্ট্য |
|-----|------|-----------|
| **ArgoCD** | Pull-based | Web UI, multi-cluster, SSO, RBAC |
| **Flux v2** | Pull-based | লাইটওয়েট, GitOps Toolkit, Kustomize-native |
| **Jenkins X** | Push+Pull | CI/CD + GitOps একত্রে |
| **Rancher Fleet** | Pull-based | বড় স্কেলে multi-cluster management |

---

## 🏠 বাস্তব জীবনের উদাহরণ

### 📋 রেস্তোরাঁর রেসিপি বই

একটি বড় রেস্তোরাঁ চেইন কল্পনা করুন:

- **রেসিপি বই = Git Repository**: প্রতিটি খাবারের রেসিপি (desired state) একটি কেন্দ্রীয় বইতে লেখা আছে
- **শেফ = GitOps Agent (ArgoCD)**: শেফ বইয়ের রেসিপি দেখে খাবার তৈরি করে (reconciliation)
- **খাবার = Running Application**: বাস্তবে তৈরি হওয়া খাবার (actual state)
- **রেসিপি পরিবর্তন = Git Commit**: নতুন রেসিপি বা পরিবর্তন বইতে লেখা হয়
- **Quality check = Diff detection**: শেফ নিয়মিত চেক করে যে খাবার রেসিপি অনুযায়ী হচ্ছে কিনা

কোনো শেফ নিজে নিজে রেসিপি বদলাতে পারে না — আগে বইতে লিখতে হবে, অনুমোদন নিতে হবে (PR review), তারপর শেফ সেই অনুযায়ী রান্না করবে।

---

## 🔧 ArgoCD কনফিগারেশন

### Application Manifest

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service
  namespace: argocd
spec:
  project: production
  source:
    repoURL: https://github.com/company/k8s-manifests.git
    targetRevision: main
    path: services/order-service/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true        # Git-এ নেই কিন্তু cluster-এ আছে — মুছে ফেলবে
      selfHeal: true      # কেউ ম্যানুয়ালি cluster বদলালে ফিরিয়ে আনবে
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Flux Kustomization

```yaml
# flux-kustomization.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: order-service
  namespace: flux-system
spec:
  interval: 5m
  path: ./services/order-service/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: k8s-manifests
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: order-service
      namespace: production
  timeout: 3m
```

---

## 🐘 PHP উদাহরণ — GitOps-compatible Deployment Manifests

```php
<?php
/**
 * PHP অ্যাপ্লিকেশনের জন্য Kubernetes manifest তৈরির স্ক্রিপ্ট।
 * এই স্ক্রিপ্ট CI pipeline-এ চলে এবং output GitOps repo-তে commit হয়।
 */

class KubernetesManifestGenerator
{
    private string $appName;
    private string $imageTag;
    private string $environment;

    public function __construct(string $appName, string $imageTag, string $environment)
    {
        $this->appName = $appName;
        $this->imageTag = $imageTag;
        $this->environment = $environment;
    }

    /**
     * Deployment manifest তৈরি করে।
     * এই manifest GitOps repo-তে push হবে এবং ArgoCD deploy করবে।
     */
    public function generateDeployment(): array
    {
        $replicas = match ($this->environment) {
            'production' => 3,
            'staging' => 2,
            default => 1,
        };

        return [
            'apiVersion' => 'apps/v1',
            'kind' => 'Deployment',
            'metadata' => [
                'name' => $this->appName,
                'namespace' => $this->environment,
                'labels' => [
                    'app' => $this->appName,
                    'managed-by' => 'gitops',
                ],
            ],
            'spec' => [
                'replicas' => $replicas,
                'selector' => ['matchLabels' => ['app' => $this->appName]],
                'template' => [
                    'metadata' => ['labels' => ['app' => $this->appName, 'version' => $this->imageTag]],
                    'spec' => [
                        'containers' => [[
                            'name' => $this->appName,
                            'image' => "registry.example.com/{$this->appName}:{$this->imageTag}",
                            'ports' => [['containerPort' => 8080]],
                            'resources' => [
                                'requests' => ['cpu' => '100m', 'memory' => '128Mi'],
                                'limits' => ['cpu' => '500m', 'memory' => '512Mi'],
                            ],
                            'livenessProbe' => [
                                'httpGet' => ['path' => '/health', 'port' => 8080],
                                'initialDelaySeconds' => 15,
                                'periodSeconds' => 10,
                            ],
                            'readinessProbe' => [
                                'httpGet' => ['path' => '/ready', 'port' => 8080],
                                'initialDelaySeconds' => 5,
                                'periodSeconds' => 5,
                            ],
                            'envFrom' => [
                                ['configMapRef' => ['name' => "{$this->appName}-config"]],
                                ['secretRef' => ['name' => "{$this->appName}-secrets"]],
                            ],
                        ]],
                    ],
                ],
            ],
        ];
    }

    /**
     * সম্পূর্ণ manifest সেট YAML ফাইলে লেখে।
     * GitOps repo-তে commit করার জন্য প্রস্তুত।
     */
    public function writeManifests(string $outputDir): void
    {
        if (!is_dir($outputDir)) {
            mkdir($outputDir, 0755, true);
        }

        $deployment = $this->generateDeployment();
        $yamlContent = $this->arrayToYaml($deployment);
        file_put_contents("{$outputDir}/deployment.yaml", $yamlContent);

        echo "✅ Manifests generated in: {$outputDir}\n";
        echo "📌 Image tag: {$this->imageTag}\n";
        echo "🌍 Environment: {$this->environment}\n";
    }

    private function arrayToYaml(array $data, int $indent = 0): string
    {
        // সরলীকৃত YAML generator
        return yaml_emit($data);
    }
}

// CI Pipeline-এ ব্যবহার:
// php generate-manifests.php order-service abc123f production
$generator = new KubernetesManifestGenerator(
    appName: $argv[1] ?? 'my-app',
    imageTag: $argv[2] ?? 'latest',
    environment: $argv[3] ?? 'development'
);
$generator->writeManifests("./manifests/{$argv[3]}/{$argv[1]}");
```

---

## 🟨 JavaScript উদাহরণ — GitOps Automation Script

```javascript
/**
 * GitOps workflow automation — CI pipeline থেকে চালানো হয়।
 * নতুন image build হলে GitOps repo-তে manifest আপডেট করে PR তৈরি করে।
 */

const { Octokit } = require('@octokit/rest');
const yaml = require('js-yaml');

class GitOpsUpdater {
  constructor(config) {
    this.octokit = new Octokit({ auth: config.githubToken });
    this.owner = config.manifestRepoOwner;
    this.repo = config.manifestRepoName;
    this.baseBranch = config.baseBranch || 'main';
  }

  /**
   * নতুন image tag দিয়ে deployment manifest আপডেট করে।
   * ArgoCD/Flux এই পরিবর্তন detect করে auto-deploy করবে।
   */
  async updateImageTag(serviceName, newTag, environment) {
    const manifestPath = `services/${serviceName}/overlays/${environment}/kustomization.yaml`;

    // বর্তমান manifest পড়া
    const { data: fileData } = await this.octokit.repos.getContent({
      owner: this.owner,
      repo: this.repo,
      path: manifestPath,
      ref: this.baseBranch,
    });

    const content = Buffer.from(fileData.content, 'base64').toString('utf8');
    const kustomization = yaml.load(content);

    // Image tag আপডেট করা
    if (!kustomization.images) {
      kustomization.images = [];
    }

    const imageEntry = kustomization.images.find(img => img.name === serviceName);
    if (imageEntry) {
      imageEntry.newTag = newTag;
    } else {
      kustomization.images.push({
        name: serviceName,
        newName: `registry.example.com/${serviceName}`,
        newTag: newTag,
      });
    }

    const updatedContent = yaml.dump(kustomization);

    // Production-এ PR তৈরি, staging-এ সরাসরি commit
    if (environment === 'production') {
      await this.createPullRequest(serviceName, newTag, manifestPath, updatedContent, fileData.sha);
    } else {
      await this.directCommit(manifestPath, updatedContent, fileData.sha, serviceName, newTag);
    }
  }

  async createPullRequest(serviceName, newTag, path, content, fileSha) {
    const branchName = `deploy/${serviceName}-${newTag}`;

    // নতুন branch তৈরি
    const { data: ref } = await this.octokit.git.getRef({
      owner: this.owner, repo: this.repo, ref: `heads/${this.baseBranch}`,
    });

    await this.octokit.git.createRef({
      owner: this.owner, repo: this.repo,
      ref: `refs/heads/${branchName}`,
      sha: ref.object.sha,
    });

    // Manifest আপডেট commit
    await this.octokit.repos.createOrUpdateFileContents({
      owner: this.owner, repo: this.repo, path,
      message: `chore: update ${serviceName} to ${newTag}`,
      content: Buffer.from(content).toString('base64'),
      sha: fileSha,
      branch: branchName,
    });

    // Pull Request তৈরি
    const { data: pr } = await this.octokit.pulls.create({
      owner: this.owner, repo: this.repo,
      title: `🚀 Deploy ${serviceName}:${newTag} to production`,
      body: `## Automated Deployment PR\n\n- **Service**: ${serviceName}\n- **Tag**: ${newTag}\n- **Environment**: production\n\nArgoCD will auto-sync after merge.`,
      head: branchName,
      base: this.baseBranch,
    });

    console.log(`✅ PR তৈরি হয়েছে: ${pr.html_url}`);
    return pr;
  }

  async directCommit(path, content, fileSha, serviceName, newTag) {
    await this.octokit.repos.createOrUpdateFileContents({
      owner: this.owner, repo: this.repo, path,
      message: `chore: auto-deploy ${serviceName}:${newTag} to staging`,
      content: Buffer.from(content).toString('base64'),
      sha: fileSha,
      branch: this.baseBranch,
    });

    console.log(`✅ Staging-এ সরাসরি deploy হচ্ছে: ${serviceName}:${newTag}`);
  }
}

// CI Pipeline থেকে ব্যবহার
async function main() {
  const updater = new GitOpsUpdater({
    githubToken: process.env.GITHUB_TOKEN,
    manifestRepoOwner: 'company',
    manifestRepoName: 'k8s-manifests',
  });

  const serviceName = process.env.SERVICE_NAME || 'order-service';
  const imageTag = process.env.IMAGE_TAG || 'latest';
  const environment = process.env.DEPLOY_ENV || 'staging';

  await updater.updateImageTag(serviceName, imageTag, environment);
}

main().catch(err => {
  console.error('❌ GitOps update ব্যর্থ:', err.message);
  process.exit(1);
});
```

---

## 📊 GitOps Workflow ডায়াগ্রাম

```
সম্পূর্ণ GitOps Workflow:

┌─────────────┐     ┌─────────────┐     ┌──────────────────┐
│  Developer  │────►│  App Repo   │────►│  CI Pipeline     │
│  Code Push  │     │  (source)   │     │  Build & Test    │
└─────────────┘     └─────────────┘     └────────┬─────────┘
                                                  │
                                          Docker Image Push
                                                  │
                                                  ▼
                                        ┌──────────────────┐
                                        │ Container        │
                                        │ Registry         │
                                        └────────┬─────────┘
                                                  │
                                        Update image tag
                                                  │
                                                  ▼
┌──────────────────┐     ┌──────────────────────────────────┐
│  K8s Cluster     │◄────│  GitOps Repo (manifests)         │
│  (actual state)  │sync │  Deployment, Service, ConfigMap  │
│                  │     │  = single source of truth        │
└────────┬─────────┘     └──────────────────────────────────┘
         │                           ▲
         │ watch & reconcile         │ watch
         │                           │
         └───────┐            ┌──────┘
                 ▼            │
            ┌────────────────────┐
            │   ArgoCD / Flux    │
            │   (GitOps Agent)   │
            └────────────────────┘
```

---

## ✅ কখন ব্যবহার করবেন

| পরিস্থিতি | কারণ |
|-----------|------|
| **Kubernetes-ভিত্তিক ডেপ্লয়মেন্ট** | GitOps Kubernetes-এর declarative nature-এর সাথে পুরোপুরি মানানসই |
| **Audit trail দরকার** | প্রতিটি পরিবর্তন Git history-তে থাকে — কে, কখন, কী বদলেছে |
| **Multi-environment management** | dev, staging, production — সব কনফিগারেশন একই repo-তে |
| **Disaster recovery** দ্রুত করতে চান | Git repo থেকে পুরো cluster পুনরুদ্ধার সম্ভব |
| **Compliance ও security requirement** | PR review-এর মাধ্যমে infrastructure পরিবর্তন অনুমোদন |
| **টিম বড় এবং coordination দরকার** | সবাই একই Git workflow ফলো করে |

## ❌ কখন ব্যবহার করবেন না

| পরিস্থিতি | কারণ |
|-----------|------|
| **Non-Kubernetes পরিবেশ** | GitOps টুলস মূলত Kubernetes-কেন্দ্রিক |
| **খুব ছোট প্রজেক্ট** | অপ্রয়োজনীয় overhead — সরাসরি deploy করাই ভালো |
| **Stateful infrastructure** পরিচালনা | Database schema migration GitOps-এ জটিল |
| **Real-time পরিবর্তন দরকার** | Git commit → sync cycle-এ কিছুটা delay থাকে |
| **টিমের Git expertise কম** | Branching, merge conflict সমাধান জানা দরকার |

---

## 🔑 মূল শিক্ষা

1. **Git হলো single source of truth** — cluster-এ সরাসরি `kubectl apply` করবেন না
2. **App repo আর manifest repo আলাদা রাখুন** — separation of concerns
3. **Production-এ PR workflow বাধ্যতামূলক করুন** — staging-এ auto-sync চলতে পারে
4. **Sealed Secrets বা External Secrets Operator ব্যবহার করুন** — plain-text secret Git-এ রাখবেন না
5. **Drift detection alert সেট করুন** — কেউ ম্যানুয়ালি cluster বদলালে জানতে পারবেন
