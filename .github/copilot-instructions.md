# GitHub Action: Docker Build and Push

> **Purpose**: TypeScript wrapper over Docker Buildx for building and pushing container images in GitHub Actions workflows.
> 
> **Complexity Level**: Medium - TypeScript facade with version-aware logic and lifecycle management
> 
> **Key Dependencies**: @docker/actions-toolkit v0.62.1 (primary), Yarn 3.6.3 with PnP

---

## 🎯 Quick Reference

### Architecture Flow
```
GitHub Workflow → action.yml → src/main.ts → @docker/actions-toolkit → docker buildx build
```

### Essential Commands
```bash
yarn install                  # Install dependencies (Yarn 3 PnP - NOT npm!)
yarn build                    # Build TypeScript → dist/ (uses @vercel/ncc)
yarn test                     # Run Jest test suite
docker buildx bake test       # Run tests with coverage in container
docker buildx bake pre-checkin # Pre-commit checks (vendor + format + build)
```

### Critical Files
- `action.yml` - 40+ input parameters, output definitions
- `src/main.ts` - Main/post lifecycle hooks via actionsToolkit.run()
- `src/context.ts` - Input parsing, Handlebars templating, Buildx args construction
- `src/state-helper.ts` - State passing between main→post phases

---

## 🏗️ Architecture Principles

### 1. Facade Pattern Over @docker/actions-toolkit
This action doesn't reimplement Docker logic - it orchestrates the toolkit:
- Validates inputs → Constructs arguments → Delegates to toolkit → Handles outputs
- **When to modify**: Add new features by checking toolkit capabilities first, then add wrapper logic

### 2. Lifecycle Management (Main + Post Phases)
```typescript
// src/main.ts
actionsToolkit.run(
  async (toolkit) => { /* Main: Build image */ },
  async (toolkit) => { /* Post: Generate summary, upload artifacts */ }
);
```
**State persists** via `state-helper.ts` using environment variables prefixed with `STATE_`.

### 3. Version-Aware Feature Gating
Many features check Buildx version before enabling:
```typescript
if (await toolkit.buildx.versionSatisfies('>=0.12.0')) {
  args.push('--annotation', ...); // Only in v0.12+
}
```
**Impact**: New features require version checks to maintain backward compatibility.

### 4. Handlebars Template Expansion
Users can write dynamic contexts like `{{defaultContext}}:mysubdir`, which expands at runtime.
See [INPUT_HANDLING.md](.github/INPUT_HANDLING.md) for details.

---

## 🔧 Development Workflows

### Quick Start: Adding a New Feature
1. Check if `@docker/actions-toolkit` supports it (consult toolkit docs)
2. Add input to `action.yml` with description, type, required flag
3. Parse input in `src/context.ts` getInputs()
4. Add argument logic in `src/context.ts` getBuildArgs/getCommonArgs
5. Add test cases in `__tests__/context.test.ts` using test.each()
6. Run `docker buildx bake pre-checkin` before committing

**For detailed testing patterns**, see [TESTING.md](.github/TESTING.md)

### Common Development Tasks
- **Local testing**: `yarn test` (fast) or `docker buildx bake test` (CI-like)
---

## 💡 Key Patterns & Anti-Patterns

### ✅ DO: Leverage Existing Toolkit Methods
```typescript
// Good: Use toolkit's built-in validation
await toolkit.buildx.versionSatisfies('>=0.12.0')

// Bad: Manually parse version strings
```

### ✅ DO: Preserve State for Post-Execution
```typescript
// In main phase
setBuildRef(buildRef);

// In post phase (different process!)
const ref = buildRef; // Reads from process.env['STATE_buildRef']
```

### ❌ DON'T: Assume Buildx Version
Always gate new features:
```typescript
if (await toolkit.buildx.versionSatisfies('>=X.Y.Z')) {
  // Use new feature
} else {
  core.warning("Feature requires Buildx >= X.Y.Z");
}
```

