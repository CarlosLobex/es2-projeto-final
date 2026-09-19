# Plano de Refactoring — Parte 2, Tarefa 2.2

Dos 6 smells catalogados em `CODE_SMELLS.md`, os 4 abaixo foram
escolhidos para tratamento nesta parte. O Smell 1 (duplicação entre
`Category.get_all`/`get_forums`) fica registrado como candidato para
uma futura rodada — não foi descartado, só priorizado depois dos
outros quatro, que têm maior impacto imediato (um deles corrige uma
inconsistência de nome de campo já existente no código).

---

## Refatoração 1 — Extract Method: `Forum.set_last_post(post)`

- **Smell tratado:** Smell 2 (Duplicated Code / Data Clumps — "último
  post do fórum" atribuído manualmente em 4 lugares) e, como
  consequência direta, o Smell 6 (Primitive Obsession).
- **Resultado esperado:** os 4 pontos que hoje atribuem os 5 campos
  `last_post*` um a um passam a chamar um único método
  `Forum.set_last_post(post)` (ou `set_last_post(None)` para limpar),
  eliminando a duplicação e corrigindo de vez a inconsistência
  `last_post_user` vs. `last_post_user_id`.
- **Riscos antecipados:** os 4 pontos de chamada não são idênticos —
  um deles ("limpar" último post) passa `None`; é preciso garantir que
  o novo método aceite esse caso sem quebrar. Risco de efeito colateral
  em `Post.delete`/`hide`/`unhide`, que dependem indiretamente desse
  estado — mitigado rodando a suíte completa após o commit.

---

## Refatoração 2 — Extract Method: `_calculate_read_cutoff()`

- **Smell tratado:** Smell 3 (Duplicated Code — cálculo de
  `read_cutoff` repetido em `Topic.tracker_needs_update` e
  `Forum.update_read`).
- **Resultado esperado:** as duas cópias do cálculo passam a chamar
  uma única função auxiliar, então uma mudança futura na regra do
  `TRACKER_LENGTH` só precisa ser feita em um lugar.
- **Riscos antecipados:** baixo — é um cálculo puro (sem efeito
  colateral), então o risco principal é só garantir que a função
  auxiliar fique acessível dos dois pontos de chamada (mesmo módulo,
  sem problema de import).

---

## Refatoração 3 — Extract Method dentro de `Forum.update_read`

- **Smell tratado:** Smell 4 (Long Method — `Forum.update_read` com
  ~87 linhas e 3 responsabilidades misturadas).
- **Resultado esperado:** a query de contagem de tópicos não lidos
  sai para um método próprio (ex.: `_count_unread_topics(user,
  read_cutoff)`), deixando `update_read` só com a lógica de decisão
  (criar/atualizar/ignorar o `ForumsRead`), mais curta e mais fácil de
  entender de uma vez.
- **Riscos antecipados:** é o método com **menor cobertura de teste**
  do arquivo (0% na baseline), então o risco real não é quebrar um
  teste (não existe nenhum ainda) e sim mudar o comportamento sem que
  ninguém perceba. Mitigação: validar manualmente antes/depois via
  `flask shell` ou teste ad-hoc, além de rodar a suíte completa.

---

## Refatoração 4 — Remoção de código morto (ramo `else` inalcançável)

- **Smell tratado:** Smell 5 (Dead Code — `else: updated = False` em
  `Topic.update_read`, inalcançável porque `if topicsread` /
  `elif not topicsread` já cobre os dois únicos casos possíveis).
- **Resultado esperado:** o `if/elif/else` de 3 ramos vira um
  `if/else` de 2 ramos, sem nenhuma mudança de comportamento — só
  remove a confusão de um ramo que nunca executa.
- **Riscos antecipados:** mínimo. Único cuidado é confirmar, antes de
  remover, que `topicsread` realmente não pode ser um valor "falsy
  mas não None" que escapasse da lógica — checagem rápida pelo tipo
  (`TopicsRead | None`) confirma que só há os dois casos.

---

## Backlog (não tratado nesta parte)

- **Smell 1** (Duplicated Code em `Category.get_all`/`get_forums`):
  candidato a uma futura `Extract Method` (`_query_forums_for_user`),
  fica documentado aqui para retomada posterior.
