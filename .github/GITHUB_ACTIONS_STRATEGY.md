# GitHub Actions Build Strategy

This document explains the optimized GitHub Actions workflows for MacDown.

## Overview

MacDown uses a **smart build strategy** to balance:
- ✅ Fast feedback during development (Apple Silicon only on push)
- ✅ Full validation before merging (both architectures on PR)
- ✅ Comprehensive releases (both architectures on tags)
- ✅ Resource efficiency (minimal unnecessary builds)

## Build Matrix Strategy

```
Push Event
│
├─ Apple Silicon (arm64)  ✅ Always builds
│  ├─ Runner: macos-14
│  ├─ Speed: 5-10 minutes
│  └─ Feedback: Fast (ideal for development)
│
└─ Intel (x86_64)  ⏭️  Skipped (save resources)

Pull Request
│
├─ Apple Silicon (arm64)  ✅ Builds
│  ├─ Runner: macos-14
│  └─ Speed: 5-10 minutes
│
└─ Intel (x86_64)  ✅ Builds
   ├─ Runner: macos-13
   └─ Speed: 10-15 minutes
   → Full validation before merge

Release Tag (v*.*.*)
│
├─ Apple Silicon (arm64)  ✅ Builds
│  ├─ Runner: macos-14
│  └─ Creates: MacDown-v1.0.0-arm64.dmg
│
└─ Intel (x86_64)  ✅ Builds
   ├─ Runner: macos-13
   └─ Creates: MacDown-v1.0.0-x86_64.dmg
   → Both DMGs in GitHub release
```

## Scenario Examples

### Scenario 1: Daily Development (Push to branch)

```
$ git push origin feature/new-feature
```

**Workflow runs:**
- ✅ Build for Apple Silicon (arm64)
  - Completes in ~5-10 minutes
  - Fast feedback on whether your changes break things

**Skips:**
- ⏭️ Intel (x86_64) build
  - Saves time and resources
  - Not needed for quick iteration

**Best for:** Quick iteration, early feedback

### Scenario 2: Code Review (Pull Request)

```
$ gh pr create
```

**Workflow runs:**
- ✅ Build for Apple Silicon (arm64)
- ✅ Build for Intel (x86_64)
- ✅ Run tests on both architectures
- ✅ Artifacts available in PR

**Why both:** Full validation before merging ensures compatibility

**Best for:** Quality assurance, code review

### Scenario 3: Release (Create Tag)

```
$ git tag v1.0.0
$ git push origin v1.0.0
```

**Workflow runs:**
- ✅ Build Apple Silicon
  - Creates: MacDown-v1.0.0-arm64.dmg
- ✅ Build Intel
  - Creates: MacDown-v1.0.0-x86_64.dmg
- ✅ Combine artifacts
- ✅ Create GitHub release with both DMGs

**Result:** Users can download appropriate version

**Best for:** Production releases with broad compatibility

### Scenario 4: Testing Intel Build (Manual)

If you need to test Intel build without creating a PR:

1. Go to: **Actions → Build → Run workflow**
2. Check: **"Also build Intel (x86_64)?"**
3. Click: **"Run workflow"**

**Result:**
- ✅ Apple Silicon builds (always)
- ✅ Intel builds (manually triggered)
- ✅ Both artifacts available

**Best for:** Debugging Intel-specific issues

## Build Time Comparison

```
Scenario                     Build Time    Wait Time
─────────────────────────────────────────────────────
Push (Apple Silicon only)    5-10 min      2-5 min
                             ────────      ─────────
                             Total: 7-15 min

PR (Both architectures)      5-10 + 10-15  2-5 min
                             ──────────    ─────────
                             Total: 17-30 min

Release (Both architectures) 5-10 + 10-15  2-5 min
                             ──────────    ─────────
                             Total: 17-30 min
```

## Workflow Configuration

### build.yml (Push & PR)

```yaml
on:
  push:
    branches: [ master, main, develop ]
  pull_request:
    branches: [ master, main, develop ]
  workflow_dispatch:  # Manual trigger
    inputs:
      build_intel:
        description: 'Also build Intel?'
        type: boolean
        default: false

jobs:
  build:
    strategy:
      matrix:
        include:
          - os: macos-14
            arch: arm64
            name: "Apple Silicon"
          - os: macos-13
            arch: x86_64
            name: "Intel"
            skip_on_push: true  # ← Skip on push

    # Conditional: Skip Intel on push events
    if: |
      !matrix.skip_on_push ||
      github.event_name == 'pull_request' ||
      github.event_name == 'workflow_dispatch' && 
      github.event.inputs.build_intel == 'true'
```

