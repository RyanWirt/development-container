# Python AI development container

Pre-built Ubuntu images for general Python, OpenAI agent, machine learning, and
AI development, built and published automatically via GitHub Actions.
The image includes Python 3.12 with a ready-to-use virtual environment:

- OpenAI's Python SDK and Agents SDK for agentic applications.
- CPU PyTorch and Hugging Face Transformers for deep learning and language models.
- NumPy, pandas, SciPy, scikit-learn, and Matplotlib for data analysis and CPU ML.
- JupyterLab and ipykernel for notebooks.
- pytest and Ruff for Python testing and linting.
- Git, C/C++ build tools, Python headers, Ubuntu's Node.js/npm, and CLI utilities.

Direct Python dependency versions are pinned in
[`ubuntu-24.04/requirements.txt`](ubuntu-24.04/requirements.txt), with PyTorch
pinned in the Dockerfile and installed separately from its official CPU wheel
index. This avoids bundling CUDA runtime dependencies.
It runs as the non-root `vscode` user with passwordless sudo for development.
This is a development environment, not a hardened production image.

## Images

| Image | Source | Registry |
|-------|--------|----------|
| Ubuntu 24.04 Python AI base | [`ubuntu-24.04/Dockerfile`](ubuntu-24.04/Dockerfile) | `ghcr.io/ryanwirt/python-ai-ubuntu-24.04` |

## Repository structure

```text
.
├── ubuntu-24.04/
│   ├── Dockerfile
│   └── requirements.txt
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
| Push of `python-ai-ubuntu-24.04-v1.2.3` | Yes | `1.2.3`, `latest`, `sha-<short-sha>` |

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
   git tag python-ai-ubuntu-24.04-v1.2.3
   git push origin python-ai-ubuntu-24.04-v1.2.3
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
  "name": "Python AI development",
  "image": "ghcr.io/ryanwirt/python-ai-ubuntu-24.04:main",
  "remoteUser": "vscode",
  "containerEnv": {
    "OPENAI_API_KEY": "${localEnv:OPENAI_API_KEY}"
  },
  "forwardPorts": [8888],
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-python.python",
        "ms-toolsai.jupyter",
        "charliermarsh.ruff"
      ],
      "settings": {
        "python.defaultInterpreterPath": "/home/vscode/.venv/bin/python"
      }
    }
  }
}
```

Commit that file to the project, install VS Code's Dev Containers extension,
and choose **Dev Containers: Reopen in Container**. Docker must be running.
For reproducible environments, use a release tag such as `:1.2.3`, a
`:sha-<short-sha>` tag (subject to cleanup), or pin the image by digest:
`ghcr.io/ryanwirt/python-ai-ubuntu-24.04@sha256:<digest>`.
The workflow build summary includes the published digest.

The image sets `PATH` and `VIRTUAL_ENV` to `/home/vscode/.venv`, which is writable
by `vscode`, including after Dev Containers matches its UID to a Linux host user.
`python` and `pip` use this environment without manual activation:

```sh
python -m pip install -r requirements.txt
python -m pytest
ruff check .
```

For projects needing independent dependency versions, create a project-local
environment and select `.venv/bin/python` in VS Code:

```sh
python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
```

This separate environment does not inherit the image's AI packages; install
your project's dependencies into it. Ubuntu manages system Python; do not use
`sudo pip` or `--break-system-packages`. Install Node dependencies with `npm install`.

### OpenAI agents and credentials

Set `OPENAI_API_KEY` in the host environment before opening VS Code so the
devcontainer configuration can pass it through at runtime. Recreate the container
after changing the host value. Never put credentials in the Dockerfile,
requirements, Git, or image. Projects without OpenAI access can still use the
offline ML and notebook tools.

An agent application can use the preinstalled SDK:

```python
from agents import Agent, Runner

agent = Agent(name="Assistant", instructions="Help with Python development.")
result = Runner.run_sync(agent, "Explain Python virtual environments.")
print(result.final_output)
```

This example calls the OpenAI API and requires an API key, network access, and
an account with billing/access to the selected model.

### Notebooks and deep learning

Use VS Code's Jupyter extension with `/home/vscode/.venv/bin/python`, or run:

```sh
jupyter lab --ip=0.0.0.0 --port=8888 --no-browser
```

Open the token-bearing URL from Jupyter's output using the forwarded port.
Keep token authentication enabled and the forwarded port private.

This base image includes CPU PyTorch and Transformers for training and inference.
For example, a small randomly initialized transformer can run entirely offline:

```python
import torch
from transformers import GPT2Config, GPT2LMHeadModel

config = GPT2Config(vocab_size=100, n_positions=32, n_embd=32, n_layer=1, n_head=2)
model = GPT2LMHeadModel(config)
tokens = torch.randint(0, config.vocab_size, (1, 8))
loss = model(input_ids=tokens, labels=tokens).loss
loss.backward()
print(loss.item())
```

Pretrained model weights and datasets are not bundled. Loading them with
`from_pretrained` downloads them at runtime and may require network access,
Hugging Face authentication, sufficient storage, and acceptance of model licenses.
TensorFlow and CUDA are not bundled. GPU workloads require a compatible CUDA image,
host NVIDIA drivers, NVIDIA Container Toolkit, and GPU device access; installing
a Python framework alone does not enable GPU support.

### Local build

From the repository root:

```sh
docker build -t python-ai-ubuntu-24.04:local -f ubuntu-24.04/Dockerfile ubuntu-24.04
docker run --rm python-ai-ubuntu-24.04:local python -c \
  'import openai, agents, torch, transformers, numpy, pandas, scipy, sklearn; print("Python AI environment ready")'
```

You can use `"image": "python-ai-ubuntu-24.04:local"` in your devcontainer
configuration before publishing.

## Add a new image

1. Add a top-level folder (for example `ubuntu-22.04/`) with a `Dockerfile`.
2. Copy `.github/workflows/ubuntu-24.04.yml` to `ubuntu-22.04.yml`.
3. Update the workflow name, job name, `paths`, `tags`, `image-name`,
   `dockerfile`, `context`, and `tag-prefix` values.
4. Open a PR for build validation, then merge to `main` to publish and clean up.
   Use the new image's matching version-tag prefix for releases.