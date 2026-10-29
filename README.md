# Guardener demo repo

A small repo that stands in for a customer's codebase, for demoing Guardener and the [Chainguard Console Insights dashboard](https://www.chainguard.dev/unchained/introducing-the-chainguard-console-insights-dashboard-powered-by-guardener). It has upstream container images in the places Guardener reads: a Dockerfile, a Compose file, Kubernetes manifests and Helm values.

Guardener is in Early Access and only works with GitHub. Commands and behavior below come from the [Guardener docs](https://edu.chainguard.dev/chainguard/guardener/) as of late October 2026. They can change, so skim the docs before you present.

## What is in it

| File | Upstream image | Expected result |
|---|---|---|
| `Dockerfile` | `python:3.12` | Migrated. No `RUN`, exec-form `ENTRYPOINT`, no `CMD`, so it is a clean swap. |
| `docker-compose.yml` | `redis:7`, plus an image you choose on line 11 (`rabbitmq:3` placeholder) | `redis:7` migrated. Your line 11 image listed as **Not migrated** because it is not in your catalog. |
| `k8s/web.yaml` | `nginx:1.27` | Migrated. |
| `k8s/cache.yaml` | `redis:7` | Migrated. |
| `helm/demo-chart/values.yaml` | `nginx` + `1.27` | **Not migrated**. The image is split across Helm `repository` and `tag` keys, which Guardener does not rewrite. Templates are not read at all. |
| `scripts/make_guardrail_branch.sh` | `memcached:1.6` | Only for people who cloned the repo. Step 5 below does the same thing in the GitHub web UI. |
| `.chainguard/images.yaml.example` | n/a | The switch that turns Guardener on. Nothing happens until you rename it in step 3. |
| `app/app.py` | n/a | Tiny standard-library service, so the Dockerfile needs no `pip install` and no `RUN`. |

## Before you demo

Do this at least a day ahead. Scanning is continuous but the timing is not documented, and the dashboard needs data to show.

### What you need

- [ ] Owner of a GitHub organization. Installing the app and authorizing the link both need it. A personal account also works: pass your username to `--github-org`.
- [ ] Owner of a Chainguard organization, or the `guardener.association.manage` capability on it.
- [ ] The latest `chainctl`, authenticated. Older releases lack the `chainctl guardener` commands, and `migrate create` needs `0.2.375` or later.
- [ ] Access to the Chainguard Console for that organization.

### 1. Make your own copy of this repo

1. On this repo's GitHub page, select **Use this template** > **Create a new repository**.
2. Pick the GitHub organization you will link and give the copy a name.
3. Make it **Public**. Guardener's default repository visibility scope is `PUBLIC`, which skips private repos. If your copy must be private, confirm `chainctl guardener github status` shows scope `ALL` first. There is no self-service command to change the scope, so ask your account team.

### 2. Install the Chainguard App and link your organization

Follow [Getting started with Guardener](https://edu.chainguard.dev/chainguard/guardener/github/getting-started/). In short:

1. Install the Chainguard App on the GitHub organization and give it access to your copy of the repo (or all repos). Installing it changes nothing until you enable a capability.
2. Link your Chainguard organization. A browser opens to authorize with GitHub, and the first link asks you to accept the Guardener terms.
   ```bash
   chainctl guardener github link --github-org <github-org-login> --group <chainguard-org-name>
   ```
3. Verify the link and note the visibility scope:
   ```bash
   chainctl guardener github status --group <chainguard-org-name>
   ```

### 3. Add the images to your catalog

Guardener never adds images to your organization. A reference is only swapped when the replacement image **and the matching tag** are already in your catalog.

1. In the Console,  go to **Chainguard catalog** > **Add to org** (needs the `owner` role). Add `python`, `nginx`, `redis` and `memcached`. 
2. **Pick an image that is not in your catalog** for the "Not migrated" line in the demo. Most organizations already have common images like `postgres`, so choose one you have not added. In the GitHub web UI, open `docker-compose.yml` on your copy, select the pencil (**Edit this file**), replace `rabbitmq:3` on line 11 with your image and tag, and commit directly to `main`. Keep `rabbitmq:3` if it is not in your catalog. Do not add your chosen image to the catalog.
3. Tags take a few minutes to sync. Then check every image resolves as `entitled`, replacing `<your-line-11-image>` with the image you put on line 11:
   ```bash
   chainctl images discover --parent <chainguard-org-name> \
     --image python:3.12 --image nginx:1.27 --image redis:7 --image memcached:1.6 --image <your-line-11-image>
   ```
   Your line 11 image should show `available to add`, not `entitled`. If it shows `entitled`, it is already in your catalog and Guardener will migrate it, so pick a different image. If any other image shows `no maintained tag for <tag>`, that tag has aged out. Change the tag in this repo to a maintained one, or Guardener will leave it alone.

### 4. Confirm Guardener has scanned your copy

```bash
chainctl guardener scans list --parent <chainguard-org-name> --repo <github-org>/<repo>
```

When a scan shows up here, the repo should appear in the Insights dashboard. If the Insights dashboard is not yet visible in your Console, use the screenshots from the launch post and say it is rolling out.

### 5. Turn migration on

Do this in the GitHub web UI on your copy:

1. Open `.chainguard/images.yaml.example` and select the pencil (**Edit this file**).
2. Change the filename to `.chainguard/images.yaml`.
3. **Commit directly to `main`.** Guardener only reads this file from the default branch, and the push that adds it starts the first migration.

The file turns on two things: `enabled: true` is Container Image Suggestions (the guardrail in step 5), and `migrate.enabled: true` is Container Image Migration (the pull request in step 4).

If the Console offers it, you can show enabling migration from the Insights dashboard instead. The YAML file is the documented path and always works. (Make sure to do this a day in advance, it might take 24 hours to see results).

## Demo flow

### 1. Insights dashboard

Open the Chainguard Console and show the Insights dashboard for your organization: the per-image inventory across repos, the suggested Chainguard equivalent for each, the CVE difference, and the mitigated CVE summary. Point out that this came from the GitHub App reading the repo. No agent, no CI change.

Fallback if the dashboard is not live for you: `chainctl guardener scans list` output plus the launch post screenshots.

### 2. Review the migration pull request

A pull request titled **Migrate container images to Chainguard equivalents** appears from a branch under `guardener/images/`. If it has not shown up after about ten minutes, trigger it:

```bash
chainctl guardener github migrate create <github-org>/<repo>
```

Walk through the PR:

- **Changed files:** `Dockerfile`, `docker-compose.yml` (only `redis`), `k8s/web.yaml`, `k8s/cache.yaml`. Each upstream image became a `cgr.dev/chainguard/...` image with the same tag.
- **Not migrated section:** your line 11 image in `docker-compose.yml` because it is not in your catalog, and `helm/demo-chart/values.yaml` because the image is split across Helm keys. Guardener explains each one. This is the honest part of the demo: it is conservative, and it tells you why.
- **It swaps references, nothing else.** Guardener does not build or run anything. CI should run before merge, same as any PR.
- Do not push to the `guardener/images/` branch yourself unless you first add the `skip:guardener/images` label or assign someone. Guardener force-pushes that branch.

Merge the PR.

### 5. Guardrail: a new upstream image in a pull request

Still in the GitHub web UI:

1. On `main`, select **Add file** > **Create new file**.
2. Name it `k8s/memcached.yaml` and paste:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: memcached
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: memcached
     template:
       metadata:
         labels:
           app: memcached
       spec:
         containers:
           - name: memcached
             image: memcached:1.6
   ```
3. Select **Commit changes**, choose **Create a new branch for this commit and start a pull request**, name it `demo/add-memcached`, and open the pull request. Do not open it as a draft; Guardener waits for drafts to be marked ready.

Guardener posts a non-blocking review comment on the `image: memcached:1.6` line naming the Chainguard replacement, with a one-click suggested change. The `actions-recommender-images` check finishes as success or neutral, never failed. Accept the suggestion to show the fix landing in one click.

### 6. Back to the dashboard

Return to the Insights dashboard and show the repo's inventory moving toward Chainguard images. It may lag the merge; say so.

## Talking points to expect

- **Clean swap:** `python:3.12` becomes the Chainguard Python image because the Dockerfile has no `RUN`, uses exec-form `ENTRYPOINT`, and sets no `CMD`. Guardener skips stages that break those rules, because Chainguard images are minimal and set their own entrypoint.
- **Not migrated is a feature:** the image you chose for line 11 of `docker-compose.yml` shows what happens before an image is entitled. The Helm chart shows Guardener refusing to guess when an image is assembled from several keys.
- **Guardener does not run your build:** it swaps references and opens a PR. Review and CI are still yours.
- **Nothing to install in your clusters:** it is a GitHub App plus a YAML file. Every capability is off until you opt in.
- **Org-wide rollout:** the same `.chainguard/images.yaml` committed to the org's `.github` repo applies to every repo that lacks its own.

## Resetting for another run

Fastest: delete your copy and create a new one from the template. Otherwise, close or delete the `guardener/images/...` and `demo/add-memcached` branches, revert the merged migration PR on `main`, and rename `images.yaml` back to `images.yaml.example` (or delete it).

## Check before you present

- Run the `chainctl images discover --image ...` check from prep step 3 again. Tags age out, and a missing tag turns a clean swap into "Not migrated". Also confirm your line 11 image still shows `available to add`, in case someone added it to the catalog since you set up.
- `chainctl guardener github status` still shows your GitHub organization linked.
- `chainctl guardener scans list` shows a recent scan of your copy.
- Check whether the Insights dashboard is visible in your Console, so you know which version of step 1 you are giving.
- Guardener is Early Access and GitHub only. Say so.

## References

- [Introducing the Chainguard Console Insights Dashboard](https://www.chainguard.dev/unchained/introducing-the-chainguard-console-insights-dashboard-powered-by-guardener)
- [Guardener overview](https://edu.chainguard.dev/chainguard/guardener/overview/)
- [Getting started with Guardener](https://edu.chainguard.dev/chainguard/guardener/github/getting-started/)
- [Configuration model](https://edu.chainguard.dev/chainguard/guardener/github/configuration/)
- [Container Image Migration](https://edu.chainguard.dev/chainguard/guardener/github/image-migration/)
- [Container Image Suggestions](https://edu.chainguard.dev/chainguard/guardener/github/image-suggestions/)
- [App connections](https://edu.chainguard.dev/chainguard/guardener/github/app-connections/)
- [chainctl images discover](https://edu.chainguard.dev/chainguard/containers/migration/migration-tools/images-discover/)
- [Adding an image to your catalog](https://edu.chainguard.dev/chainguard/containers/troubleshooting/container-version-troubleshooting/#the-container-isnt-in-your-organizations-catalog)
