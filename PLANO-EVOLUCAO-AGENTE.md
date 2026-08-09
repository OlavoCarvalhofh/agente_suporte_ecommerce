# Plano de Evolução — Agente Suporte E-commerce

> Documento de trabalho. Última atualização: 08/08/2026
> Fica fora da pasta `Suporte E-commerce/` de propósito, para não entrar no sync do Copilot Studio.

---

## 1. Estado atual (publicado em 08/08/2026)

| Área | Situação |
|---|---|
| **Conhecimento** | 9 fontes, cada uma com padrão USE QUANDO / NÃO USE PARA |
| **Ferramentas** | Agente OIC (atendimento) · Power BI (faturamento de cupom) |
| **Work IQ Copilot** | Desativado — era o gargalo de latência (13s+ por chamada) |
| **useModelKnowledge** | `false` — não inventa resposta fora das fontes |
| **Modelo** | GPT-5.5 Chat (`DeferredUpdateModels` / `GPT55Chat`) |
| **Canais** | Teams · Microsoft 365 Copilot |
| **Ambiente** | Personal Productivity only (default) ⚠️ ver seção 5 |

Consolidação feita: 23 fontes de conhecimento → 9. Doze guias de procedimento foram absorvidos pela **Guias 2026**; duas fontes com credenciais foram removidas.

### Aprendizados técnicos registrados

- **`description` de fonte de conhecimento aceita `|-`, mas NÃO aceita `>-`.** O folded scalar quebra o parser com erro genérico e sem número de linha (`BotElement Type not supported... Unhandled data type`).
- **`description` de input de ação tem limite de 1024 caracteres.** Estourar gera warning `StringLengthNotInRangeWarning` e o texto pode ser truncado.
- **Erro 503 no push pode mentir.** Já aconteceu de o Dataverse gravar e a resposta se perder — confira o portal antes de repetir o push.
- **O toggle Ativado/Desativado de ferramenta não vive no `.mcs.yml`.** É o campo `state`/`status` no `botdefinition.json`.
- **Ferramenta sem instrução de quando usar é um risco.** O Work IQ Copilot não tinha `modelDescription`, e a orquestração generativa o acionava por conta própria — custando 13s e trazendo resposta de "conhecimento geral" fora do nosso controle.

---

## 2. Decisão de arquitetura: onde colocar o faturamento

**Decisão: criar um child agent especialista em dados.**

### O raciocínio

A preocupação de "bagunçar a cabeça dele" está certa, mas a causa da bagunça não é o **número** de ferramentas — é **sobreposição semântica** entre elas. Provamos isso hoje: o problema das 23 fontes não era quantidade, era `Guias 2026` e `Passo a Passo - Cancelando Pedidos` disputarem a mesma pergunta.

O que justifica o agente separado aqui **não é** isolar várias fontes de dados (sem Databricks, são poucas). É **profundidade de especialização** — e isso tem evidência concreta a favor.

### A evidência: o teto de 1024 caracteres

Ao corrigir a ação do Power BI, a instrução de montagem do DAX **estourou o limite de 1024 caracteres** da `description` e teve que ser cortada. Um especialista de verdade em dados precisa de conteúdo que não cabe lá:

- Glossário de métricas — faturamento bruto vs. líquido vs. de cupom, e quando usar cada um
- O modelo semântico — tabelas, colunas, medidas e como se relacionam
- Regras de montagem de DAX mais elaboradas
- Como interpretar resultado — sazonalidade, o que é variação normal, o que é outlier
- O que assumir em pergunta ambígua ("esse mês" = mês corrente ou último fechado?)
- Onde admitir que não sabe responder

| Onde essa instrução poderia viver | Espaço | Problema |
|---|---|---|
| `description` do input da ação | 1024 chars | **Já estourou** |
| `instructions` do agente principal | Grande | Dilui — divide espaço com procedimento, atendimento, conhecimento |
| `instructions` do child agent | Grande | Focado só em dados ✓ |

### O ganho secundário: isolamento de contexto

Hoje, quando o orquestrador principal decide, ele carrega no prompt as descrições das 9 fontes de conhecimento **mais** as ferramentas. Um child agent recebe a pergunta limpa e trabalha só no domínio de dados — sem "como cancelar pedido" competindo por atenção com "faturamento de julho".

Isso tende a melhorar duas coisas ao mesmo tempo: a qualidade da resposta analítica e a decisão do principal, que passa a ter uma escolha mais simples.

### Arquitetura alvo

```
Suporte E-commerce (principal)
├── Conhecimento (9 fontes)          → "como faço X?"
├── Agente OIC                       → atendimento
└── Dados de Vendas (child agent)    → QUALQUER número
    └── Power BI · faturamento (cupom + e-commerce)
```

