# Pull Request Review: #17 - temporarily add polyfill for serde_json

## Summary
This PR proposes to replace the standard `serde_json` dependency with a git-based temporary fix to address compilation issues in certain configurations.

## Changes Overview
1. **Cargo.toml**: Replaces `serde_json = "1.0.89"` with a git dependency pointing to `https://github.com/slin99/tmpfix`
2. **src/main.rs**: Adds a test case to verify the fix works with account names containing special characters (e.g., "a:a")

## Review Findings

### 🔴 Critical Issues

1. **Dependency from Unverified Source**
   - The PR introduces a git dependency from a personal repository (`https://github.com/slin99/tmpfix`)
   - This repository is not verified or audited
   - **Concern**: This introduces a supply chain security risk. The git repository could be:
     - Modified or deleted at any time
     - Could contain malicious code
     - May not receive security updates
   - **Impact**: High - This affects the entire application's security posture

2. **Missing Context and Documentation**
   - The PR description mentions "weird json that crashes serde on some configs" but provides no:
     - Error messages or stack traces
     - Reproduction steps
     - Specific configurations that fail
     - Link to the mailing list discussion referenced
   - **Impact**: Medium - Makes it difficult to verify the issue exists or that the fix is appropriate

3. **No Verification of the Fix**
   - The temporary fix repository is not examined
   - No comparison provided between the fix and standard serde_json
   - Unknown what changes were made to serde_json to create this "polyfill"
   - **Impact**: High - Cannot verify correctness or safety of the fix

### ⚠️ Major Issues

4. **Test Quality**
   - The test in `src/main.rs` appears to test the `export` function but:
     - Test location: Should be in a tests module or the export module itself
     - No assertions: The test calls `export()` but doesn't verify anything
     - Return value ignored: The test doesn't check if export succeeded or failed
     - Side effects: The test likely creates files without cleanup
   ```rust
   #[test]
   fn test_json() {
       use crate::account::export::{export, Config};
       // good
       export(Config { account_name: "a".into() });
       // crashed before
       export(Config { account_name: "a:a".into() });
   }
   ```

5. **Temporary Solution Without Exit Strategy**
   - PR title says "temporarily" but there's no:
     - Timeline for when this will be replaced
     - Issue filed upstream
     - TODO comments in code
     - Plan to migrate back to official serde_json

### 💡 Minor Issues

6. **Code Formatting**
   - Missing space in `Cargo.toml`: `serde_json=` should be `serde_json =`

## Root Cause Analysis

Looking at the code in `src/account/export.rs`, the `normalize_name` function already handles special characters:
```rust
fn normalize_name(account_name: &str) -> String {
    account_name
        .chars()
        .map(|c| if char::is_alphanumeric(c) { c } else { '_' })
        .collect()
}
```

This function converts special characters (including `:`) to underscores, so account name "a:a" becomes "a_a" in the archive filename. The serialization of account data to JSON should not be affected by the account name containing special characters, as the name is just a string value.

**Question**: What exactly is crashing in serde_json? Is it:
- The serialization of the account name itself?
- The serialization of some other data structure?
- A specific platform/configuration issue?

## Recommendations

### 🛑 Do Not Merge As-Is

This PR should **NOT** be merged in its current state due to security concerns and lack of proper verification.

### ✅ Recommended Actions

1. **Investigate the Root Cause**
   - Provide specific error messages and stack traces
   - Create a minimal reproducible example
   - Determine if this is truly a serde_json issue or an application issue

2. **If It's a serde_json Bug**
   - File an issue in the official serde_json repository: https://github.com/serde-rs/json
   - Check if newer versions of serde_json (currently at 1.0.89 in this repo, latest is 1.0.132+) fix the issue
   - Engage with the serde maintainers to get an official fix

3. **Security Alternative**
   - If a temporary workaround is absolutely necessary:
     - Fork serde_json to a verified/official organization account
     - Document the exact changes made
     - Add checksums/verification
     - Set up automated security scanning
     - Create a migration plan

4. **Improve Testing**
   - Move test to appropriate location (either `src/account/export.rs` or `tests/` directory)
   - Add proper assertions to verify behavior
   - Add cleanup logic for any side effects
   - Consider using a test helper that creates temporary directories

5. **Better Documentation**
   - Add a TODO comment in Cargo.toml explaining the temporary dependency
   - Link to the upstream issue
   - Add a deadline or version target for reverting to official serde_json
   - Update README.md if this affects users

### 📋 Example of Improved Test

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use tempfile::TempDir;

    #[test]
    fn test_export_with_special_characters() {
        // This test verifies that account names with special characters
        // can be exported without causing serde_json errors
        
        let test_dir = TempDir::new().unwrap();
        // Setup test account...
        
        let config = Config { 
            account_name: "test:account".into() 
        };
        
        let result = export(config);
        
        // Verify the export succeeded
        assert!(result.is_ok(), "Export should succeed with special characters");
        
        // Verify the archive was created with normalized name
        assert!(test_dir.path().join("gdrive_export-test_account.tar").exists());
    }
}
```

## Questions for PR Author

1. Can you provide the specific error message you're encountering?
2. What configuration causes the crash? (OS, Rust version, etc.)
3. Can you link to the mailing list discussion mentioned?
4. What changes did you make in the `slin99/tmpfix` repository?
5. Have you tested with newer versions of serde_json (1.0.90+)?
6. Have you considered reporting this to the serde_json maintainers?
7. What is the timeline for replacing this temporary fix?

## Conclusion

While the PR attempts to address a real issue, the approach taken introduces significant security and maintainability concerns. The issue needs more investigation, and if a workaround is truly needed, it should be implemented in a more transparent and secure manner.

**Recommendation**: Request Changes - Do not merge until security concerns are addressed and proper verification is provided.
