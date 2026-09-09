# Tutorial 2 — Arquitetura do Node.js e Suas Aplicações

> **Pré-requisitos:** Tutorial 1 concluído.
> **Duração estimada:** 2 aulas de 50 minutos.
> **Ao final deste tutorial, o estudante será capaz de:** explicar por que o Node.js é considerado assíncrono e de thread única, prever a ordem de execução de operações assíncronas, distinguir operações bloqueantes de não bloqueantes, escolher entre CommonJS e ESM e identificar em que situações o Node.js é a tecnologia adequada.

---

## 1. Os componentes internos

O Node.js não é uma linguagem nem um *framework*. É um programa em C++ que integra quatro componentes:

```
┌─────────────────────────────────────────────────────────────┐
│                  APLICAÇÃO (código JavaScript)              │
├─────────────────────────────────────────────────────────────┤
│         BIBLIOTECA PADRÃO (fs, http, path, crypto, os)      │
├─────────────────────────────────────────────────────────────┤
│                    BINDINGS (ponte C++ ↔ JS)                │
├──────────────────────────┬──────────────────────────────────┤
│           V8             │            libuv                 │
│  Compila e executa       │  Event loop, thread pool,        │
│  JavaScript              │  E/S assíncrona, rede            │
├──────────────────────────┴──────────────────────────────────┤
│                    SISTEMA OPERACIONAL                      │
└─────────────────────────────────────────────────────────────┘
```

**V8** — motor JavaScript do Google. Compila o código-fonte diretamente em linguagem de máquina, sem etapa intermediária de interpretação. Gerencia a memória e o coletor de lixo. É o mesmo motor do Chrome e do Microsoft Edge.

**libuv** — biblioteca em C que fornece ao Node.js a capacidade de realizar operações de entrada e saída sem bloquear a execução. Contém o *event loop* e um conjunto de threads auxiliares (*thread pool*), com quatro threads por padrão.

**Bindings** — camada de tradução que permite ao código JavaScript invocar funções escritas em C++.

**Biblioteca padrão** — os módulos nativos (`fs`, `http`, `path`, `crypto`, `os`, `stream`, entre outros), escritos parcialmente em JavaScript e parcialmente em C++.

Verificação prática das versões de cada componente:

```javascript
// versoes.js
console.log(process.versions);
```

```
{
  node: '22.22.2',
  v8: '12.4.254.21-node.22',
  uv: '1.48.0',
  openssl: '3.0.13',
  ...
}
```

---

## 2. Uma thread, muitas conexões

### 2.1 O problema que o Node.js resolve

Servidores web tradicionais, como o Apache em sua configuração clássica, atribuem **uma thread do sistema operacional a cada conexão**. Cada thread consome cerca de 1 MB de memória e exige uma troca de contexto do processador para ser alternada. Com 10.000 conexões simultâneas, o custo torna-se proibitivo — cenário conhecido na literatura como *problema C10K*.

Observa-se, contudo, um fato decisivo: em uma aplicação web típica, a maior parte do tempo é gasta **esperando**. Espera-se pela resposta do banco de dados, pela leitura do disco, pela resposta de uma API externa. Durante essa espera, a thread permanece ociosa, mas continua ocupando memória.

O Node.js adota a estratégia oposta: **uma única thread atende a todas as conexões**. Quando uma operação de espera é iniciada, a thread não aguarda: registra o que deve ser feito quando o resultado chegar e passa imediatamente a atender outra requisição.

### 2.2 A analogia da lanchonete

Considere-se uma lanchonete com um único atendente.

**Modelo bloqueante:** o atendente recebe o pedido do primeiro cliente, dirige-se à cozinha, permanece parado observando o preparo por oito minutos, entrega o lanche e só então atende o segundo cliente. A fila cresce sem que exista trabalho efetivo sendo realizado.

**Modelo do Node.js:** o atendente recebe o pedido, repassa-o à cozinha, entrega uma senha ao cliente e imediatamente atende o próximo. Quando a cozinha sinaliza que um pedido está pronto, o atendente chama a senha correspondente. Um único atendente processa dezenas de clientes, porque não desperdiça tempo esperando.

A cozinha corresponde ao sistema operacional e ao *thread pool* do libuv; a senha corresponde ao *callback* ou à *Promise*; o atendente corresponde à thread principal do JavaScript.

**Limite do modelo:** se um cliente solicitar ao atendente que descasque cem batatas no balcão, todos os demais ficarão parados. Tarefas que exigem processamento intensivo — e não espera — bloqueiam a thread única. Esse é o principal ponto fraco do Node.js, tratado na Seção 7.