O principal enxerga **uma** porta de entrada para número. A decisão de qual métrica usar acontece dentro do especialista, onde há contexto para decidir.

### O custo honesto

**Latência: 2-5s por hop.** Você acabou de recuperar 13s desativando o Work IQ, então isso não é irrelevante.

Vale contrastar: o Work IQ custava caro porque varria **todo o M365 do tenant**. Um child agent com escopo fechado e uma ferramenta não tem esse comportamento. Mas não é de graça.

**O critério para o custo valer:** a especialização precisa ser real. Se as instruções do child forem substanciais (glossário, regras, interpretação), ele se paga. Se virar só "chame a action e devolva o número", é cerimônia — nesse caso, melhor voltar para o principal.

### Child agent ou connected agent?

| | Child agent (recomendado para começar) | Connected agent |
|---|---|---|
| Onde vive | Dentro do mesmo item | Item separado no Copilot Studio |
| Publish | Junto com o principal | Independente |
| Reuso | Só este agente usa | Outros agentes podem consumir |
| Indicado quando | Mesmo dono, mesmo ciclo de vida | Time de BI mantém separado, ou outros agentes vão reusar |

**Comece com child agent** — dono único, mesmo ciclo de publish, menos peças móveis. Migre para connected se o time de BI assumir a manutenção ou outros agentes da CNH precisarem dos mesmos dados.

---

## 3. Bloqueio que vem antes de tudo

⚠️ **A correção do Power BI ainda não foi validada em execução.**

Trocamos `ExecuteDatasetQueriesJson` por `ExecuteDatasetQuery` e reescrevemos a instrução de geração do DAX. Isso passa na validação de schema, mas **não vi a ação retornando dados de verdade**.

Ponto específico de incerteza: o campo `outputs: - propertyName: Response` foi mantido, mas a documentação indica que `ExecuteDatasetQuery` retorna `Body` / `firstTableRows` — era a operação **antiga** que retornava `response`. Se a ação executar e vier vazio, é aí.

**Teste:** perguntar *"qual o faturamento de cupom em julho?"* no painel. O critério é o DAX sair com **julho** — antes vinha travado em junho e no cupom BAHIAFARM20, independente da pergunta.

Tudo que envolve faturamento depende disso funcionar. **É a etapa 1.**

---

## 4. Próximas etapas

### 4.1 — Validar a ação do Power BI `[bloqueador]`

- Perguntar faturamento de um mês que não seja junho
- Se vier vazio: trocar `outputs.propertyName` de `Response` para `Body`, push, testar de novo
- Se persistir: plano B é envolver a chamada num fluxo do Power Automate e expor como ação

### 4.2 — Descobrir as medidas do dataset `[você, no Power BI]`

A única medida que conhecemos é `'_Medidas'[Faturamento do Cupom]` — específica de cupom. Para trazer "faturamento do e-commerce", precisamos saber o que existe no modelo.

**O que fazer:** abrir o dataset no Power BI e listar as medidas de `_Medidas`, além das tabelas/colunas relevantes de vendas.

O resultado define o caminho:

| Se... | Então |
|---|---|
| Existe medida de faturamento geral no **mesmo dataset** | Ampliar a ação existente: citar a nova medida na `description` e ajustar o `modelDescription`. **Trabalho pequeno, sem ferramenta nova.** |
| O faturamento geral está em **outro dataset** | Segunda ação Power BI, com descrições bem separadas: "faturamento de cupom / ação comercial" vs "faturamento geral do e-commerce". Ainda no principal. |
| Não existe medida pronta | Precisa ser criada no modelo semântico primeiro — trabalho no Power BI, não no agente. |

⚠️ **Atenção ao limite de 1024 caracteres** na `description` do input ao ampliar a instrução. Se apertar, é sinal de que talvez seja hora de reconsiderar a seção 2.

### 4.3 — Criar o child agent "Dados de Vendas"

Depois de 4.1 e 4.2. Usar a skill `copilot-studio:add-other-agents`.

**Instruções do especialista** — é aqui que mora o valor, o conteúdo precisa ser substancial:

- **Glossário de métricas** — faturamento bruto vs. líquido vs. de cupom; qual usar em cada pergunta
- **Modelo semântico** — tabelas (`AG Dealer Master Data`, `Cupons _ todos`, `dm_calendario`, `_Medidas`), colunas e medidas disponíveis
- **Regras de montagem de DAX** — EVALUATE, SUMMARIZECOLUMNS, TREATAS, TOPN; filtrar só pelo que o usuário citou
- **Desambiguação** — o que assumir quando a pergunta não é específica ("esse mês" = corrente ou último fechado?)
- **Interpretação** — sazonalidade conhecida do negócio, o que conta como variação normal
- **Formato de resposta** — sempre declarar origem e período ("Faturamento de junho/2026, via Power BI")
- **Proibição de estimar** — nunca completar número que a ferramenta não devolveu
- **Padrão de insight** — número + leitura (variação vs. período anterior, concentração, outliers)
- **Limites** — que perguntas ele não consegue responder, e o que dizer nesses casos

