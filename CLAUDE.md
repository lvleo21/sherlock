# CLAUDE.md

Instruções para revisão de Pull Requests do GitHub neste projeto (**Sherlock**). Estas instruções têm prioridade sobre o comportamento padrão sempre que a tarefa for revisar um PR.

## Contexto do projeto

O Sherlock existe para revisar Pull Requests do GitHub **de projetos Django** com apoio de assistentes de código e skills. O fluxo típico: o usuário cola um link de PR do GitHub e espera uma auditoria independente, avaliando também convenções e boas práticas do Django/DRF. A saída são os **achados**, publicados como uma review no próprio PR do GitHub. Não há veredito nem arquivo de relatório.

Skills instaladas em `.claude/skills/` (via `skills-lock.json`), disponíveis como apoio mas não substituem o processo abaixo. As skills Django/DRF são o núcleo da auditoria; as demais são apoio geral:
- `django-reviewer` — revisão de mudanças Django/Python focada em clareza, consistência, manutenibilidade e anti-patterns de ORM/DRF, preservando o comportamento existente. Skill principal para qualquer PR revisado pelo Sherlock.
- `django-expert` — orientação sobre boas práticas de backend Django (models, views, serializers, APIs, ORM, migrations, autenticação, testes). Apoia a auditoria para avaliar se o código segue os padrões modernos do Django.
- `django-safe-migration` — revisão de migrations Django/PostgreSQL quanto à segurança de deploy sem downtime (locks, `SeparateDatabaseAndState`, `AddIndexConcurrently`, FK `NOT VALID` + `VALIDATE`, `db_default`, `lock_timeout`). Usar sempre que o PR alterar migrations.
- `django-celery-expert` — orientação sobre tasks assíncronas com Celery em Django (workers, retries, tratamento de erro, Celery Beat, monitoramento). Útil quando o PR mexe em processamento assíncrono/background.
- `cdrf-expert` — orientação sobre class-based views do Django REST Framework (APIView, GenericAPIView, mixins, ViewSets, MRO, `create` vs `perform_create` etc.), usando Classy DRF como referência. Útil quando o PR mexe em views/serializers da DRF.
- `code-review` — revisão em dois eixos (Standards/Spec) de um diff `git`. Assume um checkout local e um ponto fixo (`git diff <fixed-point>...HEAD`); útil se o PR já estiver com checkout local feito.
- `requesting-code-review` — template para despachar um subagente revisor com contexto isolado (Critical/Important/Minor).
- `humanizer` — remove marcas de texto gerado por IA da prosa dos achados antes de publicar. Trabalha junto com o markdown da review, sem removê-lo (ver etapa 6).

Nenhuma dessas skills busca o PR no GitHub nem publica a review sozinha — isso é o que este arquivo cobre.

## Princípio central

**Não confie na descrição do autor do PR.** A descrição, o título e os comentários do autor são hipóteses a verificar, não fatos. A auditoria deve ser feita lendo o diff e o código real, comparando nos dois sentidos o que o PR *diz* que faz com o que ele *de fato* faz.

## Processo ao receber um link de PR

### 1. Coletar os dados do PR

Use o `gh` CLI (nunca assuma o conteúdo do PR sem buscar):

```bash
gh pr view <url> --json number,title,body,author,baseRefName,headRefName,headRefOid,url,additions,deletions,files,commits,closingIssuesReferences
gh pr diff <url>
```

Se o PR fecha uma issue (`closingIssuesReferences`), leia a issue com `gh issue view`: ela faz parte do que o PR promete.

Busque também a thread completa de comentários do PR (reviews, comentários gerais e threads inline com respostas, status de resolvida e de desatualizada). Ela é usada na etapa 5:

```bash
gh api graphql -F owner=<owner> -F repo=<repo> -F number=<numero> -f query='
query($owner: String!, $repo: String!, $number: Int!) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $number) {
      reviews(first: 50) { nodes { author { login } state body submittedAt url } }
      comments(first: 100) { nodes { author { login } body createdAt url } }
      reviewThreads(first: 100) {
        nodes {
          isResolved isOutdated path line
          comments(first: 50) { nodes { author { login } body createdAt url } }
        }
      }
    }
  }
}'
```

Se for necessário rodar testes, ler arquivos fora do diff ou navegar o repo completo, faça o checkout:

```bash
gh pr checkout <url>   # ou gh repo clone <owner>/<repo> se o repo não estiver local
```

Faça isso em um diretório separado (ex.: `/tmp` ou um worktree) — nunca dentro deste diretório do Sherlock, que não é o repositório sendo revisado.

### 2. Conferir o código contra a descrição do PR

Compare o diff com o título, a descrição e a issue vinculada, nos dois sentidos:

- **Mudanças não descritas** — liste tudo o que o diff de fato faz: comportamento novo ou alterado, endpoints, migrations, settings, variáveis de ambiente, dependências, permissões, refatorações, arquivos removidos. Cada item deve estar descrito no PR. O que muda comportamento e não está descrito é um achado, porque quem revisa ou faz deploy não vai saber que aquilo mudou.
- **Promessas não cumpridas** — cada coisa que a descrição ou a issue diz que o PR faz precisa existir no código. O que foi prometido e não foi implementado, ou foi implementado só em parte, é um achado.

