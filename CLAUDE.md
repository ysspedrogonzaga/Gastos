# Projeto Gastos — Guia para o Claude Code

Este documento descreve o app de controle de gastos pessoais do Pedro (Jota). Serve como referência de arquitetura, convenções e do fluxo mensal de atualização. Leia por completo antes de mexer no código.

## O que é o projeto

App de página única (um arquivo `index.html`, sem build step) que rastreia receitas e despesas mensais, marca contas como pagas, detalha faturas de cartão por categoria e mostra gráficos comparativos. Hospedado no **GitHub Pages** (`https://ysspedrogonzaga.github.io/Gastos`). Os dados vivem no próprio código (`MESES_CFG`); não há banco ativo.

- Repositório: `github.com/ysspedrogonzaga/Gastos` (arquivo servido: `index.html`)
- Deploy: automático pelo GitHub Pages ao dar push na branch `main`
- Stack: HTML + CSS + JavaScript puro (vanilla), Motion (versão vanilla do Framer Motion, `motion@11.18.2` via CDN, global `Motion`) para micro-interações. Gráficos em SVG próprio (sem Chart.js). Sem framework, sem bundler.
- Fonte: Neue Montreal (Regular 400, Bold 700) servida de `fonts/`. Não usar o peso Light (300); o mínimo é Regular.

## Arquitetura do arquivo `index.html`

Tudo vive num arquivo só (mais a pasta `fonts/`), em três blocos: `<style>`, marcação do topo/dock de abas, e um `<script>` no final. O padrão é **data-driven**: cada mês é descrito por arrays de configuração e renderizado por funções genéricas. Para adicionar um mês, adiciona-se dados + uma função `buildX`, não marcação nova.

### Abas
`const TABS = ['maio','junho','julho','agosto','setembro','outubro','graficos','historico']`. Os meses ficam num dock flutuante (`.dock`) no rodapé, com nomes completos e rolagem horizontal dentro da pílula (`.dock-sc`, bordas esmaecidas por `dockEdges()`); o indicador branco desliza via `moveDock()` e some em Gráficos/Histórico. Gráficos e Histórico são links no topo (`.top-nav`, ao lado do wordmark "Gastos"). Os IDs `tab-<aba>` continuam iguais nos dois lugares. `showTab(t)` alterna a classe `active` nas abas e páginas e chama `buildMes(cfg)` / `buildGraficos()` na primeira visita (lazy render, controlado por `dataset.built`), seguido de `enterPage()` (entrada orquestrada). `buildHist()` também é lazy e lê de `MESES_CFG`. Ao carregar, o app abre no mês mais recente (`showTab(MESES_CFG.at(-1).key)`).

### Modelo de dados de um mês
Cada mês salvo é um objeto:
```js
{ mes: 'Julho 2026', mesKey: '2026-07', fields: {
    salario, vr, vho, paula,          // receitas
    aluguel, cond, iptu, luz, internet, terapia, cartao   // despesas
} }
```
`calc(f)` deriva: `tot` (receitas), `desp` (despesas fixas, sem cartão), `total` (desp+cartao), `sobrou` (tot - desp - cartao).

### Funções genéricas de render (reutilizar sempre)
Cada mês é descrito em `MESES_CFG` (receitas, despesas, faturas, saldos, `mode:'input'|'contas'`, `parcial`, `infoAlert`) e renderizado por `buildMes(cfg)`; `recalcMes(cfg)` atualiza números no lugar (com contagem animada).
- `heroHtml` → card "Resumo" (preto com gradiente animado). A barra de composição é um seletor: tocar numa fatia (ou na pílula da legenda) chama `setHeroView(K, 'sobra'|'fixas'|'cartao')` e `paintHero(cfg)` troca legenda, valor e variação. Sempre abre em `'sobra'` (`cfg._view` volta a `'sobra'` ao reentrar no mês). Fica bordô (`.is-neg`) quando o mês fecha negativo; o número só fica vermelho na visão "Faltou".
- `summaryHtml(cfg)` → 4 cards brancos de métricas (receitas, despesas fixas, cartão, saldo) com variação vs. mês anterior.
- `inputRow(label, id, val, badge, ev)` → linha editável. `badge` é `'auto'` ou `'rep'`.
- `checkRow(key,label,val,pago,edit,ph,prefix,ev,prevVal)` → conta com check "pago". `prefix` isola os IDs por mês (ex.: `j`, `jl`, `ag`).
- `progCard` + `updProg(contas, prefix, alertId, lblId, pctId, barId)` → card preto de pagamentos (% grande + segmentos).
- `faturaHtml(fat, span)` + `toggleCat(id)` → fatura por categoria (accordion animado). `fat.banco` = 'Inter' | 'Personnalité'; `fat.prev` aponta para a fatura anterior do mesmo banco para mostrar variação por categoria.
- `buildPie(hostId, legendId, cats)` → anéis concêntricos (até 8; o resto vira "Demais"), maior categoria = 270°. `buildCartaoBar(hostId, mesDestaque)` → barras em pílula de `CARTAO_MESES` (derivado de `MESES_CFG`). Estes são os DOIS gráficos padrão do detalhe de cartão.
- `regChart(id, draw)` registra o gráfico para redesenhar no resize.
- Dinheiro grande: `moneyHTML(n)` (R$ e centavos em sobrescrito) + `setMoney(el, v)` (anima do valor anterior). Variações: `deltaHTML(d, 'gasto'|'ganho', mesRef)` e `dchip(d, kind)` — subir gasto = vermelho, subir ganho = verde.

