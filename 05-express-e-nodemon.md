# Tutorial 5 — Aplicação Web com Express e nodemon

> **Pré-requisitos:** Tutorial 4 concluído.
> **Duração estimada:** 4 aulas de 50 minutos.
> **Ao final deste tutorial, o estudante será capaz de:** instalar e configurar o Express, definir rotas e parâmetros, compreender o encadeamento de *middlewares*, servir arquivos estáticos, gerar páginas com o mecanismo de modelos EJS, processar formulários, tratar erros de forma centralizada, organizar o projeto em camadas e utilizar o nodemon com variáveis de ambiente.

---

## 1. Por que utilizar um framework

No Tutorial 4, o servidor de arquivos estáticos exigiu aproximadamente cem linhas de código: identificação de tipos MIME, resolução segura de caminhos, tratamento de erros e roteamento por encadeamento de condicionais.

O Express reduz o mesmo servidor a:

```javascript
const express = require("express");
const app = express();

app.use(express.static("publico"));
app.listen(3000);
```

Quatro linhas. O Express não substitui o módulo `http` — ele o utiliza internamente, acrescentando três abstrações:

1. **Roteamento declarativo:** `app.get("/cursos/:id", ...)` em lugar de condicionais aninhadas;
2. **Middlewares:** funções encadeadas que processam a requisição em etapas;
3. **Utilidades de resposta:** `res.json()`, `res.render()`, `res.redirect()`, `res.status()`.

---

## 2. Instalação e primeiro servidor

```powershell
mkdir app-express
cd app-express
npm init -y
npm install express
npm install --save-dev nodemon
code .
```

**Arquivo `app.js`:**

```javascript
// app.js — primeiro servidor com Express
const express = require("express");

// A chamada a express() cria a aplicação: um objeto que concentra
// rotas, middlewares e configurações.
const app = express();
const PORTA = process.env.PORT || 3000;

// Definição de rota: método HTTP + caminho + função manipuladora.
// req (request)  -> dados da requisição
// res (response) -> métodos para construir a resposta
app.get("/", (req, res) => {
  // res.send infere o Content-Type a partir do argumento e encerra a
  // resposta automaticamente — não é necessário chamar res.end().
  res.send("<h1>Servidor Express em funcionamento</h1>");
});

app.get("/sobre", (req, res) => {
  res.send("<h1>Sobre</h1><p>Aplicação construída com Express.</p>");
});

app.get("/api/hora", (req, res) => {
  // res.json define Content-Type como application/json e serializa
  // o objeto automaticamente.
  res.json({
    horario: new Date().toISOString(),
    formatado: new Date().toLocaleString("pt-BR"),
  });
});

app.listen(PORTA, () => {
  console.log(`Servidor disponível em http://localhost:${PORTA}`);
});
```

```powershell
node app.js
```

**Comparação direta com o Tutorial 4:**

| Tarefa | Módulo `http` | Express |
|---|---|---|
| Roteamento | `if (url.pathname === "/")` | `app.get("/", ...)` |
| Tipo MIME | Definido manualmente | Inferido por `res.send` |
| Resposta JSON | `JSON.stringify` + cabeçalho | `res.json(objeto)` |
| Arquivos estáticos | ~60 linhas | `express.static("publico")` |
| 404 | Bloco `else` final | *Middleware* posicionado ao fim |

---

## 3. nodemon: reinício automático

Cada alteração no código exige reiniciar o servidor manualmente. O **nodemon** monitora os arquivos do projeto e reinicia o processo automaticamente — utilizando, internamente, o mesmo `fs.watch` estudado no Tutorial 3.

Configuração dos scripts em `package.json`:

```json
{
  "name": "app-express",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "dev": "nodemon app.js"
  },
  "dependencies": {
    "express": "^4.19.2"
  },
  "devDependencies": {
    "nodemon": "^3.1.0"
  }
}
```

```powershell
npm run dev
```

```
[nodemon] 3.1.0
[nodemon] watching path(s): *.*
[nodemon] watching extensions: js,mjs,cjs,json
[nodemon] starting `node app.js`
Servidor disponível em http://localhost:3000
```

Ao salvar qualquer alteração:

```
[nodemon] restarting due to changes...
[nodemon] starting `node app.js`
```

Comandos do nodemon durante a execução: `rs` seguido de `Enter` força o reinício; `Ctrl+C` encerra.

### 3.1 Configuração avançada

**Arquivo `nodemon.json`:**

```json
{
  "watch": ["app.js", "rotas/", "controladores/", "views/"],
  "ext": "js,json,ejs,html,css",
  "ignore": ["node_modules/", "publico/uploads/", "*.test.js", "dados/*.sqlite"],
  "delay": 500,
  "env": {
    "NODE_ENV": "development"
  }
}
```

| Chave | Função |
|---|---|
| `watch` | Pastas e arquivos monitorados |
| `ext` | Extensões que disparam o reinício |
| `ignore` | Caminhos ignorados |
| `delay` | Intervalo em milissegundos antes de reiniciar, evitando reinícios múltiplos |
| `env` | Variáveis de ambiente injetadas no processo |

**Advertência importante:** o arquivo do banco de dados SQLite deve constar em `ignore`. Sem essa exclusão, cada gravação no banco provoca um reinício do servidor, produzindo um ciclo infinito.

**Distinção de uso:** `npm run dev` (nodemon) destina-se ao desenvolvimento; `npm start` (node) destina-se à produção. O nodemon é uma dependência de desenvolvimento e não deve ser utilizado em servidores de produção.

---

## 4. Middlewares

### 4.1 Conceito

Um *middleware* é uma função com a assinatura `(req, res, next)`, executada durante o processamento de uma requisição. Os *middlewares* formam uma cadeia: cada um pode inspecionar a requisição, alterá-la, encerrá-la ou repassá-la ao próximo por meio de `next()`.

```
Requisição
    │
    ▼
