---
title: "Red Hat Project Experience"
layout: "single"
url: "/resume-en/"
summary: "Detailed Red Hat project experience"
ShowToc: false
draft: false
hideMeta: true
---


### Cluster Management

**[Cluster-Proxy](https://github.com/open-cluster-management-io/cluster-proxy) Core Owner** — [106 merged PRs](https://github.com/open-cluster-management-io/cluster-proxy/pulls?q=is%3Apr+is%3Amerged+author%3Axuezhaojun)

- Under OCM's Hub-Spoke architecture, enables users to directly access target Services on the Agent side from the Hub side via reverse tunnels (including KubeAPI Server), solving the core issue that the Hub cannot actively connect to managed clusters in Pull Mode
- Core ACM components such as Console, Application, and Observability all rely on this component as the underlying network layer for Hub→Spoke access

**[Import Controller](https://github.com/stolostron/managedcluster-import-controller) Core Developer** — [183 merged PRs](https://github.com/stolostron/managedcluster-import-controller/pulls?q=is%3Apr+is%3Amerged+author%3Axuezhaojun)

- A critical bridging component from open-source OCM to enterprise ACM: provides auto-import automatic onboarding and cross-cloud multi-vendor cluster integration capabilities (AWS/Azure/GCP/private cloud), making ACM viable for real production environments

**[OCM](https://github.com/open-cluster-management-io/ocm) Core Maintainer** — [58 merged PRs, 28 PR reviews](https://github.com/open-cluster-management-io/ocm/pulls?q=is%3Apr+author%3Axuezhaojun), covering Registration, Workload, Placement, Add-on Framework, SDK, and other core modules. Full-stack Go development, 5 years of CRD + Controller / Operator pattern practice.

- Registration module [Approver](https://github.com/open-cluster-management-io/ocm/blob/d770e1655234d34fd8df03ab5d297a34b5d42ce2/pkg/registration/OWNERS#L4): responsible for cluster registration and identity authentication (CSR issuance, automatic certificate rotation, Lease heartbeat monitoring), in charge of code review and quality assurance for this module
- Switch Hub: implements online migration of clusters between Hubs, supporting Global Hub (Hub of Hub) horizontal scaling, breaking through the single Hub cluster count limit

---

### AI + K8s Platform Capabilities

**[KubeOpenCode](https://github.com/kubeopencode/kubeopencode)** — K8s-native AI Agent platform, independently completed all work

- **K8s-native design**: Agent, Task, and CronTask are all CRDs, managing AI workload lifecycle and scheduling through the K8s API
- Architecture design + full-stack development + documentation site + internal/external promotion, 99% AI-assisted development
- Actively promoted by a Distinguished Engineer to be included as one of Red Hat's internal AI incubation projects

**[Repo-as-Agent](https://xuezhaojun.github.io/posts/repo-as-agent/) Methodology** — Independently proposed and implemented, Git repo = Agent entity (identity, skills, knowledge, workflows all under version control), 28 reusable skills, covering 20+ repos ([example repo](https://github.com/xuezhaojun/server-foundation-agent))

**Production Use Cases**:
- Tekton image change automation: integrated with [konflux-build-catalog](https://github.com/stolostron/konflux-build-catalog) ([workflow L247](https://github.com/stolostron/konflux-build-catalog/blob/8eb3352732e44d44ea9ec2923d50bd2731871a47/.github/workflows/update-tekton-task-bundles.yaml#L247)), detection → analysis → modification → PR → merge fully automated
- CronTask scheduled jobs: daily Scrum status auto-analysis, daily new bug auto-analysis, weekly bot PR auto-processing

---

### Initiative

**Agile Adoption**: Proactively obtained PSM certification and drove the team's agile development process adoption

**Efficiency Improvement**: Created the [konflux-build-catalog](https://github.com/stolostron/konflux-build-catalog) centralized solution, eliminating hundreds of duplicate Tekton update PRs across 60–70 repos every week, adopted by the entire org

**Cost Optimization**: Proactively reviewed the team's AWS test cluster configurations (storage type io1→gp3 reduction 96%, instance type m5→t3, layered by test scenario into HA/Lite clusters — HA for high-availability scenarios, Lite for regular tests), monthly cost $5,000 → $2,000, saving $36K annually

**Maintainability**: Led the merge of Registration/Work/Placement and other multi-repos into a [Mono Repo](https://github.com/open-cluster-management-io/ocm/issues/128#issuecomment-1552536628), unifying dependency management and CI/CD processes, reducing cross-repo maintenance overhead

**Documentation & Community**:
- Used AI technology to create a [Dashboard](https://github.com/open-cluster-management-io/lab/tree/main/dashboard) for the OCM upstream community, already adopted by external teams with plans to contribute new features back upstream
- Led the OCM community documentation site refactoring ([PR #429](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/pull/429), +1,856 / −11,626 lines) — migrated to Google Docsy theme (standard choice for K8s/Istio/gRPC and other CNCF projects), removed unmaintained Chinese docs (48/55 files had only Chinese titles), reduced file count by 46%, lowering the barrier for community participation

---

### Early Career

**Aimeigou** — Software Engineer (2019–2020) Cross-border e-commerce platform development, Shenzhen

**Gongji Technology** — Software Engineer (2017–2019) Data center infrastructure management software, Shenzhen
