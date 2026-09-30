# Agent Note: A walked manifest no longer fails a request as a malformed package

Status: implemented

English | [中文](2026-09-30-walked-manifest-loose-module.zh.md)

## Problem

Every official DeepSeek request from the `web` profile failed with `DeepSeek request extension preparation failed REQUEST_EXTENSION`, on the first turn and on every turn after it. `~/.dsh/profiles/web/cordis.patch.yml` inserts `dsh-probe-plugin`, whose entry is the relative path `./probe-plugin.mjs`. Walking up from that file reaches `~/.dsh/profiles/web/package.json`, the profile root, which declares `"name": "dsh-profile-web"` and `"private": true` and declares no `version`.

The inventory read any nearest manifest as a package identity, so a name without a version was malformed package metadata, and the throw became the request's `REQUEST_EXTENSION` failure. The root cause is that a `dsh` profile root is a container for plugin modules, not the package that owns them; npm lets a private root omit `version`, so requiring one there rejects a valid profile.

## Decision

- Whether a manifest owns its entry is the deciding fact, not what the manifest happens to declare. Node resolution proved ownership for a bare package specifier, so that manifest must carry a non-empty `name` and `version` and still fails request preparation otherwise.
- A relative or absolute module reached its manifest by walking parent directories, and that walk stops at the first manifest whether or not it owns the module. An incomplete walked manifest yields no package identity, which marks the entry a loose module and omits it from `dsh_plugin_packages`.
- The incomplete-manifest failure is unchanged for owning manifests, so a published package that drops `name` or `version` remains a loud request failure.

## Alternatives considered

**Drop the fail-loud check for every manifest.** Rejected because it would hide a genuinely broken published package: a bare specifier that resolved to a manifest without both fields has no usable identity, and the request cannot describe it.

**Validate the profile root's own declaration instead.** Rejected because a profile root is written by `dsh plugin` and by hand, and the harness already accepts its `name`-only shape everywhere else. The request path cannot be stricter about it than the loader that composed it.

**Fix the probe plugin's specifier to a bare package name.** Rejected because that repairs one profile and leaves every relative plugin module beneath any versionless root broken.

## Testing

[inventory.spec.ts](../../../../packages/llm/plugin-package-inventory-deepseek/tests/inventory.spec.ts) pins the reported failure three ways: a bare package whose manifest omits `version` still fails preparation, a relative module under a named private root with no `version` is omitted instead, and a relative module whose nearest manifest declares both fields still contributes its identity.

## Related

[The DeepSeek request extensions decision](../architecture/2026-08-21-deepseek-llm-api-request-extensions.md) owns the field, its registry, and the `REQUEST_EXTENSION` failure envelope; this note owns only which manifests may be read as a package identity.

## Consequences

A relative or absolute plugin module beneath a versionless or unnamed root contributes no entry, so a deployment inventory under-reports that plugin instead of failing every request. A bare package that cannot be resolved at all already failed before this change and still does.
