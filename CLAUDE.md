# CLAUDE.md

Instruções para revisão de Pull Requests do GitHub neste projeto (**Sherlock**). Estas instruções têm prioridade sobre o comportamento padrão sempre que a tarefa for revisar um PR.

## Contexto do projeto

O Sherlock existe para revisar Pull Requests do GitHub **de projetos Django** com apoio de assistentes de código e skills. O fluxo típico: o usuário cola um link de PR do GitHub e espera uma auditoria completa e independente — avaliando também convenções e boas práticas específicas do Django/DRF — encerrando em um relatório em markdown.

Skills instaladas em `.claude/skills/` (via `skills-lock.json`), disponíveis como apoio mas não substituem o processo abaixo. As skills Django/DRF são o núcleo da auditoria; as demais são apoio geral:
- `django-reviewer` — revisão de mudanças Django/Python focada em clareza, consistência, manutenibilidade e anti-patterns de ORM/DRF, preservando o comportamento existente. Skill principal para qualquer PR revisado pelo Sherlock.
- `django-expert` — orientação sobre boas práticas de backend Django (models, views, serializers, APIs, ORM, migrations, autenticação, testes). Apoia a auditoria para avaliar se o código segue os padrões modernos do Django.
- `django-safe-migration` — revisão de migrations Django/PostgreSQL quanto à segurança de deploy sem downtime (locks, `SeparateDatabaseAndState`, `AddIndexConcurrently`, FK `NOT VALID` + `VALIDATE`, `db_default`, `lock_timeout`). Usar sempre que o PR alterar migrations.
- `django-celery-expert` — orientação sobre tasks assíncronas com Celery em Django (workers, retries, tratamento de erro, Celery Beat, monitoramento). Útil quando o PR mexe em processamento assíncrono/background.
- `cdrf-expert` — orientação sobre class-based views do Django REST Framework (APIView, GenericAPIView, mixins, ViewSets, MRO, `create` vs `perform_create` etc.), usando Classy DRF como referência. Útil quando o PR mexe em views/serializers da DRF.
- `code-review` — revisão em dois eixos (Standards/Spec) de um diff `git`. Assume um checkout local e um ponto fixo (`git diff <fixed-point>...HEAD`); útil se o PR já estiver com checkout local feito.
- `requesting-code-review` — template para despachar um subagente revisor com contexto isolado (Critical/Important/Minor).

Nenhuma dessas skills busca o PR no GitHub sozinha — isso é o que este arquivo cobre.

## Princípio central

**Não confie na descrição do autor do PR.** A descrição, o título e os comentários do autor são hipóteses a verificar, não fatos. A auditoria deve ser feita lendo o diff e o código real, comparando o que o PR *diz* que faz com o que ele *de fato* faz.

## Processo ao receber um link de PR

### 1. Coletar os dados do PR

Use o `gh` CLI (nunca assuma o conteúdo do PR sem buscar):

```bash
gh pr view <url> --json number,title,body,author,baseRefName,headRefName,url,additions,deletions,files,commits
gh pr diff <url>
```

Se for necessário rodar testes, buscar código não incluído no diff, ou navegar o repo completo:

```bash
gh pr checkout <url>   # ou gh repo clone <owner>/<repo> se o repo não estiver local
```

Faça isso em um diretório separado (ex.: `/tmp` ou um worktree) — nunca dentro deste diretório do Sherlock, que não é o repositório sendo revisado.

### 2. Auditar o código a fundo

Leia o diff completo e o código ao redor (não só as linhas alteradas). Para cada mudança, pergunte: isso faz sentido dado o problema que o PR alega resolver?

Verifique especificamente:

- **Regressão** — a mudança quebra comportamento existente, casos de borda ou testes que passavam antes?
- **Queda de qualidade de código** — a mudança piora a legibilidade, a separação de responsabilidades, o tratamento de erros ou a manutenibilidade em relação ao que já existia?
- **Código morto** — funções, variáveis, imports, flags ou branches que ficaram sem uso após a mudança (introduzidos pelo PR ou deixados para trás).
- **Duplicação desnecessária** — lógica repetida que já existia em outro lugar do código, ou repetida dentro do próprio PR, e que deveria ser extraída.
- **Valores mágicos hardcoded** — números, strings ou timeouts embutidos direto no código que deveriam virar uma constante nomeada e/ou vir acompanhados de um comentário explicando a origem/motivo do valor.
- **Cobertura de testes** — os caminhos novos e alterados têm testes reais (não mocks vazios)? Casos de borda relevantes estão cobertos? Testes antigos foram atualizados ou ficaram obsoletos?
- **Documentação** — README, comentários, CHANGELOG, docs/ ou schemas de API foram atualizados de forma condizente com a mudança de comportamento?
- **Convenções Django/DRF** — models, migrations, querysets, views/serializers da DRF e tasks Celery seguem as boas práticas do framework (ver skills `django-expert`, `django-safe-migration`, `django-celery-expert`, `cdrf-expert`)? Queries N+1, migrations bloqueantes e uso indevido do ORM são achados de alta prioridade.

Cite sempre `arquivo:linha`. Não aponte um problema sem mostrar onde ele está.

### 3. Formar uma opinião própria

Depois da auditoria, escreva o que você, como revisor, acha que deveria ser feito — não apenas liste problemas. Isso inclui: mergear como está, mergear com ressalvas, pedir mudanças antes do merge, ou rejeitar. Justifique tecnicamente.

### 4. Escrever o relatório em um arquivo `.md`

Todo relatório de revisão deve ser salvo em um arquivo markdown, nunca apenas na resposta de chat. Salve sempre em `reviews/pr-review-<owner>-<repo>-<numero>.md` (crie a pasta `reviews/` na raiz do projeto se ainda não existir).

Estrutura do relatório:

```markdown
# Review: <título do PR> (#<numero>)

**PR:** <url>
**Autor:** <autor>
**Branch:** <head> → <base>

## Resumo

O que o PR diz que faz vs. o que ele de fato faz.

## Achados

### Críticos (bloqueiam merge)
- `arquivo:linha` — descrição do problema, por que importa, como corrigir.

### Importantes (deveriam ser corrigidos)
- ...

### Menores (nice to have)
- ...

## Regressão e queda de qualidade
...

## Cobertura de testes
...

## Código morto / duplicação / valores mágicos
...

## Documentação
...

## Recomendação
**Pronto para merge?** Sim / Não / Com ajustes
**Justificativa:** ...
```

## Regras

- Sempre buscar o PR real via `gh`, nunca inferir conteúdo a partir do título ou de suposições.
- Sempre citar `arquivo:linha` para cada achado.
- Categorizar achados por severidade real — nem tudo é crítico.
- Reconhecer o que está bem feito, não só listar problemas.
- Terminar sempre com um veredito claro e uma recomendação de próximos passos.
- O relatório final é sempre um arquivo `.md` em `reviews/`, entregue além do resumo dado ao usuário no chat.
