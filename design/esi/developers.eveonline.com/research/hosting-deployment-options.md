---
type: Research
title: "AWS Hosting & Deployment Options for the Private Astro Site"
description: "Decision-support survey of viable AWS deploy targets for the host-agnostic Astro Node-SSR container: what runs the artifact, how deploys are triggered, tuning knobs, extensibility for the future public-repo rebuild trigger, and deploy latency. Feeds open decisions D1 and D2."
tags: [design, esi, developer-portal, astro, ssr, cdn, architecture, decisions, ci, hosting, aws, pricing, cost]
---

# AWS Hosting & Deployment Options for the Private Astro Site

**Status:** Research / decision-support. This note does **not** pick a host. It feeds
[D1 — Hosting target and D2 — Rebuild trigger](../11-open-decisions.md).

**Accessed:** all AWS and Astro sources below were fetched **2026-07-21**. AWS docs move; re-check
before committing. Where a fact is *not* pinned to a first-party page, it is flagged inline and in
[§8](#8-claims-not-fully-pinned-to-a-primary-source).

## 0. What we are deploying (recap of the given constraints)

The artifact is fixed by the design ([03 — Rendering & Hosting](../03-rendering-and-hosting.md)): a
**host-agnostic container image** running Astro's **Node adapter** (`@astrojs/node`) in
**standalone mode**, serving a **hybrid** app (mostly prerendered static, plus a small set of
server-rendered routes: OAuth callback/session, authed proxy routes), with a **CDN** in front of
static assets. A **server runtime is mandatory** — static-only hosting is out
([03.2](../03-rendering-and-hosting.md#32-consequence-a-server-runtime-is-mandatory)). AWS family
only; Cloudflare/Vercel/Netlify are out.

`@astrojs/node` standalone mode is confirmed first-party: it "creates a self-contained server" that
starts when the entrypoint runs, and you can "override the host and port the standalone server runs
on by passing them as environment variables at runtime" (`HOST` and `PORT`). Standalone mode serves
both routing and static assets. Source: [Astro docs — @astrojs/node](https://docs.astro.build/en/guides/integrations-guide/node/).

This is the key discriminator below: **which targets accept a plain container running that Node
server, vs. which require a different packaging (a framework-specific adapter or a Lambda wrapper).**

We evaluate each option on the stakeholder's three axes — **(1) deploys**, **(2) tuneable knobs**,
**(3) extensible to the D2 rebuild trigger** — plus **deploy latency** and **ops ownership**.

---

## 1. AWS Amplify Hosting (SSR / compute)

**Does it run our artifact? — No, not as-is. This is the most important caveat in this note.**

Amplify Hosting does **not** accept our `@astrojs/node` container image. Amplify SSR runs on a
**managed compute layer, not a container you supply**. Its model is: "any Javascript based SSR
framework with an open source build adapter that transforms an application's build output into the
directory structure that Amplify Hosting expects"
([AWS — Deploying SSR applications with Amplify Hosting](https://docs.aws.amazon.com/amplify/latest/userguide/server-side-rendering-amplify.html)).
You connect a Git repo; Amplify builds it; the adapter's output is deployed onto Amplify's compute.

For Astro specifically, AWS is explicit: "You can deploy an Astro.js application to Amplify using a
**community adapter. We do not maintain an Amplify owned adapter for the Astro framework**" — the
adapter is `alexnguyennz/astro-aws-amplify`, "created by a member of the community and is not
maintained by AWS" ([AWS — Amplify support for Astro.js](https://docs.aws.amazon.com/amplify/latest/userguide/astro-support.html)).
Astro's own deploy guide agrees: SSR on Amplify "will need to use the third-party,
community-maintained AWS Amplify adapter" and set `output: "server"`
([Astro docs — Deploy to AWS](https://docs.astro.build/en/guides/deploy/aws/)). Per Astro's guide,
that adapter "takes your built site and wraps it in an AWS Lambda function that Amplify can deploy
and serve" — i.e. it is a **Lambda-backed** deployment, **not** our standalone Node-server
container, and **not** the standard `@astrojs/node` adapter.

**Implication for the host-agnostic-container premise:** choosing Amplify means abandoning the
one-artifact promise for this target — you swap `@astrojs/node` for the community Amplify adapter and
accept Lambda semantics (and a community, non-AWS dependency). That is a real deviation from
[03.3](../03-rendering-and-hosting.md#33-host-agnostic-deployment-artifact) and should be weighed as
such under D1.

**Deploy/trigger model.** Git-connected auto-build: connect the branch and Amplify "automatically
deploys when you push commits"
([Astro docs — Deploy to AWS](https://docs.astro.build/en/guides/deploy/aws/); AWS SSR-deploy steps
in [server-side-rendering-amplify](https://docs.aws.amazon.com/amplify/latest/userguide/server-side-rendering-amplify.html)).
Plus a first-party **incoming webhook / build hook**: in *Hosting → Build settings → Incoming
webhooks* you "Create webhook", pick the branch to build, and get a URL you can `curl` or hand to a
CMS/Zapier to "trigger deployments in the Amplify Console without requiring a code commit"
([AWS — Creating an incoming webhook to start a build](https://docs.aws.amazon.com/amplify/latest/userguide/create-incoming-webhook.html)).

**Configurability knobs.** Environment variables (Amplify console), branch-based deploys,
build settings (`amplify.yml`), custom domains, and a managed CloudFront distribution + rewrites
that Amplify provisions for SSR apps
([SSR supported features](https://docs.aws.amazon.com/amplify/latest/userguide/ssr-supported-features.html)).
Node runtime is pinned to the build's major version; supported runtimes are Node 20/22/24
(*ibid.*). CloudWatch Logs for the SSR runtime must be explicitly enabled for adapter-based (non-Next)
apps ([server-side-rendering-amplify](https://docs.aws.amazon.com/amplify/latest/userguide/server-side-rendering-amplify.html)).
Fine-grained compute knobs (memory/CPU/concurrency) are **not** clearly documented for
arbitrary-adapter SSR — see [§8](#8-claims-not-fully-pinned-to-a-primary-source).

**Extensibility for D2.** Strong and native: the incoming-webhook URL is exactly a "place to POST to
trigger a rebuild." The public repo's *settings* (not tracked tree) can point a webhook at that URL,
or a minimal `repository_dispatch` workflow can `curl` it — satisfying D2's "public repo stays
hosting-agnostic." (First-party mechanism: [create-incoming-webhook](https://docs.aws.amazon.com/amplify/latest/userguide/create-incoming-webhook.html).)

**Deploy latency.** Git-push → build → deploy; latency ≈ full Astro build time + Amplify deploy
step. No first-party per-build number. SSR is Lambda-backed, so **cold starts** apply to
server routes. (Not pinned — [§8](#8-claims-not-fully-pinned-to-a-primary-source).)

**Ops ownership.** Lowest — fully managed build + host + CDN. Trade-off: least control, a
community/non-AWS Astro adapter, and divergence from the container artifact.

---

## 2. AWS App Runner (container image / ECR)

**Availability blocker — read first.** "**AWS App Runner is no longer open to new customers.**
Existing customers can continue to use the service as normal, including creating new resources and
services." AWS adds: "we do not plan to introduce new features," and explicitly recommends **Amazon
ECS Express Mode** for anyone migrating from / would-be-adopting App Runner
([AWS — App Runner availability change](https://docs.aws.amazon.com/apprunner/latest/dg/apprunner-availability-change.html)).
**Practical consequence:** unless CCP's AWS account is already an existing App Runner customer, this
option is likely **not adoptable for a new service** — verify account eligibility before it goes on
the D1 shortlist. It is documented here because it is the closest "managed container" fit and its
successor (ECS Express Mode, [§3a](#3a-amazon-ecs-express-mode-app-runners-successor)) inherits its ergonomics.

**Does it run our artifact? — Yes (if eligible).** App Runner takes "a ready-to-use container image
that App Runner can deploy" from **Amazon ECR** (or builds from a GitHub/Bitbucket source repo)
([App Runner architecture & concepts](https://docs.aws.amazon.com/apprunner/latest/dg/architecture.html)).
Our standalone Node container on a port is exactly the image-source case.

**Deploy/trigger model.** Two methods
([Deploying a new application version](https://docs.aws.amazon.com/apprunner/latest/dg/manage-deploy.html)):
- **Automatic** (`AutoDeploymentsEnabled = true`): "Whenever you push a new image version to your
  image repository … App Runner automatically deploys it." This is the "push image to ECR → it
  deploys" model. Caveat: no auto-deploy for ECR **Public** images or cross-account ECR repos
  (*ibid.*).
- **Manual**: `StartDeployment` API/CLI or console button (*ibid.*).

**Configurability knobs.** vCPU/memory from 0.25 vCPU/0.5 GB up to 4 vCPU/12 GB; env vars;
custom domains (note `*.awsapprunner.com` is on the Public Suffix List); VPC connectors; observability
config ([architecture](https://docs.aws.amazon.com/apprunner/latest/dg/architecture.html)).
Auto-scaling knobs are **Max concurrency** (requests per instance before scale-up), **Min size**
(always-provisioned instances — the reserve that mitigates cold start; you pay memory on all, CPU on
active), and **Max size**
([Managing auto scaling](https://docs.aws.amazon.com/apprunner/latest/dg/manage-autoscaling.html)).
App Runner terminates TLS and provides an HTTPS endpoint itself; a separate CloudFront layer is
optional for our static-asset CDN.

**Extensibility for D2.** Good: the trigger is "push a new image to ECR." A public-repo webhook or
`repository_dispatch` drives a GitHub Action that builds + pushes the image; App Runner auto-deploys
on the ECR push. The public repo learns nothing about hosting. Alternatively call `StartDeployment`.

**Deploy latency.** Auto-deploy fires on ECR push; migration/maintenance uses blue-green traffic
shifting ([architecture](https://docs.aws.amazon.com/apprunner/latest/dg/architecture.html)). No
first-party per-deploy or cold-start number ([§8](#8-claims-not-fully-pinned-to-a-primary-source));
**Min size ≥ 1** keeps warm instances to blunt cold starts.

**Ops ownership.** Low (managed), but constrained by the availability blocker above.

---

## 3. Amazon ECS on Fargate

**Does it run our artifact? — Yes, cleanly.** ECS runs standard container images as tasks; Fargate
is the serverless launch type (no EC2 to manage). Our Node-server image runs unchanged. ECS resolves
image tags to digests and keeps all tasks on identical images for "version consistency"
([ECS — Deploy by replacing tasks (rolling update)](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-ecs.html)).

**Deploy/trigger model.** ECS itself does not watch a Git repo; deployment is driven by
`UpdateService` / a new task-definition revision / `forceNewDeployment`
([deployment-type-ecs](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-ecs.html)).
Concrete trigger mechanisms (all first-party):
- **CI pushes an image to ECR, then updates the service** — e.g. a GitHub Action (build → push to
  ECR → deploy). AWS documents exactly this pattern for ECS Express Mode with the
  `aws-actions/amazon-ecs-deploy-express-service` action
  ([App Runner→ECS Express migration, source-based section](https://docs.aws.amazon.com/apprunner/latest/dg/apprunner-availability-change.html)).
- **CodePipeline + CodeDeploy blue/green**, triggered by an ECR source action: "quickly configure a
  continuous delivery pipeline that will automatically trigger a blue/green deployment when you
  upload a new image to Amazon ECR" — with rolling, canary, and linear traffic shifting
  ([AWS — CodeDeploy blue/green for ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-bluegreen.html);
  [AWS blog — canary/linear for ECS](https://aws.amazon.com/blogs/containers/aws-codedeploy-now-supports-linear-and-canary-deployments-for-amazon-ecs)).

**Configurability knobs.** Task CPU/memory, env vars, task count, ALB health checks, custom domain
via ALB + ACM + Route 53, autoscaling policies, VPC/networking. Rolling-update knobs are
`minimumHealthyPercent` / `maximumPercent`; failure detection via deployment circuit breaker or
CloudWatch alarms, both supporting rollback to the previous revision
([deployment-type-ecs](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-ecs.html)).
CloudFront sits in front for static-asset CDN.

**Extensibility for D2.** Strong. Public-repo webhook / `repository_dispatch` → GitHub Action builds
and pushes image + `UpdateService` (or lets CodePipeline's ECR source action fire). Public repo stays
hosting-agnostic; all hosting logic lives in the private pipeline.

**Deploy latency.** Rolling replacement of tasks governed by min/max-healthy percentages;
blue/green shifts traffic gradually. No universal first-party per-deploy number
([§8](#8-claims-not-fully-pinned-to-a-primary-source)); with warm tasks there is no cold start once
running. (ECS Express Mode reports ~3–5 min for *initial* stack provisioning — see [§3a](#3a-amazon-ecs-express-mode-app-runners-successor).)

**Ops ownership.** Medium — you own the service/task defs, ALB, pipeline. More moving parts than
Amplify/App Runner, but standard CCP-friendly primitives and full control.

### 3a. Amazon ECS Express Mode (App Runner's successor)

AWS's recommended replacement for App Runner, and a genuinely viable option worth surfacing on its
own. "With a single API call, you provide a container image and two IAM roles, and Amazon ECS
provisions a complete application stack in your AWS account, including an ECS service on Fargate, an
Application Load Balancer, auto scaling, and networking. There is no additional charge for using
Amazon ECS Express Mode"
([App Runner availability change](https://docs.aws.amazon.com/apprunner/latest/dg/apprunner-availability-change.html)).
It **requires a container image** (`create-express-gateway-service` takes image, port, env vars,
`--health-check-path`, `--scaling-target minTaskCount/maxTaskCount`), and **initial provisioning
"typically takes 3–5 minutes"** (*ibid.* — a first-party latency datapoint, for provisioning not
per-deploy). AWS documents a GitHub Actions flow (build → push ECR → deploy) that "replicates App
Runner's automatic deployment on code push," which maps directly onto D2 (*ibid.*). This gives
App-Runner-like simplicity with our container artifact and without the closed-to-new-customers
blocker.

---

## 4. Amazon EKS (our own Kubernetes cluster)

**Does it run our artifact? — Yes.** EKS is "certified Kubernetes-conformant, so you can deploy
Kubernetes-compatible applications without refactoring." Our container runs as a Deployment/Pod
behind a Service/Ingress. AWS manages the control plane (EKS standard); you own the workloads (and,
outside Auto Mode, the nodes) under a shared-responsibility model
([AWS — What is Amazon EKS?](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)).

**Deploy/trigger model.** No built-in Git-connected build. Deploy via `kubectl`/Helm rollouts or,
first-party, **GitOps with Argo CD**, which EKS offers as a managed Capability providing
"declarative, GitOps-based continuous deployment"
([what-is-eks](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)). Trigger = push
new image to ECR and update the manifest/tag (CI or Argo image updater).

**Configurability knobs.** The full Kubernetes surface — replicas/HPA, resource requests/limits,
probes, ConfigMaps/Secrets (env), Ingress/ALB, node types (EC2 incl. Graviton) or Fargate profiles.
Most knobs of any option; also the most to own ([what-is-eks](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)).

**Extensibility for D2.** Strong but indirect: public-repo webhook / `repository_dispatch` → CI
pushes image + bumps manifest (or Argo CD auto-syncs). Public repo stays hosting-agnostic.

**Deploy latency.** K8s rolling update; warm pods, no cold start once running. No first-party
per-deploy number ([§8](#8-claims-not-fully-pinned-to-a-primary-source)).

**Ops ownership.** Highest. AWS manages the control plane; you own cluster ops, upgrades, add-ons,
node lifecycle (unless Auto Mode). Justified only if CCP already runs EKS and the portal folds into
existing cluster practice (relevant to D1's "parity with existing deployment practice" criterion).

---

## 5. AWS Lambda (container image / Lambda Web Adapter) — brief

**Does it run our artifact? — Technically yes, with an adapter shim; caveats.** Lambda accepts
**container images** (OCI/Docker manifest) up to **10 GB** uncompressed, pulled from **ECR**; the
image must include a **runtime interface client** implementing the Lambda runtime API
([AWS — Create a Lambda function using a container image](https://docs.aws.amazon.com/lambda/latest/dg/images-create.html)).
Astro has **no first-party Lambda adapter**; the route is `@astrojs/node` standalone + the
**AWS Lambda Web Adapter** (an AWS Labs/`awslabs` project), which lets you "run web applications on
AWS Lambda" unmodified — "build web apps (http api) with familiar frameworks … anything speaks HTTP
1.1/1.0 and run it on AWS Lambda," added to a container via one `COPY` line from
`public.ecr.aws/awsguru/aws-lambda-adapter`, listening on a configurable port
([AWS Lambda Web Adapter — GitHub](https://github.com/awslabs/aws-lambda-web-adapter)). The Web
Adapter repo is AWS-owned but is a **tool/sample, not a first-party managed service contract** —
treat as semi-primary.

**Deploy/trigger model.** Push image to ECR → `update-function-code`. After an image update Lambda
**optimizes** the image (function sits `Pending` → `Active`, "can take a few seconds") before it can
serve ([images-create — Function lifecycle](https://docs.aws.amazon.com/lambda/latest/dg/images-create.html)).

**Configurability knobs.** Memory/CPU (coupled), timeout, env vars, concurrency/provisioned
concurrency; needs API Gateway/Lambda function URL + CloudFront in front. (General Lambda knobs;
config-page specifics not fetched here.)

**Extensibility for D2.** Same pattern as the container options: webhook → CI pushes image →
`update-function-code`.

**Deploy latency & the catch.** **Cold starts** are the concern for a hybrid site's server routes;
containers up to 10 GB plus image-optimization mean first-invocation latency unless **provisioned
concurrency** is used (added cost). Idle functions go `Inactive` and re-optimize on next call
([images-create](https://docs.aws.amazon.com/lambda/latest/dg/images-create.html)). Fit is best when
server traffic is spiky/low; for steady auth traffic the always-warm container options ([§2](#2-aws-app-runner-container-image--ecr)/[§3](#3-amazon-ecs-on-fargate)) are simpler to reason about.

**Ops ownership.** Low-to-medium, but the adapter shim + fronting API Gateway/CloudFront add
assembly, and it drifts from the "plain container" artifact.

---

## 6. Comparison table

| Option | Runs our `@astrojs/node` container as-is? | Primary trigger mechanism(s) | Key tuning knobs | D2 rebuild-trigger fit | Deploy latency / cold start | Ops ownership |
|---|---|---|---|---|---|---|
| **Amplify Hosting (SSR)** | **No** — needs community `astro-aws-amplify` adapter; Lambda-backed, not a container | Git auto-build; **incoming webhook** (build hook URL) | env vars, `amplify.yml`, custom domain, managed CloudFront; Node 20/22/24 | **Native** (webhook URL) | build+deploy; Lambda cold starts on server routes; no first-party number | Lowest (fully managed) |
| **App Runner** | **Yes** (ECR image) — **but closed to new customers** | ECR push auto-deploy; `StartDeployment` | 0.25–4 vCPU / 0.5–12 GB, max-concurrency, min/max size, VPC, custom domain | Good (push to ECR) | blue-green; min-size warm reserve; no first-party number | Low (managed) — eligibility blocker |
| **ECS on Fargate** | **Yes** (ECR image) | `UpdateService`/`forceNewDeployment`; CodePipeline+CodeDeploy on ECR push; GitHub Action | task CPU/mem, env, min/max-healthy %, ALB health, autoscaling, circuit breaker | Strong (CI push + UpdateService) | rolling/blue-green; warm tasks, no cold start; no per-deploy number | Medium |
| **ECS Express Mode** | **Yes** (ECR image) | single API call; GitHub Action (build→push→deploy) | image, port, env, health-check path, min/max task count | Strong (documented GH Action) | **~3–5 min initial provisioning** (first-party); warm tasks | Low–medium |
| **EKS (own k8s)** | **Yes** (pod) | kubectl/Helm; **Argo CD GitOps**; image push + manifest bump | full K8s surface (HPA, probes, limits, Ingress) | Strong but indirect | rolling; warm pods; no per-deploy number | Highest |
| **Lambda (container + Web Adapter)** | **Partially** — needs Lambda Web Adapter shim; no first-party Astro adapter | ECR push + `update-function-code` | memory/timeout/env, (provisioned) concurrency; needs API GW/CloudFront | Same push pattern | image-optimize `Pending`→`Active` (secs); **cold starts** unless provisioned concurrency | Low–medium (+ assembly) |

---

## 7. Synthesis (decision-support only — not the decision)

Against the stakeholder's four tests — **deploys · tuneable · extensible · fast-ish feedback** — for
a **standalone private-repo deploy as a first step**:

- **Best fit for the host-agnostic-container premise:** **ECS on Fargate** and its **ECS Express
  Mode** front-end. Both run our exact `@astrojs/node` container unchanged, expose rich knobs, and
  extend cleanly to D2 (build+push image → deploy). Express Mode gives App-Runner-like one-call
  simplicity (and the only first-party latency figure we found: ~3–5 min to provision) while keeping
  the door open to full ECS control later. This pairing most directly honours
  [03.3](../03-rendering-and-hosting.md#33-host-agnostic-deployment-artifact).
- **App Runner** would be the tidiest managed-container answer, **but is closed to new customers** —
  only viable if CCP's account is already an existing customer. Otherwise treat Express Mode as its
  stand-in.
- **Amplify** is the lowest-ops and has the **most native D2 trigger** (an incoming-webhook URL out
  of the box), **but it does not run our container** — it requires the *community* `astro-aws-amplify`
  adapter and a Lambda-backed deployment, breaking the single-artifact promise and adding a non-AWS
  dependency. Attractive for speed-to-first-deploy; a strategic compromise on artifact portability.
- **EKS** offers the most control and the cleanest fit *if the portal is meant to live inside CCP's
  existing cluster practice* (a D1 criterion), at the highest ops cost. Overkill purely for a
  standalone first step unless that cluster already exists.
- **Lambda (container + Web Adapter)** can run the artifact via a shim but adds cold-start risk on
  the auth/proxy routes and fronting-infra assembly; best where server traffic is low/spiky, weakest
  where auth traffic is steady.

**Fastest feedback loop for iterating alone first:** Amplify (git-push auto-build, one webhook) and
ECS Express Mode (one API call, documented GitHub Action) are the two lowest-friction starting
points; ECS Express Mode keeps the artifact intact, Amplify does not.

For **D2** specifically, every non-Amplify option converges on the same pattern — *public-repo
webhook / `repository_dispatch` → private CI builds+pushes a container image → target redeploys* —
which satisfies "the public repo learns nothing about hosting." Amplify additionally offers a
zero-CI **incoming-webhook URL**. So D2's shape is largely **independent of D1** except that Amplify
uniquely removes the need for a CI image-build step.

---

## 8. Claims NOT fully pinned to a primary source

Flagged per the research brief:

- **Per-deploy / commit→live latency numbers** for Amplify, App Runner, ECS (rolling & blue/green),
  and EKS: no first-party quantified figure found. Only **ECS Express Mode's ~3–5 min *initial
  provisioning*** is first-party ([App Runner availability change](https://docs.aws.amazon.com/apprunner/latest/dg/apprunner-availability-change.html)),
  and that is provisioning, not steady-state per-deploy. Treat all latency comparisons as qualitative.
- **Cold-start magnitudes** (Amplify SSR Lambda, Lambda-container): AWS confirms cold-start *exists*
  (image optimize `Pending`→`Active` "can take a few seconds";
  [images-create](https://docs.aws.amazon.com/lambda/latest/dg/images-create.html)) but gives no
  duration for our workload.
- **Amplify SSR compute limits for arbitrary (non-Next.js) adapters** — memory/CPU/timeout/response
  size for the current WEB_COMPUTE layer: **not clearly documented.** The
  [SSR supported features](https://docs.aws.amazon.com/amplify/latest/userguide/ssr-supported-features.html)
  page documents mainly the legacy *Classic (Next.js 11)* provider (50 MB deploy limit, 1 MB
  Lambda@Edge response limit). Whether those exact limits bind a community Astro adapter on current
  Amplify compute is **ambiguous** — verify directly before relying on it.
- **`astro-aws-amplify` "wraps in a Lambda function"**: sourced from
  [Astro's deploy guide](https://docs.astro.build/en/guides/deploy/aws/) (first-party for adapter
  facts) and the community adapter; AWS's own Amplify page confirms only that the adapter is
  *community-maintained, not AWS-owned*
  ([astro-support](https://docs.aws.amazon.com/amplify/latest/userguide/astro-support.html)).
- **AWS Lambda Web Adapter**: [awslabs/aws-lambda-web-adapter](https://github.com/awslabs/aws-lambda-web-adapter)
  is AWS-owned but a tool/sample repo, not a managed-service doc — semi-primary.

---

## 9. Pricing implications (region: us-east-1; rates accessed 2026-07-21)

Feeds **[D1 — Hosting target](../11-open-decisions.md)**. This section is **decision-support, not a
decision** — it does not pick a winner. Every **rate** below is quoted from a first-party AWS pricing
page (linked inline and in [Sources](#sources-all-accessed-2026-07-21)); every **monthly total** is a
**worked ESTIMATE** built on explicitly stated assumptions, *not* a quote. All rates are **us-east-1
(N. Virginia)**. Rates whose exact figure could not be read off the (JavaScript-rendered) primary page
are flagged in [§9.5](#95-rates-not-confirmable-from-a-rendered-primary-page).

> **Region note.** us-east-1 is the canonical/cheapest US region. EU regions plausibly relevant to CCP
> (**eu-west-1** Ireland, **eu-central-1** Frankfurt) run **somewhat higher** on Fargate/App Runner
> compute, data transfer and (for eu-central-1) most line items; the EKS control-plane fee and Lambda
> request/GB-s rates are the same across standard commercial regions. Not priced per-region here.

### 9.0 Shared assumptions for the two scenarios

- **Hours/month** = 730 (24×30.4).
- **Scenario (a) — standalone first deploy:** one always-on small instance/task at **0.25 vCPU / 0.5 GB**,
  **low traffic** (the stakeholder's "private-repo first deploy" starting point). CloudFront/data-transfer
  volumes sit inside free tiers.
- **Scenario (b) — modest steady state:** container sized **0.5 vCPU / 1 GB**; **2,000,000 site
  requests/month** of which ~90 % (1.8 M) are static assets served by **CloudFront** and ~10 %
  (**200,000**) hit the **server** routes; **~50 GB/month CloudFront egress**; server-route responses
  ~**10 GB/month**; container image ~**0.7 GB** in ECR; SSR/Lambda work averages **~300 ms at 1 GB** per
  server request. These are illustrative — restate them if the real profile differs.
- **CI builds and pushes the image externally (GitHub Actions)**, so AWS-side build cost (CodeBuild /
  App Runner source-build fee / Amplify build minutes) is largely **not** incurred; noted per-option but
  not carried into the totals unless the option forces an AWS build (Amplify does).

### 9.1 Cross-cutting cost components (apply regardless of compute choice)

| Component | Rate (us-east-1, first-party) | Free tier | Notes for this workload |
|---|---|---|---|
| **Amazon ECR** (private) | **$0.10 / GB-month** storage ([ECR pricing](https://aws.amazon.com/ecr/pricing/)) | 500 MB-month for 12 months | ~0.7 GB image ⇒ **~$0.07/mo** (or $0 in year 1 for a lean image). Data transfer **in is free**; pull to same-region compute is free. Negligible either scenario. |
| **Amazon CloudFront** (CDN) | Pay-as-you-go per-GB egress + per-10k HTTPS requests, North America tier — **exact figures not readable from the rendered page**, see [§9.5](#95-rates-not-confirmable-from-a-rendered-primary-page) | **1 TB egress + 10,000,000 HTTP/HTTPS requests per month, always free** ([AWS Free Tier expansion](https://aws.amazon.com/blogs/aws/aws-free-tier-data-transfer-expansion-100-gb-from-regions-and-1-tb-from-amazon-cloudfront-per-month/)) | **Origin→CloudFront transfer is free.** Both scenarios (50 GB, ≤2 M requests) sit **well inside the perpetual free tier ⇒ ~$0**. CloudFront cost only appears above 1 TB / 10 M req. |
| **Data transfer out to internet (from a Region, not via CloudFront)** | tiered ~**$0.09/GB** first 10 TB *(commonly published; not confirmable from rendered page — [§9.5](#95-rates-not-confirmable-from-a-rendered-primary-page))* | **100 GB/month free**, aggregated across services ([Free Tier expansion](https://aws.amazon.com/blogs/aws/aws-free-tier-data-transfer-expansion-100-gb-from-regions-and-1-tb-from-amazon-cloudfront-per-month/); usage-type detail [CUR data-transfer doc](https://docs.aws.amazon.com/cur/latest/userguide/cur-data-transfers-charges.html)) | Server responses that bypass CloudFront (~10 GB/mo) stay inside the 100 GB free tier ⇒ **~$0** in both scenarios. |
| **Application Load Balancer (ALB)** — needed by ECS/EKS on Fargate | **$0.0225 / ALB-hour** + **$0.008 / LCU-hour** ([ELB pricing](https://aws.amazon.com/elasticloadbalancing/pricing/)) | none | **$0.0225 × 730 = ~$16.43/mo fixed**, before any traffic. LCU = max of {new conn 25/s, active conn 3,000, 1 GB/h processed, 1,000 rule-evals/s} per LCU. Low traffic ⇒ ~1 LCU ⇒ **+~$1–6/mo**. **This fixed ~$16–22/mo often dwarfs the task compute at low traffic.** |
| **NAT Gateway** — *the classic hidden cost* if the task runs in a **private subnet** needing egress | **$0.045 / hour** + **$0.045 / GB processed** ([VPC pricing](https://aws.amazon.com/vpc/pricing/)) | none | **$0.045 × 730 = ~$32.85/mo fixed** + per-GB. Avoidable by placing the task in a public subnet with a public IP, or via VPC/gateway endpoints. Flag on any ECS/EKS/App Runner-VPC design. |
| **CodePipeline / CodeBuild** | only if AWS-side CI is chosen | — | Out of scope: GitHub Actions builds/pushes the image. Mentioned, not priced. |

### 9.2 Per-option pricing model, idle cost, free tier

**1. AWS Amplify Hosting (SSR / WEB_COMPUTE)** — [Amplify pricing](https://aws.amazon.com/amplify/pricing/)
- **Model:** build & deploy **$0.01/build-minute**; hosting storage **$0.023/GB-month**; **data transfer
  out $0.15/GB served**; SSR **request count $0.30 per 1 M requests**; SSR **duration $0.20 per GB-hour**.
- **Idle cost:** SSR compute is request/duration-metered (Lambda-backed) → **scales toward zero**; no
  always-on instance charge. Storage for deployed assets persists (tiny).
- **Free tier:** 1,000 build-min, 5 GB storage, **15 GB data transfer**, 500,000 SSR requests,
  **100 GB-hours** SSR duration — **per month**.
- **Gotcha:** egress at **$0.15/GB is ~1.7× CloudFront's ~$0.085/GB** and there is **no 1 TB free tier** —
  data transfer is the sting as traffic grows. Also (per [§1](#1-aws-amplify-hosting-ssr--compute))
  Amplify does **not** run our container; it forces an AWS-side build + community Lambda adapter.

**2. AWS App Runner** — [App Runner pricing](https://aws.amazon.com/apprunner/pricing/)
- **Model:** **provisioned (idle) memory $0.007/GB-hour**; **active compute $0.064/vCPU-hour** +
  **active memory $0.007/GB-hour**. Per-second billing; **1-minute minimum vCPU charge** each time a
  provisioned instance starts handling requests.
- **Idle cost — the key nuance:** with **Min size ≥ 1** (to keep a warm instance and avoid cold starts)
  you **pay memory 24/7 on the provisioned instance(s) but CPU only while actively serving**. So idle is
  **memory-only**: 0.5 GB ⇒ 0.5 × 730 × $0.007 = **~$2.56/mo** floor; 1 GB ⇒ **~$5.11/mo**. This is a real
  but small idle floor. (Min size 0 is not offered — App Runner always keeps ≥1 provisioned.)
- **Free tier:** none.
- **Gotchas:** active vCPU **$0.064/hr is ~1.6× Fargate's ~$0.0405/hr** — under sustained full load App
  Runner costs more per vCPU than Fargate. Optional **automation fee** (per-app/month) if auto-deploy is
  enabled and **source build fee** (per build-minute) — **exact figures not readable, [§9.5](#95-rates-not-confirmable-from-a-rendered-primary-page)**; the source build fee is **$0 for us** (ECR image
  source). And per [§2](#2-aws-app-runner-container-image--ecr) App Runner is **closed to new customers**.

**3. Amazon ECS on Fargate** — [Fargate pricing](https://aws.amazon.com/fargate/pricing/)
- **Model:** **per vCPU-hour + per GB-hour**, per-second (1-min min). X86: **$0.04048/vCPU-hour**,
  **$0.004446/GB-hour** (from the page's $0.000011244/vCPU-s and $0.000001235/GB-s). **Graviton/ARM ~20 %
  cheaper**: **~$0.03238/vCPU-hour**, **~$0.003560/GB-hour**. 20 GB ephemeral storage free per task.
- **Idle cost:** **YES — a Fargate task bills 24/7 whether or not anyone hits it** (always-on compute).
  Plus the **ALB (~$16–22/mo)** runs 24/7 regardless. This is the structural opposite of Lambda.
- **Free tier:** none for Fargate compute.
- **Gotchas:** **ALB fixed cost** (see §9.1) and **NAT Gateway** if private-subnet. **Fargate Spot** (up to
  ~70 % off) exists but is interruptible — unsuitable for the single always-on server task.

**4. Amazon ECS Express Mode** — no service surcharge ([App Runner→ECS Express](https://docs.aws.amazon.com/apprunner/latest/dg/apprunner-availability-change.html)); compute = Fargate rates above
- **Model:** **"no additional charge for using Amazon ECS Express Mode"** — you pay the **underlying
  Fargate compute + the auto-provisioned ALB + networking** at their normal rates.
- **Idle cost:** same as ECS/Fargate — **always-on task + always-on ALB bill 24/7**.
- **Free tier:** none (inherits Fargate).
- **Gotcha:** the convenience hides an **ALB you're billed for (~$16–22/mo)** and possibly a **NAT
  Gateway** in the managed VPC — same hidden fixed costs as hand-rolled ECS.

**5. Amazon EKS (own Kubernetes)** — [EKS pricing](https://aws.amazon.com/eks/pricing/)
- **Model:** **$0.10 per cluster per hour** control-plane (standard support) = **~$73/mo**, **plus** the
  Fargate or EC2 compute under it, **plus** ALB, **plus** any NAT. Extended-support surcharge **$0.60/hr**
  once a version ages out (adds ~$365/mo — avoid).
- **Idle cost:** **YES, and it is the highest structural floor:** the **~$73/mo control plane bills 24/7
  regardless of traffic or even whether any pod runs**, before compute or ALB.
- **Free tier:** none.
- **Gotcha:** the **$73/mo floor** makes EKS the most expensive option for a low-traffic standalone deploy;
  only rational if the portal folds into an **existing** CCP cluster that already carries that cost.

**6. AWS Lambda (container image + Web Adapter)** — [Lambda pricing](https://aws.amazon.com/lambda/pricing/)
- **Model:** **$0.20 per 1 M requests** + **$0.0000166667 per GB-second** duration (x86; container images
  billed identically to zip). Optional **provisioned concurrency $0.0000041667/GB-s** + a lower duration
  rate **$0.0000097222/GB-s** while enabled.
- **Idle cost:** **~$0 — scales to zero** (nothing runs, nothing bills). **Unless** you enable
  **provisioned concurrency** to kill cold starts on the auth/proxy routes, which **reintroduces an idle
  floor**: 1 GB kept warm 24/7 = 1 × 730 × 3600 × $0.0000041667 = **~$10.95/mo** (+ its duration rate) —
  defeating the scale-to-zero advantage.
- **Free tier:** **1,000,000 requests + 400,000 GB-seconds per month.** (Function URL is free; API Gateway,
  if used instead, adds cost.)
- **Gotcha:** cold starts on a container image (10 GB max; image-optimize `Pending`→`Active`) push toward
  provisioned concurrency, i.e. toward paying an idle floor.

### 9.3 Worked monthly estimates (ESTIMATES on §9.0 assumptions — not quotes)

**Scenario (a) — 0.25 vCPU / 0.5 GB, always-on, low traffic**

| Option | Arithmetic (us-east-1) | Est. $/mo |
|---|---|---|
| **Lambda** | within free tier (≤1 M req, ≤400k GB-s); Function URL free | **~$0** (~**$11** if 1 GB provisioned concurrency) |
| **Amplify SSR** | within all free tiers (build/storage/transfer/SSR) | **~$0** (but forces AWS build + non-container adapter) |
| **App Runner** | idle mem 0.5×730×$0.007=$2.56 + light active CPU ~$0.58 (+~$1 automation) | **~$4** |
| **ECS on Fargate** | compute: 0.25×730×$0.04048=$7.39 + 0.5×730×$0.004446=$1.62 = **$9.01**; + ALB ~$16.4 + ~$1–6 LCU | **~$27** (or **~$9** if no ALB) |
| **ECS Express Mode** | same Fargate compute $9.01 + auto ALB ~$16.4 + LCU | **~$27** |
| **EKS** | control plane $73 + Fargate compute $9.01 + ALB ~$16.4 | **~$99** |

**Scenario (b) — 0.5 vCPU / 1 GB, 2 M req/mo (200k server), ~50 GB CDN egress**

| Option | Arithmetic (us-east-1) | Est. $/mo |
|---|---|---|
| **Lambda** | 200k req < 1 M free; 200k×0.3s×1 GB=60k GB-s < 400k free ⇒ compute $0; CloudFront in free tier | **~$0** (~**$11+** with provisioned concurrency) |
| **Amplify SSR** | egress 50−15 free=35 GB×$0.15=**$5.25**; SSR req/dur & storage within free | **~$5** (egress-dominated) |
| **App Runner** | idle mem 1×730×$0.007=$5.11 + active CPU ~20 %×0.5×730×$0.064≈$4.67 (+~$1) | **~$11** |
| **ECS on Fargate** | compute 0.5×730×$0.04048=$14.78 + 1×730×$0.004446=$3.24 = **$18.02**; + ALB ~$16.4 + ~$8 LCU | **~$42** |
| **ECS Express Mode** | same as Fargate (auto ALB) | **~$42** |
| **EKS** | control $73 + compute $18.02 + ALB ~$24 | **~$115** |

CloudFront egress (50 GB) and server-route egress (10 GB) are **$0** in both scenarios — both sit inside
the perpetual 1 TB CloudFront / 100 GB Region free tiers.

### 9.4 Cost-scaling shape & comparison

| Option | Pricing model | Idle floor (low traffic) | Free tier | Est. baseline (a) $/mo | Scaling shape |
|---|---|---|---|---|---|
| **Amplify SSR** | build-min + storage/GB + **egress $0.15/GB** + SSR $0.30/1M + $0.20/GB-hr | **~$0** (metered, ~scale-to-zero) | generous (build/storage/15 GB/500k/100 GB-hr) | **~$0** | grows with requests **and egress**; egress the sting (no 1 TB free) |
| **App Runner** | idle mem $0.007/GB-hr + active $0.064/vCPU-hr | **memory-only** (~$2.6 @0.5 GB) | none | **~$4** | low floor + per-active-CPU; pricier per vCPU than Fargate at full load |
| **ECS Fargate** | $0.04048/vCPU-hr + $0.004446/GB-hr + ALB | **task + ALB bill 24/7** | none | **~$27** (w/ ALB) | ~flat vs traffic until scale-out; cheapest at **high sustained** load |
| **ECS Express** | Fargate rates, **no surcharge**, + auto ALB | **task + ALB 24/7** | none | **~$27** | same as Fargate |
| **EKS** | **$0.10/cluster-hr (~$73/mo)** + compute + ALB | **~$73 control plane 24/7** | none | **~$99** | highest fixed floor; only amortizes at scale / existing cluster |
| **Lambda** | $0.20/1M req + $0.0000166667/GB-s (+ prov. concurrency) | **~$0 (scale-to-zero)** | 1M req + 400k GB-s | **~$0** | linear with req×duration; cheapest at low/spiky; crosses above always-on at high sustained load; provisioned concurrency adds ~$11/GB-mo floor |

**Synthesis (decision-support for D1 — not a pick):**
- **Cheapest for a low-traffic standalone first deploy:** **Lambda** and **Amplify** (~$0, riding free
  tiers / scale-to-zero), then **App Runner (~$4)** as the cheapest *always-warm container*. But Lambda's
  ~$0 relies on tolerating cold starts (else ~$11 provisioned concurrency), and Amplify's ~$0 **abandons
  the container artifact** ([§1](#1-aws-amplify-hosting-ssr--compute)). So among options that run our
  `@astrojs/node` container as-is and stay warm, **App Runner is cheapest** — subject to its
  closed-to-new-customers blocker ([§2](#2-aws-app-runner-container-image--ecr)).
- **ECS on Fargate / Express Mode (~$27)** carry a **fixed ALB tax (~$16–22/mo)** that dominates the tiny
  task compute at low traffic — but they run the exact container, stay warm, and their cost is
  **~flat** as traffic grows, making them the value option at **modest-to-high sustained** traffic.
- **EKS (~$99)** is the most expensive at low traffic because the **~$73/mo control-plane floor bills
  24/7**; it only makes economic sense if the portal joins an **existing** cluster.
- **Crossover as traffic grows:** Lambda is cheapest until request×duration volume rises enough that its
  per-request cost overtakes an always-on task; App Runner's active-CPU cost climbs faster than Fargate's;
  Fargate/Express flatten out and win at sustained load; EKS only competes once its fixed floor is spread
  over enough workloads.
- **Hidden costs to watch:** **ALB** (~$16–22/mo fixed on ECS/EKS), **NAT Gateway** (~$33/mo + $0.045/GB if
  a task sits in a private subnet — the classic surprise), **EKS control-plane floor** (~$73/mo), **Amplify
  egress** ($0.15/GB, no 1 TB free), **Lambda provisioned concurrency** (~$11/GB-mo if used), and
  **cross-AZ / Region egress** generally. CloudFront's perpetual **1 TB + 10 M-request** free tier keeps
  CDN cost at $0 for this workload's foreseeable traffic.

### 9.5 Rates NOT confirmable from a rendered primary page

The exact figures below sit in **JavaScript-rendered tables** the fetch could not read; the *free tiers
and pricing models* around them **were** confirmed first-party. Verify in-console before relying on the
numbers — none is load-bearing for this workload (all sit inside free tiers or are $0 for us):
- **CloudFront pay-as-you-go per-GB egress (~$0.085/GB NA) and per-10k-HTTPS-request rate** — only the
  **1 TB + 10 M-request perpetual free tier** was confirmed ([Free Tier blog](https://aws.amazon.com/blogs/aws/aws-free-tier-data-transfer-expansion-100-gb-from-regions-and-1-tb-from-amazon-cloudfront-per-month/)). Workload stays inside the free tier, so unused.
- **Region Data-Transfer-Out-to-internet per-GB tier (commonly ~$0.09/GB first 10 TB, us-east-1)** — only
  the **100 GB/month free tier** was confirmed. Workload stays inside it.
- **App Runner automation fee (per-app/month) and source build fee (per build-minute)** — the *model* is
  on the page but the dollar figures did not render. The **source build fee is $0 for us** (we deploy an
  **ECR image**, not source); the automation fee (small, per-app) applies only if ECR auto-deploy is on.



**AWS — first-party docs**
- Deploying SSR applications with Amplify Hosting — https://docs.aws.amazon.com/amplify/latest/userguide/server-side-rendering-amplify.html
- Amplify support for Astro.js — https://docs.aws.amazon.com/amplify/latest/userguide/astro-support.html
- Amplify SSR supported features — https://docs.aws.amazon.com/amplify/latest/userguide/ssr-supported-features.html
- Amplify — Creating an incoming webhook to start a build — https://docs.aws.amazon.com/amplify/latest/userguide/create-incoming-webhook.html
- App Runner — availability change (closed to new customers; ECS Express Mode migration; ~3–5 min provisioning) — https://docs.aws.amazon.com/apprunner/latest/dg/apprunner-availability-change.html
- App Runner — architecture & concepts (image/source, vCPU/memory table, custom domain) — https://docs.aws.amazon.com/apprunner/latest/dg/architecture.html
- App Runner — deploying a new version (automatic vs manual, ECR auto-deploy) — https://docs.aws.amazon.com/apprunner/latest/dg/manage-deploy.html
- App Runner — managing auto scaling (max concurrency / min / max size) — https://docs.aws.amazon.com/apprunner/latest/dg/manage-autoscaling.html
- ECS — Deploy by replacing tasks (rolling update; min/max-healthy %; image digest resolution) — https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-ecs.html
- ECS — CodeDeploy blue/green deployments — https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-bluegreen.html
- Amazon EKS — What is Amazon EKS? (managed control plane, Argo CD GitOps, conformance) — https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html
- Lambda — Create a function using a container image (10 GB, ECR, RIC, function lifecycle) — https://docs.aws.amazon.com/lambda/latest/dg/images-create.html

**Astro — first-party docs**
- Deploy your Astro Site to AWS (Amplify SSR = community adapter, Lambda-wrapped) — https://docs.astro.build/en/guides/deploy/aws/
- @astrojs/node adapter (standalone vs middleware; HOST/PORT env) — https://docs.astro.build/en/guides/integrations-guide/node/

**AWS-owned but tool/sample (semi-primary)**
- AWS Lambda Web Adapter — https://github.com/awslabs/aws-lambda-web-adapter
- AWS blog — CodeDeploy canary/linear for ECS — https://aws.amazon.com/blogs/containers/aws-codedeploy-now-supports-linear-and-canary-deployments-for-amazon-ecs

**AWS — pricing pages (§9; all accessed 2026-07-21, region us-east-1)**
- AWS Fargate pricing (vCPU-hour / GB-hour, X86 & Graviton, ephemeral storage) — https://aws.amazon.com/fargate/pricing/
- AWS App Runner pricing (provisioned/idle memory, active vCPU/memory, build & automation fees) — https://aws.amazon.com/apprunner/pricing/
- AWS Lambda pricing (per-request, GB-second, provisioned concurrency, free tier) — https://aws.amazon.com/lambda/pricing/
- AWS Amplify pricing (build minutes, storage, data transfer, SSR request/duration, free tier) — https://aws.amazon.com/amplify/pricing/
- Amazon EKS pricing ($0.10/cluster-hour control plane; extended support surcharge) — https://aws.amazon.com/eks/pricing/
- Amazon ECR pricing (private storage $0.10/GB-month, 500 MB free 12 mo) — https://aws.amazon.com/ecr/pricing/
- Amazon CloudFront pricing (free-tier & flat-rate plans; pay-as-you-go per-GB/request table JS-rendered) — https://aws.amazon.com/cloudfront/pricing/
- Elastic Load Balancing pricing (ALB $0.0225/hr + $0.008/LCU-hr; LCU dimensions) — https://aws.amazon.com/elasticloadbalancing/pricing/
- Amazon VPC pricing (NAT Gateway $0.045/hr + $0.045/GB) — https://aws.amazon.com/vpc/pricing/
- Amazon EC2 On-Demand pricing (data transfer out to internet; 100 GB/mo free) — https://aws.amazon.com/ec2/pricing/on-demand/
- AWS Free Tier data-transfer expansion (100 GB Regions + 1 TB CloudFront + 10 M requests/mo, perpetual) — https://aws.amazon.com/blogs/aws/aws-free-tier-data-transfer-expansion-100-gb-from-regions-and-1-tb-from-amazon-cloudfront-per-month/
- AWS CUR — understanding data-transfer charges (usage-type detail) — https://docs.aws.amazon.com/cur/latest/userguide/cur-data-transfers-charges.html
