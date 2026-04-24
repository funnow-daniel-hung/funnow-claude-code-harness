---
name: swagger-conventions
description: Use when writing or reviewing Swagger/swaggo doc comments in Go API handlers. Covers correct param types, header declarations, formatting tool, and common mistakes.
---

# Swagger Conventions

## Accept-Language Header

**WRONG:**
```go
// @Param       Accept-Language  header  string  false  "en-US, zh-TW, zh-CN, ja-JP, ms-MY, th-TH"
```

**CORRECT:**
```go
// @Param       Accept-Language header   lang.ReqHeader         false "Language Tag"
```

Use `lang.ReqHeader` as the type, not `string`. The description is just `"Language Tag"`.

## Quick Reference

| Field       | Value            |
|-------------|------------------|
| param name  | `Accept-Language` |
| location    | `header`          |
| type        | `lang.ReqHeader`  |
| required    | `false` (usually) |
| description | `"Language Tag"`  |

## Common Mistakes

- Using `string` as type → use `lang.ReqHeader`
- Listing language codes in description → just use `"Language Tag"`

## Formatting: Always Run `go-swag fmt` After Changes

After writing or modifying ANY swagger annotation, run:

```bash
go-swag fmt <directory>
```

Examples:
```bash
go-swag fmt api/v1/branch
go-swag fmt api/admin/branch
```

The `<directory>` is the directory containing the modified `.go` file(s). This tool ensures consistent column alignment in swagger comments and prevents pre-push hook failures from `gci`/`swag fmt` checks.

**Rule**: Never commit swagger annotation changes without running `go-swag fmt` first.
