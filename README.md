# Agente Suporte E-commerce

Código-fonte do agente **Suporte E-commerce** do Microsoft Copilot Studio, que atende o time de e-commerce da CNH via Microsoft Teams e Microsoft 365 Copilot.

## O que o agente faz

Responde dúvidas sobre como utilizar o e-commerce, consulta documentos do SharePoint do departamento e traz dados de faturamento e atendimento.

| Recurso | Descrição |
|---|---|
| **9 fontes de conhecimento** | Documentos e planilhas do SharePoint, cada uma com instrução de quando usar |
| **Ação Power BI** | Consulta DAX no semantic model de vendas — faturamento de cupom |
| **Ação Agente OIC** | Fluxo do Power Automate com dados de atendimento |

## Estrutura

```
Suporte E-commerce/
├── agent.mcs.yml               # instruções e propósito do agente
├── settings.mcs.yml            # canais, autenticação, configurações de IA
├── connectionreferences.mcs.yml
├── knowledge/                  # 9 fontes de conhecimento
├── actions/                    # Power BI e Agente OIC
├── topics/                     # 13 tópicos de sistema
├── agents/                     # agentes conectados
└── workflows/                  # fluxo do Power Automate (Agente OIC)

PLANO-EVOLUCAO-AGENTE.md        # próximas etapas e decisões de arquitetura
```

A pasta `.mcs/` não é versionada — contém estado de sincronização local (o `changetoken.txt` muda a cada sync). Esse é o padrão do produto.

## Como trabalhar neste repositório

O agente é editado como arquivos YAML e sincronizado com o Copilot Studio. É preciso ter a [extensão Copilot Studio do VS Code](https://github.com/microsoft/vscode-copilotstudio) instalada — ela fornece o binário `LanguageServerHost` usado para validar, puxar e enviar.

```bash
# validar antes de enviar
node <plugin>/scripts/manage-agent.bundle.js validate --workspace "Suporte E-commerce" ...

# enviar (cria rascunho)
node <plugin>/scripts/manage-agent.bundle.js push --workspace "Suporte E-commerce" ...
```

Push cria um **rascunho**. As mudanças só ficam visíveis para os usuários após **publicar** no portal.

## Convenção das fontes de conhecimento

Cada fonte descreve quando deve e quando não deve ser usada. É isso que faz o orquestrador escolher a fonte certa:

```yaml
mcs.metadata:
  componentName: Nome da fonte
  description: |-
    O que a fonte contém.
    USE QUANDO: situações e exemplos de pergunta real entre aspas.
    NÃO USE PARA: o que não é o caso, apontando para a fonte correta.
```

Apontar para onde ir no `NÃO USE PARA` é importante — mandar embora sem endereço faz o modelo escolher qualquer outra fonte.

## Armadilhas conhecidas

Coisas que custaram tempo para descobrir:

- **`description` aceita `|-` mas NÃO aceita `>-`.** O folded scalar quebra o parser com erro genérico e sem número de linha: `BotElement Type not supported... Unhandled data type`.
- **`description` de input de ação tem limite de 1024 caracteres.** Estourar gera `StringLengthNotInRangeWarning` e o texto pode ser truncado.
- **Erro 503 no push pode mentir.** Já houve caso do Dataverse gravar e a resposta se perder — confira o portal antes de repetir o push.
- **O toggle Ativado/Desativado de ferramenta não vive no `.mcs.yml`.** É o campo `state`/`status` no `botdefinition.json`, que não é versionado.
- **Ferramenta sem `modelDescription` é acionada a esmo.** Com `GenerativeActionsEnabled: true`, o orquestrador chama por conta própria qualquer ferramenta habilitada. Toda ferramenta precisa dizer quando deve ser usada — ou ser desativada.

## Ambiente

O agente vive no ambiente **Personal Productivity only (default)** do Power Platform. Antes de virar ferramenta oficial do time, vale avaliar migração para um ambiente de produção: no ambiente default todo usuário licenciado do tenant tem papel de Maker, e não há garantia de backup.

Ver [PLANO-EVOLUCAO-AGENTE.md](PLANO-EVOLUCAO-AGENTE.md) para as próximas etapas.