┌───────────────┐
│  Registro     │  console.log(método, caminho)
└───────┬───────┘
        │ next()
        ▼
┌───────────────┐
│  Estáticos    │  arquivo existe? → responde e encerra a cadeia
└───────┬───────┘
        │ next()
        ▼
┌───────────────┐
│  Corpo (JSON) │  converte o corpo em req.body
└───────┬───────┘
        │ next()
        ▼
┌───────────────┐
│  Autenticação │  token válido? → next() ; senão → 401
└───────┬───────┘
        │ next()
        ▼
┌───────────────┐
│  Rota         │  res.send(...)
└───────────────┘
```

**Regra decisiva: a ordem de registro determina a ordem de execução.** Um *middleware* de autenticação registrado após as rotas jamais será executado para elas.

### 4.2 Implementação

**Arquivo `middlewares.js`:**

```javascript
// middlewares.js — demonstração da cadeia de middlewares
const express = require("express");
const app = express();

// ---------- MIDDLEWARE GLOBAL 1: registro de requisições ----------
// Aplicado a todas as rotas por não receber um caminho específico.
app.use((req, res, next) => {
  const inicio = Date.now();

  // O evento "finish" ocorre quando a resposta termina de ser enviada.
  res.on("finish", () => {
    const duracao = Date.now() - inicio;
    console.log(`${req.method} ${req.originalUrl} → ${res.statusCode} (${duracao}ms)`);
  });

  next(); // sem esta chamada, a requisição fica suspensa indefinidamente
});

// ---------- MIDDLEWARE GLOBAL 2: interpretação do corpo ----------
// Converte corpos JSON em req.body. O limite protege contra
// requisições excessivamente grandes.
app.use(express.json({ limit: "1mb" }));

// Converte corpos de formulários HTML (application/x-www-form-urlencoded).
app.use(express.urlencoded({ extended: true }));

// ---------- MIDDLEWARE GLOBAL 3: dado compartilhado ----------
app.use((req, res, next) => {
  // Propriedades acrescentadas a req ficam disponíveis nos middlewares
  // e rotas seguintes.
  req.identificador = Math.random().toString(36).slice(2, 10);
  res.setHeader("X-Requisicao-Id", req.identificador);
  next();
});

// ---------- MIDDLEWARE DE ROTA: verificação de credencial ----------
// Aplicado apenas às rotas em que for explicitamente incluído.
function exigirCredencial(req, res, next) {
  const credencial = req.headers["x-credencial"];

  if (credencial !== "curso-nodejs-2026") {
    // Encerrar a cadeia: não chamar next().
    return res.status(401).json({
      erro: "Credencial ausente ou inválida",
      dica: "Informe o cabeçalho X-Credencial",
    });
  }

  req.usuario = { nome: "Aluno", perfil: "estudante" };
  next();
}

// ---------- ROTAS ----------

app.get("/", (req, res) => {
  res.json({
    mensagem: "Rota pública",
    identificador: req.identificador,
  });
});

// O middleware é inserido entre o caminho e a função final.
app.get("/protegida", exigirCredencial, (req, res) => {
  res.json({
    mensagem: "Acesso autorizado",
    usuario: req.usuario,
  });
});

