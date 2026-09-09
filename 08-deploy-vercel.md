# Tutorial 8 — Implantação na Vercel

> **Pré-requisitos:** Tutoriais 5, 6 e 7 concluídos; conta no GitHub; Git instalado.
> **Duração estimada:** 3 aulas de 50 minutos.
> **Ao final deste tutorial, o estudante será capaz de:** explicar o modelo de execução *serverless* e suas implicações, adaptar uma aplicação Express para a Vercel, publicar a API e a aplicação web, configurar variáveis de ambiente na plataforma, substituir o SQLite local por um banco de dados adequado à nuvem e diagnosticar as falhas mais comuns de implantação.

> **Advertência sobre atualidade.** A Vercel altera sua interface e suas opções de configuração com frequência. Os conceitos apresentados — funções *serverless*, ausência de sistema de arquivos persistente, variáveis de ambiente — permanecem válidos; nomes de menus e telas podem divergir. A documentação oficial encontra-se em <https://vercel.com/docs>.

---

## 1. O modelo serverless

### 1.1 Servidor tradicional versus função serverless

Nos tutoriais anteriores, o comando `node servidor.js` iniciava um processo que permanecia em execução, ocupando memória e mantendo uma porta aberta, à espera de requisições.

A Vercel adota modelo distinto. O código é empacotado em **funções**, que permanecem inativas até a chegada de uma requisição. Nesse momento, a plataforma inicia uma instância, processa a requisição, devolve a resposta e, após um período de inatividade, descarta a instância.

| Aspecto | Servidor tradicional | Função serverless |
|---|---|---|
| Execução | Contínua | Sob demanda |
| Custo | Por tempo de disponibilidade | Por invocação e tempo de processamento |
| Escalabilidade | Manual ou configurada | Automática |
| Estado entre requisições | Preservado em memória | **Não preservado** |
| Sistema de arquivos | Persistente | **Somente leitura**, exceto `/tmp`, efêmero |
| Latência do primeiro acesso | Nenhuma | *Cold start* de centenas de milissegundos |

### 1.2 Consequências práticas

**Variáveis em memória não persistem.** O vetor `mensagens` do Tutorial 5 seria reiniciado a cada invocação, e duas requisições consecutivas podem ser atendidas por instâncias diferentes.

**O arquivo SQLite não é gravável.** O sistema de arquivos da função é somente leitura, com exceção de `/tmp`, cujo conteúdo é descartado ao encerrar a instância. Um `INSERT` executado em uma requisição não estará disponível na seguinte. Esta é a limitação central deste tutorial, tratada na Seção 5.

**`app.listen()` não é utilizado.** A plataforma gerencia a rede. A aplicação deve **exportar** o objeto Express, não iniciá-lo.

**Há limite de tempo de execução.** Processos longos são interrompidos. Tarefas demoradas exigem arquitetura assíncrona com filas.

---

## 2. Preparação do repositório

A Vercel implanta a partir de um repositório Git. Cada envio ao ramo principal dispara uma nova implantação.

### 2.1 Instalação e configuração do Git

```powershell
winget install Git.Git
```

Após abrir um novo terminal:

```powershell
git config --global user.name "Nome do Estudante"
git config --global user.email "email@exemplo.com"
git config --global init.defaultBranch main
```

### 2.2 Verificação do `.gitignore`

Antes do primeiro envio, confirmar que o arquivo `.gitignore` contém:

```gitignore
node_modules/
.env
.env.local
.vercel
dados/*.sqlite
*.log
```

**Verificação obrigatória:** o arquivo `.env` contém credenciais. Uma vez enviado a um repositório público, deve ser considerado comprometido — remover o arquivo em um envio posterior não apaga o histórico. A conferência antes do primeiro `git push` é indispensável.

```powershell
git init
git add .
git status          # conferir que .env e node_modules NÃO aparecem
git commit -m "Versão inicial da API de tarefas"
```

### 2.3 Envio ao GitHub

Criar um repositório em <https://github.com/new>, **sem** README, `.gitignore` ou licença, e executar:

```powershell
git remote add origin https://github.com/USUARIO/api-tarefas.git
git branch -M main
git push -u origin main
```

---

## 3. Adaptação da API

### 3.1 Estrutura exigida

A Vercel reconhece automaticamente arquivos situados na pasta `api/` como funções *serverless*.