Mudanças puramente mecânicas (formatação, renomear variável local) não precisam estar na descrição.

### 3. Conferir a documentação na versão do PR

Leia a documentação do repositório **como ela fica no head do PR**, não na base, e verifique se ela continua verdadeira depois da mudança. Isso vale também para arquivos que o PR não tocou: um README que não foi alterado pode ter ficado errado justamente porque o código mudou.

Documentos a verificar: `README.md`, `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`, `CHANGELOG`, `docs/`, `.env.example`, schemas de API (OpenAPI), docstrings e comentários próximos ao código alterado.

Para ler um arquivo na versão do PR sem checkout:

```bash
gh api "repos/<owner>/<repo>/contents/<caminho>?ref=<headRefOid>" --jq .content | base64 -d
```

Procure por:
- comandos, passos de setup, variáveis de ambiente, endpoints, nomes de settings ou de tasks que o PR mudou, renomeou ou removeu, mas que a documentação ainda cita;
- comportamento novo que deveria estar documentado e não está;
- instruções para agentes (`CLAUDE.md`, `AGENTS.md`) que contradizem a nova estrutura do código.

### 4. Auditar o código a fundo

Leia o diff completo e o código ao redor (não só as linhas alteradas). Para cada mudança, pergunte: isso faz sentido dado o problema que o PR alega resolver?

Verifique especificamente:

- **Regressão** — a mudança quebra comportamento existente, casos de borda ou testes que passavam antes?
- **Queda de qualidade de código** — a mudança piora a legibilidade, a separação de responsabilidades, o tratamento de erros ou a manutenibilidade em relação ao que já existia?
- **Código morto** — funções, variáveis, imports, flags ou branches que ficaram sem uso após a mudança (introduzidos pelo PR ou deixados para trás).
- **Duplicação desnecessária** — lógica repetida que já existia em outro lugar do código, ou repetida dentro do próprio PR, e que deveria ser extraída.
- **Valores mágicos hardcoded** — números, strings ou timeouts embutidos direto no código que deveriam virar uma constante nomeada e/ou vir acompanhados de um comentário explicando a origem/motivo do valor.
- **Cobertura de testes** — os caminhos novos e alterados têm testes reais (não mocks vazios)? Casos de borda relevantes estão cobertos? Testes antigos foram atualizados ou ficaram obsoletos?
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

### 5. Confrontar com a thread de comentários

Leia a thread inteira buscada na etapa 1, incluindo respostas e threads já resolvidas, e confronte cada ponto com os achados da nova revisão e com o código no head do PR:

- **Apontado e corrigido de fato:** não vira achado.
- **Dado como corrigido, mas o problema continua:** a thread foi resolvida ou o autor respondeu "corrigido", mas o código no head ainda tem o problema. É um achado. Cite o link da thread e mostre em `arquivo:linha` onde o problema continua.
- **Ainda em aberto e válido:** não crie um comentário duplicado. Liste o link da thread no corpo da review, em "Pontos anteriores ainda em aberto".
- **Autor justificou a decisão:** leve a justificativa em conta antes de apontar o mesmo ponto. Só reaponte se o código contradizer a justificativa, explicando por quê e citando a thread.
- **Combinado na thread:** decisões como "fica para outro PR" ou "vou adicionar os testes" são promessas. Verifique se foram cumpridas (a issue foi aberta, os testes existem) e trate o que não foi cumprido como achado.
- **Correções feitas em resposta a comentários:** audite também os commits que responderam à review anterior. Uma correção pode introduzir um problema novo.

Ao terminar, cada achado da nova revisão é novo ou traz o link da thread que ele retoma.

### 6. Escrever os achados

Só achados. Não há resumo executivo, seção "O que está bem", veredito nem arquivo de relatório.

Cada achado vira um comentário no formato abaixo. A severidade vai no título: **Crítico** (quebra funcionalidade, segurança ou dados; bloqueia o merge), **Importante** (deveria ser corrigido antes do merge) ou **Menor** (opcional).

````markdown
**[Crítico] IntegrityError não tratado ao criar usuários**

**O que acontece:** O método `create()` chama `User.objects.create_user()` sem verificar se o email já existe. Se dois requests chegarem com o mesmo email, um deles quebra com `IntegrityError`.

**Por que importa:** O usuário recebe um erro 500 em vez de uma mensagem 400 explicando o problema, e o fluxo de cadastro quebra.

**Como corrigir:** Valide o email no serializer:
```python
def validate_email(self, value):
    if User.objects.filter(email=value).exists():
        raise serializers.ValidationError("Email já registrado.")
    return value
```
````

