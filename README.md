# Automação Inteligente de Cadastro de Itens

Sistema de automação para cadastro completo de itens a partir de e-mails com anexos em PDF, utilizando extração estruturada de dados com inteligência artificial, validação fiscal de NCM, classificação contábil automatizada por banco (IB, TH e Salinas) e inserção direta em banco de dados.

---

## Sumário

- [Sobre o Projeto](#sobre-o-projeto)
- [Arquitetura do Fluxo](#arquitetura-do-fluxo)
- [Funcionamento](#funcionamento)
- [Data Tables](#data-tables)
- [Regras de Negócio](#regras-de-negócio)
- [Tecnologias Utilizadas](#tecnologias-utilizadas)
- [Tratamento de Erros](#tratamento-de-erros)
- [Benefícios](#benefícios)
- [Evoluções Futuras](#evoluções-futuras)
- [Status](#status)

---

## Sobre o Projeto

Este projeto foi desenvolvido para automatizar o processo de cadastro de itens recebidos por e-mail, eliminando atividades manuais, reduzindo falhas operacionais e garantindo padronização das informações cadastradas em múltiplos bancos de dados simultaneamente.

O sistema realiza:

- Recebimento de solicitações via formulário
- Identificação e processamento de anexos em PDF
- Extração estruturada de dados via IA (Information Extractor)
- Iteração item a item com Loop Over Items
- Validação do NCM via IA dedicada e consulta ao banco de dados
- Consulta de código padrão em planilhas de controle (IB, TH e Salinas)
- Classificação contábil inteligente por banco (Classificador Conta IB, TH e Salinas)
- Inserção automática em Microsoft SQL para cada banco
- Atualização de registros após inserção
- Envio de e-mail com resumo do processamento

---

## Arquitetura do Fluxo

```
Form Submission (Trigger)
  ↓
Edit Fields → Extract from File (PDF)
  ↓
IA - Information Extractor (Google Gemini)
  ↓
Split Out → Edit Fields → Aggregate → Code in JavaScript
  ↓
If (tem código preenchido?)
  ├── [NÃO] → Responde e-mail ao solicitante
  └── [SIM] → Loop Over Items
                ↓
              IA - Validador NCM
                ├── [INVÁLIDO] → Tabela 101 → E-mail suporte + E-mail CNPJ errado
                └── [VÁLIDO]  → Decisão do Banco de Dados
                                  ├── [IB]      → SQL → Classificador IB      → SQL Insert → Update → E-mail
                                  ├── [TH]      → SQL → Classificador TH      → SQL Insert → Update → E-mail
                                  └── [Salinas] → SQL → Classificador Salinas → SQL Insert → Update → E-mail
```

---

## Funcionamento

### 1. Recebimento via Formulário

O fluxo é iniciado por um **Form Submission**. Os dados passam pelo **Edit Fields** e o PDF é extraído com o node **Extract from File**.

### 2. Extração de Dados com IA

O texto do PDF é processado pela IA **Information Extractor** (Google Gemini), que identifica os campos necessários ao cadastro: descrição do item, NCM, unidade de medida, código do fornecedor e demais atributos.

### 3. Preparação dos Dados

Os dados passam pelo **Split Out**, **Edit Fields**, **Aggregate** e um **Code in JavaScript** para normalização e verificação de campos preenchidos.

### 4. Verificação de Código Preenchido

Um **If** verifica se ao menos um dos códigos (`codigo_IB`, `codigo_TH`, `codigo_salinas`) está preenchido. Caso nenhum esteja, o sistema responde automaticamente ao solicitante via e-mail.

### 5. Iteração por Item

O **Loop Over Items** percorre cada item extraído individualmente para as etapas seguintes.

### 6. Validação do NCM

A IA **Validador NCM** analisa o NCM informado e decide se ele é compatível com a descrição do item.

- **Inválido**: passa pela Tabela 101, envia e-mail ao suporte informando incompatibilidade e segundo e-mail sobre problema com CNPJ.
- **Válido**: segue para a decisão do banco de dados.

### 7. Decisão do Banco de Dados

Um **If** direciona o item para o banco correto. Cada banco possui fluxo independente com os mesmos passos:

**Para cada banco (IB / TH / Salinas):**

1. Consulta ao **Microsoft SQL** para verificar se o NCM já existe
2. **Get Mold** busca o modelo correspondente ao item
3. **Code node** separa e formata o código conforme padrão do banco
4. **Edit Fields** ajusta os valores para inserção
5. **Classificador Conta** (IA dedicada por banco) retorna a conta contábil mais adequada — se nenhuma se encaixar, e-mail é enviado ao responsável
6. **Code node** monta o comando SQL de inserção
7. **Microsoft SQL** executa o INSERT
8. **If** valida o sucesso da inserção
9. **Edit Field + Update Mold** atualizam o registro na planilha de controle
10. **Send a message** envia confirmação por e-mail

---

## Data Tables

A planilha de dados serve como base auxiliar com duas funções:

- Armazenar as **contas contábeis** utilizadas pelos Classificadores de IA
- Manter em **uma única linha** os códigos dos itens de todos os bancos (IB, TH e Salinas), servindo como referência cruzada para identificação e padronização

---

## Regras de Negócio

- Nenhum item é cadastrado sem NCM válido e compatível com a descrição
- A classificação contábil é obrigatória para todos os bancos
- Se nenhuma conta contábil se encaixar, o responsável é notificado por e-mail
- Cada banco (IB, TH, Salinas) possui fluxo de validação, classificação e inserção independente
- Todo item processado é registrado e atualizado para auditoria
- O fluxo só finaliza após o processamento completo de todos os itens do anexo
- Em caso de erro crítico, o processo é interrompido e o responsável é notificado

---

## Tecnologias Utilizadas

- **n8n** — Orquestração e automação do workflow
- **Google Gemini** — IA para extração estruturada e validação de NCM
- **IA Classificador Conta** — Classificação contábil por banco (IB, TH e Salinas)
- **Microsoft SQL Server** — Banco de dados relacional
- **Google Sheets** — Planilha auxiliar para contas contábeis e referência de códigos
- **Gmail** — Envio e recebimento de e-mails
- **PDF Extractor** — Extração de texto de arquivos PDF

---

## Tratamento de Erros

- **NCM inválido** → e-mail ao suporte e ao solicitante com a inconsistência
- **NCM incompatível com a descrição** → notificação automática via e-mail
- **Nenhuma conta contábil compatível** → e-mail enviado ao responsável para análise manual
- **Nenhum código preenchido** → e-mail automático ao solicitante
- **Falha na inserção** → verificação via If pós-inserção com tratamento do erro
- **Erros não tratados** → registrados para análise posterior

---

## Benefícios

- Eliminação de cadastros manuais em múltiplos bancos simultaneamente
- Padronização e validação fiscal automatizada via IA
- Classificação contábil inteligente e auditável
- Rastreabilidade completa (planilha + banco + e-mail)
- Escalabilidade para novos bancos sem redesenho do fluxo principal
- Redução de erros humanos e retrabalho

---

## Evoluções Futuras

- Integração com validação automática da tabela TIPI atualizada
- Cache de classificações contábeis recorrentes
- Dashboard de monitoramento em tempo real
- Sistema de logs centralizado
- Versionamento de regras de cadastro
- Ativação dos nodes SQL atualmente em modo desativado para produção
- Transformação em solução interna escalável para outros tipos de cadastro

---

## Status

Projeto em operação com evolução contínua. Nodes SQL em modo **Deactivated** indicam ambiente de homologação — ativação para produção é etapa planejada nas próximas evoluções.