```
api-tarefas/
├── api/
│   └── index.js          ← ponto de entrada da função
├── app.js                ← aplicação Express (inalterada)
├── servidor.js           ← execução local (não utilizado na Vercel)
├── config/
├── modelos/
├── rotas/
├── servicos/
├── controladores/
├── middlewares/
├── vercel.json
└── package.json
```

A separação entre `app.js` e `servidor.js`, adotada no Tutorial 7, revela agora sua utilidade: `app.js` não invoca `listen()` e pode ser exportado diretamente.

### 3.2 O ponto de entrada

**Arquivo `api/index.js`:**

```javascript
// api/index.js — ponto de entrada para a Vercel.
// A plataforma importa este arquivo e utiliza o valor exportado como
// manipulador das requisições. Não se deve chamar app.listen() aqui:
// o gerenciamento da rede é responsabilidade da plataforma.

const app = require("../app");

module.exports = app;
```

### 3.3 Configuração da plataforma

**Arquivo `vercel.json`:**

```json
{
  "version": 2,
  "rewrites": [
    {
      "source": "/(.*)",
      "destination": "/api"
    }
  ]
}
```

A regra encaminha todas as requisições, qualquer que seja o caminho, à função `api/index.js`. O roteamento interno permanece a cargo do Express, exatamente como em ambiente local.

**Observação:** desde meados de 2026, a Vercel oferece detecção automática de aplicações Express: basta exportar o objeto `app` a partir de `index.js`, `server.js` ou `app.js` na raiz do projeto, dispensando a pasta `api/` e o arquivo `vercel.json`. A configuração explícita apresentada acima continua funcionando e é preferível em contexto didático, por tornar visível o mecanismo de roteamento.

### 3.4 Ajustes no `package.json`

```json
{
  "name": "api-tarefas",
  "version": "1.0.0",
  "main": "servidor.js",
  "engines": {
    "node": ">=20.0.0"
  },
  "scripts": {
    "start": "node servidor.js",
    "dev": "nodemon servidor.js"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "express": "^4.19.2",
    "helmet": "^7.1.0",
    "morgan": "^1.10.0",
    "pg": "^8.11.5",
    "sequelize": "^6.37.3"
  },
  "devDependencies": {
    "nodemon": "^3.1.0"
  }
}
```

**Ponto de atenção:** o campo `engines` declara a versão mínima do Node.js. Ausente essa declaração, a plataforma pode selecionar uma versão distinta da utilizada em desenvolvimento.

**Ponto de atenção adicional:** o pacote `sqlite3` compila código nativo. Ainda que a implantação seja bem-sucedida, o banco não persistirá, pelas razões da Seção 1.2. A Seção 5 substitui o SQLite por uma alternativa adequada.

---

## 4. Implantação

### 4.1 Método 1 — Interface web

1. Acessar <https://vercel.com> e criar uma conta autenticando-se com o GitHub.
2. Selecionar **Add New → Project**.
3. Escolher o repositório `api-tarefas` e clicar em **Import**.
4. Verificar as configurações:
   - *Framework Preset*: **Other**;
   - *Root Directory*: `./`;
   - *Build Command*: deixar em branco;
   - *Output Directory*: deixar em branco;
   - *Install Command*: `npm install`.
5. Expandir **Environment Variables** e cadastrar as variáveis necessárias (Seção 4.3).
6. Clicar em **Deploy** e aguardar de um a três minutos.

Ao final, é apresentada uma URL no formato `https://api-tarefas-<identificador>.vercel.app`.

### 4.2 Método 2 — Linha de comando

```powershell
npm install -g vercel

vercel login          # autenticação
vercel                # implantação de pré-visualização
vercel --prod         # implantação de produção
```

Comandos complementares:

```powershell
vercel ls                        # lista as implantações
vercel logs <url>                # exibe os registros de execução
vercel env ls                    # lista as variáveis de ambiente
vercel env add NOME_DA_VARIAVEL  # adiciona uma variável
vercel rollback                  # retorna à implantação anterior
vercel domains ls                # lista os domínios
```

### 4.3 Variáveis de ambiente

O arquivo `.env` não é enviado ao repositório e, portanto, não chega à plataforma. As variáveis devem ser cadastradas em **Settings → Environment Variables**.

