# Biome LSP pnpm node_modules Bug Reproduction

This repository reproduces a bug in Biome 2.2.2 where the Language Server Protocol (LSP) completely ignores configuration exclusions in pnpm monorepos and watches all node_modules directories despite explicit `!**/node_modules` exclusions.

## 🐛 Bug Summary

- **Biome CLI**: ✅ Correctly respects `biome.jsonc` configuration  
- **Biome LSP**: ❌ Ignores configuration, watches 730+ directories including 670+ in node_modules
- **Result**: Excessive CPU usage, memory bloat, and file system watcher overload

## 📋 Environment

- **Biome Version**: 2.2.2
- **Node.js**: 24.0.2  
- **pnpm**: 10.13.1
- **VS Code**: Latest with Biome extension
- **OS**: macOS (affects `fseventsd` CPU usage)

## 🔄 Reproduction Steps

### 1. Clone and Setup
```bash
git clone https://github.com/schickling-repros/2025-09-biome-lsp-pnpm-node-modules-bug
cd 2025-09-biome-lsp-pnpm-node-modules-bug
pnpm install
```

### 2. Verify CLI Works Correctly
```bash
npx biome check --verbose
```

**Expected output**: Processes 6 files, excludes node_modules ✅

### 3. Trigger LSP Bug
1. Open this directory in VS Code
2. The bug triggers immediately when Biome LSP starts
3. Monitor CPU usage: `top -pid $(pgrep biome)`

**Expected bug behavior**: 
- Biome process consumes high CPU
- `fseventsd` process also spikes to 60-90% CPU

### 4. Verify LSP Watches Excessive Directories
```bash
# Check Biome server logs (macOS)
grep -c "watch_folder" ~/Library/Caches/dev.biomejs.biome/biome-logs/server.log.*

# Count node_modules directories being watched
grep "watch_folder" ~/Library/Caches/dev.biomejs.biome/biome-logs/server.log.* | grep "node_modules" | wc -l
```

## 📊 Evidence

### CLI Behavior ✅ (Correct)
```
Files processed: 6
- biome.jsonc
- package.json
- packages/@test/core/package.json
- packages/@test/utils/package.json  
- packages/@test/core/src/index.ts
- packages/@test/utils/src/index.ts

Checked 6 files in 3ms.
```

### LSP Behavior ❌ (Bug)
```
Directories watched: 730-884
node_modules directories watched: 670+
Memory usage: 42MB (vs 17MB expected)
```

**LSP watches 147x more directories than necessary!**

## ⚙️ Configuration Being Ignored

The LSP completely ignores this configuration:
```jsonc
{
  "files": {
    "ignoreUnknown": true,
    "includes": [
      "packages/**/*.{js,jsx,ts,tsx,json}",
      "*.{js,jsx,ts,tsx,json}",
      "!**/node_modules",  // ← This exclusion is ignored by LSP
      "!**/dist"
    ]
  }
}
```

## 🎯 Root Cause

The bug is triggered by **pnpm's nested node_modules structure**:
```
node_modules/.pnpm/package@version/node_modules/...
```

The LSP's file watching logic fails to apply configuration exclusions when encountering this nested structure, while the CLI correctly applies the same configuration.

## 🔧 Workaround

Disable Biome LSP in VS Code until this is fixed:
```json
{
  "biome.enabled": false,
  "editor.formatOnSave": false
}
```

Use CLI for formatting/linting: `npx biome format --write .`

## 🐛 Related GitHub Issue

- **Issue**: [Link will be added after issue creation]
- **Repository**: https://github.com/schickling-repros/2025-09-biome-lsp-pnpm-node-modules-bug

## 🔍 Expected Behavior

LSP should respect `biome.jsonc` configuration like CLI does:
- Watch only configured source directories  
- **Exclude** all node_modules directories as specified
- Match CLI performance and behavior exactly

---

**This bug affects any pnpm monorepo using Biome LSP in VS Code.**