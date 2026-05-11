# Automacao de Cadastro de Itens de Consumo

Este repositorio contem dois workflows de automacao desenvolvidos em n8n para o cadastro de itens de consumo fiscal nos bancos de dados das empresas do grupo (INBRANDS, TOMMY e SALINAS). Cada workflow atende a um publico e proposito diferente.

---

## Visao Geral dos Workflows

### 1. Cadastro itens de consumo completo (fluxo principal — usuarios externos)

**Arquivo:** `Cadastro_itens_de_consumo_completo.json`

Este e o fluxo principal da automacao. Ele e voltado para usuarios que submetem solicitacoes de cadastro de novos itens por meio de um formulario, podendo enviar multiplos itens de uma vez via PDF ou planilha. O processo inclui validacao de NCM, classificacao contabil por IA e insercao direta no banco de dados.

#### Fluxo de execucao

**Entrada e extracao de dados**

O processo se inicia com o envio de um formulario (`On form submission`). Um no de extracao de texto (`Extract from File`) le o PDF ou arquivo enviado. Um agente de IA (`Information Extractor`, via Google Gemini) identifica os campos relevantes do documento e retorna os dados estruturados (nome do item, NCM, CST, banco de destino, e-mail do solicitante, entre outros).

**Iteracao por item**

Os dados extraidos passam por um `Split Out` e depois por um `Loop Over Items`, garantindo que cada item do documento seja processado individualmente e de forma sequencial.

**Validacao do NCM por IA**

Cada item passa por um agente de IA (`Validar NCM`) que verifica se o NCM informado e compativel com a descricao do item. O resultado determina o caminho seguinte:

- NCM valido e coerente com o item: segue para a verificacao no banco.
- NCM invalido ou inconsistente com a descricao: um e-mail automatico e enviado ao suporte informando o problema. O item nao e cadastrado.
- NCM inexistente na tabela `CLASSIF_FISCAL` do banco: um e-mail automatico e enviado ao solicitante informando que o NCM esta errado e que o CNPJ esta arredado (bloqueado). O item nao e cadastrado.

**Verificacao do NCM no banco de dados**

O NCM valido e consultado na tabela `CLASSIF_FISCAL` via Microsoft SQL. Se o NCM nao existir no banco, o fluxo segue para o envio de notificacao de erro. Se existir, o fluxo continua para a decisao de banco de destino.

**Decisao do banco de destino**

Um no Switch (`Decisao do Banco de dados`) identifica a empresa de destino com base no campo `BANCO` do item:

- `09` — INBRANDS
- `15` — TOMMY
- `06` — SALINAS

Cada ramo segue para a conexao SQL correspondente ao banco da empresa.

**Classificacao da conta contabil por IA**

Para cada empresa, um agente de IA (`Classificador conta IB`, `Classificador conta TH`, `Classificador conta Salinas`) recebe a descricao do item e decide qual conta contabil deve ser utilizada, consultando uma lista de contas pre-definida como ferramenta (`Contas IB`, `Contas TH`, `Contas Salinas`).

- Se a conta contabil for identificada: o fluxo segue para geracao do codigo do item e insercao no banco.
- Se nenhuma conta se encaixar: um e-mail e enviado a um responsavel para tratativa manual.

**Geracao do codigo do item**

O ultimo codigo cadastrado e recuperado do banco (`Get row(s)`). Um no de codigo JavaScript extrai o prefixo alfabetico e o numero sequencial, incrementa o numero e gera o novo codigo formatado (ex.: `IF66480` torna-se `IF66480`).

**Insercao no banco de dados**

O comando `INSERT INTO CADASTRO_ITEM_FISCAL` e executado com todos os campos necessarios:

- `ITEM_DESCRICAO`
- `CODIGO_ITEM`
- `UNIDADE` (fixo: `UN`)
- `TRIBUT_ORIGEM` (CST)
- `CLASSIF_FISCAL` (NCM)
- `CONTA_CONTABIL`
- `INDICADOR_CFOP` (fixo: `13`)
- `ITEM_FISCAL_GRUPO`
- `TIPO_ITEM_SPED` (fixo: `07`)
- `IDENT_UTILIZACAO_IMOB` (fixo: `9`)

Em caso de erro na insercao, o workflow continua e registra a falha sem travar o loop.

**Confirmacao por e-mail**

Apos a insercao bem-sucedida, um e-mail de com os códigos e as descrições dos itens e enviado ao usuario que submeteu o formulario, informando que o item foi cadastrado com sucesso.

---

### 2. Cadastro itens de consumo (fluxo interno — uso dos analistas)

**Arquivo:** `Cadastro_itens_de_consumo.json`

Este fluxo e destinado ao uso interno pelos analistas da equipe fiscal. Ele e acionado manualmente via formulario interno e pula toda a etapa de validacao de NCM por IA, partindo diretamente da decisao do banco de destino e da classificacao de conta contabil. E utilizado em situacoes onde o analista ja tem os dados validados e precisa cadastrar um item de forma direta e rapida.

#### Diferencas em relacao ao fluxo completo

- Nao ha leitura de PDF nem extracao por IA de campos.
- Nao ha validacao de NCM (nem por IA nem por consulta ao banco antes do cadastro).
- O trigger e um formulario interno (`On form submission1`) preenchido diretamente pelo analista.
- Os dados ja chegam estruturados, sem necessidade de iteracao por loop.
- O fluxo segue diretamente para a decisao do banco de destino e classificacao da conta contabil por IA.

#### Fluxo de execucao

1. Formulario interno preenchido pelo analista com os dados do item.
2. Switch decide o banco de destino (INBRANDS, TOMMY ou SALINAS) com base no campo `Banco de dados`.
3. Agente de IA classifica a conta contabil adequada para o item.
4. Caso nenhuma conta se encaixe, um e-mail e enviado ao responsavel.
5. Codigo do item e gerado incrementando o ultimo codigo cadastrado no banco.
6. `INSERT INTO CADASTRO_ITEM_FISCAL` e executado no banco correspondente.
7. E-mail de confirmacao e enviado ao analista.

---

## Estrutura dos Bancos de Dados

Os tres bancos de destino sao identificados pelos 2 primeiros números do CNPJ que esta na nota e cada um possui sua propria conexao SQL configurada no n8n:

| Empresa  | Codigo |
|----------|--------|
| INBRANDS | 09     |
| TOMMY    | 15     |
| SALINAS  | 06     |

---

## Notificacoes e Tratativas de Erro

| Situacao | Acao automatica |
|---|---|
| NCM invalido ou inconsistente com o item | E-mail enviado ao suporte com detalhes do problema |
| NCM nao encontrado no banco (`CLASSIF_FISCAL`) | E-mail enviado ao solicitante informando que o CNPJ esta arredado |
| Nenhuma conta contabil compativel encontrada | E-mail enviado a um responsavel para tratativa manual |
| Insercao no banco bem-sucedida | E-mail de confirmacao enviado ao solicitante |

---

## Quando usar cada workflow

Use o **fluxo completo** (`Cadastro_itens_de_consumo_completo.json`) quando:
- Um usuario externo ou solicitante precisa submeter itens para cadastro.
- O processo precisa incluir validacao automatica do NCM.
- Ha um PDF ou arquivo com multiplos itens a serem processados.

Use o **fluxo interno** (`Cadastro_itens_de_consumo.json`) quando:
- Um analista precisa cadastrar um item de forma direta, sem passar pela validacao de NCM.
- Os dados ja estao verificados e o objetivo e apenas executar o cadastro no banco.
