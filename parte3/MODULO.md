# Módulo `flaskbb/forum/` — Parte 3, Tarefa 3.1

Números de linha citados neste documento valem para o estado do fork no
commit `2af86c5` (o das docstrings).

## 1. Propósito

O módulo `flaskbb/forum/` implementa o núcleo do fórum: a hierarquia
**Categoria → Fórum → Tópico → Post**, as páginas que o usuário navega
para ler e escrever nela, e as regras que a sustentam. Isso inclui o
controle de acesso por grupos (um fórum só é visível para os grupos
associados a ele), o *readtracker* que marca o que cada usuário já leu
(`TopicsRead`, `ForumsRead`), o acompanhamento de tópicos, as denúncias
de posts (`Report`), a moderação (trancar, destacar, ocultar e apagar
tópicos e posts) e a busca. Também expõe pontos de extensão para plugins
por meio de hooks do `pluggy`.

## 2. Mapa dos arquivos

| Arquivo | Linhas | Responsabilidade |
|---|---|---|
| `__init__.py` | 3 | Só cria o `logger` do pacote. |
| `models.py` | 1700 | Modelos SQLAlchemy (`Category`, `Forum`, `Topic`, `Post`, `Report`, `TopicsRead`, `ForumsRead`, tabela `topictracker`) e a maior parte da regra de negócio: contadores, "último post", readtracker, ocultar/apagar. |
| `views.py` | 1344 | 30 views baseadas em `MethodView` (uma por ação) e o registro de todas as rotas no blueprint `forum`. |
| `forms.py` | 212 | Formulários WTForms (`PostForm`, `ReplyForm`, `TopicForm`, `EditTopicForm`, `ReportForm`, formulários de busca) e o método `save` de cada um. |
| `locals.py` | 59 | *Proxies* `current_post`, `current_topic`, `current_forum` e `current_category`, resolvidos a partir de `request.view_args` e guardados em `g`. |
| `utils.py` | 39 | `force_login_if_needed`: exige login quando o fórum atual não aceita convidados. |

## 3. Pontos de entrada e saídas

### 3.1 Entradas

**Rotas HTTP.** A função `flaskbb_load_blueprints` (`views.py:1181-1344`)
é implementação do hook de mesmo nome (`@impl(tryfirst=True)`). Ela cria o
blueprint `forum`, registra 31 rotas (30 views; `NewPost` aparece duas
vezes) e monta o blueprint no prefixo `FORUM_URL_PREFIX` da configuração.
Antes de cada requisição do blueprint roda `force_login_if_needed`
(`views.py:1343`).

| Grupo | Rotas (sem o prefixo) |
|---|---|
| Navegação | `/`, `/category/<id>`, `/forum/<id>`, `/topic/<id>`, `/post/<id>`, `/who-is-online`, `/memberlist` |
| Criar e editar | `/<forum_id>/topic/new`, `/topic/<id>/post/new`, `/topic/<id>/post/<id>/reply`, `/topic/<id>/edit`, `/post/<id>/edit`, `/forum/<id>/edit` |
| Moderação de tópico | `/topic/<id>/delete`, `/lock`, `/unlock`, `/highlight`, `/trivialize`, `/hide`, `/unhide` |
| Moderação de post | `/post/<id>/delete`, `/post/<id>/hide`, `/post/<id>/unhide` |
| Leitura e acompanhamento | `/<forum_id>/markread`, `/topictracker`, `/topictracker/<id>/add`, `/topictracker/<id>/delete` |
| Outras | `/search`, `/post/<id>/raw`, `/post/<id>/report`, `/markdown`, `/markdown/<mode>` |

Vários caminhos aceitam também a variante `-<slug>` (por exemplo,
`/topic/<id>-<slug>`).

Nota: nas rotas registradas só existe `/<forum_id>/markread`. O ramo da
`MarkRead` que marca *todos* os fóruns como lidos (chamada sem
`forum_id`) não tem rota registrada em `views.py`; falta confirmar se
outro arquivo do flaskbb a chama.

**Hooks.** O módulo implementa `flaskbb_load_blueprints` (acima). Não
encontrei comandos CLI, envio de e-mail nem tarefas Celery neste módulo.

### 3.2 Saídas

| Destino | O que sai |
|---|---|
| Banco de dados | Tabelas `categories`, `forums`, `topics`, `posts`, `reports`, `topicsread`, `forumsread` e a tabela de associação `topictracker` (`models.py:85`). |
| HTML | 13 templates `forum/*.html` (`index`, `category`, `forum`, `topic`, `new_topic`, `new_post`, `edit_forum`, `report_post`, `memberlist`, `online_users`, `search_form`, `search_result`, `topictracker`), mais mensagens `flash` e redirecionamentos. |
| Hooks de plugins | Emitidos pelo módulo: `flaskbb_event_post_save_before/after` e `flaskbb_event_topic_save_before/after` (`models.py`), `flaskbb_form_post_save` e `flaskbb_form_topic_save` (`forms.py`), `flaskbb_form_post` e `flaskbb_form_topic`, `flaskbb_load_post_markdown_class` e `flaskbb_load_nonpost_markdown_class` (`views.py`). Os hooks são declarados fora deste módulo. |

## 4. Docstrings novas (commit `2af86c5` no fork)

Três docstrings em pontos em que o nome não explica o contrato:

| Onde | Linhas | Por que o nome não basta |
|---|---|---|
| `Post._deal_with_last_post` | `models.py:399-417` | O nome não diz o que o método muda. Só age quando o post é o último do tópico, pode reapontar também o último post do fórum (ou limpá-lo) e deve rodar antes do recálculo dos contadores. Só deixa alterações pendentes na sessão. |
| `Post._update_counts` | `models.py:451-467` | Recalcula contadores contando no banco em vez de incrementar. O resultado depende do flag `hidden`, que precisa ser alterado antes da chamada. Posts em tópicos ocultos não entram na conta. |
| `MarkRead` | `views.py:934-951` | Tem dois modos: um fórum ou todos. O modo "todos" apaga o histórico de leitura do usuário e cria um `ForumsRead` para todos os fóruns do banco, sem checar acesso. O parâmetro `slug` não é usado. |

Já existia, desde a Parte 2 (commit `412438c`), a docstring de
`Forum.set_last_post` (`models.py:1169`), que centraliza a atribuição dos
campos `last_post_*`.
