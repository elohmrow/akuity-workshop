# Akuity Workshop — Kargo + Argo CD on Akuity Platform

Promote a simple NGINX app through `dev → test → prod` using Kargo and Argo CD on the Akuity Platform.

## Prerequisites

- Akuity account (https://akuity.cloud) and access to the Argo CD and Kargo instances
- [Kind Cluster](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- Access to both Argo CD and Kargo Instance control planes
- A fork of this repo in your own GitHub account
- A GitHub personal access token (PAT) with read and write access to your fork
- CLIs: [`akuity`](https://docs.akuity.io/akuity-portal/automation/#installation), [`kargo`](https://docs.akuity.io/kargo/getting-started/access-kargo-instance#access-kargo-using-the-kargo-cli), [`task`](https://taskfile.dev/docs/installation), [`envsubst`](https://formulae.brew.sh/formula/gettext)
(`brew install akuity kargo go-task gettext`)

## Step-by-Step Instructions

### 1. Create your cluster and connect the agents

```bash
kind create cluster --name <WORKSHOP_NAME>
```

1. **Argo CD agent**: register this cluster with your Argo CD instance's control plane. Follow [Connect a Kubernetes cluster](https://docs.akuity.io/argocd/getting-started/connect-kubernetes-cluster). Note the cluster name you choose: it's your `ARGOCD_DESTINATION` in `.env`.
2. **Kargo agent**: create a Kargo agent for your Kargo instance and install it on the same cluster. Follow [Connect a Kargo agent](https://docs.akuity.io/kargo/getting-started/connect-kargo-agent).

Wait until both agents show as **Healthy** in the Akuity Platform UI.

### 2. Clone your fork and set up `.env`

```bash
git clone https://github.com/<your-github-username>/akuity-workshop.git
cd akuity-workshop
cp .env.example .env
```

Fill in `.env`. `WORKSHOP_NAME` must be **unique per participant** (for example, `workshop-shivam`). It's used as your Kargo Project name and as the prefix for your Argo CD Applications.

> Never commit `.env`. It contains your PAT.

> **Shortcut:** `task setup` runs steps 3–6 in one go. We recommend going through them one by one the first time, so you see what each step creates in Argo CD and Kargo.

### 3. Log in and check your config

```bash
akuity login
task check
```

### 4. Create the Argo CD AppProject

```bash
task apply-argocd-project
```

This creates the shared `akuity-workshop` AppProject.

### 5. Create the Argo CD Applications

```bash
task apply-applicationset
```

In the Argo CD dashboard you should see three Applications:

`<WORKSHOP_NAME>-nginx-dev`, `<WORKSHOP_NAME>-nginx-test`, `<WORKSHOP_NAME>-nginx-prod`

They show as **Unknown** and aren't synced. That's expected: each one points at a `stage/<env>` branch that doesn't exist yet. Kargo creates those branches when you promote.

### 6. Create the Kargo resources

```bash
task apply-kargo
```

This applies, in order:

| Resource | What it does |
|---|---|
| Project | Your own Kargo Project, named `<WORKSHOP_NAME>` |
| Secret | Git credentials so Kargo can clone your fork, create `stage/*` branches, and push commits |
| Warehouse | Watches `public.ecr.aws/nginx/nginx` for new tags (`^1.27.0`) |
| PromotionTask | Defines how Freight is promoted: clone, set the image, `kustomize build`, commit, push, and sync Argo CD |
| Stages | `dev → test → prod`. Each stage can only promote Freight that passed the stage before it |

The pipeline is ready. Freight discovered by the Warehouse can now be promoted through the stages.

### 7. Your first promotion

1. **dev**: in the Kargo UI, drag the Freight onto the `dev` stage.
2. **test**: in the Kargo UI, click the truck icon on the `test` stage and pick the Freight.
3. **prod**: use the CLI:

   ```bash
   kargo login https://<your-kargo-instance-url> --sso
   kargo get freight --project <WORKSHOP_NAME>
   kargo promote --project <WORKSHOP_NAME> --freight <freight-name> --stage prod
   ```

After each promotion, Kargo pushes to `stage/<env>` and syncs the matching Argo CD Application.

### 8. Check the app

Each stage runs in its own namespace, `<WORKSHOP_NAME>-<stage>`. Port-forward to a stage's Service:

```bash
task port-forward STAGE=dev
```

Or with `kubectl` directly:

```bash
kubectl port-forward svc/nginx 8080:80 -n <WORKSHOP_NAME>-dev
```

Open http://localhost:8080. You should see **NGINX - DEV**.

Check which image is running:

```bash
kubectl get deploy nginx -n <WORKSHOP_NAME>-dev \
-o jsonpath='{.spec.template.spec.containers[0].image}'
```

The tag should match the Freight you promoted. Repeat for `test` and `prod`, using a different local port for each, for example `task port-forward STAGE=test PORT=8081`.


## Repository Structure

```text
.
├── app/                 # NGINX Kustomize base + dev/test/prod overlays
├── argocd/              # AppProject, ApplicationSet
├── kargo/               # Project, Warehouse, PromotionTask, Stages
├── secret.yaml          # Kargo Git credentials (filled from .env)
├── Taskfile.yaml
└── .env.example
```

# Bonus tasks

### Add a custom promotion step

Kargo on the Akuity Platform can run your own container as a promotion step, using a [`CustomPromotionStep`](https://docs.kargo.io/user-guide/reference-docs/promotion-steps/custom-steps). The one in [kargo/custom-step.yaml](kargo/custom-step.yaml) runs `alpine` and prints a message with the stage and image tag.

1. Register the custom step:

   ```bash
   task apply-kargo-custom-step
   ```

2. Add it to [kargo/promotiontask.yaml](kargo/promotiontask.yaml), between the `kustomize-build` and `git-commit` steps:

   ```yaml
       - uses: ${WORKSHOP_NAME}-hello
         as: hello
         config:
           stage: ${{ ctx.stage }}
           tag: ${{ imageFrom(vars.imageRepo).Tag }}
   ```

   Leave `${WORKSHOP_NAME}` as it is. The task fills it in from your `.env`.

3. Apply the updated PromotionTask:

   ```bash
   task apply-kargo-promotion-task
   ```

4. Promote a Freight to `dev`. In the Kargo UI, open the Promotion: the `hello` step runs after `kustomize-build`, and its output shows `Hello from dev, promoting <tag>`.

> Bonus: change the `git-commit` message to `${{ task.outputs.hello.message }}`, apply again, and promote. Your `stage/dev` commit on GitHub now uses the message from your custom step.

### Use Argo CD to manage your Kargo resources

So far you've applied your Kargo resources with `akuity kargo apply`. In this task, Argo CD syncs them from Git instead.

1. **Register your Kargo instance with Argo CD and create the Application.** Follow [Managing Kargo resources with Argo CD](https://docs.akuity.io/kargo/managing-instances/managing-kargo-resources-with-argocd). Point the Application at the `kargo/` folder of your fork. You can use [argocd/kargo-resources-application.yaml](argocd/kargo-resources-application.yaml) as a starting point.

2. **Replace the placeholders.** Argo CD applies files exactly as they are in Git, so it can't fill in `${...}` values from your `.env`. In the files under `kargo/`, replace `${WORKSHOP_NAME}` and `${GITOPS_REPO_URL}` with your real values, then commit and push to your fork.

3. **Manage the Git credentials secret.** Don't commit `secret.yaml` with your PAT in it. Set up the secret by following [Secrets](https://docs.akuity.io/argocd/managing-instances/settings/features/secrets).

4. **Sync and test.** Sync the Application in the Argo CD UI. Then change `discoveryLimit` in `kargo/warehouse.yaml` from `5` to `3`, commit and push, and watch Argo CD apply the change to Kargo.

### Turn on auto-promotion

Set up auto-promotion for your `dev` and `test` stages by following [Promotion policies](https://docs.kargo.io/user-guide/how-to-guides/working-with-projects#promotion-policies) in the Kargo docs.

### Send a notification from a promotion

Add a [`send-message`](https://docs.kargo.io/user-guide/reference-docs/promotion-steps/send-message) step to your PromotionTask to post to Slack or email when a promotion runs.

### Update a ServiceNow ticket

Use [`snow-update`](https://docs.kargo.io/user-guide/reference-docs/promotion-steps/snow-update) (and `snow-create`) in your PromotionTask to create and update a ServiceNow ticket as part of each promotion.

### Verify a Stage with an AnalysisTemplate

Add an [`AnalysisTemplate`](https://docs.kargo.io/user-guide/reference-docs/analysis-templates) to a Stage's `verification` so Kargo runs checks after each promotion. Freight that fails verification can't move on to the next Stage.

### Roll back a Stage

There are two ways to go back to an earlier version:

- **Manually:** promote an older Freight to the Stage again, for example by dragging it onto `prod` in the Kargo UI. Kargo runs the same PromotionTask with the older image tag, and Argo CD syncs it.
- **Automatically:** turn on [auto-rollback](https://docs.kargo.io/user-guide/how-to-guides/working-with-projects#auto-rollback) for a Stage in your Project's `ProjectConfig`. If verification fails after a promotion, Kargo promotes the Stage back to the last Freight that passed verification. This needs verification on the Stage (see the bonus task above) and at least one earlier Freight that passed it.

### Blue-green deployments with Kargo

Try [bluegreen-demo](https://github.com/shimagrawal/bluegreen-demo): a blue-green release driven by Kargo, using two Deployments and Service selectors instead of Argo Rollouts.