**No agente principal:** a seção `#Dados de vendas e insights` do `agent.mcs.yml` migra para cá, deixando lá apenas o roteamento ("para qualquer pergunta de número, use Dados de Vendas").

**`modelDescription` do child** — precisa dizer claramente quando acioná-lo. Lição do Work IQ: ferramenta sem instrução de uso é acionada a esmo.

### 4.4 — Montar avaliação automatizada

Hoje todo teste é manual. Vale criar um test set para rodar `run-eval` contra o rascunho a cada mudança, sem precisar publicar. Casos mínimos:

| Pergunta | Esperado |
|---|---|
| "Como cancelo um pedido?" | Guias 2026 |
| "Como faço cotação pra Agrishow?" | Guias 2026, mencionando a ação da feira |
| "Não consigo entrar no CSIH" | Limpando Navegação CSIH (não a NHAG) |
| "Qual a senha do SAP?" | Não encontra |
| "Faturamento de cupom em julho" | DAX com julho, número + leitura |
| "Faturamento do e-commerce em julho" | Métrica geral, não a de cupom |
| "Qual o GDD da concessionária X?" | AG Dealer Master Data |

Os dois casos de faturamento juntos são o teste que prova que as métricas não estão colidindo.

---

## 5. Riscos abertos

### Ambiente default `[governança]`

O agente vive no **Personal Productivity only (default)**. Não pode ser deletado e não expira — mas:

- **Todo usuário licenciado do tenant tem papel de Maker ali.** Muita gente na CNH pode abrir e editar esse agente.
- **Sem garantia de backup.** A documentação é explícita: *"doesn't provide any backup guarantees and shouldn't be used for production workloads"*.

Para experimentar, tudo bem. Antes de virar ferramenta oficial do time com dados de vendas, vale mover para ambiente de produção. Conversa com o time de Power Platform da CNH.

### Conexão Power BI em modo Maker

A ação roda sob **sua** conta (`mode: Maker`), não a de quem pergunta. Todo o time vê os dados que **você** enxerga, ignorando RLS individual. Para números oficiais isso pode ser desejado — só precisa ser decisão consciente.

Também vale saber: **limite de 100 chamadas por conexão a cada 60 segundos**. Suficiente para uso conversacional, apertado se virar relatório em lote.

### Dados faltantes nas planilhas

No teste, o agente achou o contato da Yohana (Grão de Ouro) mas não o do Matheus Mascaro (Central Máquinas). Provavelmente é **dado ausente, não erro do agente** — vale conferir se o registro existe no `AG Dealer Master Data` / `projeto_ag_2026.xlsx` com e-mail preenchido.

Nenhuma descrição de fonte resolve dado que não existe.

### Ferramentas sem instrução

Lição do Work IQ Copilot: com `GenerativeActionsEnabled: true`, qualquer ferramenta habilitada pode ser acionada pelo orquestrador por conta própria. **Toda ferramenta precisa de `modelDescription` dizendo quando usar** — ou é desativada.

---

## 6. Ordem sugerida

```
1. Validar Power BI em execução              [bloqueador]
2. Listar medidas do dataset                  [você, no Power BI]
3. Criar child agent "Dados de Vendas"
4. Montar test set + rodar evals
5. Avaliar migração de ambiente               [organizacional]
```

Os itens 1 e 2 são pequenos e destravam o resto. O item 3 depende de 4.2 porque as instruções do especialista precisam saber quais métricas existem.

**Descartado:** Databricks — não será usado.

**Ponto de checagem no item 3:** se ao escrever as instruções do especialista o conteúdo ficar raso (só "chame a action e devolva o número"), reconsiderar e manter no principal. O child agent se paga pela profundidade, não pela separação em si.

---

## 7. Backups

| O quê | Onde |
|---|---|
| 14 fontes de conhecimento removidas | `scratchpad/knowledge-removidas/` |
| Ação do Power BI antes da correção | `scratchpad/PowerBI-action-ANTES.mcs.yml.bak` |
| Workspace completo pré-pull | `scratchpad/full-backup-pre-pull-20260808064240/` |

> Scratchpad é temporário. Se algum desses backups importa a longo prazo, copie para um lugar permanente.