// Múltiplos middlewares podem ser encadeados.
function registrarAcesso(req, res, next) {
  console.log(`  [auditoria] ${req.usuario.nome} acessou ${req.path}`);
  next();
}

app.get("/relatorio", exigirCredencial, registrarAcesso, (req, res) => {
  res.json({ relatorio: "Dados restritos", geradoEm: new Date() });
});

// ---------- 404: registrado APÓS todas as rotas ----------
// Se a execução alcançar este ponto, nenhuma rota correspondeu.
app.use((req, res) => {
  res.status(404).json({ erro: "Rota não encontrada", caminho: req.originalUrl });
});

// ---------- TRATAMENTO DE ERROS: quatro parâmetros ----------
// O Express identifica middlewares de erro pela quantidade de parâmetros.
// A omissão de "next" impede o reconhecimento e a função não é invocada.
app.use((erro, req, res, next) => {
  console.error("Erro capturado:", erro.message);
  res.status(erro.status || 500).json({
    erro: "Erro interno do servidor",
    identificador: req.identificador,
  });
});

app.listen(3000, () => console.log("http://localhost:3000"));
```

Testes com PowerShell:

```powershell
curl.exe http://localhost:3000/
curl.exe http://localhost:3000/protegida
curl.exe -H "X-Credencial: curso-nodejs-2026" http://localhost:3000/protegida
curl.exe http://localhost:3000/inexistente
```

**Observação:** no PowerShell, `curl` é um apelido para `Invoke-WebRequest`, cuja sintaxe difere. Deve-se utilizar `curl.exe` explicitamente para invocar a ferramenta padrão.

### 4.3 Middlewares de terceiros

```powershell
npm install morgan cors helmet compression
```

```javascript
const morgan = require("morgan");        // registro de requisições
const cors = require("cors");            // compartilhamento entre origens
const helmet = require("helmet");        // cabeçalhos de segurança
const compression = require("compression"); // compressão gzip

app.use(helmet());                        // primeiro: segurança
app.use(cors());                          // controle de origem
app.use(compression());                   // compressão das respostas
app.use(morgan("dev"));                   // registro colorido no terminal
app.use(express.json());                  // interpretação do corpo
app.use(express.static("publico"));       // arquivos estáticos
// ... rotas
```

---

## 5. Roteamento

### 5.1 Parâmetros de rota

```javascript
// parametros.js
const express = require("express");
const app = express();

// Segmento iniciado por ":" é um parâmetro, disponível em req.params.
app.get("/cursos/:id", (req, res) => {
  // Todo parâmetro chega como texto e deve ser convertido quando necessário.
  const id = Number(req.params.id);

  if (!Number.isInteger(id) || id < 1) {
    return res.status(400).json({ erro: "O identificador deve ser inteiro positivo" });
  }

  res.json({ id, nome: `Curso ${id}` });
});

// Múltiplos parâmetros.
app.get("/cursos/:cursoId/turmas/:turmaId", (req, res) => {
  const { cursoId, turmaId } = req.params;
  res.json({ cursoId, turmaId });
});

// Parâmetro opcional, indicado por "?".
app.get("/relatorio/:ano?", (req, res) => {
  const ano = req.params.ano || new Date().getFullYear();
  res.json({ ano: Number(ano) });
});

// Query string: /busca?termo=node&pagina=2&porPagina=10
app.get("/busca", (req, res) => {
  const { termo = "", pagina = 1, porPagina = 10 } = req.query;

  res.json({
    termo,
    pagina: Number(pagina),
    porPagina: Math.min(Number(porPagina), 100), // limite superior
  });
});

app.listen(3000);
```

**Ordem das rotas:** o Express avalia as rotas na ordem de registro e utiliza a primeira que corresponder. Rotas específicas devem preceder rotas com parâmetros:

```javascript
app.get("/cursos/destaques", ...); // ANTES
app.get("/cursos/:id", ...);       // DEPOIS

// Na ordem inversa, "/cursos/destaques" seria capturada por "/cursos/:id",
// com req.params.id valendo "destaques".
```

### 5.2 Modularização com Router

Concentrar todas as rotas em um único arquivo torna o projeto ingerenciável. O `express.Router` permite agrupá-las por assunto.

**Arquivo `rotas/cursos.js`:**

```javascript
// rotas/cursos.js — rotas relativas a cursos
const express = require("express");
const router = express.Router();

