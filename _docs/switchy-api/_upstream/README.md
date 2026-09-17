# `_upstream/` — fonte original, sem alterações

Cópia fiel do conteúdo de `docs/` do repositório oficial <https://github.com/Switchy-io/api-docs> (branch `master`), tal como estava em **2026-09-17** (`last-modified` do site: 2026-09-10).

Não editar. Serve para diff quando se refrescar o clone.

| Ficheiro | Página publicada |
|---|---|
| `overview/index.md` | <https://developers.switchy.io/docs/overview/index> |
| `overview/authentication.md` | <https://developers.switchy.io/docs/overview/authentication> |
| `overview/root-endpoint.md` | <https://developers.switchy.io/docs/overview/root-endpoint> |
| `overview/schema-introspection.md` | <https://developers.switchy.io/docs/overview/schema-introspection> |
| `guides/how-to-query.md` | <https://developers.switchy.io/docs/guides/how-to-query> |
| `guides/how-to-create-a-link.md` | <https://developers.switchy.io/docs/guides/how-to-create-a-link> |
| `guides/how-to-update-a-link.md` | <https://developers.switchy.io/docs/guides/how-to-update-a-link> |
| `changelog/index.md` | <https://developers.switchy.io/docs/changelog/index> |
| `sidebars.js` | estrutura de navegação do site |

## Ficheiros do upstream NÃO copiados

Existem no repositório mas estão fora do sidebar e sem conteúdo útil:

- `docs/overview/about-graphql.md` — só front-matter, corpo vazio (está comentado no `sidebars.js`)
- `docs/overview/rate-limits.md` — contém apenas a linha `## Coucou` (placeholder); os rate limits reais estão no rodapé dos dois guias de link
- `docs/guides/doc3.md` — lorem ipsum do template Docusaurus

## Como refrescar

```bash
curl -sL https://codeload.github.com/Switchy-io/api-docs/tar.gz/refs/heads/master -o api-docs.tgz
tar xzf api-docs.tgz
diff -ru _docs/switchy-api/_upstream/overview api-docs-master/docs/overview
diff -ru _docs/switchy-api/_upstream/guides  api-docs-master/docs/guides
```