| Variável | Valor | Ambientes |
|---|---|---|
| `NODE_ENV` | `production` | Production |
| `DATABASE_URL` | *string* de conexão do banco | Production, Preview |
| `ORIGENS_PERMITIDAS` | domínios autorizados, separados por vírgula | Production, Preview |
| `API_KEY` | chave de escrita, se implementada | Production, Preview |

**Regra operacional:** após alterar uma variável, é necessário **reimplantar** o projeto. A modificação não é aplicada automaticamente às implantações existentes.

Os três ambientes disponíveis são **Production** (ramo principal), **Preview** (demais ramos e *pull requests*) e **Development** (execução local com `vercel dev`).

---

## 5. Substituição do banco de dados

Este é o ponto crítico do tutorial. A aplicação funciona localmente com SQLite, porém, na Vercel, as gravações são perdidas.

### 5.1 Diagnóstico do problema

Ao implantar a API com SQLite, observa-se o seguinte comportamento:

1. `GET /api/tarefas` responde `200` com lista vazia;
2. `POST /api/tarefas` aparentemente cria a tarefa e responde `201`;
3. `GET /api/tarefas`, executado em seguida, devolve novamente a lista vazia;
4. em outros casos, o próprio `POST` falha com `SQLITE_READONLY` ou `EROFS: read-only file system`.

**Causa:** a função *serverless* executa em um sistema de arquivos somente leitura. A pasta `/tmp` é gravável, porém seu conteúdo é descartado quando a instância é encerrada, e instâncias distintas não o compartilham.

### 5.2 Alternativas

| Solução | Compatível com Sequelize | Nível gratuito | Observação |
|---|---|---|---|
| **PostgreSQL gerenciado** (Neon, Supabase) | Sim (`pg`) | Sim | Caminho recomendado neste tutorial |
| **MySQL gerenciado** (PlanetScale, Aiven) | Sim (`mysql2`) | Variável | Alternativa equivalente |
| **Turso / libSQL** | Parcial | Sim | SQLite na nuvem; exige cliente próprio |
| **SQLite somente leitura** | Sim | — | Adequado apenas a dados que não mudam |
| **Hospedagem com servidor persistente** (Render, Railway, Fly.io) | Sim | Variável | Mantém o SQLite; não é *serverless* |

Adota-se aqui o **PostgreSQL gerenciado**, por ser compatível com o Sequelize sem alteração da lógica da aplicação — exatamente como ocorreu na alternância entre SQLite e MySQL no Tutorial 6.

### 5.3 Criação do banco

1. Acessar <https://neon.tech> e criar conta gratuita.
2. Criar um projeto; a região mais próxima reduz a latência.
3. Copiar a *connection string*, no formato:

```
postgresql://usuario:senha@ep-nome-123456.us-east-2.aws.neon.tech/neondb?sslmode=require
```

**Alternativa integrada:** o painel da Vercel oferece, em **Storage**, a criação de bancos com configuração automática das variáveis de ambiente.

### 5.4 Adaptação do código

```powershell
npm install pg pg-hstore
```

**Arquivo `config/banco.js` (versão final):**

```javascript
// config/banco.js — conexão compatível com desenvolvimento local
// (SQLite) e produção (PostgreSQL), selecionada por variável de ambiente.
require("dotenv").config();

const { Sequelize } = require("sequelize");
const path = require("node:path");
const fs = require("node:fs");

let sequelize;

if (process.env.DATABASE_URL) {
  // ---------- PRODUÇÃO: PostgreSQL gerenciado ----------
  sequelize = new Sequelize(process.env.DATABASE_URL, {
    dialect: "postgres",
    logging: false,

    dialectOptions: {
      ssl: {
        require: true,
        // Provedores gerenciados utilizam certificados próprios.
        // Em contexto didático, aceita-se a cadeia sem verificação;
        // em produção real, deve-se fornecer o certificado do provedor.
        rejectUnauthorized: false,
      },
    },

    // Em ambiente serverless, cada instância mantém seu próprio pool.
    // Muitas instâncias simultâneas podem esgotar o limite de conexões
    // do provedor, razão pela qual o pool é mantido reduzido.
    pool: {
      max: 2,
      min: 0,
      idle: 10_000,
      acquire: 30_000,
    },
  });
} else {
  // ---------- DESENVOLVIMENTO: SQLite local ----------
  const caminho = path.resolve(process.env.DB_STORAGE || "./dados/tarefas.sqlite");
  fs.mkdirSync(path.dirname(caminho), { recursive: true });

  sequelize = new Sequelize({
    dialect: "sqlite",
    storage: caminho,
    logging: process.env.NODE_ENV === "development" ? console.log : false,
  });
}

module.exports = { sequelize };
```

