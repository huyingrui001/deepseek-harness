# Agent Note: 走查所得的 manifest 不再作为格式错误的包使请求失败

Status: implemented

[English](2026-09-30-walked-manifest-loose-module.md) | 中文

## Problem

`web` profile 发出的每个官方 DeepSeek 请求都以 `DeepSeek request extension preparation failed REQUEST_EXTENSION` 失败，首轮失败，之后每轮同样失败。`~/.dsh/profiles/web/cordis.patch.yml` 插入了 `dsh-probe-plugin`，其配置项是相对路径 `./probe-plugin.mjs`。从该文件向上走查会到达 `~/.dsh/profiles/web/package.json`，即 profile 根；它声明了 `"name": "dsh-profile-web"` 与 `"private": true`，但没有声明 `version`。

清单把任何“最近 manifest”都读成包身份，因此没有版本的具名 manifest 就成了格式错误的包元数据，该抛错又变成请求的 `REQUEST_EXTENSION` 失败。根因是：`dsh` profile 根是插件模块的容器，而不是拥有这些模块的包；npm 允许私有根省略 `version`，因此在这里要求版本会拒绝一个合法的 profile。

## Decision

- 决定性事实是 manifest 是否拥有它对应的配置项，而不是 manifest 恰好声明了什么。裸包说明符由 Node 解析证明所有权，因此该 manifest 必须声明非空 `name` 与 `version`，否则仍会使请求准备失败。
- 相对与绝对模块的 manifest 来自向上走查父目录，该走查会停在第一个 manifest，无论它是否拥有该模块。走查所得的 manifest 若不完整，则不产生包身份，这会把该配置项标记为松散模块并从 `dsh_plugin_packages` 中省略。
- 拥有方 manifest 的不完整检查保持不变，因此丢失 `name` 或 `version` 的已发布包仍是响亮的请求失败。

## Alternatives considered

**对所有 manifest 取消 fail-loud 检查。** 已否决：这会隐藏真正损坏的已发布包。一个解析到缺少这两个字段的 manifest 的裸说明符没有可用身份，请求也无法描述它。

**改为校验 profile 根自身的声明。** 已否决：profile 根由 `dsh plugin` 与人工书写，而 harness 在其他所有位置都接受它只有 `name` 的形态。请求路径不能比组合出它的 loader 更严格。

**把 probe 插件的说明符改成裸包名。** 已否决：那只会修好一个 profile，而任何无版本根之下的相对插件模块仍然损坏。

## Testing

[inventory.spec.ts](../../../../packages/llm/plugin-package-inventory-deepseek/tests/inventory.spec.ts) 从三个方向钉住已报告的失败：省略 `version` 的裸包 manifest 仍使准备失败；没有 `version` 的具名私有根之下的相对模块改为被省略；最近的 manifest 两个字段齐备的相对模块仍贡献其身份。

## Related

[DeepSeek 请求扩展决策](../architecture/2026-08-21-deepseek-llm-api-request-extensions.zh.md) 拥有该字段、其注册表与 `REQUEST_EXTENSION` 失败信封；本 note 只拥有哪些 manifest 可以被读成包身份。

## Consequences

位于无版本或匿名根之下的相对与绝对插件模块不贡献任何条目，因此部署清单会少报该插件，而不是让每个请求都失败。完全无法解析的裸包在本次改动前就已失败，现在仍然失败。