// Dados em memória. No Tutorial 6, serão substituídos por um banco.
const cursos = [
  { id: 1, nome: "Informática para Internet", turno: "Integral", vagas: 40 },
  { id: 2, nome: "Administração", turno: "Integral", vagas: 35 },
  { id: 3, nome: "Logística", turno: "Noturno", vagas: 30 },
];

// Middleware aplicado a todas as rotas deste router.
router.use((req, res, next) => {
  console.log(`  [cursos] ${req.method} ${req.originalUrl}`);
  next();
});

// GET /cursos
router.get("/", (req, res) => {
  const { turno } = req.query;
  const resultado = turno ? cursos.filter((c) => c.turno === turno) : cursos;
  res.json({ total: resultado.length, dados: resultado });
});

// GET /cursos/:id
router.get("/:id", (req, res) => {
  const curso = cursos.find((c) => c.id === Number(req.params.id));
  if (!curso) {
    return res.status(404).json({ erro: "Curso não encontrado" });
  }
  res.json(curso);
});

// POST /cursos
router.post("/", (req, res) => {
  const { nome, turno, vagas } = req.body;

  if (!nome || !turno) {
    return res.status(400).json({ erro: "Os campos nome e turno são obrigatórios" });
  }

  const novo = {
    id: Math.max(0, ...cursos.map((c) => c.id)) + 1,
    nome,
    turno,
    vagas: Number(vagas) || 0,
  };

  cursos.push(novo);

  // 201 Created, com o cabeçalho Location apontando para o novo recurso.
  res.status(201).location(`/cursos/${novo.id}`).json(novo);
});

module.exports = router;
```

**Arquivo `app.js` (atualizado):**

```javascript
const express = require("express");
const rotasCursos = require("./rotas/cursos");

const app = express();

app.use(express.json());

// Todas as rotas do router são prefixadas por "/cursos".
app.use("/cursos", rotasCursos);

app.listen(3000, () => console.log("http://localhost:3000"));
```

---

## 6. Mecanismo de modelos: EJS

Concatenar HTML dentro de strings JavaScript, como no Tutorial 4, é ilegível e propenso a erros. Um **mecanismo de modelos** (*template engine*) separa a estrutura HTML da lógica de programação.

```powershell
npm install ejs
```

**Arquivo `app.js`:**

```javascript
const express = require("express");
const path = require("node:path");

const app = express();

// Configuração do mecanismo de modelos.
app.set("view engine", "ejs");
app.set("views", path.join(__dirname, "views"));

app.use(express.static(path.join(__dirname, "publico")));
app.use(express.urlencoded({ extended: true }));

const cursos = [
  { id: 1, nome: "Informática para Internet", turno: "Integral", vagas: 40, ativo: true },
  { id: 2, nome: "Administração", turno: "Integral", vagas: 35, ativo: true },
  { id: 3, nome: "Eventos", turno: "Noturno", vagas: 30, ativo: false },
];

app.get("/", (req, res) => {
  // res.render carrega views/index.ejs e substitui as variáveis
  // pelos valores do objeto fornecido como segundo argumento.
  res.render("index", {
    titulo: "Página inicial",
    totalCursos: cursos.length,
    totalVagas: cursos.reduce((s, c) => s + c.vagas, 0),
  });
});

app.get("/cursos", (req, res) => {
  const { turno } = req.query;
  const lista = turno ? cursos.filter((c) => c.turno === turno) : cursos;

  res.render("cursos", {
    titulo: "Cursos ofertados",
    cursos: lista,
    filtroAtivo: turno || "todos",
  });
});

app.get("/cursos/:id", (req, res, next) => {
  const curso = cursos.find((c) => c.id === Number(req.params.id));
  if (!curso) {
    const erro = new Error("Curso não encontrado");
    erro.status = 404;
    return next(erro); // encaminha ao middleware de erro
  }
  res.render("curso-detalhe", { titulo: curso.nome, curso });
});

app.listen(3000, () => console.log("http://localhost:3000"));
```

### 6.1 Estrutura das views

```
views/
├── parciais/
│   ├── cabecalho.ejs
│   └── rodape.ejs
├── index.ejs
├── cursos.ejs
├── curso-detalhe.ejs
└── erro.ejs
```

**Arquivo `views/parciais/cabecalho.ejs`:**

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <!-- <%= %> insere o valor com escape automático de HTML -->
  <title><%= titulo %> | Curso de Node.js</title>
  <link rel="stylesheet" href="/css/estilo.css">
</head>
<body>
  <header class="cabecalho">
    <div class="container cabecalho__conteudo">
      <a href="/" class="marca">&lt;/&gt; Curso de Node.js</a>
      <nav class="navegacao">
        <a href="/">Início</a>
        <a href="/cursos">Cursos</a>
        <a href="/contato">Contato</a>
      </nav>
    </div>
  </header>
  <main class="container">
```

