# Central de HTMLs — LL Mídia

Repositório central para alojar ficheiros HTML da empresa (planeamentos, diretrizes,
briefings, brandbooks, documentos estratégicos) e servi-los como **páginas web públicas**
via GitHub Pages. Ninguém precisa de abrir HTML localmente nem de andar a passar ficheiros
de mão em mão: cada documento ganha uma URL fixa.

- **Central (navegável):** https://dicasvisuais.github.io/llmidia/
- **Base das URLs:** `https://dicasvisuais.github.io/llmidia/<caminho-do-ficheiro>`

A página inicial (`index.html`) é uma central tipo "drive": mostra as pastas, deixa entrar
nelas, e abre os HTMLs. A lista **atualiza-se sozinha** sempre que um novo ficheiro é
enviado ao repositório (é lida ao vivo a partir do GitHub).

---

## Convenção de pastas

Cada projeto vive na sua própria pasta (com subpastas, se fizer sentido), e os HTMLs ficam
lá dentro. A regra é simples:

```
/<area-ou-projeto>/<subpasta-opcional>/<ficheiro>.html
```

### Exemplos já no repositório

| Caminho no repositório | URL pública |
|---|---|
| `funis/minicurso/funil-minicurso-v3.html` | https://dicasvisuais.github.io/llmidia/funis/minicurso/funil-minicurso-v3.html |
| `mas26/Brandbook_MasterAndScale_2026.html` | https://dicasvisuais.github.io/llmidia/mas26/Brandbook_MasterAndScale_2026.html |

---

## Como adicionar um projeto novo

1. Cria a pasta do projeto (e subpastas, se precisares).
2. Coloca lá o(s) ficheiro(s) `.html`.
3. Envia para o repositório:
   ```bash
   git add .
   git commit -m "Adiciona <nome do projeto>"
   git push
   ```
4. ~1 minuto depois, o ficheiro está no ar na sua URL e aparece automaticamente na central.

> Quem preferir, pode fazer o upload diretamente pelo site do GitHub
> (botão **Add file → Upload files**) — a central reflete na mesma.

---

## Boas práticas de nomenclatura

- **Pastas:** minúsculas, sem espaços nem acentos. Usa hífen para separar (`master-scale`, `funis`).
- **Ficheiros:** evita espaços e acentos; usa hífen ou underscore (`brandbook-2026.html`).
- Mantém **um assunto por pasta** — facilita encontrar e partilhar a URL certa.

---

## Notas importantes

- **O repositório é público.** Qualquer pessoa com o link de um ficheiro consegue abri-lo.
  A palavra-passe da central é apenas um **portão simbólico** (fica visível no código-fonte
  e não impede o acesso direto por URL). **Não coloques aqui dados confidenciais**
  (senhas, dados de clientes, contratos, números internos sensíveis).
- Para conteúdo que precise de acesso restrito de verdade, fala com quem gere o repositório
  (dá para migrar essa parte para uma solução com login por email autorizado).
- `index.html`, `README.md` e `.gitignore` não aparecem na navegação da central.