---

## 3. Demonstração de código: bloqueante versus não bloqueante

**Arquivo `bloqueio.js`:**

```javascript
// bloqueio.js — comparação entre operação bloqueante e não bloqueante
const fs = require("node:fs");

console.log("=== VERSÃO BLOQUEANTE (readFileSync) ===");
console.time("bloqueante");

// A execução permanece detida nesta linha até que o arquivo seja lido
// por completo. Nada mais ocorre no programa durante esse intervalo.
const conteudo = fs.readFileSync(__filename, "utf8");
console.log(`Arquivo lido: ${conteudo.length} caracteres`);
console.log("Esta linha só é alcançada após a leitura terminar.");

console.timeEnd("bloqueante");

console.log("\n=== VERSÃO NÃO BLOQUEANTE (readFile) ===");
console.time("nao-bloqueante");

// A leitura é delegada ao sistema. A função devolve o controle
// imediatamente, e o callback será invocado quando o dado estiver pronto.
fs.readFile(__filename, "utf8", (erro, dados) => {
  if (erro) {
    console.error("Falha na leitura:", erro.message);
    return;
  }
  console.log(`[callback] Arquivo lido: ${dados.length} caracteres`);
  console.timeEnd("nao-bloqueante");
});

console.log("Esta linha é executada ANTES do callback acima.");
console.log("A thread principal não ficou parada esperando o disco.");
```

Saída:

```
=== VERSÃO BLOQUEANTE (readFileSync) ===
Arquivo lido: 1043 caracteres
Esta linha só é alcançada após a leitura terminar.
bloqueante: 0.842ms

=== VERSÃO NÃO BLOQUEANTE (readFile) ===
Esta linha é executada ANTES do callback acima.
A thread principal não ficou parada esperando o disco.
[callback] Arquivo lido: 1043 caracteres
nao-bloqueante: 1.317ms
```

**Análise:** na versão não bloqueante, as duas mensagens finais foram impressas antes do *callback*, embora apareçam depois dele no código-fonte. A ordem do texto no arquivo não corresponde à ordem de execução. Compreender essa distinção é o núcleo da programação assíncrona.

---

## 4. O event loop

O *event loop* é um laço infinito, implementado pelo libuv, que verifica continuamente se há trabalho pendente a ser executado. Ele organiza esse trabalho em **fases**, percorridas em ordem fixa a cada volta (denominada *tick*):

```
   ┌───────────────────────────┐
┌─▶│           timers          │  callbacks de setTimeout e setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │     pending callbacks     │  callbacks de erros de E/S do sistema
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │       idle, prepare       │  uso interno do Node.js
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │           poll            │  aguarda e processa eventos de E/S
│  └─────────────┬─────────────┘     (leitura de arquivos, rede)
│  ┌─────────────▼─────────────┐
│  │           check           │  callbacks de setImmediate
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
└──┤      close callbacks      │  eventos de fechamento (socket.on('close'))
   └───────────────────────────┘
```

Entre **cada** fase, o Node.js esvazia duas filas prioritárias, chamadas **microtarefas**:

1. a fila de `process.nextTick()`;
2. a fila de *Promises* (`.then`, `.catch`, `.finally` e `await`).

Essas filas têm precedência sobre qualquer fase do *event loop*.

### 4.1 Experimento de ordenação

**Arquivo `ordem.js`:**

```javascript
// ordem.js — demonstração da ordem de execução das filas
console.log("1. Síncrono: início");

setTimeout(() => console.log("6. setTimeout 0ms (fase timers)"), 0);

setImmediate(() => console.log("7. setImmediate (fase check)"));

Promise.resolve().then(() => console.log("5. Promise (microtarefa)"));

process.nextTick(() => console.log("4. process.nextTick (prioridade máxima)"));

// Uma função assíncrona executa de forma síncrona até encontrar o primeiro
// await. Tudo o que vem depois do await torna-se uma microtarefa.
(async function () {
  console.log("2. Síncrono: dentro da função async, antes do await");
  await null;
  console.log("5b. Após o await (também é microtarefa)");
})();

console.log("3. Síncrono: fim");
```

Saída:

```
1. Síncrono: início
2. Síncrono: dentro da função async, antes do await
3. Síncrono: fim
4. process.nextTick (prioridade máxima)
5. Promise (microtarefa)
5b. Após o await (também é microtarefa)
6. setTimeout 0ms (fase timers)
7. setImmediate (fase check)
```

