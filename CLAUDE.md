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

**IMPORTANTE: Cite sempre `arquivo:linha`. Não aponte um problema sem mostrar exatamente onde ele está no código.**

#### Linguagem natural + rigor técnico

Cada achado deve ter **descoberta** (o que está errado), **contexto** (por que importa em termos práticos) e **como corrigir** (passo concreto).

Exemplos:

- ❌ **Evite:** "Query N+1 detectada"
- ✅ **Prefira:** "Em `views.py:156`, a view carrega usuários sem `select_related('profile')`, causando uma query por usuário. Com 1.000 usuários, isso vira 1.001 queries ao invés de 2. Solução: adicione `.select_related('profile')` na linha 142 onde o queryset é criado."

- ❌ **Evite:** "Falta validação"
- ✅ **Prefira:** "O serializer em `serializers.py:89` aceita `email` duplicado sem rejeitar. Se dois usuários tiverem o mesmo email, a view vai quebrar no `create()` (linha 91) com IntegrityError não tratado. Teste: adicione um caso que POST email já existente e verifique que retorna 400, não 500."

- ❌ **Evite:** "Código duplicado"
- ✅ **Prefira:** "As funções `get_user_avatar_url()` em `utils.py:45` e `User.get_avatar()` em `models.py:120` fazem a mesma coisa: buscam o avatar ou retornam uma URL default. Uma delas deveria ser removida, e o código deveria usar apenas a outra em todo o projeto."

### 3. Formar uma opinião própria

Depois da auditoria, escreva o que você, como revisor, acha que deveria ser feito — não apenas liste problemas. 

**Decisão:** Pode ser mergear agora, mergear com ressalvas, pedir mudanças, ou rejeitar. Cada um tem uma razão:

- **Mergear agora** — não tem achados críticos, testes cobrem bem, documentação está. Um PR limpo.
- **Mergear com ressalvas** — tem achados menores que não bloqueiam a funcionalidade. Autor pode abrir issue de follow-up.
- **Pedir mudanças** — tem achados importantes ou críticos que prejudicam qualidade/segurança. O autor deveria corrigir antes de mergear.
- **Rejeitar** — o PR não resolve o problema que alega, ou a solução é fundamentalmente errada. Merece reconsideração completa.

**Justificativa:** Uma frase ou duas explicando *por quê*. Exemplo: "Pedir mudanças — o IntegrityError não tratado quebra a API. O resto é sólido; as mudanças são 10 minutos de trabalho."

Isso vai direto no **Veredito Final** do relatório.

### 4. Escrever o relatório em um arquivo `.md`

Todo relatório de revisão deve ser salvo em um arquivo markdown, nunca apenas na resposta de chat. Salve sempre em `reviews/pr-review-<owner>-<repo>-<numero>.md` (crie a pasta `reviews/` na raiz do projeto se ainda não existir).

#### Estrutura do relatório

```markdown
# Review: <título do PR> (#<numero>)

**PR:** <url>
**Autor:** <autor>
**Branch:** <head> → <base>

## Resumo Executivo

Uma ou duas frases: o que o PR alega fazer vs. o que de fato faz. Se faz o que promete, diga. Se não faz ou vai além, descreva a divergência.

**Exemplo:** "O PR alega otimizar queries na view de usuários. De fato, adiciona `select_related('profile')` mas deixa uma query N+1 em feedback que continua não otimizada."

## O Que Está Bem

Reconheça o que o PR faz certo — testes bem estruturados, migrations seguras, documentação clara, etc. Isso encoraja e mostra que não é só crítica.

**Exemplo:** "A cobertura de testes é completa (91%+) e inclui casos de erro. As migrations estão feitas com `SeparateDatabaseAndState` e `AddIndexConcurrently`, sem risco de downtime."

## Achados

Organize por severidade e cite `arquivo:linha` para tudo. Cada achado tem três partes:

### Críticos (bloqueiam merge)

**1. Título conciso do problema**

**Localização:** `arquivo:linha` 

**O que acontece:** Descreva em uma frase simples o que o código faz — não se assume conhecimento da mudança.

**Por que importa:** Contexto prático — qual é a consequência dessa mudança no produção? Queima requests? Cai autenticação? Dados inconsistentes?

**Como corrigir:** Passo concreto, não genérico. Exemplo: "Na linha 156, adicione `.select_related('profile')` onde o queryset é criado" é melhor que "otimize a query".

---

**Exemplo completo:**

**IntegrityError não tratado ao criar usuários**

**Localização:** `views.py:91`

**O que acontece:** O método `create()` chama `User.objects.create_user()` sem verificar se o email já existe. Se dois requests chegarem simultaneamente com o mesmo email, um vai quebrar com `IntegrityError`.

**Por que importa:** Isso vira uma resposta 500 para o usuário, não uma mensagem amigável 400. Quebra o fluxo de registro.

**Como corrigir:** Na linha 85, valide o email antes de criar: 
```python
if User.objects.filter(email=request.data['email']).exists():
    return Response({'email': 'Já registrado'}, status=400)