**Arquivo `views/parciais/rodape.ejs`:**

```html
  </main>
  <footer class="rodape">
    <div class="container">
      <p>Instituto Federal de Brasília &middot;
         Página gerada em <%= new Date().toLocaleString("pt-BR") %></p>
    </div>
  </footer>
</body>
</html>
```

**Arquivo `views/cursos.ejs`:**

```html
<%- include("parciais/cabecalho") %>

<h1>Cursos ofertados</h1>

<div class="filtros">
  <a href="/cursos" class="<%= filtroAtivo === 'todos' ? 'ativo' : '' %>">Todos</a>
  <a href="/cursos?turno=Integral"
     class="<%= filtroAtivo === 'Integral' ? 'ativo' : '' %>">Integral</a>
  <a href="/cursos?turno=Noturno"
     class="<%= filtroAtivo === 'Noturno' ? 'ativo' : '' %>">Noturno</a>
</div>

<%# Comentário EJS: não aparece no HTML enviado ao navegador %>

<% if (cursos.length === 0) { %>
  <p class="aviso">Nenhum curso corresponde ao filtro selecionado.</p>
<% } else { %>
  <div class="cartoes">
    <% cursos.forEach(function (curso) { %>
      <article class="cartao">
        <h2><%= curso.nome %></h2>
        <p>Turno: <strong><%= curso.turno %></strong></p>
        <p>Vagas: <strong><%= curso.vagas %></strong></p>
        <span class="etiqueta <%= curso.ativo ? 'etiqueta--ativa' : 'etiqueta--inativa' %>">
          <%= curso.ativo ? "Com inscrições" : "Encerrado" %>
        </span>
        <p><a href="/cursos/<%= curso.id %>" class="botao">Detalhes</a></p>
      </article>
    <% }); %>
  </div>
<% } %>

<%- include("parciais/rodape") %>
```

### 6.2 Sintaxe do EJS

| Sintaxe | Função |
|---|---|
| `<%= valor %>` | Insere o valor **com escape de HTML** — uso padrão |
| `<%- valor %>` | Insere o valor **sem escape** — apenas para HTML confiável |
| `<% código %>` | Executa JavaScript sem produzir saída (`if`, `for`) |
| `<%# texto %>` | Comentário, removido da saída |
| `<%- include("arquivo") %>` | Insere outro modelo |

**Alerta de segurança:** `<%- %>` não escapa o conteúdo. Utilizá-lo com dados fornecidos pelo usuário reintroduz a vulnerabilidade XSS discutida no Tutorial 4. Deve ser reservado a `include` e a HTML gerado pela própria aplicação.

---

## 7. Processamento de formulários

**Arquivo `views/contato.ejs`:**

```html
<%- include("parciais/cabecalho") %>

<h1>Formulário de contato</h1>

<%# Exibição de mensagens de erro devolvidas pelo servidor %>
<% if (typeof erros !== "undefined" && erros.length > 0) { %>
  <div class="aviso" style="border-left-color:#c1121f;background:#fff1f1">
    <strong>Corrija os itens abaixo:</strong>
    <ul>
      <% erros.forEach(function (e) { %><li><%= e %></li><% }); %>
    </ul>
  </div>
<% } %>

<form class="formulario" method="POST" action="/contato">
  <div class="campo">
    <label for="nome">Nome completo</label>
    <%# O atributo value repopula o campo, preservando o que já foi digitado %>
    <input type="text" id="nome" name="nome" required
           value="<%= typeof dados !== 'undefined' ? dados.nome || '' : '' %>">
  </div>

  <div class="campo">
    <label for="email">Correio eletrônico</label>
    <input type="email" id="email" name="email" required
           value="<%= typeof dados !== 'undefined' ? dados.email || '' : '' %>">
  </div>

  <div class="campo">
    <label for="assunto">Assunto</label>
    <select id="assunto" name="assunto">
      <option value="duvida">Dúvida sobre o conteúdo</option>
      <option value="erro">Relato de erro</option>
      <option value="sugestao">Sugestão</option>
    </select>
  </div>

  <div class="campo">
    <label for="mensagem">Mensagem</label>
    <textarea id="mensagem" name="mensagem" required><%=
      typeof dados !== "undefined" ? dados.mensagem || "" : "" %></textarea>
  </div>

  <button type="submit" class="botao">Enviar</button>
</form>

<%- include("parciais/rodape") %>
```

