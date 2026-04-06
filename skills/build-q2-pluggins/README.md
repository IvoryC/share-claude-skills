# build-q2-pluggins

A Claude Code skill for building QIIME 2 plugins — wrapping new tools or existing tools to operate in the QIIME 2 ecosystem and track provenance.

## Origin

This skill was created from the following prompt:

> I want to build qiime2 pluggins. I have done this before with Claude, and I asked Claude to make notes for the next time it comes up, and that was the basis for the built-q2-pluggins skill. But that has a few issues... it references paths that are specific to me but I want to make this a widely shareable skill; it is not very comprehensive, I think more of the information from the qiime2 pluggin documentation needs to be brought into the markdown; and it is is just one file but I think once it is more comprehensive it will be too big, and it would be better if some parts of the skill were pulled off into markdowns in a reference subfolder. I would like you to start with the information in the current skill, and expand the skill based on the current documentation. Keep in mind, a lot of qiime2 documentation is flagged as being out of date; it is important to link back to the qiime2 teams official documentation throughout the skill markdown text.

## Files

| File | Contents |
|---|---|
| [`SKILL.md`](SKILL.md) | Main entry point — quick start, repo structure, type system summary, action registration patterns, testing, dev workflow, gotchas |
| [`references/package-structure.md`](references/package-structure.md) | Full directory layout, `pyproject.toml` entry point, `plugin_setup.py` skeleton, `Makefile` conventions, scaling up as plugin grows |
| [`references/type-system.md`](references/type-system.md) | Semantic types → Python format class mappings, primitive types + predicates, `Metadata`/`MetadataColumn`, `TypeMatch`, artifact collections, custom artifact class pattern |
| [`references/action-registration.md`](references/action-registration.md) | Complete examples for Method, Visualizer, and Pipeline registration; citations; `TypeMatch`; custom semantic type registration |
| [`references/testing.md`](references/testing.md) | `TestPluginBase`, fixtures, loading artifacts, transformers in tests, skipping external-tool tests, common assertions, CI setup |
| [`references/external-tools.md`](references/external-tools.md) | subprocess pattern, temp directories, per-sample naming, R scripts, rJava/JVM heap, Java tools, tool availability checking, conda packaging |

## Official Documentation

QIIME 2 is actively developed and some older docs are flagged as outdated. Always refer to the canonical sources:

- Developer book: [develop.qiime2.org](https://develop.qiime2.org/en/latest/)
- Plugin template: [github.com/caporaso-lab/plugin-template](https://github.com/caporaso-lab/plugin-template)
- Semantic types reference: [docs.qiime2.org/2024.10/semantic-types](https://docs.qiime2.org/2024.10/semantic-types/)
- Developer forum: [forum.qiime2.org](https://forum.qiime2.org/c/dev/16)
