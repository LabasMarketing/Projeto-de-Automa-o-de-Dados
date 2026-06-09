# 🤖 Automação de JSON para Excel e Dashboard no Power BI

Projeto de automação desenvolvido no **n8n** para processar arquivos JSON brutos contendo dados empresariais, normalizar essas informações com **IA**, gerar planilhas **XLSX** organizadas e construir um **dashboard interativo no Power BI**.

## 👨‍💻 Desenvolvedor

- [Gabriel Labarca Del Bianco](https://github.com/LabasMarketing)

---

## 🧠 Visão Geral

A ideia central do projeto é construir um pipeline capaz de:

1. Ler um arquivo JSON bruto do disco
2. Extrair e quebrar os dados em lotes menores
3. Enviar cada lote para um AI Agent no n8n
4. Normalizar os dados com IA
5. Reunir todos os lotes processados
6. Separar os dados em **funcionários** e **produtos**
7. Gerar duas planilhas XLSX independentes
8. Usar um arquivo `modelo.xlsx` como base visual
9. Gerar o relatório final `relatorio_final.xlsx`
10. Importar no Power BI e criar um dashboard com os dados tratados

---

## 🛠️ Tecnologias Utilizadas

| Tecnologia | Função |
|---|---|
| **n8n** | Orquestração do fluxo de automação |
| **JavaScript** | Lógica de criação de lotes e separação de dados |
| **AI Agent (n8n)** | Normalização e limpeza dos dados brutos |
| **OpenAI Chat Model** | Modelo de linguagem usado pelo agente |
| **Excel / XLSX** | Geração das planilhas de saída |
| **Power BI** | Criação do dashboard analítico |
| **JSON** | Formato de entrada dos dados brutos |

---

## 🏗️ Arquitetura do Fluxo

O projeto está organizado em 4 grandes etapas:

### 1. Leitura e preparação dos dados

Responsável por:

- ler o arquivo JSON do disco
- extrair os dados brutos
- quebrar os arrays em lotes de até 5 registros

### 2. Processamento com IA

Responsável por:

- iterar sobre cada lote via loop
- enviar o lote ao AI Agent
- aguardar a normalização
- controlar o ritmo das chamadas com o node Wait

### 3. Reunião e separação dos dados

Responsável por:

- reunir todos os outputs do agente
- converter o campo `output` em JSON válido
- separar os registros em `funcionarios` e `produtos`

### 4. Geração das planilhas

Responsável por:

- processar cada categoria separadamente
- converter para XLSX
- salvar os arquivos no disco
- gerar o relatório final com base no `modelo.xlsx`

---

## 🛠️ Etapas do Desenvolvimento

### 🗄️ Estrutura do Fluxo no n8n

O fluxo completo no n8n segue esta sequência:

```text
When clicking "Execute workflow"
↓
Read/Write Files from Disk
↓
Extract from File
↓
Code in JavaScript       ← cria lotes de até 5 registros
↓
Loop Over Items
↓
AI Agent                 ← normaliza cada lote
↓
Wait                     ← controla o ritmo das chamadas
↓
Retorno para Loop Over Items
↓
Done Branch
↓
Code in JavaScript1      ← reúne e separa em funcionários e produtos
↓
├── Funcionarios
│   ↓
│   Funcionarios1 - Split Out
│   ↓
│   Funcionarios2 - Convert to XLSX
│   ↓
│   Funcionarios3 - Write File to Disk
│
└── Produtos
    ↓
    Produtos1 - Split Out
    ↓
    Produtos2 - Convert to XLSX
    ↓
    Produtos3 - Write File to Disk
```

---

### 📌 1. Exemplo de JSON de Entrada

O JSON bruto de entrada segue esta estrutura:

```json
{
  "data": {
    "empresa": "TechNova Solutions",
    "cnpj": "83.705.508/0001-21",
    "fundacao": "2022-07-10",
    "sede": {
      "cidade": "Recife",
      "estado": "SP",
      "pais": "Brasil"
    },
    "receita_anual": 22995486.49,
    "funcionarios": [
      {
        "id": 1,
        "nome": "Larissa Silva",
        "cargo": "Gerente de Marketing",
        "departamento": "Marketing",
        "salario": 5472.93,
        "ativo": true
      }
    ],
    "produtos": [
      {
        "id": "P523",
        "nome": "ProdTech 24",
        "tipo": "SaaS",
        "preco_mensal": 978.82,
        "usuarios_ativos": 4404,
        "dia": 15,
        "mês": "Junho",
        "ano": 2024
      }
    ]
  }
}
```

---

### 📌 2. Code in JavaScript — Criação dos Lotes

O primeiro nó de código é responsável por **quebrar os arrays grandes em lotes menores** para evitar sobrecarregar o AI Agent com muitos registros de uma vez.

```javascript
const input = $input.first().json;
const data = input.data ?? input;

const tamanhoLote = 5;

const empresa = {
  empresa: data.empresa,
  cnpj: data.cnpj,
  fundacao: data.fundacao,
  sede: data.sede,
  receita_anual: data.receita_anual,
};

const funcionarios = data.funcionarios ?? [];
const produtos = data.produtos ?? [];

const lotes = [];

function criarLotes(tipo, itens) {
  for (let i = 0; i < itens.length; i += tamanhoLote) {
    lotes.push({
      json: {
        tipo,
        numero_lote: Math.floor(i / tamanhoLote) + 1,
        empresa,
        itens: itens.slice(i, i + tamanhoLote)
      }
    });
  }
}

criarLotes('funcionarios', funcionarios);
criarLotes('produtos', produtos);

return lotes;
```

**O que esse código faz:**

- lê o JSON bruto de entrada
- extrai os dados de contexto da empresa
- cria lotes de até 5 registros para funcionários
- cria lotes de até 5 registros para produtos
- retorna todos os lotes prontos para o loop

Cada lote carrega os campos `tipo`, `numero_lote` e `itens`, que são essenciais para a separação correta depois do processamento.

---

### 📌 3. AI Agent — Normalização dos Dados

O AI Agent recebe cada lote e retorna os dados limpos e normalizados.

O prompt usado garante que o agente sempre preserva a estrutura necessária para a separação posterior:

```text
INSTRUÇÃO DE SAÍDA OBRIGATÓRIA:

Você receberá um lote de dados JSON.

O JSON recebido poderá ter qualquer estrutura, mas normalmente virá com campos de controle como:

{
  "tipo": "...",
  "numero_lote": 1,
  "empresa": {...},
  "itens": [...]
}

Sua tarefa é processar apenas os registros principais do lote, normalmente encontrados no array "itens".

Retorne apenas JSON válido.
Não use blocos de código Markdown.
Não inclua explicações, introduções ou texto adicional.

FORMATO OBRIGATÓRIO DA RESPOSTA:

{
  "tipo": "...",
  "numero_lote": 1,
  "dados": []
}

Regras obrigatórias:
- Preserve exatamente o valor recebido em "tipo".
- Preserve exatamente o valor recebido em "numero_lote".
- O campo "dados" deve ser sempre um array.
- Cada objeto dentro de "dados" deve representar uma linha ou registro tabular.
- Use somente os campos encontrados no JSON recebido.
- Não invente campos.
- Não misture tipos diferentes de registro no mesmo retorno.
- Não retorne um array solto.
- Não retorne texto fora do JSON.
- Se existir um array "itens", processe os objetos desse array.
- Se não existir "itens", procure o array principal mais relevante do JSON recebido.
- Se não houver nenhum array, transforme o objeto principal em um único registro dentro de "dados".
- Adapte-se à estrutura real do JSON recebido.
- Normalize nomes de campos apenas quando fizer sentido, mantendo nomes claros e consistentes.
- Não repita dados de contexto, como empresa, sede ou organização, dentro de cada item,
  a menos que esses dados já estejam no próprio registro ou sejam necessários para a tabela.

Dados Brutos para Processar:
{{ JSON.stringify($json) }}
```

**Por que os campos `tipo`, `numero_lote` e `dados` são obrigatórios:**

Esses três campos são a âncora de todo o fluxo. Sem eles, o código posterior não consegue identificar se o lote pertence a funcionários ou produtos, e a separação falha. O prompt foi escrito de forma imperativa justamente para garantir que o agente nunca os omita.

---

### 📌 4. Code in JavaScript1 — Reunião e Separação dos Dados

Depois que todos os lotes passam pelo `Done Branch`, o segundo nó de código é responsável por **reunir tudo e separar em dois arrays finais**.

```javascript
const items = $input.all();

const funcionarios = [];
const produtos = [];

for (const item of items) {
  let json = item.json;

  if (json.output) {
    try {
      json = JSON.parse(json.output);
    } catch (error) {
      throw new Error('O output do AI Agent não é um JSON válido: ' + json.output);
    }
  }

  if (Array.isArray(json.funcionarios)) {
    funcionarios.push(...json.funcionarios);
  }

  if (Array.isArray(json.produtos)) {
    produtos.push(...json.produtos);
  }

  if (json.tipo === 'funcionarios' && Array.isArray(json.dados)) {
    funcionarios.push(...json.dados);
  }

  if (json.tipo === 'produtos' && Array.isArray(json.dados)) {
    produtos.push(...json.dados);
  }

  if (json.tipo === 'funcionarios' && Array.isArray(json.itens)) {
    funcionarios.push(...json.itens);
  }

  if (json.tipo === 'produtos' && Array.isArray(json.itens)) {
    produtos.push(...json.itens);
  }

  if (Array.isArray(json)) {
    for (const registro of json) {
      const keys = Object.keys(registro);

      const pareceFuncionario =
        keys.includes('id_funcionario') ||
        keys.includes('nome_funcionario') ||
        keys.includes('cargo') ||
        keys.includes('departamento') ||
        keys.includes('salario');

      const pareceProduto =
        keys.includes('id_produto') ||
        keys.includes('nome_produto') ||
        keys.includes('produto') ||
        keys.includes('categoria') ||
        keys.includes('preco') ||
        keys.includes('preco_mensal') ||
        keys.includes('usuarios_ativos') ||
        keys.includes('estoque');

      if (pareceProduto && !pareceFuncionario) {
        produtos.push(registro);
      } else if (pareceFuncionario) {
        funcionarios.push(registro);
      }
    }
  }
}

return [
  {
    json: {
      funcionarios,
      produtos
    }
  }
];
```

**O que esse código faz:**

- lê todos os outputs do `Done Branch`
- converte o campo `output` do AI Agent em JSON real
- junta todos os lotes de funcionários em um único array
- junta todos os lotes de produtos em outro array
- possui fallback por heurística de campos (caso o agente retorne array solto)
- retorna um objeto final com `funcionarios` e `produtos` prontos para as planilhas

---

### 📌 5. Geração das Planilhas

Após o `Code in JavaScript1`, o fluxo se divide em dois caminhos paralelos.

#### Caminho de Funcionários

```text
Funcionarios         ← prepara o campo de funcionários
↓
Funcionarios1        ← Split Out: separa cada funcionário em uma linha
↓
Funcionarios2        ← Convert to XLSX: converte os dados para planilha
↓
Funcionarios3        ← Write File to Disk: salva "funcionarios_json.xlsx"
```

#### Caminho de Produtos

```text
Produtos             ← prepara o campo de produtos
↓
Produtos1            ← Split Out: separa cada produto em uma linha
↓
Produtos2            ← Convert to XLSX: converte os dados para planilha
↓
Produtos3            ← Write File to Disk: salva "produtos_json.xlsx"
```

Cada caminho é totalmente independente. Os dados não se misturam em nenhum momento após a separação.

---

### 📌 6. Arquivo modelo.xlsx e Relatório Final

Existe um arquivo `modelo.xlsx` que serve como **base visual** para o relatório final.

O modelo define:

- cabeçalhos das colunas
- cores e estilos da tabela
- bordas e largura de colunas
- estrutura visual das abas
- formatação aplicada automaticamente

Um script Python usa esse modelo para gerar o arquivo `relatorio_final.xlsx`, que contém duas abas organizadas:

- `Funcionarios` — com todos os registros de colaboradores
- `Produtos` — com todos os registros de produtos

Esse arquivo é o que vai direto para o Power BI.

---

### 📌 7. Dashboard no Power BI

O arquivo `relatorio_final.xlsx` foi importado como fonte de dados no Power BI.

O dashboard contém:

- 📊 Cartão com soma total do preço mensal dos produtos
- 📈 Gráfico de usuários ativos por ano
- 📉 Gráfico de usuários ativos por produto
- 📋 Tabela completa de produtos
- 🔍 Filtros por ano e mês
- 📌 Visualizações para análise dos produtos e da empresa

---

## 📂 Estrutura do Repositório

```text
projeto-json-powerbi/
│
├── README.md
│
├── data/
│   ├── entrada.json
│   ├── funcionarios_json.xlsx
│   ├── produtos_json.xlsx
│   └── relatorio_final.xlsx
│
├── templates/
│   └── modelo.xlsx
│
├── n8n/
│   ├── workflow.json
│   └── screenshots/
│       ├── 01_fluxo_completo.png
│       ├── 02_code_lotes.png
│       ├── 03_ai_agent.png
│       ├── 04_saida_done_branch.png
│       ├── 05_funcionarios.png
│       └── 06_produtos.png
│
├── scripts/
│   └── gerar_relatorio.py
│
├── powerbi/
│   ├── dashboard.pbix
│   └── screenshots/
│       └── dashboard_final.png
│
└── docs/
    └── explicacao_fluxo.md
```

---

## ⚠️ Desafios Encontrados

Durante o desenvolvimento, alguns problemas exigiram atenção especial:

**1. Uso incorreto de `.map()` em objeto**
A tentativa inicial de usar `.map()` diretamente sobre o objeto de entrada causava erro, pois o método só funciona em arrays. Foi necessário acessar corretamente os campos `data.funcionarios` e `data.produtos`.

**2. AI Agent descartando produtos**
Em versões anteriores do prompt, o agente processava apenas o array de funcionários e ignorava os produtos. A solução foi reformular o prompt para deixar explícito que o campo `tipo` deve ser preservado e que o agente não deve misturar nem descartar registros.

**3. Retorno de array solto pelo agente**
O AI Agent retornava um array sem a estrutura de controle (`tipo`, `numero_lote`, `dados`), impossibilitando a separação posterior. O prompt foi reescrito com regras imperativas para forçar o formato correto.

**4. Perda de estrutura com Edit Fields**
O node Edit Fields criava apenas um campo `dados_limpos` e descartava o `tipo` e o `numero_lote`, quebrando toda a lógica de separação. Ele foi substituído pelo `Code in JavaScript1`.

**5. Separação incorreta de categorias**
Sem os campos `tipo` e `numero_lote` preservados, era impossível distinguir lotes de funcionários de lotes de produtos. A solução final foi garantir no prompt que esses campos sempre fossem mantidos, e no código, tratar também casos em que o agente retorna array solto via heurística de campos.

---

## 🔮 Melhorias Futuras

Algumas evoluções planejadas para o projeto:

- Detectar automaticamente todos os arrays do JSON, não apenas `funcionarios` e `produtos`
- Criar uma planilha automaticamente para cada array detectado
- Melhorar a validação do JSON de entrada antes do processamento
- Adicionar logs de processamento por lote para facilitar o debug
- Implementar tratamento para falhas e retentativas do AI Agent
- Automatizar a atualização do Power BI após geração do relatório
- Criar versão do fluxo sem dependência de IA para estruturas JSON simples
- Adicionar documentação de instalação e configuração do n8n
- Exportar e versionar o workflow do n8n em JSON

---

## 🧠 Aprendizados

Este projeto foi uma ótima oportunidade para praticar:

- Orquestração de fluxos complexos com n8n
- Uso de AI Agents para normalização de dados estruturados
- Engenharia de prompts com restrições imperativas de formato
- Processamento em lotes para contornar limitações de contexto do LLM
- Separação de dados por heurística de campos
- Geração e formatação de planilhas XLSX
- Construção de dashboards analíticos no Power BI
- Debugging de fluxos híbridos entre IA, código e arquivos

---

## 🚀 Como Executar

### Requisitos

- n8n instalado (local ou Docker)
- Credencial da OpenAI configurada no n8n
- Python 3.10+ (para o script de geração do relatório)
- Power BI Desktop instalado

### 1. Clonar o repositório

```bash
git clone <URL_DO_REPOSITORIO>
cd projeto-json-powerbi
```

### 2. Importar o workflow no n8n

Acesse o n8n, clique em **Import from file** e selecione o arquivo:

```
n8n/workflow.json
```

### 3. Configurar as credenciais

No n8n, configure a credencial da **OpenAI** no node AI Agent.

### 4. Colocar o JSON de entrada na pasta correta

Salve seu arquivo JSON como:

```
data/entrada.json
```

### 5. Executar o workflow

Clique em **Execute Workflow** e acompanhe o processamento dos lotes no painel do n8n.

### 6. Gerar o relatório final

Execute o script Python para gerar o `relatorio_final.xlsx` com base no modelo:

```bash
python scripts/gerar_relatorio.py
```

### 7. Abrir no Power BI

Abra o arquivo `powerbi/dashboard.pbix` no Power BI Desktop e atualize a fonte de dados apontando para `data/relatorio_final.xlsx`.

---

## 📄 Licença

Este projeto foi desenvolvido com fins educacionais, de estudo e prática em:

- automação de processos com n8n
- engenharia de prompts e AI Agents
- manipulação e geração de planilhas Excel
- integração entre ferramentas de dados
- construção de dashboards no Power BI
