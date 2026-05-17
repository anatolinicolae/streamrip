# Release a new patch version of streamrip

## Steps

1. **Find all version strings**
   ```bash
   grep -r "x\.y\.z" --include="*.py" --include="*.toml" -l .
   ```
   Version lives in exactly two places:
   - `pyproject.toml` → `version = "x.y.z"`
   - `streamrip/__init__.py` → `__version__ = "x.y.z"`

2. **Bump both files** to the next patch version (e.g. 2.3.2 → 2.3.3)

3. **Run tests**
   ```bash
   uv sync --group dev
   uv run pytest
   ```
   All must pass (7 credential-gated skips are expected).

4. **Commit**
   ```bash
   git add pyproject.toml streamrip/__init__.py uv.lock
   git commit -m "chore: bump version to vX.Y.Z"
   ```

5. **Tag and push**
   ```bash
   git tag vX.Y.Z
   git push origin dev
   git push origin vX.Y.Z
   ```

6. **Create GitHub release** (triggers publish workflow → PyPI)
   ```bash
   gh release create vX.Y.Z --repo anatolinicolae/streamrip --title "vX.Y.Z" --generate-notes --latest
   ```

## Notes

- The publish workflow (`.github/workflows/publish.yml`) fires on `release: published` and runs `uv build`. Ensure PyPI credentials are configured as repo secrets if upload is wired up.
- The CI ruff format check must be clean before pushing — run `uv run ruff format streamrip/ tests/` if needed.
