# Changelog — Simulador de Bordado (protótipo)

Histórico de toda mudança feita neste protótipo (`index.html`, e docs quando relevante). Entrada
nova sempre no **topo**, com a data do dia em que a mudança foi feita. Categorias: `Adicionado` /
`Alterado` / `Corrigido` / `Removido`, iguais ao padrão do resto do ecossistema Lava.

A regra de quando e como atualizar este arquivo está em
[`docs/06-processo.md`](docs/06-processo.md).

## 2026-09-04

### Adicionado
- Protótipo criado do zero: página única `index.html`, sem dependências, com o pipeline completo
  de simulação de bordado (preparo da imagem, quantização de cores por k-means, separação em
  cetim/tatami, sprites de fio iluminados, tecidos procedurais). Documentação técnica completa em
  [`docs/`](docs/README.md).
- Logo padrão da Sixtini (`LOGO PADRÃO.png`, enviado pelo Kewin) agora carrega automaticamente ao
  abrir a página — embutido como base64 (`DEFAULT_LOGO_URI`) direto no `index.html`, e não por
  caminho de arquivo relativo (isso "contaminaria" o canvas ao abrir via `file://` e quebraria
  toda a extração de cores). Ver [`docs/07-bugs.md`](docs/07-bugs.md) se algum problema de
  carregamento aparecer no futuro.
- Doc de regras/processo do projeto ([`docs/06-processo.md`](docs/06-processo.md)) e registro de
  bugs ([`docs/07-bugs.md`](docs/07-bugs.md)).

### Alterado
- Cor padrão do tecido trocada de marinho (`#1d2a44`) para branco (`#f4f4f1`, mesmo tom do preset
  "Branco" já existente na lista de cores de tecido).

### Adicionado (mesmo dia, sessão seguinte)
- **Emendas entre letras**: simula o fio que a máquina de bordar não corta entre dois elementos
  próximos da mesma cor (ex.: entre o "S" e o "I" de SIXTINI), com slider de intensidade 0-100%
  (0% = tudo aparado, como antes). Novas funções `findComponents`/`planJumps`/`drawJumpThread`.
  Ver [`docs/03-pipeline-bordado.md`](docs/03-pipeline-bordado.md), Etapa 11.
- **Altura do bordado independente da largura**: com "Manter proporção do logo" desligado
  (ligado por padrão), a altura vira um controle próprio e o resultado final estica na vertical
  pra caber no tamanho exato — simula precisar encaixar num espaço de altura fixa (bolso, aba).
  Ver Etapa 12 no mesmo doc.
- **Orçamento estimado**: campo "Preço por ponto" e total calculado ao vivo (pontos × preço),
  numa caixa destacada no painel. Ver seção "Orçamento" em
  [`docs/04-parametros.md`](docs/04-parametros.md).
