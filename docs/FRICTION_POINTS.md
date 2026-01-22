# Friction Points

Issues encountered during development and refactoring tasks.

## 2026-01-19: Code Compliance Audit

### sed Variable Replacement Hits String Literals

**Problem:** Using `sed` with word boundary matching (`\b`) to rename variables like `n` → `written` also matched escape sequences inside string literals like `\n`, turning them into `\written`.

**Example:**
```c
// Before sed replacement
fprintf(file_handle, "\n");

// After sed 's/\bn\b/written/g'
fprintf(file_handle, "\written");  // BROKEN!
```

**Root cause:** `\b` in sed matches word boundaries, but `\n` in a C string has `n` preceded by `\` which is not considered a word character, so the `n` matches as a word boundary.

**Workaround:** After bulk sed replacements, run a fixup:
```bash
sed -i 's/\\written/\\n/g' file.c
```

**Better approach:** Use more specific patterns that include context:
```bash
# Instead of:
sed 's/\bn\b/written/g'

# Use patterns that require C variable context:
sed 's/int n;/int written;/g'
sed 's/size_t n;/size_t written;/g'
```

**Recommendation:** For variable renaming in C code, prefer tools that understand C syntax (like `clang-rename` or semantic search/replace) over regex-based tools. Alternatively, use sift_edit with SQL mode for more precise matching.

---

### sift_edit vs sed for Bulk Replacements

**Observation:** For bulk variable renaming across files, `sed -i` was faster to invoke but more error-prone. `sift_edit` with `replace_all: true` would be safer but requires more invocations.

**Trade-off:**
- `sed`: Fast, but no syntax awareness, can break code
- `sift_edit`: Safer fuzzy matching, but one file at a time

**Potential improvement:** A `sift_refactor` tool that understands C variable scoping could make bulk renames safer.

---

### Magic Number Replacement Scope

**Challenge:** Replacing magic numbers like `[256]` with named constants requires understanding the semantic purpose of each buffer (error message vs filename vs SQL query).

**Approach taken:**
1. Grep for patterns to understand contexts
2. Define semantically meaningful constants (`ERROR_BUFFER_SIZE`, `SQL_BUFFER_SIZE`, etc.)
3. Replace in batches by context

**Friction:** Some buffer sizes are used for multiple purposes, making it hard to choose the "right" constant name. Settled on the most common use case for each.

---

### sed Line Insertion Position

**Problem:** Using `sed -i '1a\...'` to insert lines at the start of a file inserts after line 1, which in C files with copyright headers puts the code inside comments or before #includes.

**Example:**
```bash
# Intended to add defines to a .c file
sed -i '1a\#define FOO 42' file.c

# Result: defines inserted after first line of copyright comment
/* Copyright...
#define FOO 42  <- WRONG POSITION
 * ...
```

**Workaround:** Find the correct insertion point (after includes, before code):
```bash
# Insert after last #include
sed -i '/^#include/a\#define FOO 42' file.c
```

**Better approach:** Use `sift_edit` with `insert_after` specifying a line number after reviewing the file structure.

---

### Array Size vs Buffer Size Distinction

**Observation:** Not all `[N]` patterns are buffer sizes. Some are:
- Lookup table dimensions: `base64_decode_table[256]`
- 2D array dimensions: `words[16][32]`
- Array indices: `array[count - 1]`
- Small fixed counts: `int flags[10]`

**Friction:** Bulk replacement of `[256]` → `[NAME_BUFFER_SIZE]` can break non-buffer uses.

**Recommendation:** Review grep output carefully before bulk replacement. Consider separate patterns for different semantic uses.

---

### File Overwrite Ordering with unexpand

**Problem:** Using `unexpand` for tab conversion followed by `sed` for brace style fixes requires careful ordering. The temp file approach can cause changes to be lost if not properly sequenced.

**Example failure pattern:**
```bash
unexpand -t 2 file.c > /tmp/temp.c && mv /tmp/temp.c file.c
# Later sed or sift_update changes get lost if another unexpand runs
```

**Root cause:** The `unexpand` command reads from disk, so any in-progress edits not yet flushed can be lost.

**Solution:** Use `cat temp > file` instead of `mv temp file`, and verify changes between operations:
```bash
unexpand -t 2 file.c > /tmp/temp.c && cat /tmp/temp.c > file.c
# Verify with: od -c file.c | head
```

---

### Function Brace Style Fix Complexity

**Problem:** Fixing function opening brace style (from same line to new line) requires:
1. Multi-line function signatures need joining first
2. Single-line function definitions (e.g., `int foo() { return 0; }`) need different handling
3. Pattern must not match control structures (if/for/while)

**Solution used:**
```bash
# 1. Join wrapped signatures
awk '/^(static )?(int|void|...) \*?[a-z_]+\([^)]*$/ {
    line = $0; while (line !~ /\)/) { getline next; line = line " " next }
    print line; next
} { print }' file.c

# 2. Fix standard function braces
sed -E -i 's/^((static )?(int|void|...) \*?[a-z_]+\([^)]*\)) \{$/\1\n{/' file.c

# 3. Manually fix single-line functions
```

---

### Trailing Comments Volume

**Problem:** CODING_STANDARDS.md requires comments above code, not trailing. The codebase had 183 trailing comments.

**Example:**
```c
// Wrong (trailing)
int line_width = 4; /* Minimum width */

// Correct (above)
/* Minimum width */
int line_width = 4;
```

**Solution:** Used awk to automatically move trailing comments:
```bash
awk '
/;\s*\/\*.*\*\/$/ {
    match($0, /\/\*.*\*\//)
    comment = substr($0, RSTART, RLENGTH)
    code = substr($0, 1, RSTART - 1)
    gsub(/\s+$/, "", code)
    match($0, /^[\t ]*/)
    indent = substr($0, RSTART, RLENGTH)
    print indent comment
    print code
    next
}
{ print }
' file.c > temp.c && cp temp.c file.c
```

**Result:** All 183 trailing comments successfully moved above their respective code lines.
