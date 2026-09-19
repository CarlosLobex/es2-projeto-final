# Catálogo de Code Smells — Parte 2, Tarefa 2.1

**Arquivo analisado:** `flaskbb/forum/models.py` (1665 linhas)

Este arquivo foi escolhido como hotspot da Parte 2 por dar continuidade
ao módulo trabalhado na Parte 1 (`flaskbb/forum/`) e por já contar com
cobertura de testes maior (58% na baseline) do que a alternativa
`forum/views.py` (7%), reduzindo o risco de as refatorações
introduzirem regressões não detectadas pela suíte.

---

## Smell 1 — Duplicated Code: `Category.get_all` e `Category.get_forums`

**Localização:** `flaskbb/forum/models.py:1526-1665`

```python
@classmethod
def get_all(cls, user: "User"):
    from flaskbb.user.models import Group

    if user.is_authenticated:
        user_groups = [gr.id for gr in user.groups]
        user_forums = (
            db.select(Forum)
            .filter(Forum.groups.any(Group.id.in_(user_groups)))
            .subquery()
        )
        forum_alias = aliased(Forum, user_forums)
        forums = (
            db.session.execute(
                db.select(cls, forum_alias, ForumsRead)
                .join(forum_alias, cls.id == forum_alias.category_id)
                .outerjoin(ForumsRead, db.and_(
                    ForumsRead.forum_id == forum_alias.id,
                    ForumsRead.user_id == user.id,
                ))
                .add_columns(forum_alias)
                .add_columns(ForumsRead)
                .order_by(Category.position, Category.id, forum_alias.position)
            ).unique().all()
        )
    else:
        # ... bloco quase idêntico para guest_group ...
    return get_categories_and_forums(forums, user)

@classmethod
def get_forums(cls, category_id: int, user: "User"):
    from flaskbb.user.models import Group

    if user.is_authenticated:
        user_groups = [gr.id for gr in user.groups]
        user_forums = (
            db.select(Forum)
            .filter(Forum.groups.any(Group.id.in_(user_groups)))
            .subquery()
        )
        forum_alias = aliased(Forum, user_forums)
        forums = (
            db.session.execute(
                db.select(cls, forum_alias, ForumsRead)
                .filter(cls.id == category_id)
                .join(forum_alias, cls.id == forum_alias.category_id)
                .outerjoin(ForumsRead, db.and_(
                    ForumsRead.forum_id == forum_alias.id,
                    ForumsRead.user_id == user.id,
                ))
                .add_columns(forum_alias)
                .add_columns(ForumsRead)
                .order_by(forum_alias.position)
            ).unique().all()
        )
    else:
        # ... bloco quase idêntico para guest_group ...
    if not forums:
        abort(404)
    return get_forums(forums, user)
```

**Por que é um smell:** os dois métodos de classe têm ~95% do código
idêntico — a mesma lógica de filtrar fóruns por grupo do usuário
autenticado vs. grupo de convidado, montar o `aliased(Forum, ...)` e
executar a query com `ForumsRead`. A única diferença real é o filtro
por `category_id` em `get_forums` e a função final de formatação do
resultado (`get_categories_and_forums` vs. `get_forums`). Qualquer
mudança na regra de visibilidade por grupo (ex.: adicionar um novo
critério de permissão) precisa ser replicada manualmente nos dois
métodos, com alto risco de as duas cópias divergirem com o tempo.

---

## Smell 2 — Duplicated Code / Data Clumps: informações de "último post"

**Localização:** `flaskbb/forum/models.py:317-325` (`Post.save`),
`388-421` (`Post._deal_with_last_post`), `495-500`
(`Post._restore_post_to_topic`) e `1158-1170`
(`Forum.update_last_post`)

```python
# Post.save (linha ~317)
topic.forum.last_post = self
topic.forum.last_post_user = self.user
topic.forum.last_post_title = topic.title
topic.forum.last_post_username = user.username
topic.forum.last_post_created = created

# Post._deal_with_last_post (linha ~410)
self.topic.forum.last_post = second_last_post
self.topic.forum.last_post_title = second_last_post.topic.title
self.topic.forum.last_post_user = second_last_post.user
self.topic.forum.last_post_username = second_last_post.username
self.topic.forum.last_post_created = second_last_post.date_created

# Forum.update_last_post (linha ~1159, ramo "encontrado")
self.last_post = last_post
self.last_post_title = last_post.topic.title
self.last_post_user_id = last_post.user_id      # <- nome diferente!
self.last_post_username = last_post.username
self.last_post_created = last_post.date_created
```

**Por que é um smell:** o grupo de 5 campos
(`last_post`, `last_post_title`, `last_post_user`/`last_post_user_id`,
`last_post_username`, `last_post_created`) sempre "anda junto" e é
atribuído em bloco em pelo menos **4 lugares diferentes** do arquivo —
um caso clássico de *Data Clumps* combinado com *Duplicated Code*. O
custo já se manifestou como um bug latente: em três desses quatro
lugares o campo se chama `last_post_user`, mas em
`Forum.update_last_post` ele é atribuído como `last_post_user_id`
(linha 1159) — uma inconsistência que só existe porque a lógica foi
copiada e colada em vez de centralizada em um único método.

