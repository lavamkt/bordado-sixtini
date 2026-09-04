# Documentação do Simulador de Bordado (protótipo)

Guia pra um agente de IA (ou pessoa) entrar no projeto e entender tudo sem precisar ler o
código inteiro antes. Leia nesta ordem:

| Arquivo | O que tem |
|---|---|
| [01-visao-geral.md](01-visao-geral.md) | O que é, pra quem é, stack, como abrir e rodar, o que está e o que não está pronto |
| [02-arquitetura.md](02-arquitetura.md) | Estrutura do `index.html`, objeto de estado, fluxo de dados, mapa de todas as funções com linha |
| [03-pipeline-bordado.md](03-pipeline-bordado.md) | Cada etapa do algoritmo em detalhe: preparo da imagem, cores, regiões, cetim x tatami, orientação, sprites, tecido, relevo |
| [04-parametros.md](04-parametros.md) | Todos os controles da interface, unidades, valores padrão, o que cada um dispara |
| [05-como-trabalhar.md](05-como-trabalhar.md) | Convenções de código e de texto, como testar (navegador e Edge headless), como depurar, armadilhas conhecidas, ideias de evolução |
| [06-processo.md](06-processo.md) | Regras obrigatórias de registro: atualizar o `CHANGELOG.md` a cada mudança, registrar todo bug corrigido em `07-bugs.md`, o que fazer quando um bug sobrevive à primeira correção |
| [07-bugs.md](07-bugs.md) | Registro de bugs encontrados e corrigidos: sintoma, causa raiz, correção |

Resumo em 5 linhas:

1. É **um único arquivo** `index.html` (HTML + CSS + JavaScript puro, sem build, sem dependências).
2. O usuário sobe um logo, a página o reduz a poucas cores e simula o bordado em Canvas 2D.
3. Todo o trabalho pesado é feito em `TypedArray`s (máscaras, mapas de distância) e desenho de sprites no canvas.
4. Existem dois níveis de recomputação: `full` (refaz tudo) e `render` (só redesenha), controlados por `run(kind)`.
5. Estado global fica no objeto `state`, exposto em `window.__sim` pra depuração.

Contexto do cliente: este protótipo pertence à pasta da **Sixtini** (cliente da Lava, uniformes
profissionais). Antes de escrever qualquer texto visível ao usuário, ler
`_Dev/_docs/escrita-geral.md` e `_Dev/Sixtini/ESCRITA.md` (sem travessão, sem emoji, tom B2B
direto). O `README.md` na raiz da pasta do protótipo é a versão curta pra humanos.
