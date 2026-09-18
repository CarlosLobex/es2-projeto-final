# Novos Testes — Parte 1, Tarefas 1.3, 1.4 e 1.5

Todos os casos abaixo foram adicionados em um único arquivo novo:

**Arquivo:** `tests/unit/forum/test_forum_forms.py`
**Módulo testado:** `flaskbb/forum/forms.py` (0% de cobertura na baseline)
**Commit no fork:** `aa2511b` (https://github.com/CarlosLobex/flaskbb/commit/aa2511b)

## Lista de casos (Tarefa 1.3 — mínimo 8 casos)

| # | Caso | Classe | Linha aprox. | Tipo |
|---|---|---|---|---|
| 1 | `test_valid_inputs_creates_and_saves_topic` | `TestTopicForm` | 12 | Caminho feliz |
| 2 | `test_title_and_content_validation` (4 combinações, ver Tarefa 1.4) | `TestTopicForm` | 30 | Feliz + Borda/erro (parametrizado) |
| 3 | `test_track_topic_calls_user_track_topic` | `TestTopicForm` | 50 | Caminho feliz (com dublê) |
| 4 | `test_no_track_topic_calls_user_untrack_topic` | `TestTopicForm` | 66 | Borda (com dublê) |
| 5 | `test_valid_input_creates_and_saves_post` | `TestPostForm` | 83 | Caminho feliz |
| 6 | `test_empty_content_is_invalid` | `TestPostForm` | 93 | Erro |
| 7 | `test_valid_input_saves_report` | `TestReportForm` | 100 | Caminho feliz |
| 8 | `test_empty_reason_is_invalid` | `TestReportForm` | 109 | Erro |

Total: **8 funções de teste**, gerando **11 casos executados** (a
parametrização do item 2 expande em 4 execuções). Todos rodam
isolados entre si (cada um cria seus próprios dados via fixtures do
pytest) e não alteram nem desabilitam nenhum teste pré-existente da
suíte.

## Teste parametrizado (Tarefa 1.4)

**Caso:** `TestTopicForm::test_title_and_content_validation`

Combina 4 entradas diferentes para os campos `title` e `content` do
`TopicForm`, misturando entradas válidas e inválidas conforme exigido:

1. Título e conteúdo válidos, sem marcar "acompanhar tópico" → válido.
2. Título e conteúdo válidos, marcando "acompanhar tópico" → válido.
3. Título vazio, conteúdo válido → inválido.
4. Título válido, conteúdo vazio → inválido.

## Teste com dublê / mock (Tarefa 1.5)

**Casos:** `TestTopicForm::test_track_topic_calls_user_track_topic` e
`TestTopicForm::test_no_track_topic_calls_user_untrack_topic`.

**Por que o dublê foi necessário:** `TopicForm.save()` chama
`user.track_topic(topic)` ou `user.untrack_topic(topic)` dependendo do
campo `track_topic` do formulário, mas essas chamadas por si só não
alteram nenhum valor visível diretamente no objeto `Topic` retornado
— o efeito colateral fica só no relacionamento interno do usuário com
os tópicos rastreados. Testar apenas o resultado final exigiria
inspecionar uma tabela associativa do banco, o que tornaria o teste
mais frágil e menos direto. Usando `mocker.patch.object` para
substituir os métodos `track_topic`/`untrack_topic` do objeto `user`
por dublês, o teste verifica diretamente a **interação** esperada
(`assert_called_once()`), isolando o comportamento do formulário da
implementação de como o rastreamento de tópicos é persistido.
