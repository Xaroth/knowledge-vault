---
type: Research
title: "AWS Hosting & Deployment Options for the Private Astro Site"
description: "Decision-support survey of viable AWS deploy targets for the host-agnostic Astro Node-SSR container: what runs the artifact, how deploys are triggered, tuning knobs, extensibility for the future public-repo rebuild trigger, and deploy latency. Feeds open decisions D1 and D2."
tags: [design, esi, developer-portal, astro, ssr, cdn, architecture, decisions, ci, hosting, aws]
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

## Sources (all accessed 2026-07-21)

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