- Em comentários inline, a localização é a própria linha onde o comentário está ancorado. Cite outras linhas relacionadas como `arquivo:linha`.
- Quando a correção cabe nas linhas comentadas, use um bloco ` ```suggestion ` do GitHub, que o autor pode aplicar com um clique.
- Achados que retomam um ponto da thread de comentários trazem o link da thread original.
- Achados fora do diff (documentação não alterada, promessa não cumprida, mudança não descrita) levam uma linha **Localização:** com `arquivo:linha` e um permalink para o head do PR: `https://github.com/<owner>/<repo>/blob/<headRefOid>/<caminho>#L<inicio>-L<fim>`.

#### Humanizer e markdown

Antes de publicar, passe o texto de cada achado pelo skill `humanizer` no modo *embedded*. O humanizer trabalha **junto** com o markdown da review e nunca o remove:

- **Preservar sempre:** títulos com a severidade (`**[Crítico] ...**`), os rótulos em negrito (**O que acontece:**, **Por que importa:**, **Como corrigir:**, **Localização:**), listas, tabelas, links, permalinks, `arquivo:linha`, código inline, blocos de código e blocos `suggestion`. Nessas estruturas, esta regra tem prioridade sobre os padrões §19 e §20 do humanizer (negrito e títulos).
- **Reescrever apenas a prosa:** as frases dentro de cada rótulo, removendo frases de efeito, "não é X, é Y", travessões, palavras infladas e fechamentos que repetem o ponto.
- O humanizer não pode alterar o conteúdo técnico: nomes, números, linhas, trechos de código e a severidade ficam como estão.

### 7. Publicar a review no GitHub

1. Mostre os achados no chat, em uma lista curta (severidade, título, `arquivo:linha`), e peça confirmação antes de publicar. Publicar é visível para o autor e para o time do repositório.
2. Após a confirmação, publique **uma única review** com todos os achados. Use sempre `"event": "COMMENT"`: não aprovar nem pedir mudanças, porque o Sherlock não dá veredito.
3. Monte o JSON fora deste diretório (ex.: `/tmp`) e envie com `gh api`:

```bash
gh api repos/<owner>/<repo>/pulls/<numero>/reviews --method POST --input /tmp/review.json
```

```json
{
  "commit_id": "<headRefOid>",
  "event": "COMMENT",
  "body": "<contagem, achados fora do diff e pontos anteriores em aberto>",
  "comments": [
    { "path": "app/views.py", "line": 156, "side": "RIGHT", "body": "<achado>" },
    { "path": "app/views.py", "start_line": 140, "line": 156, "side": "RIGHT", "body": "<achado em várias linhas>" }
  ]
}
```

- **Comentário inline** (`comments`): achados ligados a linhas que aparecem no diff. `line` é o número da linha no arquivo novo (`side: "RIGHT"`); para linhas removidas, use o número no arquivo antigo com `side: "LEFT"`. Para um trecho, use `start_line` + `line`.
- **Corpo da review** (`body`): uma linha com a contagem (ex.: "5 achados: 1 crítico, 3 importantes, 1 menor."), os achados que não podem ser ancorados no diff e, se houver, a lista "Pontos anteriores ainda em aberto" com os links das threads (ver etapa 5).
- Se a API recusar um comentário inline (erro 422, linha fora do diff), mova esse achado para o corpo da review com permalink e publique de novo.
- Se não houver achados nem pontos anteriores em aberto, não publique nada; diga isso ao usuário no chat.

Ao terminar, responda no chat com o link da review publicada (`html_url` da resposta da API) e a lista curta de achados.

## Regras

**Coleta e Rigor:**
- Sempre buscar o PR real via `gh`, nunca inferir conteúdo a partir do título ou de suposições.
- Sempre citar `arquivo:linha` para cada achado.
- Categorizar achados por severidade real — nem tudo é crítico.
- Ler a thread de comentários inteira e confrontá-la com a nova revisão: não repetir o que já foi apontado e está em aberto, e reapontar (com link) o que foi dado como corrigido mas continua no código.

**Linguagem e Tom:**
- Escrever em **linguagem natural clara**, não jargão técnico hermético. Se você tiver que usar um termo técnico (N+1, IntegrityError, etc.), explique em uma frase o que significa no contexto prático.
- Cada achado deve responder: **o que está errado** (descoberta), **por que importa** (contexto), **como corrigir** (ação). Não deixe o leitor adivinhar.
- Evitar uma-liners como "refatore isso" ou "query N+1 aqui". Sempre descrever o impacto real (performance, segurança, manutenibilidade).
- Usar exemplos de código, pseudocódigo ou blocos `suggestion` para deixar claro o que fazer.

**Saída:**
- Apenas achados, publicados como uma review `COMMENT` no PR após confirmação do usuário.
- Sem veredito, sem resumo executivo, sem seção de elogios, sem arquivo de relatório.
- A prosa passa pelo `humanizer`; o markdown da review é preservado.

**Evitar:**
- Jargão sem explicação ("code smell", "tight coupling" sem contexto).
- Crítica pessoal ou tom condescendente.
- Listar achados sem linha/arquivo.
- Misturar descoberta com recomendação (separe com clareza).
- Deixar ambigüidade: sempre seja específico sobre o que, onde e por quê.
