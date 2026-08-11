<p align="center">
  <img src="./.github/assets/sherlock-banner.png" alt="Sherlock banner" width="100%">
</p>

# Sherlock

Projeto para revisar Pull Requests do GitHub de **projetos Django** com apoio do Claude Code. Cole o link de um PR e receba uma auditoria completa e independente do código, encerrando em um relatório em markdown salvo em `reviews/`.

## Como funciona

O fluxo de revisão (documentado em detalhe em [`CLAUDE.md`](./CLAUDE.md)) segue estes passos:

1. **Coleta dos dados do PR** via `gh` CLI (`gh pr view`, `gh pr diff`, e `gh pr checkout` quando é preciso rodar testes ou navegar o repo completo).
2. **Auditoria a fundo do código** — não só o diff, mas o código ao redor — verificando regressão, queda de qualidade, código morto, duplicação, valores mágicos hardcoded, cobertura de testes e documentação.
3. **Formação de uma opinião própria** sobre o PR: mergear como está, mergear com ressalvas, pedir mudanças ou rejeitar.
4. **Escrita do relatório** em `reviews/pr-review-<owner>-<repo>-<numero>.md`.

O princípio central: **a descrição do autor do PR nunca é levada como verdade**. Título, descrição e comentários são hipóteses a verificar contra o diff e o código real.

## Estrutura do projeto

```
.
├── CLAUDE.md          # instruções completas do processo de revisão (fonte da verdade)
├── README.md          # este arquivo
├── LICENSE             # licença MIT
├── .github/assets/     # imagens usadas na documentação (ex.: banner)
├── skills-lock.json   # skills instaladas via Claude Code
├── .claude/skills/     # skills disponíveis como apoio à revisão
└── reviews/            # relatórios de revisão gerados (git-ignorado)
```

## Skills de apoio

Instaladas em `.claude/skills/`, disponíveis para reforçar a auditoria mas **não substituem** o processo acima — nenhuma delas busca o PR no GitHub sozinha:

| Skill | Uso |
|---|---|
| `code-review` | Revisão em dois eixos (Standards/Spec) de um diff `git` local |
| `requesting-code-review` | Despacha um subagente revisor com contexto isolado (Critical/Important/Minor) |
| `django-reviewer` | Revisão de mudanças Django/Python (clareza, ORM/DRF, manutenibilidade) — núcleo da auditoria do Sherlock |
| `django-expert` | Boas práticas de backend Django |
| `django-safe-migration` | Segurança de migrations Django/PostgreSQL sem downtime |
| `django-celery-expert` | Tasks assíncronas com Celery em Django |
| `cdrf-expert` | Class-based views do Django REST Framework (Classy DRF) |

## Uso

Basta colar o link de um PR do GitHub na conversa com o Claude Code neste diretório. O relatório final é entregue como arquivo `.md` em `reviews/`, além de um resumo no chat.

> **Atenção:** o checkout do repositório sendo revisado deve ser feito fora deste diretório (ex.: `/tmp` ou um worktree) — este diretório é apenas o projeto de revisão, não o repositório revisado.

## Licença

Distribuído sob a licença [MIT](./LICENSE).