### 5.5 Reaproveitamento da conexão entre invocações

Uma instância *serverless* pode atender a várias requisições antes de ser descartada. Abrir uma nova conexão a cada requisição desperdiça tempo e esgota o limite do provedor. A solução consiste em armazenar a conexão em uma variável do módulo, que sobrevive enquanto a instância existir.

**Arquivo `app.js` — acréscimo:**

```javascript
const { sequelize } = require("./config/banco");

// Variável de módulo: preservada enquanto a instância permanecer ativa.
let conexaoPronta = null;

async function garantirConexao() {
  if (!conexaoPronta) {
    conexaoPronta = sequelize.authenticate().catch((erro) => {
      // Em caso de falha, limpa a referência para permitir nova tentativa
      // na próxima requisição, em vez de manter uma promise rejeitada.
      conexaoPronta = null;
      throw erro;
    });
  }
  return conexaoPronta;
}

// Middleware registrado ANTES das rotas.
app.use(async (req, res, next) => {
  try {
    await garantirConexao();
    next();
  } catch (erro) {
    console.error("Falha ao conectar ao banco:", erro.message);
    res.status(503).json({
      sucesso: false,
      erro: { codigo: 503, mensagem: "Banco de dados temporariamente indisponível" },
    });
  }
});
```

### 5.6 Criação das tabelas em produção

`sequelize.sync()` **não** deve ser executado dentro da função *serverless*: seria invocado a cada requisição, com custo desnecessário e risco de condição de corrida.

A criação das tabelas é realizada uma única vez, a partir da máquina local, apontando para o banco de produção:

```powershell
# PowerShell — variável válida apenas nesta sessão do terminal
$env:DATABASE_URL="postgresql://usuario:senha@host/neondb?sslmode=require"
node scripts/inicializar.js
```

Em projetos com migrações configuradas (Tutorial 6, Seção 9):

```powershell
$env:DATABASE_URL="postgresql://..."
npx sequelize-cli db:migrate --env production
```

### 5.7 Verificação

```powershell
$base = "https://api-tarefas-SEU-IDENTIFICADOR.vercel.app/api"

curl.exe "$base/saude"
curl.exe "$base/tarefas"

curl.exe -X POST "$base/tarefas" `
  -H "Content-Type: application/json" `
  -d '{\"titulo\":\"Primeira tarefa em producao\",\"prioridade\":\"alta\"}'

# Confirmação da persistência: a tarefa deve constar da listagem
curl.exe "$base/tarefas"
```

A presença da tarefa na segunda listagem confirma que o problema de persistência foi resolvido.

---

## 6. Implantação da aplicação web

A aplicação com EJS (Tutorial 5) requer atenção adicional: as *views* devem ser incluídas no pacote enviado à função.

**Arquivo `vercel.json`:**

```json
{
  "version": 2,
  "functions": {
    "api/index.js": {
      "includeFiles": "views/**"
    }
  },
  "rewrites": [
    { "source": "/(.*)", "destination": "/api" }
  ]
}
```

O campo `includeFiles` instrui a plataforma a incluir arquivos que não são alcançados pela análise estática das importações — caso dos modelos EJS, carregados dinamicamente em tempo de execução.

**Arquivo `app.js` — configuração das views:**

```javascript
const path = require("node:path");

// process.cwd() aponta para a raiz do projeto na função serverless.
// __dirname pode divergir conforme o empacotamento, razão pela qual
// se emprega cwd para localizar as views.
app.set("views", path.join(process.cwd(), "views"));
app.set("view engine", "ejs");
```

**Arquivos estáticos.** Recursos de `publico/` são melhor servidos pela rede de distribuição da plataforma, e não pela função. A configuração abaixo separa os dois casos:

```json
{
  "version": 2,
  "functions": {
    "api/index.js": { "includeFiles": "views/**" }
  },
  "rewrites": [
    { "source": "/css/(.*)", "destination": "/publico/css/$1" },
    { "source": "/js/(.*)", "destination": "/publico/js/$1" },
    { "source": "/imagens/(.*)", "destination": "/publico/imagens/$1" },
    { "source": "/(.*)", "destination": "/api" }
  ]
}
```