**Regras extraídas do experimento:**

1. Todo o código **síncrono** executa primeiro, até o fim.
2. Em seguida, a fila de `process.nextTick`.
3. Em seguida, a fila de *Promises*.
4. Somente então o *event loop* avança para as fases: `timers` (`setTimeout`), `poll` (E/S), `check` (`setImmediate`).
5. O código posterior a um `await` é, para todos os efeitos, o conteúdo de um `.then()`: é uma microtarefa.

**Advertência sobre `process.nextTick`:** por ter prioridade absoluta, o uso recursivo desse recurso impede que o *event loop* avance, deixando a aplicação sem resposta. Em código de aplicação, recomenda-se `setImmediate`.

### 4.2 Simulação de operações concorrentes

**Arquivo `concorrencia.js`:**

```javascript
// concorrencia.js — três operações lentas executadas em paralelo
// setTimeout simula a latência de um banco de dados ou de uma API externa.

function operacaoLenta(nome, duracaoMs) {
  console.log(`  → ${nome} iniciada`);
  return new Promise((resolver) => {
    setTimeout(() => {
      console.log(`  ← ${nome} concluída após ${duracaoMs}ms`);
      resolver(`resultado de ${nome}`);
    }, duracaoMs);
  });
}

async function emSequencia() {
  console.log("\n[A] EXECUÇÃO SEQUENCIAL (cada await aguarda o anterior)");
  console.time("sequencial");
  await operacaoLenta("consulta ao banco", 800);
  await operacaoLenta("chamada à API externa", 600);
  await operacaoLenta("leitura de arquivo", 400);
  console.timeEnd("sequencial");
}

async function emParalelo() {
  console.log("\n[B] EXECUÇÃO PARALELA (as três iniciam simultaneamente)");
  console.time("paralelo");
  // Promise.all recebe promises já iniciadas e aguarda a conclusão de todas.
  const resultados = await Promise.all([
    operacaoLenta("consulta ao banco", 800),
    operacaoLenta("chamada à API externa", 600),
    operacaoLenta("leitura de arquivo", 400),
  ]);
  console.timeEnd("paralelo");
  console.log("  Resultados:", resultados);
}

(async () => {
  await emSequencia();
  await emParalelo();

  console.log("\nConclusão: a versão paralela consome o tempo da operação");
  console.log("mais lenta (800ms), e não a soma das três (1800ms).");
})();
```

Saída resumida:

```
[A] EXECUÇÃO SEQUENCIAL
  → consulta ao banco iniciada
  ← consulta ao banco concluída após 800ms
  → chamada à API externa iniciada
  ← chamada à API externa concluída após 600ms
  → leitura de arquivo iniciada
  ← leitura de arquivo concluída após 400ms
sequencial: 1810.4ms

[B] EXECUÇÃO PARALELA
  → consulta ao banco iniciada
  → chamada à API externa iniciada
  → leitura de arquivo iniciada
  ← leitura de arquivo concluída após 400ms
  ← chamada à API externa concluída após 600ms
  ← consulta ao banco concluída após 800ms
paralelo: 803.1ms
```

**Aplicação prática:** ao construir uma página que necessita de três consultas independentes ao banco de dados, o uso de `await` sequencial triplica desnecessariamente o tempo de resposta. `Promise.all` é a construção adequada quando as operações não dependem umas das outras.

Variantes úteis:

| Método | Comportamento |
|---|---|
| `Promise.all` | Falha inteiramente se qualquer promise for rejeitada |
| `Promise.allSettled` | Aguarda todas e informa o resultado individual de cada uma |
| `Promise.race` | Resolve com a primeira que terminar, seja êxito ou falha |
| `Promise.any` | Resolve com a primeira que obtiver êxito |

---

## 5. As três formas de escrever código assíncrono

O Node.js acumulou três estilos ao longo de sua história. Todos permanecem em uso, e a leitura de código alheio exige familiaridade com os três.

**Arquivo `estilos.js`:**

