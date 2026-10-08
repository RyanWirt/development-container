# docsearch-devcontainer-image

Pre-built Ubuntu images for general development, built and published automatically
via GitHub Actions. The image includes Git, C/C++ build tools, Python with pip and
venv, Ubuntu's Node.js and npm packages, SSH client tools, and common CLI utilities.
It runs as the non-root `vscode` user with passwordless sudo for development.
This is a development environment, not a hardened production image.

## Images

| Image | Source | Registry |
|-------|--------|----------|
| Ubuntu 24.04 devcontainer base | [`ubuntu-24.04/Dockerfile`](ubuntu-24.04/Dockerfile) | `ghcr.io/ryanwirt/docsearch-ubuntu-24.04` |

## Repository structure

```text
.
├── ubuntu-24.04/
│   └── Dockerfile
└── .github/
    └── workflows/
        ├── reusable-build-push.yml
        └── ubuntu-24.04.yml
```

Each image has its own folder and caller workflow. The reusable workflow handles
building, publishing, and cleanup without image-specific changes.

## Store and publish the image in GitHub

Commit the **Dockerfile and workflows** to Git; store the built image in **GitHub
Container Registry (GHCR)**, not as an image archive in the Git repository.
Publishing uses the workflow's `GITHUB_TOKEN`; no personal access token needs
to be committed.

| Event | Push to registry? | Tags |
|-------|:-----------------:|------|
| Pull request | No | Build-only validation; image discarded |
| Push to `main` | Yes | `main`, `sha-<short-sha>` |
| Push of `docsearch-ubuntu-24.04-v1.2.3` | Yes | `1.2.3`, `latest`, `sha-<short-sha>` |

The Ubuntu workflow runs when `ubuntu-24.04/**`, its caller workflow, or the
reusable workflow changes. Matching tag pushes run regardless of changed paths.
Manual runs publish only from `main` or a matching version tag.
Images currently target Linux AMD64.

1. Enable GitHub Actions in the repository and permit workflows to write
   packages. Push these files to `main` to publish the first image.
2. In the GitHub package settings, verify that this repository has Actions
   access and admin permission on the package (required for cleanup). A package
   first published by this workflow normally inherits repository access.
3. Packages are private initially. Set the package visibility to public if
   developers should pull without authenticating. For a private package, log
   in locally using a token with `read:packages` and access to the package:

   ```sh
   printf '%s' "$GHCR_TOKEN" | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
   ```

4. To publish a release, create and push a version tag from the desired commit:

   ```sh
   git tag docsearch-ubuntu-24.04-v1.2.3
   git push origin docsearch-ubuntu-24.04-v1.2.3
   ```

For a fork or organization, replace `ryanwirt` in image references with the
lowercase repository owner. The workflow derives that owner automatically.

### GHCR image cleanup

After a successful publish, cleanup preserves the 10 newest package versions
and removes older versions **only when every tag is a `sha-*` tag**. Versions
with semantic version tags, `latest`, `main`, or any other non-SHA tag are always
preserved. Untagged versions (including provenance manifests) are left alone.
Set the caller's `keep-n-versions` to another non-negative integer to change
retention, or to `0` to disable cleanup.

## Use in a `.devcontainer`

In the project you want to develop, create `.devcontainer/devcontainer.json`:

```json
{
  "name": "Ubuntu 24.04 development",
  "image": "ghcr.io/ryanwirt/docsearch-ubuntu-24.04:main",
  "remoteUser": "vscode"
}
```

Commit that file to the project, install VS Code's Dev Containers extension,
and choose **Dev Containers: Reopen in Container**. Docker must be running.
For reproducible environments, use a release tag such as `:1.2.3`, a
`:sha-<short-sha>` tag (subject to cleanup), or pin the image by digest:
`ghcr.io/ryanwirt/docsearch-ubuntu-24.04@sha256:<digest>`.
The workflow build summary includes the published digest.

Inside the container, use a virtual environment for Python dependencies:

```sh
python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
```

Ubuntu manages the system Python installation; do not install project packages
into it. Install project-specific Node dependencies with `npm install`.

### Local build

From the repository root:

```sh
docker build -t docsearch-ubuntu-24.04:local -f ubuntu-24.04/Dockerfile ubuntu-24.04
docker run --rm docsearch-ubuntu-24.04:local bash -lc \
  'whoami && git --version && python3 --version && node --version && g++ --version'
```

You can use `"image": "docsearch-ubuntu-24.04:local"` in your devcontainer
configuration before publishing.

## Add a new image

1. Add a top-level folder (for example `ubuntu-22.04/`) with a `Dockerfile`.
2. Copy `.github/workflows/ubuntu-24.04.yml` to `ubuntu-22.04.yml`.
3. Update the workflow name, job name, `paths`, `tags`, `image-name`,
   `dockerfile`, `context`, and `tag-prefix` values.
4. Open a PR for build validation, then merge to `main` to publish and clean up.
   Use the new image's matching version-tag prefix for releases.