Rotas correspondentes:

```javascript
// Armazenamento temporário em memória. Perde-se ao reiniciar o servidor.
const mensagens = [];

app.get("/contato", (req, res) => {
  res.render("contato", { titulo: "Contato", erros: [], dados: {} });
});

app.post("/contato", (req, res) => {
  // express.urlencoded preencheu req.body com os campos do formulário.
  const { nome = "", email = "", assunto = "", mensagem = "" } = req.body;

  const erros = [];
  if (nome.trim().length < 3) {
    erros.push("O nome deve conter ao menos três caracteres.");
  }
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
    erros.push("O endereço de correio eletrônico é inválido.");
  }
  if (mensagem.trim().length < 10) {
    erros.push("A mensagem deve conter ao menos dez caracteres.");
  }

  if (erros.length > 0) {
    // Reexibe o formulário com os erros e os dados já digitados.
    return res.status(400).render("contato", {
      titulo: "Contato",
      erros,
      dados: req.body,
    });
  }

  mensagens.push({
    ...req.body,
    recebidaEm: new Date(),
    id: mensagens.length + 1,
  });

  // PADRÃO POST-REDIRECT-GET: após processar um POST, redireciona-se
  // com 302. Isso impede que a atualização da página reenvie o formulário.
  res.redirect("/contato/sucesso");
});

app.get("/contato/sucesso", (req, res) => {
  res.render("sucesso", {
    titulo: "Mensagem enviada",
    total: mensagens.length,
  });
});
```

---

## 8. Tratamento centralizado de erros

```javascript
// ---------- Registrado APÓS todas as rotas ----------

// 404: nenhuma rota correspondeu.
app.use((req, res) => {
  res.status(404).render("erro", {
    titulo: "Página não encontrada",
    codigo: 404,
    mensagem: "O endereço solicitado não existe nesta aplicação.",
  });
});

// Middleware de erro: identificado pelos quatro parâmetros.
app.use((erro, req, res, next) => {
  const codigo = erro.status || 500;

  // O detalhe técnico é registrado no servidor, não exposto ao usuário.
  console.error(`[${new Date().toISOString()}] ${codigo} ${req.originalUrl}`);
  console.error(erro.stack);

  res.status(codigo).render("erro", {
    titulo: "Erro",
    codigo,
    mensagem:
      codigo === 404
        ? erro.message
        : "Ocorreu uma falha no processamento da solicitação.",
    // A pilha de execução é revelada apenas em desenvolvimento.
    detalhe: process.env.NODE_ENV === "development" ? erro.stack : null,
  });
});
```

### 8.1 Erros em rotas assíncronas

No Express 4, exceções lançadas dentro de funções assíncronas **não** são capturadas automaticamente:

```javascript
// INCORRETO no Express 4: a falha não alcança o middleware de erro
// e a requisição permanece pendente até o tempo limite.
app.get("/dados", async (req, res) => {
  const dados = await buscarDados(); // se rejeitar, o erro se perde
  res.json(dados);
});

// CORRETO: captura explícita
app.get("/dados", async (req, res, next) => {
  try {
    res.json(await buscarDados());
  } catch (erro) {
    next(erro);
  }
});

// CORRETO E CONCISO: função auxiliar reutilizável
function assincrono(manipulador) {
  return (req, res, next) => {
    Promise.resolve(manipulador(req, res, next)).catch(next);
  };
}

app.get("/dados", assincrono(async (req, res) => {
  res.json(await buscarDados()); // erros são encaminhados automaticamente
}));
```

**Observação:** o Express 5 encaminha automaticamente as rejeições de *promises* ao *middleware* de erro, tornando a função auxiliar desnecessária. Como a versão 4 permanece amplamente utilizada, o padrão acima continua relevante.

---

## 9. Variáveis de ambiente

Credenciais e configurações não devem ser gravadas no código-fonte. O pacote `dotenv` carrega variáveis de um arquivo `.env`.

```powershell
npm install dotenv
```

**Arquivo `.env`** (jamais enviado ao repositório):

```env
PORT=3000
NODE_ENV=development
NOME_APLICACAO=Portal de Cursos
SEGREDO_SESSAO=altere-este-valor-em-producao
```

**Arquivo `.env.exemplo`** (enviado ao repositório, como documentação):

```env
PORT=3000
NODE_ENV=development
NOME_APLICACAO=
SEGREDO_SESSAO=
```