```
Ou use o serializer para fazer a validação (mais DRF way).

### Importantes (deveriam ser corrigidos antes de merge)

Mesmo formato, mas com problemas que não quebram a aplicação imediatamente — code smell, manutenibilidade, segurança menor.

### Menores (nice to have, não bloqueia)

Melhorias que seriam legais mas são opcionais.

---

## Regressão e Queda de Qualidade

- Existem testes que passavam e agora falham?
- A mudança piora a legibilidade de código existente?
- A mudança quebra abstrações ou aumenta acoplamento?

Se nada aqui, diga "Nenhuma regressão detectada" para deixar claro que você verificou.

## Cobertura de Testes

- Os caminhos novos têm testes?
- Casos de borda estão cobertos (null, lista vazia, valores limites)?
- Testes antigos foram atualizados ou deixaram de fazer sentido?

Se a cobertura está bom, diga explicitamente. Se está ruim, mostre o impacto (exemplo: "A nova feature de retry não tem teste, então se quebrar em produção ninguém vai saber").

## Documentação

- README, CHANGELOG, docstrings ou API docs foram atualizados?
- Comportamentos que mudaram estão documentados?
- Valores de configuração novos estão explicados?

## Veredito Final

**Pronto para merge?** Sim / Não / Com ajustes

**Justificativa:** Uma frase ou duas. Exemplo: "Sim, com ajustes — todos os críticos são simples de corrigir (validação de email + um `select_related`). Importantes e menores são nice-to-have e não bloqueiam."
```

## Regras

**Coleta e Rigor:**
- Sempre buscar o PR real via `gh`, nunca inferir conteúdo a partir do título ou de suposições.
- Sempre citar `arquivo:linha` para cada achado.
- Categorizar achados por severidade real — nem tudo é crítico.

**Linguagem e Tom:**
- Escrever em **linguagem natural clara**, não jargão técnico hermético. Se você tiver que usar um termo técnico (N+1, IntegrityError, etc.), explique em uma frase o que significa no contexto prático.
- Cada achado deve responder: **o que está errado** (descoberta), **por que importa** (contexto), **como corrigir** (ação). Não deixe o leitor adivinhar.
- Evitar uma-liners como "refatore isso" ou "query N+1 aqui". Sempre descrever o impacto real (performance, segurança, manutenibilidade).
- Usar exemplos de código ou pseudocódigo para deixar claro o que fazer.

**Construção do Relatório:**
- Reconhecer o que está bem feito, não só listar problemas. Um relatório equilibrado encoraja.
- Terminar sempre com um veredito claro e uma recomendação de próximos passos.
- O relatório final é sempre um arquivo `.md` em `reviews/`, entregue além do resumo dado ao usuário no chat.
- Estrutura: Resumo Executivo → O Que Está Bem → Achados (por severidade) → Regressão/Qualidade → Testes → Documentação → Veredito.

**Evitar:**
- Jargão sem explicação ("code smell", "tight coupling" sem contexto).
- Crítica pessoal ou tom condescendente.
- Listar achados sem linha/arquivo.
- Misturar descoberta com recomendação (separe com clareza).
- Deixar ambigüidade: sempre seja específico sobre o que, onde e por quê.