---

## Smell 3 — Duplicated Code: cálculo de `read_cutoff`

**Localização:** `flaskbb/forum/models.py:702-705`
(`Topic.tracker_needs_update`) e `1197-1200` (`Forum.update_read`)

```python
read_cutoff = None
if flaskbb_config["TRACKER_LENGTH"] > 0:
    read_cutoff = time_utcnow() - timedelta(
        days=flaskbb_config["TRACKER_LENGTH"]
    )
```

**Por que é um smell:** o mesmo bloco de 4 linhas, que traduz a
configuração `TRACKER_LENGTH` (em dias) em uma data de corte, aparece
idêntico nos dois métodos responsáveis por decidir se um tópico/fórum
está "não lido". É uma regra de negócio pequena, mas central para o
sistema de readtracker — duplicá-la aumenta o risco de as duas cópias
ficarem incoerentes se a regra mudar (ex.: trocar de dias para horas).

---

## Smell 4 — Long Method: `Forum.update_read`

**Localização:** `flaskbb/forum/models.py:1177-1263` (~87 linhas)

```python
def update_read(
    self, user: "User", forumsread: ForumsRead | None, topicsread: TopicsRead | None
):
    if not user.is_authenticated or topicsread is None:
        return False
    read_cutoff = None
    if flaskbb_config["TRACKER_LENGTH"] > 0:
        read_cutoff = time_utcnow() - timedelta(days=flaskbb_config["TRACKER_LENGTH"])
    unread_count = db.session.execute(
        db.select(db.func.count()).select_from(Topic)
        .outerjoin(TopicsRead, db.and_(...))
        .outerjoin(ForumsRead, db.and_(...))
        .filter(...)
    ).scalar_one()
    if unread_count == 0:
        # ... 3 sub-casos (já lido / atualizar existente / criar novo) ...
        ...
    # ... log final e return False
```

**Por que é um smell:** o método mistura, em um único bloco, três
responsabilidades distintas — (1) checagem de guarda, (2) construção
de uma query SQL de vários `outerjoin`/`filter`, e (3) a árvore de
decisão do que fazer com o registro `ForumsRead` (não mexer, atualizar
ou criar). Isso o torna difícil de testar unidade por unidade (hoje é
0% coberto) e difícil de ler de uma vez — o leitor precisa manter na
cabeça o estado de `forumsread`, `topicsread` e `unread_count`
simultaneamente até a última linha.

---

## Smell 5 — Comentário / Ramo Morto (Dead Code): `else` inalcançável em `Topic.update_read`

**Localização:** `flaskbb/forum/models.py:770-782`

```python
if topicsread:
    logger.debug("Updating existing TopicsRead '{}' object.".format(topicsread))
    topicsread.last_read = time_utcnow()
    topicsread.save()
    updated = True
elif not topicsread:
    logger.debug("Creating new TopicsRead object.")
    topicsread = TopicsRead()
    ...
    updated = True
else:
    updated = False
```

**Por que é um smell:** `if topicsread` e `elif not topicsread` juntos
já cobrem **todos** os valores possíveis de `topicsread` (verdadeiro
ou falso) — não sobra nenhum caso para o `else` final tratar. O ramo
`else: updated = False` é código morto que nunca executa, mas continua
sendo lido e mantido como se fizesse parte da lógica, o que confunde
quem tenta entender as regras de atualização do readtracker.

---

## Smell 6 — Primitive Obsession: "último post" como 5 colunas soltas

**Localização:** `flaskbb/forum/models.py` (colunas da classe `Forum`,
por volta da linha 1054-1123, usadas em todo o restante da classe e
em `Post`)

**Por que é um smell:** o conceito de "informação do último post de um
fórum" (`last_post`, `last_post_title`, `last_post_user`,
`last_post_username`, `last_post_created`) é tratado como cinco
primitivos/colunas independentes em vez de um único conceito de
domínio. Isso é a causa raiz dos Smells 2 e 3 acima: como não existe
um objeto/método único responsável por "definir o último post", cada
ponto do código que precisa atualizar essa informação reimplementa a
atribuição dos 5 campos manualmente, campo por campo.

---

## Resumo

| # | Smell | Localização principal |
|---|---|---|
| 1 | Duplicated Code | `Category.get_all` / `get_forums` (1526-1665) |
| 2 | Duplicated Code / Data Clumps | "last post" em `Post`/`Forum` (317-325, 388-421, 495-500, 1158-1170) |
| 3 | Duplicated Code | cálculo de `read_cutoff` (702-705, 1197-1200) |
| 4 | Long Method | `Forum.update_read` (1177-1263) |
| 5 | Dead Code (ramo inalcançável) | `Topic.update_read` (770-782) |
| 6 | Primitive Obsession | campos `last_post_*` de `Forum` |