### Fonte dos dados (sem banco)
`MESES_CFG` é a fonte única. `cfgFields(cfg)` extrai os valores de um mês direto da config, e dele derivam `CARTAO_MESES` (barras do cartão), `MESES_G` (resumo em Gráficos) e o Histórico — não duplicar valores à mão. O botão "Salvar" e a leitura do Supabase foram removidos em ago/2026 (o Histórico passou a mostrar todos os meses automaticamente).

Legado, sem uso no app: tabela `meses` no Supabase (`id text pk`, `dados jsonb`, `atualizado_em timestamptz`, RLS com policy pública), vazia. Credenciais caso a persistência volte (públicas por design — a publishable key só funciona sob RLS):
- `SUPA_URL = https://hzfvsohazcqgwpupmgju.supabase.co`
- `SUPA_KEY = sb_publishable_pwloxHbQF_HVlepTgoTnSA_W04qWLR8`

## Convenções de negócio (NÃO QUEBRAR)

Estas regras são a lógica central da planilha. Um erro aqui gera números errados sem parecer erro.

1. **Salário defasado.** O salário que financia as contas de um mês é o **recebido no mês anterior** (Pedro recebe dia 15 e no último dia útil; usa isso para pagar o mês seguinte). Ex.: as contas de julho são calculadas com o salário recebido em... julho? NÃO — reveja: a regra que ele definiu é que o salário do mês N financia o mês N+1. Na prática os créditos do Itaú de cada mês têm sido ~R$ 13.981,77 e o campo de salário do mês seguinte repete esse valor. Rotule sempre o campo como "Salário (Itaú — recebido em [mês anterior])".
2. **Fatura do cartão = mês em que é paga.** A fatura paga em maio conta como gasto de maio, independente de quando as compras foram feitas. O app do Inter nomeia a fatura pelo mês de pagamento (fatura "Julho" vence 07/07 e refere-se a compras de junho). Use sempre o "Valor total" da fatura no app como número oficial do campo `cartao`.
3. **Paula.** Soma de todos os Pix recebidos de "PAULA EDUARDO" dentro dos dias 1–30/31 do mês, no extrato do Inter. Só entradas.
4. **Vale Refeição / Vale Home Office.** Caem no cartão de benefícios Flash, valores estáveis — repetir o valor do mês anterior (badge `rep`).
5. **Dois cartões.** Inter (principal) e Itaú Personnalité. O campo `cartao` do mês é a **soma** das duas faturas pagas naquele mês. Cada fatura tem seu detalhamento separado.
6. **Página do mês = dias 1 a 30/31.** Tudo antes/depois é ignorado naquele mês.
7. **IPTU:** costuma zerar no fim do ano quando as parcelas acabam. Link de pagamento e índice cadastral são dados fixos do Pedro (não hardcodar no app; é contexto).
8. **Transferências pontuais** (ex.: um Pix de R$ 100.000 entre contas do próprio Pedro) são ignoradas no cálculo — não são receita.

## Convenções visuais / padrões do projeto (manter em todos os meses futuros)

- **Detalhe do cartão sempre com dois gráficos:** anéis concêntricos (distribuição por categoria) + barras (fatura ao longo dos meses). Use `buildPie` + `buildCartaoBar`, empilhados na coluna `.stack s5` ao lado das faturas (`.stack s7`).
- **Gráfico do Resumo Mensal com toggle** "Lado a lado" (grouped) / "Empilhado" (stacked). Não remover a visão grouped; o stacked é adicional e soma despesas fixas + fatura numa barra.
- **Linhas de total** ("Total receitas", "Total despesas") em tabelas levam preenchimento cinza `rgba(128,128,128,0.24)`.
- **Nomes amigáveis** nas categorias de fatura (mapa abaixo). Aplicar sempre que o lançamento reaparecer.
- **Dark mode** automático via `@media (prefers-color-scheme: dark)`. Os gráficos vivem em cards pretos, então usam cores fixas (`INK`, `MIST`, `SLATE`, hachuras SVG).
- **Hierarquia de cards:** título display gigante fora dos cards; cards brancos (`.c-w`) para informação secundária; cards pretos (`.c-b`) ou pretos com gradiente animado (`.c-b.grad` + `glow()`) para destaque.
- **Movimento:** respeitar `prefers-reduced-motion` (helper `still()`); sem animação de hover em cards.
- Formatação de moeda: `fBR(n)`; parse de string BR: `pBR(s)`.

