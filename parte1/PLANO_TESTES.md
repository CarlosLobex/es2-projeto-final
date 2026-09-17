# Plano de Testes — Parte 1, Tarefa 1.2

## Módulo escolhido: `flaskbb/forum/`

**Justificativa:** `forum/` é o núcleo de domínio do flaskbb — reúne os
modelos de Categoria, Fórum, Tópico e Post, além das rotas que
implementam a navegação e as ações do fórum (criar tópico, responder,
mover, fixar, bloquear, marcar como lido, etc.). É também o módulo
com o maior volume de código entre os três candidatos (1272 linhas) e
o que concentra o maior desequilíbrio de cobertura interna: o arquivo
de modelos (`models.py`) já parte de 58% de cobertura, enquanto o
arquivo de rotas (`views.py`, 499 linhas) está em apenas 7%. Esse
contraste torna o módulo um bom laboratório tanto para testes de
lógica de negócio (models) quanto para testes de fluxo web
(views/forms), o que atende bem aos requisitos de caminhos felizes,
bordas e mocks pedidos nas Tarefas 1.3 a 1.5. Além disso, `forum/` é
citado como hotspot sugerido para a Parte 2, o que garante
continuidade de trabalho sobre o mesmo código nas três partes do
projeto.

## Cobertura atual do módulo (baseline)

| Arquivo | Statements | Cobertos | Cobertura |
|---|---|---|---|
| `forum/__init__.py` | 2 | 0 | 0% |
| `forum/forms.py` | 91 | 0 | 0% |
| `forum/locals.py` | 29 | 11 | 38% |
| `forum/models.py` | 641 | 370 | 58% |
| `forum/utils.py` | 10 | 2 | 20% |
| `forum/views.py` | 499 | 34 | 7% |
| **Total `forum/`** | **1272** | **417** | **32,8%** |

## Meta concreta de incremento

**Meta: elevar a cobertura de linhas de `flaskbb/forum/` de 32,8%
para pelo menos 48% (+15 pontos percentuais)** até o final da Parte 1,
priorizando:

- Cobrir integralmente as funções de validação de `forum/forms.py`
  (atualmente 0%).
- Cobrir as rotas mais centrais de `forum/views.py` (visualização de
  fórum, criação de tópico, resposta a tópico), hoje quase sem testes.
- Fechar os ramos de borda ainda não testados em `forum/models.py`
  (ex.: exclusão em cascata, contagem de posts/tópicos após
  ocultação).

## Cenários ainda não cobertos (lista inicial)

| # | Cenário | Tipo | Onde |
|---|---|---|---|
| 1 | Criar um novo tópico com dados válidos através da rota de criação | Caminho feliz | `forum/views.py` |
| 2 | Responder a um tópico existente com um post válido | Caminho feliz | `forum/views.py` |
| 3 | Submeter o formulário de novo tópico com título vazio | Borda / erro | `forum/forms.py` |
| 4 | Submeter o formulário de novo tópico com conteúdo maior que o limite permitido | Borda | `forum/forms.py` |
| 5 | Usuário sem permissão tentar acessar um fórum restrito a um grupo | Erro / permissão | `forum/locals.py` + `forum/views.py` |
| 6 | Paginação da listagem de tópicos de um fórum com mais itens do que o tamanho de página | Borda | `forum/views.py` |
| 7 | Mover um tópico para um fórum de destino que não existe (id inválido) | Erro | `forum/views.py` |
| 8 | Marcar um fórum inteiro como lido e verificar que os tópicos deixam de aparecer como não lidos | Caminho feliz | `forum/models.py` |
| 9 | Excluir uma categoria que ainda contém fóruns associados | Borda | `forum/models.py` |
| 10 | Ordenação de tópicos fixados (*sticky*) aparecendo antes dos tópicos normais na listagem | Borda | `forum/views.py` |

Os 4 primeiros itens já cobrem caminho feliz + borda + erro
exigidos pela Tarefa 1.3; os itens 5 e 7 são bons candidatos para o
teste com mock/dublê (Tarefa 1.5, isolando checagem de permissão ou
consulta ao banco); o item 6 (paginação) e o item 3/4 (validação de
formulário com múltiplas combinações de entrada) são bons candidatos
para o teste parametrizado (Tarefa 1.4).