```javascript
// estilos.js — três formas de expressar a mesma operação assíncrona
const fs = require("node:fs");                 // API de callbacks
const fsPromises = require("node:fs/promises"); // API de promises

const ARQUIVO = __filename;

// ---------- ESTILO 1: CALLBACKS (2009) ----------
// Convenção "error-first": o primeiro parâmetro é sempre o erro.
function comCallback() {
  fs.readFile(ARQUIVO, "utf8", (erro, dados) => {
    if (erro) {
      console.error("[callback] erro:", erro.message);
      return;
    }
    console.log(`[callback] ${dados.split("\n").length} linhas`);
  });
}

// ---------- ESTILO 2: PROMISES (2015) ----------
function comPromise() {
  fsPromises
    .readFile(ARQUIVO, "utf8")
    .then((dados) => console.log(`[promise] ${dados.split("\n").length} linhas`))
    .catch((erro) => console.error("[promise] erro:", erro.message));
}

// ---------- ESTILO 3: ASYNC/AWAIT (2017) — recomendado ----------
async function comAsyncAwait() {
  try {
    const dados = await fsPromises.readFile(ARQUIVO, "utf8");
    console.log(`[async/await] ${dados.split("\n").length} linhas`);
  } catch (erro) {
    console.error("[async/await] erro:", erro.message);
  }
}

comCallback();
comPromise();
comAsyncAwait();
```

### 5.1 O problema do aninhamento de callbacks

O encadeamento de operações no estilo de *callbacks* produz o padrão conhecido como *callback hell*:

```javascript
// ANTIPADRÃO — apresentado apenas para reconhecimento
lerUsuario(id, (e1, usuario) => {
  if (e1) return tratar(e1);
  lerPedidos(usuario.id, (e2, pedidos) => {
    if (e2) return tratar(e2);
    lerItens(pedidos[0].id, (e3, itens) => {
      if (e3) return tratar(e3);
      calcularTotal(itens, (e4, total) => {
        if (e4) return tratar(e4);
        console.log(total);
      });
    });
  });
});
```

A mesma lógica com `async/await`:

```javascript
// FORMA RECOMENDADA
try {
  const usuario = await lerUsuario(id);
  const pedidos = await lerPedidos(usuario.id);
  const itens = await lerItens(pedidos[0].id);
  const total = await calcularTotal(itens);
  console.log(total);
} catch (erro) {
  tratar(erro);
}
```

O código assíncrono passa a ser lido de cima para baixo, e um único bloco `try/catch` trata todas as falhas. **Todos os exemplos dos tutoriais seguintes utilizam `async/await`.**

---

## 6. Sistemas de módulos: CommonJS e ESM

O Node.js suporta dois sistemas de módulos. A distinção é fonte constante de erros e precisa ser compreendida.

### 6.1 CommonJS (CJS)

Sistema histórico do Node.js. É o padrão quando o `package.json` **não** contém `"type": "module"`.

```javascript
// matematica.js
function somar(a, b) {
  return a + b;
}
const PI = 3.14159;

module.exports = { somar, PI };
```

```javascript
// principal.js
const { somar, PI } = require("./matematica");
console.log(somar(2, 3), PI);
```

Características: carregamento síncrono; o caminho do módulo pode ser calculado em tempo de execução; as variáveis `__dirname` e `__filename` estão disponíveis.

### 6.2 ECMAScript Modules (ESM)

Padrão oficial da linguagem, idêntico ao utilizado no navegador. É ativado por `"type": "module"` no `package.json` ou pela extensão `.mjs`.

```javascript
// matematica.js
export function somar(a, b) {
  return a + b;
}
export const PI = 3.14159;
export default { somar, PI };
```

```javascript
// principal.js
import { somar, PI } from "./matematica.js";  // a extensão .js é obrigatória
console.log(somar(2, 3), PI);
```

Características: análise estática das importações; suporte a `await` no nível superior do arquivo; `__dirname` e `__filename` **não** existem.

Reconstrução de `__dirname` em ESM:

```javascript
import { fileURLToPath } from "node:url";
import path from "node:path";

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

// A partir do Node.js 20.11, existe alternativa mais direta:
// const __dirname = import.meta.dirname;
```

### 6.3 Quadro comparativo

| Aspecto | CommonJS | ESM |
|---|---|---|
| Importação | `require()` | `import` |
| Exportação | `module.exports` | `export` |
| Ativação | padrão | `"type": "module"` ou `.mjs` |
| Extensão obrigatória | não | sim |
| `__dirname` | disponível | `import.meta.dirname` |
| `await` no nível superior | não | sim |
| Importa CJS? | — | sim |
| Importa ESM? | apenas via `import()` | — |

**Decisão para esta série:** os Tutoriais 3 a 7 utilizam **CommonJS**, por ser o formato predominante na maior parte da documentação e dos exemplos existentes. O Tutorial 9 utiliza **ESM**, que é o padrão atual para publicação de novos pacotes.