### release.yml (Tags)

```yaml
on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  build-and-release:
    strategy:
      matrix:
        include:
          - os: macos-14
            arch: arm64
            name: "Apple Silicon"
          - os: macos-13
            arch: x86_64
            name: "Intel"
    # Both architectures always build for releases
```

## Features

### Caching

Both workflows use CocoaPods caching:

```yaml
- name: Cache CocoaPods
  uses: actions/cache@v3
  with:
    path: Pods
    key: ${{ runner.os }}-pods-${{ hashFiles('**/Podfile.lock') }}
```

**Cache behavior:**
- ✅ **Hit**: Pods unchanged → <1 min installation
- ❌ **Miss**: Podfile.lock changed → 5-10 min installation
- 🔄 **Expires**: After 7 days of no activity

### Concurrency

Pull builds (not releases):

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

**Behavior:**
- ✅ New push cancels old builds on same branch
- ✅ Saves time and resources
- ✅ Doesn't cancel release builds

### Timeout Protection

```yaml
timeout-minutes: 60   # Build jobs
timeout-minutes: 90   # Release jobs
```

Prevents jobs from hanging indefinitely.

## Resource Impact

### Before (All push builds Intel + ARM)
- 2 runners per push × 15-20 min = 30-40 min total
- High queue time for Intel runner
- ~1 hour total per push for developer

### After (Push Apple Silicon only)
- 1 runner per push × 7-15 min = 7-15 min total
- Fast feedback
- ~15 min total per push for developer

**Savings: 50-75% faster feedback on push!**

## When Each Workflow Runs

| Event | Build.yml | Release.yml | Effect |
|-------|-----------|-------------|--------|
| `git push` | ✅ ARM64 | — | Quick feedback |
| Pull Request | ✅ ARM64 + x86_64 | — | Full validation |
| `git tag v*` | — | ✅ ARM64 + x86_64 | Create release |
| Manual trigger | ✅ ARM64 + x86_64 (optional) | — | On-demand full build |

## Troubleshooting

### "Why didn't Intel build on my push?"

✅ **Expected behavior** - Intel builds only run on PRs and releases to save resources.

**To build Intel:**
1. Create a Pull Request, or
2. Manually trigger: Actions → Build → Run workflow → Check "Also build Intel"

### "Can I build Intel without a PR?"

✅ **Yes!** Use workflow_dispatch:
1. Go to Actions tab
2. Select "Build" workflow
3. Click "Run workflow"
4. Check "Also build Intel (x86_64)?"
5. Click "Run workflow"

### "Why is my push build so fast?"

✅ **Apple Silicon only!** Takes 5-10 minutes instead of 20-30 for both architectures.

**Benefits:**
- Faster feedback during development
- More efficient resource usage
- Full validation still happens on PRs

## Best Practices

1. **Development**: Push to your feature branch → Quick ARM64 build
2. **Before Review**: Create PR → Full validation (both architectures)
3. **Release**: Create tag → Build both, create release with both DMGs
4. **Debugging**: Use workflow_dispatch for on-demand Intel builds

## Monitoring Builds

### View Build Status

1. Go to: **Actions tab**
2. Select workflow (Build or Release)
3. See status of each run

### Download Artifacts

During builds:
- Build artifacts available in PR (if enabled)
- All artifacts available in release

### Check Runner Info

Each step shows:
```
Building: Apple Silicon (arm64)
Runner: macos-14
Event: push
```

## Future Optimizations

Possible future improvements:
- [ ] Cache Xcode build artifacts
- [ ] Parallel test execution
- [ ] Build time trending
- [ ] Automatic performance reports
- [ ] Conditional builds based on changed files

## References

- [GitHub Actions Documentation](https://docs.github.com/actions)
- [Conditional Workflows](https://docs.github.com/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idif)
- [Caching Best Practices](https://docs.github.com/actions/using-workflows/caching-dependencies-to-speed-up-workflows)
- [macOS Runners](https://docs.github.com/actions/using-github-hosted-runners/about-github-hosted-runners#supported-runners-and-hardware-resources)