**Arquivo `.gitignore`:**

```gitignore
node_modules/
.env
*.log
dados/*.sqlite
```

**Uso em `app.js`:**

```javascript
// Deve ser a PRIMEIRA linha do arquivo, antes de qualquer módulo que
// dependa das variáveis de ambiente.
require("dotenv").config();

const express = require("express");
const app = express();

const PORTA = process.env.PORT || 3000;
const AMBIENTE = process.env.NODE_ENV || "development";

// Verificação de variáveis obrigatórias: falhar na inicialização é
// preferível a falhar em produção durante o atendimento a um usuário.
const OBRIGATORIAS = ["SEGREDO_SESSAO"];
const ausentes = OBRIGATORIAS.filter((v) => !process.env[v]);

if (ausentes.length > 0) {
  console.error("Variáveis de ambiente ausentes:", ausentes.join(", "));
  console.error("Copie .env.exemplo para .env e preencha os valores.");
  process.exit(1);
}

// Variável disponível em todas as views, sem necessidade de repassá-la
// em cada chamada a res.render.
app.locals.nomeAplicacao = process.env.NOME_APLICACAO || "Aplicação";

app.listen(PORTA, () => {
  console.log(`[${AMBIENTE}] http://localhost:${PORTA}`);
});
```

---

## 10. Organização em camadas

Projetos com mais de algumas centenas de linhas exigem separação de responsabilidades:

```
app-express/
├── .env
├── .env.exemplo
├── .gitignore
├── package.json
├── nodemon.json
├── app.js                    ← configuração e inicialização
├── rotas/
│   ├── index.js              ← agregador de rotas
│   ├── cursos.js
│   └── contato.js
├── controladores/
│   ├── cursosControlador.js  ← lógica de cada rota
│   └── contatoControlador.js
├── servicos/
│   └── cursosServico.js      ← regras de negócio e acesso a dados
├── middlewares/
│   ├── erros.js
│   └── validacao.js
├── views/
│   ├── parciais/
│   └── *.ejs
└── publico/
    ├── css/
    ├── js/
    └── imagens/
```

**Arquivo `servicos/cursosServico.js`:**

```javascript
// servicos/cursosServico.js — acesso e manipulação dos dados.
// No Tutorial 6, apenas o conteúdo deste arquivo mudará: o restante
// da aplicação permanecerá inalterado ao migrar para banco de dados.

const cursos = [
  { id: 1, nome: "Informática para Internet", turno: "Integral", vagas: 40, ativo: true },
  { id: 2, nome: "Administração", turno: "Integral", vagas: 35, ativo: true },
  { id: 3, nome: "Eventos", turno: "Noturno", vagas: 30, ativo: false },
];

function listar({ turno } = {}) {
  return turno ? cursos.filter((c) => c.turno === turno) : [...cursos];
}

function buscarPorId(id) {
  return cursos.find((c) => c.id === Number(id)) || null;
}

function criar({ nome, turno, vagas }) {
  const novo = {
    id: Math.max(0, ...cursos.map((c) => c.id)) + 1,
    nome,
    turno,
    vagas: Number(vagas) || 0,
    ativo: true,
  };
  cursos.push(novo);
  return novo;
}

module.exports = { listar, buscarPorId, criar };
```

**Arquivo `controladores/cursosControlador.js`:**

```javascript
// controladores/cursosControlador.js — recebe a requisição, aciona o
// serviço e produz a resposta. Não contém regras de negócio.
const servico = require("../servicos/cursosServico");

function listar(req, res) {
  const cursos = servico.listar({ turno: req.query.turno });
  res.render("cursos", {
    titulo: "Cursos ofertados",
    cursos,
    filtroAtivo: req.query.turno || "todos",
  });
}

function detalhar(req, res, next) {
  const curso = servico.buscarPorId(req.params.id);

  if (!curso) {
    const erro = new Error("Curso não encontrado");
    erro.status = 404;
    return next(erro);
  }

  res.render("curso-detalhe", { titulo: curso.nome, curso });
}

module.exports = { listar, detalhar };
```

**Arquivo `rotas/cursos.js`:**

```javascript
const express = require("express");
const controlador = require("../controladores/cursosControlador");

const router = express.Router();

router.get("/", controlador.listar);
router.get("/:id", controlador.detalhar);