---

## 7. Limitações: quando o Node.js não é adequado

A thread única é vantajosa apenas para cargas dominadas por espera. Tarefas de processamento intensivo bloqueiam a aplicação inteira.

**Arquivo `bloqueio-cpu.js`:**

```javascript
// bloqueio-cpu.js — demonstração de bloqueio por uso intensivo do processador
const http = require("node:http");

function calculoPesado(n) {
  let soma = 0;
  for (let i = 0; i < n; i++) soma += Math.sqrt(i);
  return soma;
}

const servidor = http.createServer((requisicao, resposta) => {
  if (requisicao.url === "/rapido") {
    resposta.end("Resposta imediata\n");
  } else if (requisicao.url === "/lento") {
    // Durante os segundos consumidos por este laço, o servidor NÃO responde
    // a nenhuma outra requisição, inclusive a /rapido.
    const resultado = calculoPesado(5_000_000_000);
    resposta.end(`Resultado: ${resultado}\n`);
  } else {
    resposta.end("Rotas disponíveis: /rapido e /lento\n");
  }
});

servidor.listen(3000, () => {
  console.log("Servidor em http://localhost:3000");
  console.log("Teste: abrir /lento em uma aba e, em seguida, /rapido em outra.");
});
```

**Experimento:** acessar `http://localhost:3000/lento` em uma aba e, imediatamente, `http://localhost:3000/rapido` em outra. A segunda aba permanece carregando até que a primeira conclua. Uma única requisição paralisou o servidor.

### 7.1 Soluções para carga de processador

**Worker Threads** — executam JavaScript em threads paralelas reais:

```javascript
// principal.js
const { Worker } = require("node:worker_threads");

function executarEmThread(numero) {
  return new Promise((resolver, rejeitar) => {
    const trabalhador = new Worker("./trabalhador.js", {
      workerData: numero,
    });
    trabalhador.on("message", resolver);
    trabalhador.on("error", rejeitar);
  });
}

(async () => {
  console.time("worker");
  const resultado = await executarEmThread(1_000_000_000);
  console.timeEnd("worker");
  console.log("Resultado:", resultado);
  console.log("A thread principal permaneceu livre durante o cálculo.");
})();
```

```javascript
// trabalhador.js
const { workerData, parentPort } = require("node:worker_threads");

let soma = 0;
for (let i = 0; i < workerData; i++) soma += Math.sqrt(i);

parentPort.postMessage(soma);
```

**Cluster** — replica o processo do servidor, um por núcleo do processador:

```javascript
// cluster-servidor.js
const cluster = require("node:cluster");
const os = require("node:os");
const http = require("node:http");

if (cluster.isPrimary) {
  const nucleos = os.cpus().length;
  console.log(`Processo primário ${process.pid}: criando ${nucleos} processos.`);

  for (let i = 0; i < nucleos; i++) cluster.fork();

  cluster.on("exit", (trabalhador) => {
    console.log(`Processo ${trabalhador.process.pid} encerrado. Recriando.`);
    cluster.fork();
  });
} else {
  http
    .createServer((_requisicao, resposta) => {
      resposta.end(`Atendido pelo processo ${process.pid}\n`);
    })
    .listen(3000);

  console.log(`Processo trabalhador ${process.pid} iniciado.`);
}
```

Atualizações sucessivas de `http://localhost:3000` apresentam identificadores de processo distintos, evidenciando a distribuição de carga.

### 7.2 Critérios de adequação

| Adequado ao Node.js | Inadequado ao Node.js |
|---|---|
| APIs REST e GraphQL | Processamento de vídeo e áudio |
| Aplicações em tempo real (chat, notificações) | Treinamento de modelos de aprendizado de máquina |
| Microsserviços | Cálculo numérico de alta intensidade |
| Servidores intermediários (*BFF*) | Compressão de grandes volumes de dados |
| Ferramentas de linha de comando | Renderização gráfica |
| Automação e *scraping* | |
| Aplicações de desktop (Electron) | |

Para as tarefas da coluna direita, linguagens como Python, Go, Rust, C++ e Java oferecem modelos de concorrência mais apropriados. Uma arquitetura frequente combina ambos: o Node.js atende às requisições HTTP e delega o processamento pesado a serviços especializados.

---

## 8. Aplicações do Node.js na prática

