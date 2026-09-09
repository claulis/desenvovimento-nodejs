# Tutorial 4 — Website Básico com Node.js, HTML e CSS

> **Pré-requisitos:** Tutoriais 1 a 3 concluídos; noções de HTML e CSS.
> **Duração estimada:** 3 aulas de 50 minutos.
> **Ao final deste tutorial, o estudante será capaz de:** explicar o funcionamento do protocolo HTTP, criar um servidor web utilizando apenas módulos nativos do Node.js, servir páginas HTML e folhas de estilo CSS, implementar roteamento, tratar erros 404 e gerar HTML dinamicamente a partir de dados.

---

## 1. Objetivo pedagógico deste tutorial

O Tutorial 5 apresentará o Express, *framework* que reduz a construção de um servidor a poucas linhas. Antes disso, é indispensável compreender **o que o Express automatiza**. Este tutorial constrói um servidor completo utilizando exclusivamente o módulo nativo `http`.

Trata-se de um exercício deliberadamente trabalhoso. Ao final, o estudante terá escrito manualmente o roteamento, a identificação de tipos MIME e o tratamento de erros — precisamente as tarefas que o Express resolve. O contraste entre este tutorial e o seguinte é o objetivo de aprendizagem.

---

## 2. O protocolo HTTP

### 2.1 O ciclo requisição–resposta

Toda comunicação na web segue o mesmo padrão: o cliente (navegador) envia uma **requisição**; o servidor devolve uma **resposta**. O servidor não inicia a comunicação.

```
NAVEGADOR                                          SERVIDOR
    │                                                  │
    │  GET /sobre.html HTTP/1.1                        │
    │  Host: localhost:3000                            │
    │  User-Agent: Mozilla/5.0 ...                     │
    │  Accept: text/html                               │
    ├─────────────────────────────────────────────────▶│
    │                                                  │  processa
    │  HTTP/1.1 200 OK                                 │
    │  Content-Type: text/html; charset=utf-8          │
    │  Content-Length: 1843                            │
    │                                                  │
    │  <!DOCTYPE html><html>...                        │
    │◀─────────────────────────────────────────────────┤
    │                                                  │
```

### 2.2 Componentes da requisição

- **Método**: `GET` (obter), `POST` (enviar), `PUT` (substituir), `PATCH` (alterar parcialmente), `DELETE` (remover);
- **Caminho** (*path*): `/sobre.html`, `/api/alunos`, `/`;
- **Cabeçalhos**: metadados, como o formato aceito e o idioma preferido;
- **Corpo** (*body*): dados enviados, presente em `POST`, `PUT` e `PATCH`.

### 2.3 Códigos de status

| Faixa | Significado | Exemplos frequentes |
|---|---|---|
| 2xx | Sucesso | `200 OK`, `201 Created`, `204 No Content` |
| 3xx | Redirecionamento | `301 Moved Permanently`, `302 Found`, `304 Not Modified` |
| 4xx | Erro do cliente | `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found` |
| 5xx | Erro do servidor | `500 Internal Server Error`, `503 Service Unavailable` |

**Distinção essencial:** códigos 4xx indicam que o cliente solicitou algo inválido; códigos 5xx indicam que o servidor falhou ao processar uma solicitação válida. Retornar `200` acompanhado de uma mensagem de erro no corpo é um defeito de projeto: os mecanismos automáticos que consomem a resposta interpretarão a operação como bem-sucedida.

### 2.4 Tipos MIME

O cabeçalho `Content-Type` informa ao navegador como interpretar o conteúdo recebido. Um arquivo CSS enviado como `text/plain` é exibido como texto, e não aplicado como estilo.

| Extensão | Tipo MIME |
|---|---|
| `.html` | `text/html; charset=utf-8` |
| `.css` | `text/css; charset=utf-8` |
| `.js` | `text/javascript; charset=utf-8` |
| `.json` | `application/json; charset=utf-8` |
| `.png` | `image/png` |
| `.jpg` | `image/jpeg` |
| `.svg` | `image/svg+xml` |
| `.ico` | `image/x-icon` |
| `.woff2` | `font/woff2` |

A omissão de `charset=utf-8` em conteúdo textual produz o problema clássico de acentuação incorreta (`InformaÃ§Ã£o` no lugar de `Informação`).

---

## 3. Servidor mínimo

**Arquivo `servidor-minimo.js`:**

```javascript
// servidor-minimo.js — o menor servidor HTTP funcional
const http = require("node:http");

const PORTA = 3000;

// A função de callback é executada a cada requisição recebida.
// requisicao  -> objeto com os dados enviados pelo cliente
// resposta    -> objeto utilizado para construir a resposta
const servidor = http.createServer((requisicao, resposta) => {
  console.log(`${requisicao.method} ${requisicao.url}`);

  // writeHead define o código de status e os cabeçalhos.
  resposta.writeHead(200, { "Content-Type": "text/plain; charset=utf-8" });

  // end envia o corpo e encerra a resposta.
  // Sem esta chamada, o navegador permanece aguardando indefinidamente.
  resposta.end("Servidor Node.js em funcionamento.\n");
});

servidor.listen(PORTA, () => {
  console.log(`Servidor disponível em http://localhost:${PORTA}`);
  console.log("Pressione Ctrl+C para encerrar.");
});
```

```powershell
node servidor-minimo.js
```

Ao acessar `http://localhost:3000` no navegador, observa-se no terminal o registro de **duas** requisições: a solicitação da página e a solicitação automática de `/favicon.ico`, que o navegador realiza para obter o ícone da aba.