**Ordem das regras:** as regras são avaliadas de cima para baixo. A regra genérica `/(.*)`, que captura todos os caminhos, deve constar por último; posicionada antes, impediria a aplicação das demais.

---

## 7. Configuração do CORS em produção

Com a API e a aplicação em domínios distintos, o CORS torna-se obrigatório.

Cadastrar, no painel da Vercel, a variável:

```
ORIGENS_PERMITIDAS=https://app-tarefas.vercel.app,https://www.meudominio.com.br
```

O *middleware* implementado no Tutorial 7 já consulta essa variável. Convém, porém, autorizar também os domínios de pré-visualização, gerados a cada ramo:

```javascript
app.use(
  cors({
    origin(origem, callback) {
      if (!origem) return callback(null, true);

      const permitidas = (process.env.ORIGENS_PERMITIDAS || "")
        .split(",")
        .map((o) => o.trim())
        .filter(Boolean);

      if (permitidas.includes(origem)) return callback(null, true);

      // Domínios de pré-visualização da própria plataforma.
      if (/^https:\/\/[a-z0-9-]+\.vercel\.app$/.test(origem)) {
        return callback(null, true);
      }

      callback(new Error(`Origem não autorizada: ${origem}`));
    },
    methods: ["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
  })
);
```

**Advertência:** liberar `*.vercel.app` autoriza qualquer projeto hospedado na plataforma. Em aplicação real com dados sensíveis, a lista deve ser explícita.

---

## 8. Diagnóstico de falhas

### 8.1 Registros de execução

No painel: **Deployments → selecionar a implantação → Runtime Logs**. Pela linha de comando:

```powershell
vercel logs https://api-tarefas-xxxx.vercel.app --follow
```

Todas as chamadas a `console.log` e `console.error` da aplicação aparecem nesses registros.

### 8.2 Falhas frequentes

| Sintoma | Causa provável | Correção |
|---|---|---|
| `404: NOT_FOUND` em todas as rotas | `vercel.json` ausente ou `rewrites` incorreto | Conferir a regra `/(.*)` → `/api` |
| Código-fonte exibido no navegador | Aplicação não exportada como função | `module.exports = app` em `api/index.js` |
| `FUNCTION_INVOCATION_TIMEOUT` | Processamento excedeu o limite | Otimizar consultas; verificar conectividade com o banco |
| `Cannot find module 'express'` | Dependência em `devDependencies` | Mover para `dependencies` |
| `Failed to lookup view "index"` | Views não incluídas no pacote | `includeFiles: "views/**"` e `process.cwd()` |
| `EROFS: read-only file system` | Tentativa de gravação em disco | Substituir por banco gerenciado |
| Dados desaparecem entre requisições | SQLite em ambiente efêmero | Substituir por banco gerenciado |
| `SequelizeConnectionError` | Credenciais ou SSL incorretos | Conferir `DATABASE_URL` e `dialectOptions.ssl` |
| `too many connections` | Pool excessivo por instância | Reduzir `pool.max` para 2 |
| CORS bloqueado no navegador | Origem não autorizada | Cadastrar em `ORIGENS_PERMITIDAS` e reimplantar |
| Variável de ambiente ignorada | Alterada sem reimplantar | Executar nova implantação |

### 8.3 Reprodução local do ambiente da plataforma

```powershell
vercel dev
```

O comando executa a aplicação localmente sob o mesmo modelo de funções da plataforma, reproduzindo o roteamento definido em `vercel.json`. Falhas relacionadas à configuração manifestam-se localmente, reduzindo o ciclo de correção.

### 8.4 Retorno a uma implantação anterior

Cada implantação permanece acessível por URL própria. Em caso de falha em produção:

```powershell
vercel rollback
```

Alternativamente, no painel: **Deployments → selecionar a implantação estável → Promote to Production**.

---

## 9. Integração contínua

Uma vez conectado o repositório, a Vercel adota o seguinte comportamento:

- envio ao ramo `main` → implantação em **produção**;
- envio a outro ramo → implantação de **pré-visualização**, com URL própria;
- abertura de *pull request* → comentário automático com o endereço da pré-visualização.

Fluxo recomendado para trabalho em equipe:

