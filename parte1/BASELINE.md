# Baseline do Projeto Final — Parte 1, Tarefa 1.1

## Fork utilizado

- **Fork pessoal:** https://github.com/CarlosLobex/flaskbb
- **Fork do professor (origem):** https://github.com/jeffsantos/flaskbb

## Ambiente

- **Sistema operacional:** Windows
- **Python:** 3.14.3
- **Gerenciador de pacotes:** `uv` 0.12.15
- **pytest:** 9.0.3 (com plugins `pytest-cov`, `pytest-mock`, `pytest-xdist`)

## Passos de setup executados

1. Fork de `jeffsantos/flaskbb` para `CarlosLobex/flaskbb`.
2. Clone local e configuração dos remotos (`origin` → fork pessoal,
   `upstream-prof` → fork do professor).
3. Instalação do `uv` e execução de `uv sync` para criar o ambiente
   virtual e instalar as dependências travadas em `uv.lock`.

## Obstáculos de ambiente encontrados e resolvidos

Durante a execução inicial da suíte de testes, dois problemas de
**ambiente** (não de código) precisaram ser resolvidos antes de a
baseline ficar 100% verde:

### 1. Falha por paralelismo no Windows

Ao rodar `uv run pytest` (execução padrão, com `pytest-xdist` em
paralelo), 215 dos 233 testes falharam com:

```
FileExistsError: [WinError 183] Não é possível criar um arquivo
já existente: '...\flaskbb\instance'
```

**Causa:** múltiplos workers de teste (`gw2`, `gw3`, `gw4`, `gw5`)
tentaram criar a mesma pasta `instance/` simultaneamente, causando uma
condição de corrida (*race condition*) específica do sistema de
arquivos do Windows.

**Solução:** executar a suíte sem paralelismo:

```bash
uv run pytest -n 0
```

### 2. Teste de tradução falhando por arquivos não compilados

Com o paralelismo desativado, restou 1 falha:
`tests/unit/utils/test_translations.py::test_flaskbbdomain_translations`,
por os arquivos de tradução (`.po`) não estarem compilados (`.mo`) no
ambiente recém-clonado.

**Solução:** compilar os catálogos de tradução antes de rodar os
testes (mesmo passo executado no workflow de CI oficial do flaskbb):

```bash
uv run flaskbb translations compile
```

## Resultado da baseline (100% verde)

Comando final executado:

```bash
uv run flaskbb translations compile
uv run pytest -n 0
```

Saída resumida:

```
====================== test session starts ======================
platform win32 -- Python 3.14.3, pytest-9.0.3, pluggy-1.6.0
collected 233 items

...

================ 232 passed, 1 skipped in 29.92s =================
```

O único teste "skipped" (`test_would_force_login_for_anon_in_guest_unallowed`,
em `tests/unit/forum/test_forum_utils.py`) já vem desativado
intencionalmente pelo autor original do flaskbb, com a mensagem
*"On GitHub Actions this test failed for whatever reason I cannot
identify"* — não é uma decisão nossa e não foi alterado.

## Cobertura inicial por módulo

Comando executado:

```bash
uv run pytest -n 0 --cov=flaskbb.forum --cov=flaskbb.management --cov=flaskbb.user --cov-report=term-missing
```

| Módulo | Statements | Cobertos | Faltando | Cobertura |
|---|---|---|---|---|
| `flaskbb/forum/` | 1272 | 417 | 855 | **32,8%** |
| `flaskbb/management/` | 837 | 63 | 774 | **7,5%** |
| `flaskbb/user/` | 560 | 172 | 388 | **30,7%** |
| **Total (3 módulos)** | **2669** | **652** | **2017** | **24%** |

Destaques da baseline:

- `flaskbb/forum/forms.py`, `flaskbb/management/forms.py` e
  `flaskbb/management/__init__.py` estão em **0%** de cobertura.
- `flaskbb/forum/views.py` (499 linhas) e
  `flaskbb/management/views.py` (543 linhas) são os maiores arquivos
  dos três módulos e estão entre os menos cobertos (7% e 8%,
  respectivamente).
- `flaskbb/forum/models.py` já parte de uma cobertura razoável (58%),
  a mais alta entre os arquivos grandes dos três módulos.