**Ponto de atenção:** ao alterar o código-fonte, é necessário encerrar o servidor com `Ctrl+C` e reiniciá-lo. O Node.js carrega o arquivo uma única vez. Essa limitação será eliminada pelo nodemon, no Tutorial 5.

---

## 4. Roteamento manual

**Arquivo `servidor-rotas.js`:**

```javascript
// servidor-rotas.js — respostas distintas conforme o caminho solicitado
const http = require("node:http");

const servidor = http.createServer((requisicao, resposta) => {
  // A propriedade url contém caminho e query string ("/busca?termo=node").
  // A classe URL separa esses componentes. O segundo argumento fornece a
  // base necessária para interpretar uma URL relativa.
  const url = new URL(requisicao.url, `http://${requisicao.headers.host}`);
  const caminho = url.pathname;
  const metodo = requisicao.method;

  console.log(`${metodo} ${caminho}`);

  if (caminho === "/" && metodo === "GET") {
    resposta.writeHead(200, { "Content-Type": "text/html; charset=utf-8" });
    resposta.end(`
      <h1>Página inicial</h1>
      <ul>
        <li><a href="/sobre">Sobre</a></li>
        <li><a href="/hora">Hora do servidor</a></li>
        <li><a href="/saudacao?nome=Ana">Saudação personalizada</a></li>
        <li><a href="/inexistente">Rota inexistente (404)</a></li>
      </ul>
    `);

  } else if (caminho === "/sobre" && metodo === "GET") {
    resposta.writeHead(200, { "Content-Type": "text/html; charset=utf-8" });
    resposta.end("<h1>Sobre</h1><p>Servidor construído com o módulo http.</p>");

  } else if (caminho === "/hora" && metodo === "GET") {
    // Resposta em JSON, formato utilizado por APIs.
    resposta.writeHead(200, { "Content-Type": "application/json; charset=utf-8" });
    resposta.end(JSON.stringify({
      horario: new Date().toISOString(),
      formatado: new Date().toLocaleString("pt-BR"),
      fusoHorario: Intl.DateTimeFormat().resolvedOptions().timeZone,
    }, null, 2));

  } else if (caminho === "/saudacao" && metodo === "GET") {
    // Leitura de parâmetro da query string.
    const nome = url.searchParams.get("nome") || "visitante";
    resposta.writeHead(200, { "Content-Type": "text/html; charset=utf-8" });
    resposta.end(`<h1>Olá, ${nome}!</h1><p>Experimente alterar o parâmetro
      <code>nome</code> na barra de endereços.</p>`);

  } else {
    // Nenhuma rota correspondeu: erro 404.
    resposta.writeHead(404, { "Content-Type": "text/html; charset=utf-8" });
    resposta.end("<h1>404 — Página não encontrada</h1><a href='/'>Início</a>");
  }
});

servidor.listen(3000, () => console.log("http://localhost:3000"));
```

**Observação crítica de segurança:** a rota `/saudacao` insere diretamente na página um valor fornecido pelo usuário. Um acesso a `/saudacao?nome=<script>alert(1)</script>` executaria código arbitrário no navegador — vulnerabilidade denominada **XSS** (*Cross-Site Scripting*). A Seção 7 apresenta a correção.

Constata-se, ainda, que o encadeamento de `if/else` cresce de forma insustentável. Um site com trinta páginas produziria uma função ilegível. Esse é o problema que o roteamento do Express resolve.

---

## 5. Servidor de arquivos estáticos

Um website real é composto por arquivos HTML, CSS, JavaScript e imagens. O servidor deve localizá-los no disco e entregá-los com o tipo MIME correto.

### 5.1 Estrutura do projeto

```
site-nodejs/
├── package.json
├── servidor.js
└── publico/
    ├── index.html
    ├── sobre.html
    ├── contato.html
    ├── 404.html
    ├── css/
    │   └── estilo.css
    └── js/
        └── script.js
```

```powershell
mkdir site-nodejs
cd site-nodejs
npm init -y
mkdir publico
mkdir publico\css
mkdir publico\js
code .
```

### 5.2 O servidor

**Arquivo `servidor.js`:**

```javascript
// servidor.js — servidor de arquivos estáticos com módulos nativos
const http = require("node:http");
const fs = require("node:fs/promises");
const path = require("node:path");

const PORTA = process.env.PORT || 3000;
const PASTA_PUBLICA = path.join(__dirname, "publico");

// Associação entre extensão e tipo MIME.
const TIPOS_MIME = {
  ".html": "text/html; charset=utf-8",
  ".css": "text/css; charset=utf-8",
  ".js": "text/javascript; charset=utf-8",
  ".json": "application/json; charset=utf-8",
  ".png": "image/png",
  ".jpg": "image/jpeg",
  ".jpeg": "image/jpeg",
  ".gif": "image/gif",
  ".svg": "image/svg+xml",
  ".ico": "image/x-icon",
  ".woff2": "font/woff2",
};

/**
 * Converte o caminho solicitado na URL em um caminho de arquivo no disco,
 * impedindo o acesso a arquivos fora da pasta pública.
 * Devolve null quando a tentativa é considerada maliciosa.
 */