```powershell
git checkout -b funcionalidade/filtro-por-projeto
# implementação
git add .
git commit -m "Acrescenta filtro por projeto na listagem"
git push origin funcionalidade/filtro-por-projeto
# a Vercel gera uma URL de pré-visualização para teste
# após aprovação, o pull request é integrado a main
```

Esse mecanismo permite que cada integrante teste sua funcionalidade em ambiente idêntico ao de produção, sem afetar a versão publicada.

---

## 10. Laboratório prático

### Laboratório 10.1 — Publicação da API

1. Adaptar a API do Tutorial 7 conforme as Seções 3 e 5.
2. Criar um banco PostgreSQL gerenciado gratuito.
3. Executar o script de inicialização apontando para o banco de produção.
4. Publicar na Vercel e cadastrar as variáveis de ambiente.
5. Documentar, em relatório, os testes realizados sobre a URL pública:

| Verificação | Resultado esperado |
|---|---|
| `GET /api/saude` | `200` |
| `GET /api/tarefas` | `200` com a lista |
| `POST /api/tarefas` | `201` |
| Nova listagem após o `POST` | Tarefa **persistida** |
| `GET /api/tarefas/99999` | `404` |
| `POST` com título de dois caracteres | `400` |
| `DELETE /api/tarefas/:id` | `204` |

**Entrega:** URL pública em funcionamento, endereço do repositório e relatório com capturas de tela de cada verificação.

### Laboratório 10.2 — Publicação da aplicação web

1. Publicar a aplicação com EJS do Tutorial 5, em repositório e projeto distintos da API.
2. Configurar `includeFiles` e a resolução de views.
3. Substituir os dados em memória por consultas à API publicada no Laboratório 10.1.
4. Ajustar o CORS para autorizar o domínio da aplicação.
5. Confirmar o funcionamento de todas as páginas e do formulário.

**Questão:** por que a chamada à API a partir do servidor da aplicação (com `fetch` no código Node.js) não é bloqueada pelo CORS, enquanto a mesma chamada feita pelo JavaScript do navegador o é? Explicar em um parágrafo.

### Laboratório 10.3 — Investigação do cold start

1. Aguardar quinze minutos sem acessar a API implantada.
2. Medir o tempo da primeira requisição:

```powershell
Measure-Command { curl.exe -s https://sua-api.vercel.app/api/saude }
```

3. Medir imediatamente mais dez requisições consecutivas.
4. Elaborar tabela e gráfico dos tempos obtidos.
5. Redigir análise respondendo:
   - qual a diferença entre a primeira medição e a média das demais?
   - a que se deve essa diferença?
   - em que tipo de aplicação o *cold start* é inaceitável?
   - quais estratégias existem para mitigá-lo?

### Laboratório 10.4 — Comparação de plataformas

Publicar a **mesma** API em uma plataforma com servidor persistente — Render, Railway ou Fly.io — e elaborar quadro comparativo com, no mínimo, os seguintes critérios: facilidade de configuração; necessidade de adaptação do código; suporte a SQLite; tempo da primeira requisição; tempo das requisições subsequentes; limitações do nível gratuito; adequação a uma aplicação com tráfego intermitente e a uma com tráfego constante.

Concluir com recomendação fundamentada para dois cenários: um projeto escolar de demonstração e um sistema institucional com uso diário.

---

## 11. Síntese

1. Funções *serverless* executam sob demanda, não preservam estado entre invocações e não dispõem de sistema de arquivos persistente.
2. Na Vercel, a aplicação Express é **exportada**, e não iniciada com `listen()`.
3. O arquivo `vercel.json` define o encaminhamento das requisições à função; a regra genérica deve ser a última.
4. O SQLite local não é utilizável em ambiente *serverless* com escrita; a alternativa direta é um banco gerenciado compatível com o Sequelize.
5. A conexão com o banco deve ser reaproveitada entre invocações, com pool reduzido.
6. Variáveis de ambiente são cadastradas na plataforma e exigem reimplantação após alteração.
7. Modelos EJS precisam ser explicitamente incluídos no pacote da função.
8. O comando `vercel dev` reproduz localmente o ambiente da plataforma e antecipa falhas de configuração.
9. Cada envio ao repositório gera uma implantação; ramos secundários produzem pré-visualizações isoladas.

**Próximo tutorial:** [Criação e publicação de um pacote npm](./09-pacote-npm.md)
