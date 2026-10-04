# Prompt para o agente — Santo Ramen: Ponto + Pontos de Recompensa

> Como usar: abra no VS Code o repositório do **Santo Ramen Time Tracker**
> (exportado do Lovable para o GitHub), inicie o agente (Claude Code) e cole
> tudo o que está abaixo da linha.

---

## Contexto

Você vai evoluir o app **Santo Ramen Time Tracker**, do restaurante Santo Ramen
(Calle 14 de Calacoto N8127, La Paz, Bolívia). Ele foi criado no Lovable e já
registra a entrada e a saída dos funcionários, as horas trabalhadas, as horas
extras e as faltas, além de gerar relatórios mensais. Também valida a
localização por GPS e tira uma foto na hora de bater o ponto.

O objetivo é adicionar ao app um **sistema de pontos de recompensa** integrado
ao ponto. O funcionário acompanha o próprio saldo e troca os pontos por prêmios.

## Antes de escrever código

1. Analise o projeto: a stack (provavelmente React, TypeScript, Tailwind,
   shadcn/ui e Supabase), a estrutura de pastas, o esquema do banco (pasta
   `supabase/migrations`), a autenticação, as políticas RLS e a forma como os
   registros de ponto são gravados hoje.
2. Escreva um resumo curto do que encontrou e um **plano de implementação em
   fases**. Espere minha aprovação antes de começar.
3. **Faça perguntas** sempre que algo estiver ambíguo. Não invente regras de
   negócio. Pergunte, por exemplo, quantos pontos vale cada ação.
4. Não quebre o que já funciona. Toda mudança no banco deve ser feita por uma
   nova migration, nunca editando as anteriores.

## Usuários e acesso

- Até 20 funcionários e **dois perfis**:
  - **Admin (dono):** gerencia tudo.
  - **Funcionário:** vê apenas os próprios dados.
- Duas formas de acesso:
  - **Celular pessoal:** PWA instalável, com login individual.
  - **Tablet fixo no restaurante (modo quiosque):** o funcionário escolhe o
    próprio nome e digita um **PIN de 4 a 6 dígitos** para bater o ponto e
    consultar o saldo. A sessão fecha sozinha depois de alguns segundos sem uso.
- Idiomas: **espanhol (padrão) e português**, com seletor. Todos os textos
  ficam em arquivos de tradução (i18n), nunca escritos direto no código.

## Regras de pontos (todas configuráveis pelo admin)

1. **Pontualidade e assiduidade (automático, a partir do ponto):**
   - Bônus por chegar no horário.
   - Bônus por semana ou mês sem faltas e sem atrasos (sequência).
   - Penalidade por atraso, acima de uma tolerância em minutos, e por falta
     sem justificativa.
   - Uma falta justificada ou aprovada pelo admin não gera penalidade.
2. **Lançamento manual pelo admin:** dar ou tirar pontos com **motivo
   obrigatório**, escolhido de uma lista de categorias como atendimento,
   limpeza, trabalho em equipe ou erro operacional, mais uma observação livre.
3. **Metas de vendas:** o admin cria metas (ex.: "vender 30 ramens no turno")
   com período, valor em pontos e funcionários participantes. Na primeira
   versão, o admin marca a meta como cumprida manualmente. Deixe preparado
   para integrar com um sistema de vendas (PDV) no futuro.
4. **Penalidades:** saem de qualquer uma das regras acima. O saldo pode ou não
   ficar negativo, conforme uma configuração do admin.

Os valores (quantos pontos vale cada coisa, a tolerância de atraso etc.) ficam
em uma **tela de configurações**, nunca fixos no código.

## Prêmios

- O admin mantém um **catálogo de prêmios** com nome, descrição, foto, custo
  em pontos, estoque opcional e status ativo ou inativo. Exemplos: folga extra,
  refeição grátis, bônus em dinheiro.
- O funcionário **solicita** a troca. O admin **aprova ou recusa** e depois
  marca o pedido como entregue.
- Os pontos ficam reservados quando o pedido é feito e voltam ao saldo se ele
  for recusado.

## Modelo de dados (sugestão, ajuste ao que já existe)

- `points_ledger`: um **extrato imutável**, com uma linha por movimentação.
  Campos: funcionário, valor (+/-), tipo (`attendance`, `manual`, `goal`,
  `redemption`, `refund`), motivo, referência à origem (registro de ponto,
  meta ou troca), quem lançou e data.
  O saldo é **sempre calculado** a partir do extrato (com uma view ou função),
  nunca guardado em um campo editável.
- `point_rules` / `settings`: os valores das regras.
- `reward_categories` / `reasons`: os motivos dos lançamentos manuais.
- `goals` e `goal_participants`.
- `rewards` e `redemptions`.
- PIN do quiosque guardado **com hash**, nunca em texto puro.
- Políticas **RLS** em todas as tabelas: o funcionário só lê os próprios dados
  e só o admin escreve. Os pontos automáticos de assiduidade são calculados no
  servidor (edge function ou função SQL agendada), nunca no navegador.
- O cálculo automático precisa ser **idempotente**: rodar duas vezes no mesmo
  dia não pode dar pontos em dobro.

## Telas

**Funcionário (mobile-first):**
- Início: saldo atual, botão de bater ponto e progresso das metas ativas.
- Extrato de pontos com filtro por período e o motivo de cada lançamento.
- Catálogo de prêmios e "meus pedidos".
- Minhas horas: o que o Time Tracker já mostra hoje.

**Admin:**
- Painel com o resumo da equipe: saldos, atrasos do mês e pedidos pendentes.
- Lançamento manual de pontos (um ou vários funcionários de uma vez).
- Metas, catálogo de prêmios, aprovação de trocas e configuração das regras.
- Cadastro de funcionários, incluindo o PIN do quiosque.
- Relatório mensal de pontos, exportável em CSV.

**Quiosque (tablet):** tela cheia, botões grandes e fluxo nome → PIN → bater
ponto ou ver saldo.

## Identidade visual

Siga o estilo do site do Santo Ramen, que mistura estética japonesa com cores
bolivianas, com tom vermelho escuro (#8b0000) e preto. Interface simples e
legível, pensada para quem usa o app correndo, durante o turno.

## Entregas e qualidade

- Trabalhe em fases pequenas e faça um commit por fase, com mensagens claras.
  Sugestão de ordem: (1) banco + RLS, (2) extrato e saldo, (3) lançamento
  manual, (4) pontos automáticos de assiduidade, (5) prêmios e trocas,
  (6) metas, (7) quiosque com PIN, (8) i18n e PWA.
- Escreva testes para a lógica de pontos: cálculo de atraso, idempotência,
  reserva e estorno de pontos.
- Ao final de cada fase, diga o que mudou, como testar e o que ainda falta.
- Atualize o README com o que preciso para rodar o projeto localmente e com as
  variáveis de ambiente.