function resolverCaminhoSeguro(caminhoUrl) {
  // decodeURIComponent converte sequências como %20 em espaços.
  let relativo = decodeURIComponent(caminhoUrl);

  // A raiz do site corresponde a index.html.
  if (relativo === "/") relativo = "/index.html";

  // Caminhos sem extensão recebem .html, permitindo URLs limpas
  // como /sobre em vez de /sobre.html.
  if (!path.extname(relativo)) relativo += ".html";

  const caminhoAbsoluto = path.join(PASTA_PUBLICA, relativo);

  // PROTEÇÃO CONTRA PATH TRAVERSAL.
  // Uma requisição a /../../../Windows/System32/config/sam tentaria ler
  // arquivos do sistema. Após path.join, verifica-se que o resultado
  // permanece dentro da pasta pública.
  if (!caminhoAbsoluto.startsWith(PASTA_PUBLICA)) {
    return null;
  }

  return caminhoAbsoluto;
}

/** Envia a página 404 personalizada. */
async function responder404(resposta) {
  try {
    const pagina = await fs.readFile(path.join(PASTA_PUBLICA, "404.html"));
    resposta.writeHead(404, { "Content-Type": TIPOS_MIME[".html"] });
    resposta.end(pagina);
  } catch {
    resposta.writeHead(404, { "Content-Type": "text/plain; charset=utf-8" });
    resposta.end("404 - Não encontrado");
  }
}

const servidor = http.createServer(async (requisicao, resposta) => {
  const inicio = Date.now();
  const url = new URL(requisicao.url, `http://${requisicao.headers.host}`);

  // Este servidor entrega arquivos; apenas GET e HEAD fazem sentido.
  if (requisicao.method !== "GET" && requisicao.method !== "HEAD") {
    resposta.writeHead(405, {
      "Content-Type": "text/plain; charset=utf-8",
      Allow: "GET, HEAD",
    });
    resposta.end("405 - Método não permitido");
    return;
  }

  const caminhoArquivo = resolverCaminhoSeguro(url.pathname);

  if (caminhoArquivo === null) {
    resposta.writeHead(403, { "Content-Type": "text/plain; charset=utf-8" });
    resposta.end("403 - Acesso negado");
    console.warn(`ACESSO BLOQUEADO: ${url.pathname}`);
    return;
  }

  try {
    const conteudo = await fs.readFile(caminhoArquivo);
    const extensao = path.extname(caminhoArquivo).toLowerCase();
    const tipo = TIPOS_MIME[extensao] || "application/octet-stream";

    resposta.writeHead(200, {
      "Content-Type": tipo,
      "Content-Length": conteudo.length,
      // Recursos estáticos podem ser armazenados em cache pelo navegador.
      // Durante o desenvolvimento, convém utilizar "no-cache".
      "Cache-Control": extensao === ".html" ? "no-cache" : "public, max-age=3600",
    });

    // Em uma requisição HEAD, apenas os cabeçalhos são enviados.
    resposta.end(requisicao.method === "HEAD" ? undefined : conteudo);

    console.log(`200 ${requisicao.method} ${url.pathname} (${Date.now() - inicio}ms)`);

  } catch (erro) {
    if (erro.code === "ENOENT" || erro.code === "EISDIR") {
      await responder404(resposta);
      console.log(`404 ${requisicao.method} ${url.pathname}`);
    } else {
      // Falha do servidor: registrar o detalhe internamente, mas não
      // expor informações da infraestrutura ao cliente.
      console.error("Erro interno:", erro);
      resposta.writeHead(500, { "Content-Type": "text/plain; charset=utf-8" });
      resposta.end("500 - Erro interno do servidor");
    }
  }
});

servidor.listen(PORTA, () => {
  console.log(`\n  Servidor iniciado`);
  console.log(`  Endereço : http://localhost:${PORTA}`);
  console.log(`  Pasta    : ${PASTA_PUBLICA}\n`);
});

// Encerramento controlado: aguarda a conclusão das requisições em andamento
// antes de finalizar o processo.
process.on("SIGINT", () => {
  console.log("\nEncerrando o servidor...");
  servidor.close(() => {
    console.log("Servidor encerrado.");
    process.exit(0);
  });
});
```

### 5.3 As páginas HTML

**Arquivo `publico/index.html`:**

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Início | Curso de Node.js</title>
  <link rel="stylesheet" href="/css/estilo.css">
</head>
<body>
  <header class="cabecalho">
    <div class="container cabecalho__conteudo">
      <a href="/" class="marca">&lt;/&gt; Curso de Node.js</a>
      <nav class="navegacao">
        <a href="/" class="ativo">Início</a>
        <a href="/sobre">Sobre</a>
        <a href="/contato">Contato</a>
      </nav>
    </div>
  </header>

  <main class="container">
    <section class="destaque">
      <h1>JavaScript além do navegador</h1>
      <p class="destaque__texto">
        Esta página está sendo entregue por um servidor construído
        exclusivamente com módulos nativos do Node.js — sem frameworks.
      </p>
      <a href="/sobre" class="botao">Entender como funciona</a>
    </section>

    <section class="cartoes">
      <article class="cartao">
        <div class="cartao__icone">📁</div>
        <h2>Sistema de arquivos</h2>
        <p>Leitura e escrita de arquivos, percurso de diretórios e
           processamento de grandes volumes com streams.</p>
      </article>

      <article class="cartao">
        <div class="cartao__icone">🌐</div>
        <h2>Servidores web</h2>
        <p>Criação de servidores HTTP, roteamento, entrega de arquivos
           estáticos e construção de APIs REST.</p>
      </article>

      <article class="cartao">
        <div class="cartao__icone">🗄️</div>
        <h2>Bancos de dados</h2>
        <p>Persistência de dados em SQLite e MySQL por meio de
           mapeamento objeto-relacional.</p>
      </article>
    </section>

    <section class="painel">
      <h2>Informações do servidor</h2>
      <p>Os dados abaixo são obtidos por requisição assíncrona à rota
         <code>/api/info</code>, implementada no Laboratório 4.2.</p>
      <pre id="saida-info">Carregando...</pre>
    </section>
  </main>

  <footer class="rodape">
    <div class="container">
      <p>Instituto Federal de Brasília — Ensino Médio Integrado em
         Informática para Internet</p>
    </div>
  </footer>

  <script src="/js/script.js"></script>
</body>
</html>
```