### ❌ DON'T: Modify Context Without Understanding Git vs Path
- **Git context** (default): Uses remote repo, ignores local changes
- **Path context** (requires checkout): Uses local filesystem
**See**: [Common Pitfalls](#-common-pitfalls) below

---

## 🎓 Code Navigation Guide

### When working on input parsing:
→ Read `action.yml` (input definitions) → `src/context.ts` getInputs() → understand Handlebars expansion

### When working on build arguments:
→ Read `src/context.ts` getBuildArgs() + getCommonArgs() → check version gates → review test cases

### When working on testing:
→ Read `__tests__/context.test.ts` → understand test.each() pattern → check fixtures in `__tests__/fixtures/`

### When debugging CI failures:
→ Read `.github/workflows/ci.yml` → identify which docker-bake target failed → read `docker-bake.hcl` target definition

**For detailed guides**, see:
- [INPUT_HANDLING.md](.github/INPUT_HANDLING.md) - Handlebars, secrets, version gates
- [TESTING.md](.github/TESTING.md) - Jest patterns, mocking, fixtures
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - Common errors and fixestypescript
---

## ⚠️ Common Pitfalls

### 1. Git Context vs Path Context Confusion
**Problem**: By default, action uses Git context (`https://github.com/{owner}/{repo}.git#{ref}`), so local file changes aren't visible.

**Solution**: Explicitly use path context if working with modified files:
```yaml
- uses: actions/checkout@v5  # Required!
- uses: docker/build-push-action@v6
  with:
    context: .  # Path context = local filesystem
```

### 2. Build Summaries Not Appearing
**Requirements for summaries**:
- Buildx >= 0.13.0
- Not running on GHES (GitHub Enterprise Server)
- Valid build reference in metadata
- Not a `call: check` subrequest

**Debug**: Set `DOCKER_BUILD_SUMMARY=false` to disable and check for errors.

### 3. Secrets Not Available in Build
**Cause**: Secrets must be explicitly passed and properly formatted.
# 4. Version Checks Failing in Tests
**Cause**: Test mocks not configured for version requirements.

**Fix**: Add version mock to test case:
```typescript
jest.spyOn(Buildx.prototype, 'versionSatisfies').mockResolvedValue(true);
```

### 5. Yarn Commands Don't Work
**Remember**: This project uses **Yarn 3 with PnP**, NOT npm!
- ❌ `npm install` - Won't work
- ✅ `yarn install` - Correct

Dependencies are in `.yarn/` directory, not `node_modules/`.

---

## 🔗 Related Documentation

- **[TROUBLESHOOTING.md](TROUBLESHOOTING.md)** - Detailed debugging guide for build failures
- **[CONTRIBUTING.md](CONTRIBUTING.md)** - Development workflow, PR requirements
- **[README.md](README.md)** - User-facing action documentation with usage examples
- **[action.yml](action.yml)** - Complete input/output reference

### Detailed Guides (when they exist):
- `.github/INPUT_HANDLING.md` - Handlebars templating, secrets, version gates (create if needed)
- `.github/TESTING.md` - Jest patterns, mocking strategies, fixtures (create if needed)

---

## 🎯 Success Criteria for AI Agents

**You understand this codebase when you can:**
1. Add a new input parameter end-to-end (action.yml → context.ts → tests)
2. Explain why Git context is default and when to use path context
3. Write a test case using test.each() with version-specific mocking
4. Debug a build failure using state from post-execution phase
5. Add version-gated logic for a new Buildx feature

**When in doubt:**
- Check `@docker/actions-toolkit` docs first before reimplementing
- Always add version gates for new features
- Test with both local yarn commands and docker buildx bake
- Reference existing patterns in `__tests__/context.test.ts`

---

## 📦 Project-Specific Quirks

1. **Yarn 3 PnP**: Dependencies NOT in `node_modules/`, they're in `.yarn/unplugged/`
2. **Dual Build Paths**: `yarn build` (fast, local) vs `docker buildx bake build` (CI-accurate)
3. **Handlebars Expansion**: Input contexts are Handlebars templates, not plain strings
4. **State via ENV**: Inter-phase communication uses `process.env['STATE_*']`, not in-memory variables
5. **Extensive Mocking**: Tests mock 100+ scenarios with parameterized test cases
6. **Version-First Design**: Always check Buildx version before using features
  if (res.exitCode != 0) {
    if (inputs.call === 'check') {
      // Parse check warnings from stdout
      err = new Error(res.stdout.split('\n')[0]?.trim());
    } else {
      // Extract last line of stderr
      err = new Error(`buildx failed with: ${res.stderr.match(/(.*)\s*$/)?.[0]?.trim()}`);
    }
  }
});

// Throw error after outputs are set (allows partial success)
if (err) { throw err; }
```

## Docker Bake Integration

`docker-bake.hcl` defines reusable build targets for CI:
```dockerbake
group "pre-checkin" {
  targets = ["vendor", "format", "build"]
}

target "build" {
  dockerfile = "dev.Dockerfile"
  target = "build-update"
  output = ["."]  # Extracts compiled JS to host
}
```

**Usage in workflows:** `docker buildx bake test` runs tests inside a container with coverage output.

## Project-Specific Quirks

- **No npm/npx:** Uses Yarn 3 with PnP (Plug'n'Play) - dependencies in `.yarn/` directory
- **Dual compilation paths:** `yarn build` (local) vs `docker buildx bake build` (containerized)
- **Test mocking:** Extensive use of `jest.spyOn()` to mock toolkit dependencies (100+ test cases)
- **Versioned APIs:** Many code paths check Buildx/BuildKit versions to enable/disable features
- **Handlebars in inputs:** `context` and `build-contexts` support Handlebars template expansion

## Troubleshooting

**Common issues (from TROUBLESHOOTING.md):**
- Push failures → Enable BuildKit debugging: `buildkitd-flags: --debug` in setup-buildx
- "repository name must be lowercase" → Use `docker/metadata-action` to sanitize tags
- Registry auth issues → Usually registry-side; test with containerd to isolate BuildKit

**Key debugging commands:**
```bash
docker buildx ls                    # List builders
docker buildx inspect <builder>     # Inspect builder config
docker buildx imagetools inspect <image>  # Verify pushed image
```
