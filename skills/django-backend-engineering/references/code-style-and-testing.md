# Code Style, Naming, Conventions & Testing

## Tooling & environment
- **`uv`** for env/deps: `uv run python ...`, `uv run pytest ...`, `uv add [--dev] <pkg>`, `uv sync`. Always run Python inside the uv venv.
- **Formatters/linters:** Black (line length **110**; PEP-8's 79 is *not* used), Ruff (`select = ["ALL"]` + explicit ignores), mypy `--strict`, docformatter. One command: `poe fix` (auto-fix) / `poe verify` (format + lint + types + migrations + celery checks). Run before committing.

## Naming
- **snake_case** functions/variables, **PascalCase** classes, **UPPER_SNAKE_CASE** constants, `_leading_underscore` private.
- **Avoid abbreviations** — write the full word (`configuration` not `cfg`, `user` not `usr`, `image_filter_contexts` not `imgFltCtx`). Common ones are fine: `id`, `url`, `http`. Exceptions to single-letter: `e` in `except`, `f` for file handles, comprehension vars where they genuinely help.
- **Abbreviations keep capitals in camelCase JSON:** `externalID`, `imageURL`, `recordID`, `DownloadHTTPError` (PEP-8 §id36). Applies to JSON response keys, not Python identifiers.
- **Exception names end in `Error`, not `Exception`** (`MyCoolError`), matching the stdlib (`KeyError`).
- **Settings:** prefix `APP_` for platform features, the vendor name for integrations; prefer long, verbose, explicit names.
- **Naming judgment:** analyze semantics first — what each name means, what role the thing plays. Parallel names for non-parallel concepts are worse than asymmetric names that encode a real difference. Don't lead with "matches the sibling" without checking the sibling is the same kind of thing.

## Python idioms
- **Type-hint everything** (params + returns); `Optional[T]` for nullable.
- **`@dataclass`** for internal value containers (memory-efficient, type-safe vs dicts; use `__slots__` for large collections); **`StrEnum`/`IntEnum`** for enumerations (`.choices()` for model fields); **Pydantic** for validated/external data.
- **Don't catch `Exception`** (pokemon handler; also swallows `SyntaxError`) — catch specific errors, or `RuntimeError` for a deliberate catch-all; new exceptions inherit `RuntimeError`. Chain with `raise X() from e`.
- **No `getattr(settings, "X", default)`** — define settings explicitly in `common.py`. Minimize `getattr` generally (only for genuinely dynamic attribute names).
- **Remove redundant conditionals** (`if value: x = value` → `x = value`; `if is_active == True` → `if is_active`; combine nested `if user: if user.is_active:` → `if user and user.is_active`). Keep conditionals that prevent errors or have side effects.
- **Imports at the top**, ordered (stdlib → third-party → first-party); no local imports unless unavoidable (circular) — add a comment when they are. Explicit imports (`from core.models import Record, Rule`).
- Comments explain non-obvious *why*, never restate the code. Prefer speaking code over comments; docstrings only for non-obvious public APIs.

## Commit messages
- Subject: **imperative mood** (`Add`, `Fix`, `Remove`, `Make`, `Use`…), capitalized, ≤50 chars (hard 72), no trailing period, PR ref `(#123)` and issue-tracker ref (`[#187929205]` / `ref ABC-123`).
- Body (when needed): blank line after subject, wrap at 72, explain **what & why** not how; include before/after behavior, SQL query plans when optimizing queries, and **list every newly introduced setting**; markdown tables for structured info.
- Branch from latest `master`/`main`; never commit directly to `master`. (Release flow is repo-specific — e.g. a master→staging→production flow driven by a release bot; squash normal PRs, merge release/backport PRs.)

## Testing (pytest)
- **Run from the directory containing the relevant `pytest.ini`** (e.g. per-app tests from `backend/<app>/`, shared-core tests from `backend/core/`; some services run from the repo root). Use `--reuse-db`; drop it (or `--create-db`) on DB errors, then re-add. `--snapshot-update` to update snapshots.
- **Module-level test functions, NOT `TestFoo` classes.** Name `test_<file_under_test>_<case>`.
- **Don't add `@pytest.mark.django_db`** where it's on by default; where it isn't, apply it as needed.
- **`@pytest.mark.parametrize`: the first argument is a tuple, never a plain string.** For >3 params use `pytest.param(..., id=...)`.
- **Create test data with real Django models / data factories / seed collections** (`tests/data_seed/collections`, `tests/data_factory`) — **never mock database functions or objects**. Mock external services and Celery `.delay` (`@patch('...publish_record.delay')`), and the factory for portal-specific classes.
- **No comments in test code** — if a test isn't self-explanatory, refactor it. Aim for ~100% coverage of edge cases without overtesting.
- **Make tests fail at least once** while writing them — guard against always-passing assertions.
- **Multi-portal:** test on at least two different portals (`@pytest.mark.parametrize("portal", [Portal.BRAND_A, Portal.BRAND_B])`, set `settings.APP_PORTAL_NAME`).
- Never run the full suite (too slow) — run tests relevant to your change (and the surrounding area, not just touched files). Test settings must not depend on environment variables (prefix `TEST_` if unavoidable).

## Localization
Never hard-code language codes or the default language. Use `settings.LANGUAGE_CODE` for the default and iterate `settings.LANGUAGES` for available languages. Query localized values with the localized-field query expressions. Portals default to different primary languages.