**Arquivo `publico/sobre.html`:**

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Sobre | Curso de Node.js</title>
  <link rel="stylesheet" href="/css/estilo.css">
</head>
<body>
  <header class="cabecalho">
    <div class="container cabecalho__conteudo">
      <a href="/" class="marca">&lt;/&gt; Curso de Node.js</a>
      <nav class="navegacao">
        <a href="/">Início</a>
        <a href="/sobre" class="ativo">Sobre</a>
        <a href="/contato">Contato</a>
      </nav>
    </div>
  </header>

  <main class="container">
    <h1>Como esta página chegou até o navegador</h1>

    <ol class="etapas">
      <li>
        <strong>Solicitação.</strong> O navegador enviou a requisição
        <code>GET /sobre</code> para <code>localhost</code> na porta 3000.
      </li>
      <li>
        <strong>Resolução do caminho.</strong> O servidor acrescentou a
        extensão <code>.html</code> e verificou que o caminho resultante
        permanece dentro da pasta <code>publico</code>.
      </li>
      <li>
        <strong>Leitura.</strong> O arquivo foi lido do disco com
        <code>fs.readFile</code>.
      </li>
      <li>
        <strong>Definição do tipo.</strong> A extensão <code>.html</code>
        determinou o cabeçalho
        <code>Content-Type: text/html; charset=utf-8</code>.
      </li>
      <li>
        <strong>Envio.</strong> O conteúdo foi transmitido com o código de
        status <code>200 OK</code>.
      </li>
      <li>
        <strong>Renderização.</strong> O navegador interpretou o HTML e
        solicitou os recursos referenciados: a folha de estilo e o script.
      </li>
    </ol>

    <div class="aviso">
      <strong>Observação.</strong> Cada uma dessas etapas foi programada
      manualmente. No próximo tutorial, todas serão substituídas por uma
      única linha: <code>app.use(express.static("publico"))</code>.
    </div>
  </main>

  <footer class="rodape">
    <div class="container">
      <p>Instituto Federal de Brasília</p>
    </div>
  </footer>
</body>
</html>
```

**Arquivo `publico/404.html`:**

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Página não encontrada</title>
  <link rel="stylesheet" href="/css/estilo.css">
</head>
<body>
  <main class="container erro">
    <div class="erro__codigo">404</div>
    <h1>Página não encontrada</h1>
    <p>O endereço solicitado não corresponde a nenhum arquivo da pasta
       pública do servidor.</p>
    <a href="/" class="botao">Voltar ao início</a>
  </main>
</body>
</html>
```

### 5.4 A folha de estilo

**Arquivo `publico/css/estilo.css`:**