### Mapa de renomeações (aplicar sempre)
| No extrato/fatura | Exibir como |
|---|---|
| 99 Ride | 99 |
| SUP EPA DEZESSEIS | Epa |
| Zig Mascate Flores | Mascate Floresta |
| Ativa Esporte | Adidas |
| ANTHROPIC* CLAUDE SUB | Claude Pro (Assinatura) |
| MP*MELIMAIS | Mercado Livre + (Assinatura) |

### Mapa de fornecedores → categoria de conta fixa (extrato Inter)
- Aluguel → ERA EMPREENDIMENTOS IMOBILIARIOS LTDA
- Condomínio → PJBANK **ou** PARTNERBANK (mudança de sistema; ambos = condomínio)
- IPTU → PM BELO HORIZONTE / MUNICIPIO DE BELO HORIZONTE
- Luz → CEMIG DISTRIBUICAO SA (Pix) **ou** "Debito automatico: 008133442437" (débito automático da CEMIG, ~dia 11, paga a conta do mês anterior). O app da CEMIG mostra cada conta pelo mês de referência; aqui vale a data do pagamento
- Internet + Celular → CLARO SA
- Terapia → DIALOGO PSI (desde set/2026; antes Julia Maria Pereira Alves). Valor variável; às vezes paga meses atrasados de uma vez
- Fatura cartão Inter → "Debito Automatico Fatura Cartao Inter"

## Fluxo mensal de atualização (o que muda a cada mês)

1. Fechar o mês anterior (parcial → definitivo) e abrir o novo como parcial.
2. Ler extrato Inter (período 01/mês-passado até hoje) → gastos do mês + pagamentos de contas fixas.
3. Ler extrato Itaú → salário (lembrar da regra de defasagem).
4. Ler faturas (Inter + Personnalité) → totais oficiais e detalhamento por categoria.
5. No código: adicionar objeto `FATURA_X` (com `banco` e `prev`), uma entrada em `MESES_CFG` (gráficos e histórico se atualizam sozinhos), entrada em `TABS`, uma `<section id="page-x">` e um botão no dock. Seguir o padrão de um mês existente (agosto é o mais completo: tem fatura + Personnalité + charts).
6. Testar local (abrir o arquivo no browser), depois `git add . && git commit && git push` a partir de `main` → deploy automático.

## Estado atual dos dados (até outubro/2026)

- **Maio a Setembro:** fechados. Agosto fechado em 10/10/2026 com o extrato Inter (Paula R$ 1.633,05; saldo final R$ 116.616,07).
- **Setembro:** salário R$ 13.981,77 (recebido em agosto). Cartão = Inter R$ 6.610,11 (`FATURA_AGOSTO`, compras de agosto, vence 07/09, débito 08/09) + Personnalité R$ 149,99 (03/09, sem detalhamento) = R$ 6.760,10. Terapia R$ 1.440,00 = 2 Pix "DIALOGO PSI" em 10/09; os R$ 1.260 quitam dois meses atrasados (confirmado pelo Pedro). Luz R$ 711,99 = duas contas CEMIG pagas no mês: agosto (R$ 357,99, débito automático 11/09) e setembro (R$ 354,00, Pix 10/09). Não houve pagamento em dobro.
- **Outubro:** parcial (até 10/10). Salário R$ 16.552,21 (inclui férias pagas em 03/09). Cartão = Inter R$ 5.766,10 (`FATURA_SETEMBRO`, compras de setembro, vence 07/10); Personnalité zerada. Aluguel agendado para 13/10 (conta como pago). Terapia pendente.
- **Luz por mês de referência (app CEMIG):** mar R$ 347,53 · abr R$ 335,16 · mai R$ 323,67 · jun R$ 393,94 · jul R$ 374,93 · ago R$ 357,99 · set R$ 354,00 · out R$ 329,73. A conta de outubro foi paga por Pix em 10/10, mas ainda aparece em débito automático no app da CEMIG (risco de pagar duas vezes).
- **Categorias:** "Compras para a Paula" (ex.: Sephora R$ 212,00 em set/2026, reembolsada por Pix da Paula no mesmo dia, que entra como receita "Paula"). Despesas pontuais de férias vão em "Viagem e hospedagem".
- **Pendências conhecidas:** Paula de julho não conferida (o extrato começa em 13/07); itens "Loja Lourdes", "TicketCard", "Piscinas", "Fabiola Barbosa Godinh", "Snapfast" ficam em Outros (o Pedro não sabe do que se tratam); "Tabaco" (R$ 185,00, fatura de agosto) também está em Outros, a confirmar.

## Sugestões de melhoria (se o Pedro pedir)

- Extrair CSS/JS para arquivos separados se o projeto crescer.
- Tela de "novo mês" que gera os campos a partir do mês anterior automaticamente.
- Import/export de JSON como backup dos dados.