**Servidores web e APIs.** É a aplicação predominante. Netflix, PayPal, LinkedIn, Uber e Walmart utilizam Node.js em suas camadas de serviço. O relato público do PayPal registra redução do tempo médio de resposta após a migração de Java para Node.js na camada de aplicação.

**Aplicações em tempo real.** Chats, editores colaborativos, painéis de monitoramento e notificações instantâneas beneficiam-se do modelo orientado a eventos. A biblioteca `socket.io` implementa comunicação bidirecional sobre WebSocket.

**Ferramentas de linha de comando.** O próprio npm é escrito em Node.js, assim como `eslint`, `prettier`, `vite` e `typescript`.

**Aplicações de desktop.** O Electron combina Node.js com o motor de renderização do Chromium. O Visual Studio Code, o Discord, o Slack, o Figma Desktop e o WhatsApp Desktop são construídos dessa forma.

**Automação e integração.** Scripts de organização de arquivos, extração de dados de páginas web (`puppeteer`), integração entre sistemas e rotinas agendadas.

**Internet das Coisas.** O Node.js executa em placas como Raspberry Pi, e a biblioteca `johnny-five` permite controlar sensores e atuadores.

---

## 9. Laboratório prático

### Laboratório 2.1 — Previsão da ordem de execução

Analisar o código a seguir e **registrar em papel a ordem prevista de impressão antes de executá-lo**. Em seguida, executar e justificar cada divergência.

```javascript
// laboratorio-ordem.js
const fs = require("node:fs");

console.log("A");

setTimeout(() => {
  console.log("B");
  process.nextTick(() => console.log("C"));
  Promise.resolve().then(() => console.log("D"));
}, 0);

fs.readFile(__filename, () => {
  console.log("E");
  setImmediate(() => console.log("F"));
});

Promise.resolve().then(() => {
  console.log("G");
  setTimeout(() => console.log("H"), 0);
});

process.nextTick(() => console.log("I"));

console.log("J");
```

**Questões:**
1. Por que `I` é impresso antes de `G`?
2. Por que `E` é impresso depois de `B`, ainda que a leitura do arquivo tenha sido iniciada antes do `setTimeout` de 0 ms?
3. Em qual fase do *event loop* o *callback* `F` é executado?

### Laboratório 2.2 — Medição de ganho com paralelismo

Implementar `laboratorio-paralelo.js` que:

1. Defina `buscarDados(nome, atrasoMs)`, devolvendo uma *Promise* resolvida após o atraso indicado, com o objeto `{ nome, atrasoMs }`;
2. Execute cinco chamadas sequencialmente com `await` e meça o tempo total;
3. Execute as mesmas cinco chamadas com `Promise.all` e meça o tempo total;
4. Repita com `Promise.allSettled`, incluindo uma chamada que rejeite, e demonstre que as demais não são afetadas;
5. Apresente uma tabela comparativa com os tempos e o percentual de redução.

Atrasos sugeridos: 300, 500, 200, 700 e 400 milissegundos.

**Questão:** qual seria o tempo teórico mínimo da execução paralela? O tempo medido corresponde a esse valor? Justificar eventual diferença.

### Laboratório 2.3 — Evidência do bloqueio

1. Executar `bloqueio-cpu.js` (Seção 7).
2. Documentar, com capturas de tela, a rota `/rapido` permanecendo sem resposta enquanto `/lento` é processada.
3. Reimplementar a rota `/lento` delegando o cálculo a um *Worker Thread*.
4. Repetir o experimento e documentar que `/rapido` passa a responder normalmente.
5. Redigir um parágrafo explicando a diferença observada.

---

## 10. Síntese

1. O Node.js combina o motor V8 com a biblioteca libuv, que fornece o *event loop* e a E/S assíncrona.
2. O JavaScript executa em uma única thread; as operações de espera são delegadas ao sistema operacional e ao *thread pool*.
3. O *event loop* percorre fases em ordem fixa; entre elas, esvazia as filas de `process.nextTick` e de *Promises*, que possuem prioridade.
4. Operações independentes devem ser executadas com `Promise.all`, e não com `await` sequencial.
5. `async/await` é o estilo recomendado; *callbacks* e *Promises* encadeadas permanecem presentes em código existente.
6. Tarefas de processamento intensivo bloqueiam a thread única e exigem `worker_threads` ou `cluster`.
7. CommonJS e ESM coexistem; a escolha é determinada pelo campo `type` do `package.json`.

**Próximo tutorial:** [Sistema de arquivos e recursos do Windows](./03-sistema-de-arquivos-e-windows.md)