```css
/* estilo.css — folha de estilo do site
   As variáveis CSS (custom properties) centralizam cores e medidas,
   permitindo alterar a identidade visual em um único ponto. */

:root {
  --cor-primaria: #2d6a4f;
  --cor-primaria-clara: #40916c;
  --cor-destaque: #f4a261;
  --cor-texto: #1b1b1b;
  --cor-texto-suave: #5a5a5a;
  --cor-fundo: #f8f9fa;
  --cor-superficie: #ffffff;
  --cor-borda: #e0e0e0;

  --espaco: 1rem;
  --raio: 10px;
  --sombra: 0 2px 8px rgba(0, 0, 0, 0.08);
  --largura-maxima: 1000px;
}

/* Reinicialização mínima. box-sizing: border-box faz com que padding e
   border sejam computados dentro da largura declarada. */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: "Segoe UI", system-ui, -apple-system, sans-serif;
  line-height: 1.65;
  color: var(--cor-texto);
  background-color: var(--cor-fundo);
}

.container {
  max-width: var(--largura-maxima);
  margin: 0 auto;
  padding: 0 var(--espaco);
}

/* ---------- CABEÇALHO ---------- */

.cabecalho {
  background-color: var(--cor-primaria);
  color: #fff;
  box-shadow: var(--sombra);
  position: sticky;
  top: 0;
  z-index: 10;
}

.cabecalho__conteudo {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: var(--espaco);
  padding-top: 0.9rem;
  padding-bottom: 0.9rem;
}

.marca {
  font-size: 1.15rem;
  font-weight: 700;
  color: #fff;
  text-decoration: none;
  font-family: Consolas, "Courier New", monospace;
}

.navegacao {
  display: flex;
  gap: 1.25rem;
}

.navegacao a {
  color: rgba(255, 255, 255, 0.85);
  text-decoration: none;
  padding: 0.25rem 0;
  border-bottom: 2px solid transparent;
  transition: color 0.2s, border-color 0.2s;
}

.navegacao a:hover,
.navegacao a.ativo {
  color: #fff;
  border-bottom-color: var(--cor-destaque);
}

/* ---------- DESTAQUE ---------- */

.destaque {
  text-align: center;
  padding: 3.5rem 1rem;
}

.destaque h1 {
  font-size: clamp(1.8rem, 5vw, 2.6rem);
  color: var(--cor-primaria);
  margin-bottom: var(--espaco);
}

.destaque__texto {
  max-width: 60ch;
  margin: 0 auto 1.75rem;
  color: var(--cor-texto-suave);
  font-size: 1.05rem;
}

.botao {
  display: inline-block;
  background-color: var(--cor-primaria);
  color: #fff;
  padding: 0.7rem 1.6rem;
  border-radius: var(--raio);
  text-decoration: none;
  font-weight: 600;
  border: none;
  cursor: pointer;
  font-size: 1rem;
  transition: background-color 0.2s, transform 0.1s;
}

.botao:hover {
  background-color: var(--cor-primaria-clara);
}

.botao:active {
  transform: translateY(1px);
}

/* ---------- CARTÕES ---------- */

/* auto-fit com minmax cria um layout responsivo sem media queries:
   as colunas se ajustam automaticamente ao espaço disponível. */
.cartoes {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
  gap: 1.5rem;
  padding: 2rem 0;
}

.cartao {
  background-color: var(--cor-superficie);
  border: 1px solid var(--cor-borda);
  border-radius: var(--raio);
  padding: 1.5rem;
  transition: transform 0.2s, box-shadow 0.2s;
}

.cartao:hover {
  transform: translateY(-4px);
  box-shadow: var(--sombra);
}

.cartao__icone {
  font-size: 2rem;
  margin-bottom: 0.5rem;
}

.cartao h2 {
  font-size: 1.15rem;
  color: var(--cor-primaria);
  margin-bottom: 0.5rem;
}

.cartao p {
  color: var(--cor-texto-suave);
  font-size: 0.95rem;
}

/* ---------- CONTEÚDO GERAL ---------- */

main h1 {
  color: var(--cor-primaria);
  margin: 2rem 0 1rem;
}

.etapas {
  background-color: var(--cor-superficie);
  border: 1px solid var(--cor-borda);
  border-radius: var(--raio);
  padding: 1.5rem 1.5rem 1.5rem 2.75rem;
  margin-bottom: 1.5rem;
}

.etapas li {
  margin-bottom: 0.85rem;
}

.etapas li:last-child {
  margin-bottom: 0;
}

code {
  background-color: #eef2f0;
  color: var(--cor-primaria);
  padding: 0.15em 0.4em;
  border-radius: 4px;
  font-family: Consolas, "Courier New", monospace;
  font-size: 0.9em;
}

pre {
  background-color: #1e2a24;
  color: #d8f3dc;
  padding: 1rem;
  border-radius: var(--raio);
  overflow-x: auto;
  font-family: Consolas, "Courier New", monospace;
  font-size: 0.9rem;
}

.aviso {
  background-color: #fff7e6;
  border-left: 4px solid var(--cor-destaque);
  padding: 1rem 1.25rem;
  border-radius: 0 var(--raio) var(--raio) 0;
  margin-bottom: 2rem;
}

.painel {
  background-color: var(--cor-superficie);
  border: 1px solid var(--cor-borda);
  border-radius: var(--raio);
  padding: 1.5rem;
  margin-bottom: 2rem;
}

.painel h2 {
  color: var(--cor-primaria);
  margin-bottom: 0.5rem;
}

/* ---------- FORMULÁRIO ---------- */

.formulario {
  background-color: var(--cor-superficie);
  border: 1px solid var(--cor-borda);
  border-radius: var(--raio);
  padding: 1.75rem;
  max-width: 560px;
  margin-bottom: 2rem;
}

.campo {
  margin-bottom: 1.15rem;
}

.campo label {
  display: block;
  font-weight: 600;
  margin-bottom: 0.35rem;
  font-size: 0.95rem;
}

.campo input,
.campo textarea,
.campo select {
  width: 100%;
  padding: 0.65rem 0.8rem;
  border: 1px solid var(--cor-borda);
  border-radius: 6px;
  font-family: inherit;
  font-size: 1rem;
}

.campo input:focus,
.campo textarea:focus,
.campo select:focus {
  outline: 2px solid var(--cor-primaria-clara);
  outline-offset: 1px;
  border-color: transparent;
}

.campo textarea {
  resize: vertical;
  min-height: 120px;
}

/* ---------- PÁGINA DE ERRO ---------- */

.erro {
  text-align: center;
  padding: 5rem 1rem;
}

.erro__codigo {
  font-size: 6rem;
  font-weight: 800;
  color: var(--cor-primaria);
  line-height: 1;
  opacity: 0.25;
}

.erro h1 {
  margin-bottom: 0.5rem;
}

.erro p {
  color: var(--cor-texto-suave);
  margin-bottom: 1.75rem;
}

/* ---------- RODAPÉ ---------- */

.rodape {
  background-color: #1e2a24;
  color: rgba(255, 255, 255, 0.75);
  padding: 1.75rem 0;
  margin-top: 3rem;
  text-align: center;
  font-size: 0.9rem;
}

/* ---------- ADAPTAÇÃO PARA TELAS PEQUENAS ---------- */

@media (max-width: 600px) {
  .cabecalho__conteudo {
    flex-direction: column;
    align-items: flex-start;
  }

  .navegacao {
    width: 100%;
    justify-content: space-between;
  }
}
```

