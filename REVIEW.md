# Review: empty Unix-time binding regression

## Finding — inconsistent empty-time semantics and failing test expectations

The defective change is upstream [PR #4103, “fix(time): binding time with empty value”](https://github.com/gin-gonic/gin/pull/4103), merged on May 21, 2025. The affected files are **`binding/form_mapping.go`** (runtime behavior) and **`binding/form_mapping_test.go`** (the accompanying tests). Both parts of that change are present in this work branch.

### Evidence in the reviewed tree

- In [`binding/form_mapping.go:394–406`](binding/form_mapping.go#L394-L406), `setTimeField` checks `val == ""` and assigns `time.Time{}` before dispatching on `time_format`. Consequently, empty Unix-format input succeeds with `0001-01-01 00:00:00 +0000 UTC`, not the Unix epoch. The early return bypasses all four numeric formats supported by this tree: `unix`, `unixmilli`, `unixmicro`, and `unixnano`.
- The distinction matters: explicit `"0"` reaches the numeric conversion at [`binding/form_mapping.go:405–425`](binding/form_mapping.go#L405-L425), producing the 1970 epoch; empty input takes the year-1 path instead. Go's zero `time.Time` is not Unix timestamp zero.
- [`binding/form_mapping_test.go:189–190`](binding/form_mapping_test.go#L189-L190) declares `ZeroUnixTime` and `ZeroUnixNanoTime` with the respective format tags. Lines [203–204](binding/form_mapping_test.go#L203-L204) supply present keys with empty value slices. The scalar binding path leaves `val` empty at [`binding/form_mapping.go:276–291`](binding/form_mapping.go#L276-L291), and the `time.Time` dispatch calls `setTimeField` at [329–332](binding/form_mapping.go#L329-L332).
- Yet [`binding/form_mapping_test.go:214–215`](binding/form_mapping_test.go#L214-L215) asserts `1970-01-01 00:00:00 +0000 UTC` for both fields. Those assertions contradict the implementation's unconditional year-1 assignment. This is a concrete implementation/test mismatch, not merely a preference about empty-date semantics.

The same runtime path is reached by a supplied empty form/query parameter, for example `created_at=` bound to `time.Time` with `form:"created_at" time_format:"unix"`. An entirely absent key without a default is different: `setByForm` returns without binding it. The reported regression concerns supplied empty values.

### Historical corroboration

- [Issue #4098](https://github.com/gin-gonic/gin/issues/4098) reported an error for `created_at=` with `time_format:"unix"` and explicitly expected the Unix epoch. PR #4103 attempted to address that error, but moved the generic empty-value check ahead of numeric parsing rather than making the result match its epoch expectations.
- The [introducing commit](https://github.com/gin-gonic/gin/commit/674522db91d637d179c16c372d87756ea26fa089) moved that check and added the two epoch assertions described above.
- [PR #4245](https://github.com/gin-gonic/gin/pull/4245), merged May 22, 2025, reverted PR #4103. Its [revert commit](https://github.com/gin-gonic/gin/commit/8fb3136664254d7c592127f00d52849caba18a67) changed exactly these two files: it restored the empty-value check below the Unix-format switch and removed the new Unix-empty test fields and assertions. This confirms the subsequent correction was a revert, not an implemented epoch-normalization fix.

The mirror's local history contains a single snapshot even after unshallowing; historical attribution above was checked through read-only GitHub API requests for the upstream PRs and commits, rather than inferred from local blame.

### Verification and scope

Attempted `go test ./binding -run '^TestMappingTime$' -count=1`; execution was blocked with `/bin/bash: line 1: go: command not found`. No Go executable was found in the checked standard installation locations. The expected two assertion failures are established by the source trace above, not claimed as executed test results.

No fix was implemented. Only this review document was changed.