module.exports = router;
```

**Vantagem da separação:** ao substituir o vetor em memória por um banco de dados no Tutorial 6, apenas `servicos/cursosServico.js` será modificado. Controladores, rotas e views permanecem intactos.

---

## 11. Laboratório prático

### Laboratório 5.1 — Migração do site do Tutorial 4

Reimplementar em Express o website construído no Tutorial 4, preservando a aparência e as funcionalidades.

**Requisitos:**
1. `express.static` para os recursos estáticos;
2. EJS com parciais para cabeçalho e rodapé, eliminando a duplicação de HTML;
3. rotas `/`, `/sobre`, `/cursos`, `/cursos/:id`, `/contato`;
4. formulário de contato com validação e padrão *post-redirect-get*;
5. página 404 renderizada por *middleware*;
6. `morgan` para registro das requisições;
7. `dotenv` para porta e ambiente;
8. nodemon configurado por `nodemon.json`.

**Entrega:** projeto completo (sem `node_modules`) acompanhado de um documento comparando a quantidade de linhas das duas versões e enumerando as vantagens observadas. Este documento retoma o Laboratório 4.3.

### Laboratório 5.2 — Sistema de gestão de tarefas

Desenvolver uma aplicação de gerenciamento de tarefas com armazenamento em memória.

**Funcionalidades:**

| Método | Rota | Descrição |
|---|---|---|
| GET | `/tarefas` | Lista, com filtro por situação via *query string* |
| GET | `/tarefas/nova` | Formulário de criação |
| POST | `/tarefas` | Cria a tarefa e redireciona |
| GET | `/tarefas/:id` | Detalhe |
| GET | `/tarefas/:id/editar` | Formulário preenchido |
| POST | `/tarefas/:id` | Atualiza |
| POST | `/tarefas/:id/concluir` | Alterna a situação |
| POST | `/tarefas/:id/excluir` | Remove |

**Estrutura da tarefa:**

```javascript
{
  id: 1,
  titulo: "Estudar middlewares do Express",
  descricao: "Revisar a cadeia de execução e a ordem de registro",
  prioridade: "alta",        // "baixa" | "media" | "alta"
  concluida: false,
  criadaEm: new Date(),
  prazo: "2026-09-20"
}
```

**Requisitos técnicos obrigatórios:**
1. organização em camadas (rotas, controladores, serviços);
2. validação no servidor com reexibição do formulário preenchido em caso de erro;
3. *middleware* de validação reutilizável entre criação e edição;
4. filtros combináveis: situação e prioridade;
5. contadores no cabeçalho: total, pendentes e concluídas;
6. destaque visual para tarefas com prazo vencido;
7. tratamento de 404 para identificador inexistente;
8. nodemon configurado.

### Laboratório 5.3 — Middleware de limitação de requisições

Implementar, **sem bibliotecas externas**, um *middleware* que limite cada endereço IP a vinte requisições por minuto.

**Especificação:**
1. armazenar os registros em um `Map`, com o IP como chave;
2. utilizar janela deslizante de sessenta segundos;
3. ao exceder o limite, responder `429 Too Many Requests`;
4. incluir os cabeçalhos `X-RateLimit-Limit`, `X-RateLimit-Remaining` e `Retry-After`;
5. remover periodicamente as entradas expiradas, evitando crescimento indefinido da memória;
6. tornar o limite e a janela configuráveis por parâmetros.

```javascript
// Uso previsto
app.use(limitarRequisicoes({ limite: 20, janelaMs: 60_000 }));
```

**Questão:** por que o `Map` não constitui solução adequada quando a aplicação executa em múltiplos processos (`cluster`) ou em várias instâncias? Qual seria a alternativa?

---

## 12. Síntese

1. O Express é construído sobre o módulo `http` e acrescenta roteamento declarativo, *middlewares* e utilidades de resposta.
2. A ordem de registro dos *middlewares* determina a ordem de execução; 404 e tratamento de erro devem ser os últimos.
3. Todo *middleware* deve chamar `next()` ou encerrar a resposta; a omissão de ambos suspende a requisição.
4. *Middlewares* de erro são identificados pela presença de quatro parâmetros.
5. No Express 4, erros de rotas assíncronas exigem `try/catch` ou função auxiliar.
6. O EJS separa marcação de lógica; `<%= %>` escapa o conteúdo e `<%- %>` não.
7. O padrão *post-redirect-get* impede o reenvio de formulários.
8. Configurações e credenciais pertencem ao arquivo `.env`, excluído do controle de versão.
9. A organização em camadas isola a fonte de dados do restante da aplicação.

**Próximo tutorial:** [Banco de dados com SQLite, MySQL e ORM](./06-banco-de-dados-orm.md)