### 5.5 O script do cliente

**Arquivo `publico/js/script.js`:**

```javascript
// script.js — executado pelo NAVEGADOR, e não pelo Node.js.
// Neste arquivo estão disponíveis document, window e fetch;
// não estão disponíveis require, fs ou process.

document.addEventListener("DOMContentLoaded", () => {
  const saida = document.getElementById("saida-info");
  if (!saida) return;

  // fetch realiza uma requisição HTTP em segundo plano, sem recarregar
  // a página. A rota /api/info é implementada no Laboratório 4.2.
  fetch("/api/info")
    .then((resposta) => {
      if (!resposta.ok) {
        throw new Error(`Servidor respondeu com status ${resposta.status}`);
      }
      return resposta.json();
    })
    .then((dados) => {
      saida.textContent = JSON.stringify(dados, null, 2);
    })
    .catch((erro) => {
      saida.textContent =
        `Não foi possível obter os dados.\nMotivo: ${erro.message}\n\n` +
        `A rota /api/info ainda não foi implementada (Laboratório 4.2).`;
    });
});
```

Execução:

```powershell
node servidor.js
```

Acessar `http://localhost:3000`. O site completo é entregue: HTML, CSS e JavaScript, com navegação funcional entre as páginas e página 404 personalizada.

---

## 6. Geração dinâmica de HTML

Até este ponto, os arquivos foram entregues sem modificação. A geração dinâmica consiste em produzir o HTML no servidor a partir de dados.

**Arquivo `servidor-dinamico.js`:**

