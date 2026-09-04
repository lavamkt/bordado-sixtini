# 06. Processo e regras obrigatórias

Regras de disciplina de registro pra este protótipo — separadas das convenções de código (essas
ficam em [05-como-trabalhar.md](05-como-trabalhar.md)). Pedidas pelo Kewin em 2026-09-04: são pra
**sempre** seguir ao mexer neste projeto, sem precisar que ele peça de novo a cada sessão.

## Regra 1 — Toda mudança vira uma entrada no CHANGELOG.md

Depois de qualquer alteração no `index.html` (ou nos próprios docs, quando for uma mudança
relevante de comportamento/processo), adicionar uma entrada no topo de
[`../CHANGELOG.md`](../CHANGELOG.md), **na mesma sessão em que a mudança foi feita** — não deixar
pra depois, não confiar em lembrar retroativamente.

Formato: bloco `## AAAA-MM-DD` (data de hoje; se já existe um bloco de hoje, entrar nele em vez de
criar outro) com subseções `### Adicionado` / `### Alterado` / `### Corrigido` / `### Removido`
(só as que se aplicarem). Cada item:

- Frase curta dizendo **o quê** mudou.
- Se não for óbvio, uma frase de **por quê** (o que motivou, ou o problema que evita).
- Link pro doc técnico relevante quando fizer sentido (ex.: um ajuste no pipeline referencia
  [03-pipeline-bordado.md](03-pipeline-bordado.md)).

Este protótipo não tem número de versão (não é um plugin do LWA, não passa por release) — o
changelog é organizado por **data**, não por versão semântica.

## Regra 2 — Todo bug corrigido vira uma entrada no registro de bugs

Ao encontrar e corrigir um comportamento errado (não uma feature nova, um DEFEITO: algo que
deveria funcionar de um jeito e não funcionava), registrar em
[`07-bugs.md`](07-bugs.md) antes de considerar o trabalho terminado. A entrada precisa ter, no
mínimo:

1. **O que acontecia** (sintoma, de preferência como o Kewin descreveria vendo a tela).
2. **Causa raiz** (não só "onde", o PORQUÊ técnico — é o que evita o mesmo erro de novo em outro
   lugar do código).
3. **Como foi corrigido** (a mudança de verdade, com referência a função/linha quando ajudar).
4. **Como testar/reconhecer se voltar** (o sintoma específico, ou um passo de verificação).

Numeração sequencial `BUG-001`, `BUG-002`... Bug que nunca chegou a ser publicado (achado e
corrigido antes de qualquer versão "final" existir) ainda entra no registro — o valor está em
não repetir o mesmo erro, não em ter afetado alguém.

## Regra 3 — Bug que sobrevive à primeira correção: parar e comparar, não tentar de novo às cegas

Se uma correção não resolveu o que foi reportado (o Kewin testa e o sintoma continua), a segunda
tentativa **não** pode ser mais uma suposição lida do código. Pedir um dado concreto antes de
propor o próximo fix: um valor medido no console (`window.__sim`, `getComputedStyle`, etc.), um
screenshot, ou reproduzir o problema junto. Esta regra é do ecossistema Lava inteiro (ver
`Wordpress/CLAUDE.md`, seção "Qualquer bug que sobreviva a uma 1ª correção") — vale aqui do mesmo
jeito, mesmo sem sistema de tickets.

## Regra 4 — Toda mudança publicada também vai pro repositório público (GitHub Pages)

Desde 2026-09-04, este protótipo tem um repositório próprio no GitHub da Lava
(`github.com/lavamkt/bordado-sixtini`, conta `lavamkt` — credencial já configurada no Git
Credential Manager desta máquina, ver `_Dev/_docs/credenciais.md` → "GitHub — Lava") publicado via
GitHub Pages em **https://lavamkt.github.io/bordado-sixtini/**. É o link que o Kewin manda pro
cliente ver o protótipo funcionando, sem precisar abrir o arquivo local.

Pedido explícito do Kewin: **toda vez que o `index.html` (ou qualquer outro arquivo desta pasta)
mudar, commitar e dar `git push` pro `main` desse repositório também** — não só editar o arquivo
local. O repositório é isolado, só com o conteúdo desta pasta (`git init` feito direto em
`Prototipo Bordado/`, não no `_Dev` inteiro). Identidade do commit já configurada localmente nesse
repo (`user.name "Lava"`, `user.email "claude@lavamkt.com"`, mesmo padrão usado no repositório
`assinatura-nex` da Nex) — não precisa reconfigurar. Fluxo depois de qualquer mudança:

```bash
cd "Sixtini/Simulador/Prototipo Bordado"
git add -A
git commit -m "resumo da mudança"
git push
```

O GitHub Pages rebuilda sozinho a cada push (leva menos de um minuto); não precisa de nenhum passo
extra de deploy.

## Onde isso se encaixa nas outras regras do projeto

Este arquivo cobre só o registro (changelog + bugs). As convenções de código, como testar e como
depurar continuam em [05-como-trabalhar.md](05-como-trabalhar.md) — em especial a regra já
existente ali de atualizar [03-pipeline-bordado.md](03-pipeline-bordado.md) e a tabela de linhas
de [02-arquitetura.md](02-arquitetura.md) sempre que uma mudança mexer no pipeline ou deslocar
linhas de função.
