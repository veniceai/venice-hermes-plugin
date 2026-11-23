# Publishing to PyPI

Releases publish from GitHub Actions through PyPI trusted publishing, so no
PyPI token is involved.

1. Choose a new version and update it in:
   - `pyproject.toml`
   - `src/hermes_provider_venice/__init__.py`
   - `src/hermes_provider_venice/plugin.yaml`

2. Refresh the lockfile and run the checks:

   ```bash
   uv lock
   uv run pytest -q
   uv run ruff check .
   uv run ruff format --check .
   ```

3. Merge the release to `main`, then tag the merge commit and push the tag:

   ```bash
   git switch main
   git pull --ff-only
   git tag -s v<version> -m v<version>
   git push origin v<version>
   ```

4. Create a GitHub release for the tag:

   ```bash
   gh release create v<version> --verify-tag --generate-notes
   ```

   Publishing the release runs
   [`.github/workflows/publish.yml`](../.github/workflows/publish.yml). It
   checks that the tag matches the package version and is on `main`, runs the
   checks, builds the distributions with uv and uploads them from the `pypi`
   environment.

5. Ask another maintainer to approve the `pypi` deployment, then confirm that
   the new version is on PyPI.

PyPI does not allow replacing an existing release. If publishing fails because
the version already exists, increment the version and release again.

## Release protection

The workflow runs from the tagged commit, so its checks catch mistakes, such as
a tag on an unmerged commit, but don't stop someone with write access from
publishing. Before the workflow publishes anything, these settings must be in
place:

- The `pypi` environment requires a reviewer, prevents self-review and only
  allows `v*` tags to deploy.
- A tag ruleset limits who can create, update and delete `v*` tags.
- The PyPI trusted publisher names the `pypi` environment, so a workflow
  without `environment: pypi` can't publish.

## Publishing by hand

Publish by hand only when the workflow can't run, such as for a tag that
predates it. After steps 1 and 2 above:

1. Commit the reviewed release changes and confirm the release is reproducible
   from a clean working tree:

   ```bash
   git diff --check
   git status --short
   ```

   `git status --short` should produce no output. Do not publish from an
   uncommitted or partially staged tree.

2. Build fresh distribution artifacts:

   ```bash
   uv build --clear
   ```

3. Verify the upload without publishing:

   ```bash
   uv publish --dry-run --trusted-publishing never
   ```

4. Set a PyPI token and publish:

   ```bash
   read -rsp "PyPI token: " UV_PUBLISH_TOKEN && echo
   export UV_PUBLISH_TOKEN
   uv publish
   ```

Never commit or print the PyPI token.

# Submitting to the Hermes plugin catalog

The [Hermes plugin catalog](https://hermes-agent.nousresearch.com/docs/developer-guide/plugins/catalog-submission)
installs the plugin from GitHub at an exact commit. Catalog users get a release
only after a reviewed pull request to `NousResearch/hermes-agent` pins it.

1. Merge the release to `main` and copy its full 40-character commit SHA.
   Never rewrite history past a pinned commit: the catalog delists entries whose
   commit disappears.

2. Run the catalog check from [`DEVELOPMENT.md`](../DEVELOPMENT.md) on that
   commit and fix every failure and warning.

3. Open a pull request that adds `plugin-catalog/venice.yaml`. Leave `tier` as
   `community`; Nous maintainers set `verified` for plugins published by the
   vendor itself. Keep `requires_hermes` and `version` equal to `plugin.yaml`.

   ```yaml
   name: venice
   title: Venice AI
   repo: https://github.com/veniceai/venice-hermes-plugin
   sha: <full commit SHA>
   subdir: src/hermes_provider_venice
   description: "Venice AI — privacy-first, OpenAI-compatible inference. Disclosure — sends your prompts, tool definitions, tool results and VENICE_API_KEY to api.venice.ai (or VENICE_BASE_URL); Venice passes requests for models its catalog marks anonymized, such as Claude, GPT and Gemini, on to the company that serves them. The plugin downloads Venice's model catalog with your key, sometimes on a short-lived background thread, keeps it in memory only, and reads or writes no files."
   maintainer: veniceai
   tier: community
   category: models
   requires_hermes: ">=0.22.6"
   docs_url: https://github.com/veniceai/venice-hermes-plugin#readme
   version: "<version>"
   capabilities:
     provides_tools: []
     provides_hooks: []
     provides_middleware: []
     requires_env:
       - VENICE_API_KEY
   ```

4. In the pull request description, say what the plugin does, that it uses only
   `ProviderProfile` hooks (`build_extra_body`, `build_api_kwargs_extras`,
   `fetch_models`, `live_context_length`, `get_usage_cost`,
   `invalidate_catalog_cache` and the `model_capabilities` field), and repeat the
   disclosure.

To ship a later release to catalog users, open a new pull request that changes
`sha` and `version` together.