```javascript
// servidor-dinamico.js — HTML construído a partir de uma coleção de dados
const http = require("node:http");

const CURSOS = [
  { id: 1, nome: "Informática para Internet", turno: "Integral", vagas: 40, ativo: true },
  { id: 2, nome: "Administração", turno: "Integral", vagas: 35, ativo: true },
  { id: 3, nome: "Eventos", turno: "Noturno", vagas: 30, ativo: false },
  { id: 4, nome: "Logística", turno: "Noturno", vagas: 30, ativo: true },
];

/**
 * Converte caracteres especiais em entidades HTML.
 * Impede que dados fornecidos por terceiros sejam interpretados como
 * marcação, o que caracterizaria uma vulnerabilidade de XSS.
 */
function escapar(texto) {
  return String(texto)
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;")
    .replace(/'/g, "&#39;");
}

/** Constrói a página completa a partir da lista de cursos. */
function montarPagina(cursos, filtro) {
  // map transforma cada objeto em uma linha de tabela;
  // join concatena o vetor resultante em uma única string.
  const linhas = cursos
    .map(
      (curso) => `
        <tr>
          <td>${curso.id}</td>
          <td>${escapar(curso.nome)}</td>
          <td>${escapar(curso.turno)}</td>
          <td style="text-align:right">${curso.vagas}</td>
          <td>
            <span class="etiqueta ${curso.ativo ? "etiqueta--ativa" : "etiqueta--inativa"}">
              ${curso.ativo ? "Com inscrições" : "Encerrado"}
            </span>
          </td>
        </tr>`
    )
    .join("");

  const totalVagas = cursos.reduce((soma, c) => soma + c.vagas, 0);

  return `<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Cursos ofertados</title>
  <style>
    body { font-family: "Segoe UI", sans-serif; max-width: 850px;
           margin: 2rem auto; padding: 0 1rem; color: #1b1b1b; }
    h1 { color: #2d6a4f; }
    table { width: 100%; border-collapse: collapse; margin-top: 1rem; }
    th, td { padding: 0.7rem; border-bottom: 1px solid #e0e0e0;
             text-align: left; }
    th { background: #2d6a4f; color: #fff; }
    tr:hover td { background: #f1f7f4; }
    .filtros { margin: 1rem 0; display: flex; gap: .5rem; flex-wrap: wrap; }
    .filtros a { padding: .4rem .9rem; border: 1px solid #2d6a4f;
                 border-radius: 6px; text-decoration: none; color: #2d6a4f; }
    .filtros a.ativo { background: #2d6a4f; color: #fff; }
    .etiqueta { padding: .15rem .6rem; border-radius: 999px;
                font-size: .8rem; font-weight: 600; }
    .etiqueta--ativa { background: #d8f3dc; color: #1b4332; }
    .etiqueta--inativa { background: #ffe5e5; color: #8b0000; }
    .resumo { margin-top: 1rem; color: #5a5a5a; font-size: .95rem; }
  </style>
</head>
<body>
  <h1>Cursos ofertados</h1>
  <p>Esta tabela é gerada no servidor a cada requisição, a partir de um
     vetor de objetos JavaScript.</p>

  <div class="filtros">
    <a href="/" class="${filtro === "todos" ? "ativo" : ""}">Todos</a>
    <a href="/?turno=Integral" class="${filtro === "Integral" ? "ativo" : ""}">Integral</a>
    <a href="/?turno=Noturno" class="${filtro === "Noturno" ? "ativo" : ""}">Noturno</a>
  </div>

  <table>
    <thead>
      <tr>
        <th>ID</th><th>Curso</th><th>Turno</th>
        <th style="text-align:right">Vagas</th><th>Situação</th>
      </tr>
    </thead>
    <tbody>${linhas || `<tr><td colspan="5">Nenhum curso encontrado.</td></tr>`}</tbody>
  </table>

  <p class="resumo">
    ${cursos.length} curso(s) exibido(s) &middot; ${totalVagas} vagas no total
    &middot; página gerada em ${new Date().toLocaleString("pt-BR")}
  </p>
</body>
</html>`;
}

const servidor = http.createServer((requisicao, resposta) => {
  const url = new URL(requisicao.url, `http://${requisicao.headers.host}`);

  if (url.pathname === "/") {
    const turno = url.searchParams.get("turno");
    const filtrados = turno ? CURSOS.filter((c) => c.turno === turno) : CURSOS;

    resposta.writeHead(200, { "Content-Type": "text/html; charset=utf-8" });
    resposta.end(montarPagina(filtrados, turno || "todos"));

  } else if (url.pathname === "/api/cursos") {
    // A mesma informação, entregue como dados em vez de página.
    resposta.writeHead(200, { "Content-Type": "application/json; charset=utf-8" });
    resposta.end(JSON.stringify(CURSOS, null, 2));

  } else {
    resposta.writeHead(404, { "Content-Type": "text/html; charset=utf-8" });
    resposta.end("<h1>404</h1><a href='/'>Início</a>");
  }
});

servidor.listen(3000, () => console.log("http://localhost:3000"));
```

**Comparação relevante:** a rota `/` devolve HTML pronto para exibição; a rota `/api/cursos` devolve os mesmos dados em JSON, destinados a serem consumidos por outro programa. A primeira caracteriza renderização no servidor; a segunda, uma API — objeto do Tutorial 7.

---

## 7. Recebimento de dados de formulários

**Arquivo `publico/contato.html`:**

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Contato | Curso de Node.js</title>
  <link rel="stylesheet" href="/css/estilo.css">
</head>
<body>
  <header class="cabecalho">
    <div class="container cabecalho__conteudo">
      <a href="/" class="marca">&lt;/&gt; Curso de Node.js</a>
      <nav class="navegacao">
        <a href="/">Início</a>
        <a href="/sobre">Sobre</a>
        <a href="/contato" class="ativo">Contato</a>
      </nav>
    </div>
  </header>

  <main class="container">
    <h1>Formulário de contato</h1>

    <!-- method="POST" envia os dados no corpo da requisição.
         action indica a rota que processará o envio. -->
    <form class="formulario" method="POST" action="/contato">
      <div class="campo">
        <label for="nome">Nome completo</label>
        <input type="text" id="nome" name="nome" required maxlength="100">
      </div>

      <div class="campo">
        <label for="email">Correio eletrônico</label>
        <input type="email" id="email" name="email" required>
      </div>

      <div class="campo">
        <label for="assunto">Assunto</label>
        <select id="assunto" name="assunto">
          <option value="duvida">Dúvida sobre o conteúdo</option>
          <option value="erro">Relato de erro no material</option>
          <option value="sugestao">Sugestão</option>
        </select>
      </div>

      <div class="campo">
        <label for="mensagem">Mensagem</label>
        <textarea id="mensagem" name="mensagem" required
                  maxlength="1000"></textarea>
      </div>

      <button type="submit" class="botao">Enviar mensagem</button>
    </form>
  </main>

  <footer class="rodape">
    <div class="container"><p>Instituto Federal de Brasília</p></div>
  </footer>
</body>
</html>
```

Tratamento no servidor — trecho a ser acrescentado a `servidor.js`, **antes** da entrega de arquivos estáticos:

```javascript
/**
 * Lê o corpo da requisição.
 * O corpo chega em blocos, através de eventos "data"; o evento "end"
 * sinaliza o término. O limite de tamanho impede que uma requisição
 * maliciosa esgote a memória do servidor.
 */
function lerCorpo(requisicao, limiteBytes = 1024 * 100) {
  return new Promise((resolver, rejeitar) => {
    let dados = "";
    let tamanho = 0;

    requisicao.on("data", (bloco) => {
      tamanho += bloco.length;
      if (tamanho > limiteBytes) {
        rejeitar(new Error("Corpo da requisição excede o limite permitido"));
        requisicao.destroy();
        return;
      }
      dados += bloco;
    });

    requisicao.on("end", () => resolver(dados));
    requisicao.on("error", rejeitar);
  });
}

// Dentro do createServer, antes do tratamento de arquivos estáticos:
if (url.pathname === "/contato" && requisicao.method === "POST") {
  try {
    const corpo = await lerCorpo(requisicao);

    // Formulários HTML enviam os dados no formato
    // "nome=Ana&email=ana%40exemplo.com". URLSearchParams decodifica.
    const dados = Object.fromEntries(new URLSearchParams(corpo));

    // VALIDAÇÃO NO SERVIDOR: os atributos "required" do HTML podem ser
    // contornados. Toda validação deve ser repetida no servidor.
    const erros = [];
    if (!dados.nome || dados.nome.trim().length < 3) {
      erros.push("O nome deve conter ao menos três caracteres.");
    }
    if (!dados.email || !dados.email.includes("@")) {
      erros.push("O endereço de correio eletrônico é inválido.");
    }
    if (!dados.mensagem || dados.mensagem.trim().length < 10) {
      erros.push("A mensagem deve conter ao menos dez caracteres.");
    }

    if (erros.length > 0) {
      resposta.writeHead(400, { "Content-Type": "text/html; charset=utf-8" });
      resposta.end(`
        <h1>Dados inválidos</h1>
        <ul>${erros.map((e) => `<li>${escapar(e)}</li>`).join("")}</ul>
        <a href="/contato">Voltar ao formulário</a>
      `);
      return;
    }

    console.log("Mensagem recebida:", dados);
    // Em uma aplicação real, os dados seriam gravados no banco (Tutorial 6)
    // ou encaminhados por correio eletrônico.

    resposta.writeHead(200, { "Content-Type": "text/html; charset=utf-8" });
    resposta.end(`
      <h1>Mensagem recebida</h1>
      <p>Obrigado pelo contato, ${escapar(dados.nome)}.</p>
      <p>Uma resposta será enviada para ${escapar(dados.email)}.</p>
      <a href="/">Voltar ao início</a>
    `);
  } catch (erro) {
    resposta.writeHead(413, { "Content-Type": "text/plain; charset=utf-8" });
    resposta.end("413 - Requisição muito extensa");
  }
  return;
}
```

**Princípio fundamental:** validações realizadas no navegador melhoram a experiência de uso, porém não constituem mecanismo de segurança. Qualquer pessoa pode enviar uma requisição diretamente ao servidor, sem passar pelo formulário. **Toda validação deve ser repetida no servidor.**

---

## 8. Laboratório prático

### Laboratório 4.1 — Website institucional completo

Construir um website de cinco páginas utilizando exclusivamente o módulo `http`:

1. `/` — página inicial com apresentação;
2. `/cursos` — lista de cursos gerada dinamicamente a partir de um vetor de objetos, com filtro por turno via *query string*;
3. `/cursos/detalhe?id=N` — página de detalhe de um curso, com erro 404 quando o identificador não existir;
4. `/contato` — formulário funcional com validação no servidor;
5. `/404` — página de erro personalizada.

**Requisitos técnicos:**
- folha de estilo externa compartilhada por todas as páginas;
- cabeçalho de navegação com indicação visual da página corrente;
- registro no terminal de método, caminho, status e tempo de resposta;
- proteção contra *path traversal*;
- escape de todo dado dinâmico inserido no HTML;
- layout adaptável a telas de largura inferior a 600 pixels.

### Laboratório 4.2 — Rota de informações do servidor

Implementar a rota `/api/info`, consumida pelo `script.js` da página inicial. A resposta deve ser um JSON contendo:

```json
{
  "servidor": "Node.js v22.22.2",
  "plataforma": "win32",
  "tempoAtivo": "0h 12min",
  "memoriaMB": 42.7,
  "requisicoesAtendidas": 37,
  "horario": "07/09/2026 14:32:11"
}
```

O contador de requisições deve ser incrementado a cada acesso, em variável do escopo do módulo. A página inicial deve passar a exibir os dados corretamente, substituindo a mensagem de erro.

**Extensão opcional:** atualizar os dados a cada cinco segundos com `setInterval` no script do cliente.

### Laboratório 4.3 — Análise comparativa

Redigir um documento de uma a duas páginas respondendo:

1. Quantas linhas de código foram necessárias para entregar arquivos estáticos? Quais tarefas cada bloco desempenha?
2. Que ocorreria se a verificação `caminhoAbsoluto.startsWith(PASTA_PUBLICA)` fosse removida? Descrever a requisição que exploraria a falha.
3. Por que o valor de `Content-Type` não pode ser omitido? Realizar o experimento removendo o cabeçalho de um arquivo CSS e documentar o resultado com captura de tela.
4. Enumerar cinco tarefas do `servidor.js` que se espera que um *framework* automatize.

Este documento será retomado ao final do Tutorial 5, para confronto com a implementação em Express.

---

## 9. Síntese

1. O HTTP funciona por requisição e resposta; o servidor nunca inicia a comunicação.
2. O módulo `http` permite criar servidores completos, ao custo de implementar manualmente roteamento, tipos MIME e tratamento de erros.
3. O cabeçalho `Content-Type` determina a interpretação do conteúdo pelo navegador; a omissão de `charset=utf-8` corrompe a acentuação.
4. Caminhos derivados da URL devem ser validados para impedir acesso a arquivos fora da pasta pública.
5. Dados provenientes do usuário devem ser escapados antes de compor HTML, sob pena de vulnerabilidade XSS.
6. Validações do navegador não substituem validações no servidor.
7. Os códigos de status devem refletir o resultado real da operação.

**Próximo tutorial:** [Aplicação com Express e nodemon](./05-express-e-nodemon.md)
