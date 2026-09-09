# Tutorial 3 — Sistema de Arquivos e Recursos do Windows

> **Pré-requisitos:** Tutoriais 1 e 2 concluídos.
> **Duração estimada:** 3 aulas de 50 minutos.
> **Ao final deste tutorial, o estudante será capaz de:** ler, escrever, mover e excluir arquivos e diretórios; manipular caminhos de forma independente de plataforma; processar arquivos grandes com *streams*; obter informações do hardware e do sistema; executar comandos do Windows a partir do Node.js; e implementar um utilitário de organização automática de pastas.

---

## 1. Advertência inicial

Os programas deste tutorial **modificam arquivos do disco**. Um erro de digitação em um caminho pode excluir dados relevantes.

**Regras de segurança obrigatórias:**

1. Trabalhar exclusivamente dentro de uma pasta de testes criada para este fim, por exemplo `C:\Users\<usuario>\projetos\lab-arquivos`.
2. Jamais executar os exemplos apontando para `C:\Windows`, `C:\Program Files` ou para a raiz do disco.
3. Antes de qualquer operação de exclusão ou movimentação em massa, executar o programa em **modo de simulação**, que imprime as ações sem realizá-las. O Laboratório 3.2 implementa esse modo.

---

## 2. Caminhos: o módulo `path`

### 2.1 O problema das barras

O Windows utiliza a contrabarra (`\`) como separador de diretórios; Linux e macOS utilizam a barra (`/`). Como, em JavaScript, a contrabarra inicia uma sequência de escape, a construção manual de caminhos é propensa a erros:

```javascript
// INCORRETO — "\n" é interpretado como quebra de linha
const caminho = "C:\Users\novo\arquivo.txt";

// Correto, porém dependente de plataforma
const caminho2 = "C:\\Users\\novo\\arquivo.txt";

// RECOMENDADO — o módulo path aplica o separador correto da plataforma
const path = require("node:path");
const caminho3 = path.join("C:", "Users", "novo", "arquivo.txt");
```

O Windows aceita também a barra normal em quase todos os contextos, mas a utilização de `path` elimina a necessidade de decidir caso a caso.

### 2.2 Funções principais

**Arquivo `caminhos.js`:**

```javascript
// caminhos.js — manipulação de caminhos independente de plataforma
const path = require("node:path");

const arquivo = "C:\\Users\\aluno\\projetos\\relatorio-final.pdf";

console.log("Caminho completo :", arquivo);
console.log("Diretório        :", path.dirname(arquivo));
console.log("Nome com extensão:", path.basename(arquivo));
console.log("Nome sem extensão:", path.basename(arquivo, path.extname(arquivo)));
console.log("Extensão         :", path.extname(arquivo));
console.log("Separador da plataforma:", JSON.stringify(path.sep));

// join concatena segmentos e normaliza o resultado
console.log("\njoin :", path.join("pasta", "subpasta", "..", "arquivo.txt"));
// Resultado no Windows: pasta\arquivo.txt

// resolve produz sempre um caminho absoluto, partindo da pasta atual
console.log("resolve:", path.resolve("dados", "entrada.csv"));

// parse decompõe o caminho em um objeto
console.log("\nparse:", path.parse(arquivo));

// format realiza a operação inversa
console.log("format:", path.format({
  dir: "C:\\Users\\aluno\\documentos",
  name: "contrato",
  ext: ".docx",
}));

// Variáveis disponíveis em módulos CommonJS
console.log("\n__dirname :", __dirname);   // pasta do arquivo em execução
console.log("__filename:", __filename);    // caminho completo do arquivo
console.log("process.cwd():", process.cwd()); // pasta onde o comando foi digitado
```

### 2.3 Distinção fundamental entre `__dirname` e `process.cwd()`

- `__dirname` — pasta **onde o arquivo de código está armazenado**;
- `process.cwd()` — pasta **de onde o comando `node` foi executado**.

Considere-se um arquivo em `C:\projeto\src\app.js`, executado da seguinte forma:

```powershell
cd C:\projeto
node src\app.js
```

Nesse caso, `__dirname` vale `C:\projeto\src` e `process.cwd()` vale `C:\projeto`.

**Regra prática:** ao referenciar arquivos que pertencem ao projeto (modelos, configurações, recursos estáticos), utilizar sempre `path.join(__dirname, ...)`. O uso de caminhos relativos simples torna o programa dependente da pasta a partir da qual foi invocado — causa recorrente de erros `ENOENT` que se manifestam apenas em produção.

---

## 3. O módulo `fs`

O módulo `fs` (*file system*) oferece três interfaces:

| Interface | Importação | Uso recomendado |
|---|---|---|
| Promises | `require("node:fs/promises")` | Padrão para código de aplicação |
| Callbacks | `require("node:fs")` | Código legado |
| Síncrona | `fs.readFileSync` etc. | Scripts simples e inicialização de servidores |

Este tutorial utiliza predominantemente a interface de *promises*.

### 3.1 Leitura e escrita

**Arquivo `arquivos-basico.js`:**

```javascript
// arquivos-basico.js — operações fundamentais de leitura e escrita
const fs = require("node:fs/promises");
const path = require("node:path");

// Todos os arquivos serão criados em uma subpasta "dados" do projeto.
const PASTA = path.join(__dirname, "dados");

async function principal() {
  // 1) Criação da pasta. A opção recursive evita erro caso já exista
  //    e cria também as pastas intermediárias ausentes.
  await fs.mkdir(PASTA, { recursive: true });
  console.log("Pasta preparada:", PASTA);

  // 2) Escrita de arquivo de texto. writeFile SUBSTITUI o conteúdo existente.
  const arquivoTexto = path.join(PASTA, "notas.txt");
  await fs.writeFile(arquivoTexto, "Ana;8.5\nBruno;7.0\n", "utf8");
  console.log("Arquivo criado:", arquivoTexto);

  // 3) Acréscimo ao final, sem apagar o conteúdo anterior.
  await fs.appendFile(arquivoTexto, "Carla;9.2\nDaniel;6.4\n", "utf8");

  // 4) Leitura completa.
  const conteudo = await fs.readFile(arquivoTexto, "utf8");
  console.log("\nConteúdo lido:\n" + conteudo);

  // 5) Processamento: conversão do texto em estrutura de dados.
  const alunos = conteudo
    .split("\n")
    .filter((linha) => linha.trim() !== "")
    .map((linha) => {
      const [nome, nota] = linha.split(";");
      return { nome, nota: Number(nota) };
    });

  const media = alunos.reduce((s, a) => s + a.nota, 0) / alunos.length;
  console.log(`Média da turma: ${media.toFixed(2)}`);
  const aprovados = alunos.filter((a) => a.nota >= 7).map((a) => a.nome);
  console.log("Aprovados:", aprovados.join(", "));

  // 6) Persistência em JSON. JSON.stringify com o terceiro argumento
  //    igual a 2 produz saída indentada e legível.
  const arquivoJson = path.join(PASTA, "alunos.json");
  await fs.writeFile(arquivoJson, JSON.stringify(alunos, null, 2), "utf8");
  console.log("\nJSON gravado em:", arquivoJson);

  // 7) Leitura e reconversão do JSON.
  const bruto = await fs.readFile(arquivoJson, "utf8");
  const recuperado = JSON.parse(bruto);
  console.log("Registros recuperados:", recuperado.length);
}

// Toda função assíncrona chamada no nível superior deve ter o erro tratado.
principal().catch((erro) => {
  console.error("Falha na execução:", erro.message);
  process.exitCode = 1;
});
```

**Advertência sobre codificação:** ao omitir o parâmetro `"utf8"`, `readFile` devolve um objeto `Buffer` — sequência bruta de bytes — em vez de texto. O `Buffer` é apropriado para arquivos binários (imagens, PDFs); para texto, a codificação deve ser sempre informada.

### 3.2 Verificação de existência

```javascript
const fs = require("node:fs/promises");

// FORMA RECOMENDADA: tentar a operação e tratar a falha.
async function lerConfiguracao(caminho) {
  try {
    return JSON.parse(await fs.readFile(caminho, "utf8"));
  } catch (erro) {
    if (erro.code === "ENOENT") {
      console.warn("Arquivo inexistente. Aplicando configuração padrão.");
      return { porta: 3000, ambiente: "desenvolvimento" };
    }
    throw erro; // qualquer outro erro deve ser propagado
  }
}

// Verificação explícita, quando necessária.
async function existe(caminho) {
  try {
    await fs.access(caminho);
    return true;
  } catch {
    return false;
  }
}
```

**Observação:** verificar a existência e, em seguida, abrir o arquivo introduz uma condição de corrida — o arquivo pode ser removido entre as duas operações. A prática recomendada é tentar a operação diretamente e tratar a exceção.

### 3.3 Códigos de erro relevantes

| Código | Significado | Situação típica |
|---|---|---|
| `ENOENT` | Arquivo ou diretório inexistente | Caminho incorreto |
| `EACCES` | Permissão negada | Arquivo protegido pelo sistema |
| `EEXIST` | Já existe | `mkdir` sem `recursive: true` |
| `EISDIR` | É um diretório | Tentativa de ler pasta como arquivo |
| `ENOTDIR` | Não é um diretório | Segmento intermediário é arquivo |
| `EPERM` | Operação não permitida | Arquivo em uso por outro programa |
| `ENOSPC` | Sem espaço em disco | Disco cheio |

---

## 4. Diretórios

**Arquivo `diretorios.js`:**

```javascript
// diretorios.js — criação, listagem e percurso recursivo de diretórios
const fs = require("node:fs/promises");
const path = require("node:path");

const RAIZ = path.join(__dirname, "estrutura");

// Formata um tamanho em bytes para a unidade mais legível.
function formatarTamanho(bytes) {
  const unidades = ["B", "KB", "MB", "GB"];
  let valor = bytes;
  let indice = 0;
  while (valor >= 1024 && indice < unidades.length - 1) {
    valor /= 1024;
    indice++;
  }
  return `${valor.toFixed(indice === 0 ? 0 : 1)} ${unidades[indice]}`;
}

async function criarEstrutura() {
  // Uma única chamada cria toda a hierarquia de pastas.
  await fs.mkdir(path.join(RAIZ, "documentos", "2026"), { recursive: true });
  await fs.mkdir(path.join(RAIZ, "imagens"), { recursive: true });

  await fs.writeFile(path.join(RAIZ, "leiame.txt"), "Estrutura de teste.\n");
  await fs.writeFile(
    path.join(RAIZ, "documentos", "contrato.txt"),
    "Cláusula primeira.\n".repeat(200)
  );
  await fs.writeFile(
    path.join(RAIZ, "documentos", "2026", "plano.md"),
    "# Plano de ensino\n\n- Unidade 1\n- Unidade 2\n"
  );
  await fs.writeFile(path.join(RAIZ, "imagens", "logo.svg"), "<svg></svg>");
}

// Percorre a árvore de diretórios e imprime uma representação hierárquica.
// withFileTypes evita uma chamada adicional a stat apenas para saber se o
// item é pasta ou arquivo.
async function listarArvore(diretorio, prefixo = "") {
  const itens = await fs.readdir(diretorio, { withFileTypes: true });

  // Diretórios são apresentados antes dos arquivos, em ordem alfabética.
  itens.sort((a, b) => {
    if (a.isDirectory() !== b.isDirectory()) return a.isDirectory() ? -1 : 1;
    return a.name.localeCompare(b.name, "pt-BR");
  });

  for (let i = 0; i < itens.length; i++) {
    const item = itens[i];
    const ultimo = i === itens.length - 1;
    const conector = ultimo ? "└── " : "├── ";
    const caminho = path.join(diretorio, item.name);

    if (item.isDirectory()) {
      console.log(`${prefixo}${conector}📁 ${item.name}`);
      await listarArvore(caminho, prefixo + (ultimo ? "    " : "│   "));
    } else {
      const info = await fs.stat(caminho);
      console.log(
        `${prefixo}${conector}📄 ${item.name} (${formatarTamanho(info.size)})`
      );
    }
  }
}

// Acumula estatísticas da árvore inteira.
async function estatisticas(diretorio) {
  let pastas = 0;
  let arquivos = 0;
  let bytes = 0;
  const porExtensao = {};

  async function percorrer(atual) {
    for (const item of await fs.readdir(atual, { withFileTypes: true })) {
      const caminho = path.join(atual, item.name);
      if (item.isDirectory()) {
        pastas++;
        await percorrer(caminho);
      } else {
        arquivos++;
        const info = await fs.stat(caminho);
        bytes += info.size;
        const ext = path.extname(item.name).toLowerCase() || "(sem extensão)";
        porExtensao[ext] = (porExtensao[ext] || 0) + 1;
      }
    }
  }

  await percorrer(diretorio);
  return { pastas, arquivos, bytes, porExtensao };
}

async function principal() {
  await criarEstrutura();

  console.log(`\n📁 ${path.basename(RAIZ)}`);
  await listarArvore(RAIZ);

  const dados = await estatisticas(RAIZ);
  console.log("\n--- ESTATÍSTICAS ---");
  console.log(`Pastas   : ${dados.pastas}`);
  console.log(`Arquivos : ${dados.arquivos}`);
  console.log(`Tamanho  : ${formatarTamanho(dados.bytes)}`);
  console.log("Por extensão:");
  for (const [ext, quantidade] of Object.entries(dados.porExtensao)) {
    console.log(`  ${ext.padEnd(16, ".")} ${quantidade}`);
  }
}

principal().catch(console.error);
```

Saída esperada:

```
📁 estrutura
├── 📁 documentos
│   ├── 📁 2026
│   │   └── 📄 plano.md (43 B)
│   └── 📄 contrato.txt (3.7 KB)
├── 📁 imagens
│   └── 📄 logo.svg (11 B)
└── 📄 leiame.txt (20 B)

--- ESTATÍSTICAS ---
Pastas   : 3
Arquivos : 4
Tamanho  : 3.8 KB
Por extensão:
  .md .............. 1
  .txt ............. 2
  .svg ............. 1
```

### 4.1 Cópia, movimentação e exclusão

```javascript
const fs = require("node:fs/promises");

// Cópia de arquivo
await fs.copyFile("origem.txt", "destino.txt");

// Cópia recursiva de diretório inteiro
await fs.cp("pasta-origem", "pasta-destino", { recursive: true });

// Renomeação ou movimentação (mesma operação)
await fs.rename("antigo.txt", "novo.txt");
await fs.rename("arquivo.txt", "subpasta/arquivo.txt");

// Exclusão de arquivo
await fs.unlink("descartavel.txt");

// Exclusão de diretório e todo o seu conteúdo.
// ATENÇÃO: operação irreversível, sem envio à Lixeira.
await fs.rm("pasta-temporaria", { recursive: true, force: true });
```

**Limitação de `fs.rename` no Windows:** a movimentação entre unidades diferentes (de `C:` para `D:`) falha com o erro `EXDEV`. Nesse caso, deve-se copiar e, em seguida, excluir:

```javascript
async function mover(origem, destino) {
  try {
    await fs.rename(origem, destino);
  } catch (erro) {
    if (erro.code !== "EXDEV") throw erro;
    await fs.copyFile(origem, destino);
    await fs.unlink(origem);
  }
}
```

---

## 5. Streams: processamento de arquivos grandes

A função `readFile` carrega o arquivo **inteiro** na memória. Um arquivo de 2 GB exige 2 GB de memória — comportamento inadmissível em um servidor.

Os **streams** processam o arquivo em blocos (*chunks*), tipicamente de 64 KB, mantendo o consumo de memória constante independentemente do tamanho do arquivo.

**Arquivo `streams.js`:**

```javascript
// streams.js — leitura eficiente de arquivos grandes
const fs = require("node:fs");
const fsp = require("node:fs/promises");
const readline = require("node:readline");
const path = require("node:path");
const zlib = require("node:zlib");
const { pipeline } = require("node:stream/promises");

const ARQUIVO = path.join(__dirname, "registros.log");

// Gera um arquivo de teste com 200.000 linhas.
async function gerarArquivo() {
  const niveis = ["INFO", "AVISO", "ERRO", "DEPURACAO"];
  const fluxo = fs.createWriteStream(ARQUIVO);

  for (let i = 1; i <= 200_000; i++) {
    const nivel = niveis[i % niveis.length];
    const linha = `2026-03-15T10:00:00Z [${nivel}] Evento numero ${i}\n`;

    // write devolve false quando o buffer interno está cheio.
    // Nesse caso, aguarda-se o evento "drain" antes de continuar,
    // impedindo o crescimento descontrolado da memória.
    if (!fluxo.write(linha)) {
      await new Promise((resolver) => fluxo.once("drain", resolver));
    }
  }

  fluxo.end();
  await new Promise((resolver) => fluxo.once("finish", resolver));

  const info = await fsp.stat(ARQUIVO);
  console.log(`Arquivo gerado: ${(info.size / 1024 / 1024).toFixed(2)} MB`);
}

// Processa o arquivo linha a linha, sem carregá-lo integralmente.
async function analisar() {
  const contadores = { INFO: 0, AVISO: 0, ERRO: 0, DEPURACAO: 0 };
  let totalLinhas = 0;

  const leitor = readline.createInterface({
    input: fs.createReadStream(ARQUIVO, { encoding: "utf8" }),
    crlfDelay: Infinity, // trata corretamente as quebras de linha do Windows
  });

  console.time("análise");
  for await (const linha of leitor) {
    totalLinhas++;
    const correspondencia = linha.match(/\[(\w+)\]/);
    if (correspondencia && correspondencia[1] in contadores) {
      contadores[correspondencia[1]]++;
    }
  }
  console.timeEnd("análise");

  console.log(`Linhas processadas: ${totalLinhas.toLocaleString("pt-BR")}`);
  console.table(contadores);

  const memoria = process.memoryUsage().heapUsed / 1024 / 1024;
  console.log(`Memória utilizada: ${memoria.toFixed(1)} MB`);
}

// Compacta o arquivo encadeando três streams.
async function compactar() {
  const destino = ARQUIVO + ".gz";

  // pipeline conecta os streams, propaga erros e libera os recursos
  // automaticamente, mesmo em caso de falha.
  await pipeline(
    fs.createReadStream(ARQUIVO),
    zlib.createGzip(),
    fs.createWriteStream(destino)
  );

  const original = (await fsp.stat(ARQUIVO)).size;
  const compactado = (await fsp.stat(destino)).size;
  const reducao = (1 - compactado / original) * 100;

  console.log(`\nOriginal   : ${(original / 1024 / 1024).toFixed(2)} MB`);
  console.log(`Compactado : ${(compactado / 1024 / 1024).toFixed(2)} MB`);
  console.log(`Redução    : ${reducao.toFixed(1)}%`);
}

(async () => {
  await gerarArquivo();
  await analisar();
  await compactar();
})().catch(console.error);
```

**Resultado observado:** um arquivo de aproximadamente 10 MB é integralmente analisado com consumo de memória inferior a 15 MB. Caso fosse utilizado `readFile`, o consumo cresceria proporcionalmente ao tamanho do arquivo.

**Regra de decisão:** arquivos de configuração e pequenos JSON podem ser lidos com `readFile`; registros de log, exportações CSV, mídias e uploads devem ser processados com *streams*.

---

## 6. Recursos e funcionalidades do Windows

### 6.1 Informações do sistema — módulo `os`

```javascript
// sistema.js
const os = require("node:os");

console.log("Plataforma       :", os.platform());       // win32
console.log("Versão do SO     :", os.release());
console.log("Arquitetura      :", os.arch());
console.log("Nome da máquina  :", os.hostname());
console.log("Núcleos lógicos  :", os.cpus().length);
console.log("Memória total    :", (os.totalmem() / 1024 ** 3).toFixed(2), "GB");
console.log("Memória livre    :", (os.freemem() / 1024 ** 3).toFixed(2), "GB");
console.log("Pasta pessoal    :", os.homedir());
console.log("Pasta temporária :", os.tmpdir());
console.log("Quebra de linha  :", JSON.stringify(os.EOL)); // "\r\n" no Windows

// Interfaces de rede — obtenção do endereço IPv4 local
const interfaces = os.networkInterfaces();
for (const [nome, enderecos] of Object.entries(interfaces)) {
  for (const endereco of enderecos) {
    if (endereco.family === "IPv4" && !endereco.internal) {
      console.log(`Rede ${nome}: ${endereco.address}`);
    }
  }
}
```

### 6.2 Variáveis de ambiente do Windows

```javascript
// ambiente.js
console.log("Usuário       :", process.env.USERNAME);
console.log("Pasta pessoal :", process.env.USERPROFILE);
console.log("AppData       :", process.env.APPDATA);
console.log("LocalAppData  :", process.env.LOCALAPPDATA);
console.log("ProgramFiles  :", process.env.ProgramFiles);
console.log("Temp          :", process.env.TEMP);
console.log("Domínio       :", process.env.USERDOMAIN);
console.log("Processadores :", process.env.NUMBER_OF_PROCESSORS);

// Definição de variável válida apenas durante a execução do processo
process.env.AMBIENTE_APP = "desenvolvimento";
console.log("Variável definida:", process.env.AMBIENTE_APP);
```

Pastas padrão do Windows úteis em aplicações reais:

| Finalidade | Caminho | Obtenção |
|---|---|---|
| Documentos do usuário | `C:\Users\<u>\Documents` | `path.join(os.homedir(), "Documents")` |
| Área de Trabalho | `C:\Users\<u>\Desktop` | `path.join(os.homedir(), "Desktop")` |
| Downloads | `C:\Users\<u>\Downloads` | `path.join(os.homedir(), "Downloads")` |
| Configurações da aplicação | `C:\Users\<u>\AppData\Roaming` | `process.env.APPDATA` |
| Cache | `C:\Users\<u>\AppData\Local` | `process.env.LOCALAPPDATA` |

### 6.3 Execução de comandos do Windows

O módulo `child_process` permite executar programas externos. A função `exec` executa o comando através do interpretador e captura toda a saída; `spawn` transmite a saída em fluxo, o que é preferível para comandos longos.

**Arquivo `comandos-windows.js`:**

```javascript
// comandos-windows.js — integração com comandos do sistema
const { exec, spawn } = require("node:child_process");
const { promisify } = require("node:util");

// promisify converte uma função de callback em uma que devolve Promise.
const executar = promisify(exec);

async function listarProcessos() {
  console.log("=== PROCESSOS EM EXECUÇÃO (10 primeiros) ===");

  // tasklist é um comando nativo do Windows.
  // A opção /FO CSV produz saída estruturada, mais fácil de processar.
  const { stdout } = await executar("tasklist /FO CSV /NH");

  const processos = stdout
    .split("\n")
    .filter((linha) => linha.trim())
    .slice(0, 10)
    .map((linha) => {
      // Remove as aspas e separa os campos do CSV.
      const campos = linha.split('","').map((c) => c.replace(/"/g, "").trim());
      return { nome: campos[0], pid: campos[1], memoria: campos[4] };
    });

  console.table(processos);
}

async function informacoesDisco() {
  console.log("\n=== UNIDADES DE DISCO ===");

  // O PowerShell é preferível ao wmic, descontinuado nas versões recentes.
  // ConvertTo-Json permite processar o resultado diretamente em JavaScript.
  const comando = `powershell -NoProfile -Command "Get-PSDrive -PSProvider FileSystem | Select-Object Name,Used,Free | ConvertTo-Json"`;

  const { stdout } = await executar(comando);
  const unidades = JSON.parse(stdout);
  const lista = Array.isArray(unidades) ? unidades : [unidades];

  for (const unidade of lista) {
    const usado = Number(unidade.Used || 0) / 1024 ** 3;
    const livre = Number(unidade.Free || 0) / 1024 ** 3;
    const total = usado + livre;
    if (total === 0) continue;

    const percentual = (usado / total) * 100;
    const barra =
      "█".repeat(Math.round(percentual / 5)) +
      "░".repeat(20 - Math.round(percentual / 5));

    console.log(
      `${unidade.Name}: ${barra} ${percentual.toFixed(1)}% ` +
        `(${livre.toFixed(1)} GB livres de ${total.toFixed(1)} GB)`
    );
  }
}

function pingComSaidaContinua(destino) {
  console.log(`\n=== PING PARA ${destino} ===`);

  return new Promise((resolver) => {
    // spawn não utiliza o interpretador de comandos, o que reduz o risco
    // de injeção quando os argumentos provêm do usuário.
    const processo = spawn("ping", ["-n", "3", destino]);

    processo.stdout.on("data", (dados) => process.stdout.write(dados));
    processo.stderr.on("data", (dados) => process.stderr.write(dados));
    processo.on("close", (codigo) => {
      console.log(`Comando finalizado com código ${codigo}`);
      resolver(codigo);
    });
  });
}

async function abrirRecursos() {
  console.log("\n=== ABERTURA DE RECURSOS DO SISTEMA ===");
  // start é um comando interno do interpretador cmd; exige "cmd /c".
  // As aspas vazias após start representam o título da janela.
  await executar(`cmd /c start "" "https://www.ifb.edu.br"`);
  await executar(`cmd /c start "" "${process.env.USERPROFILE}\\Documents"`);
  console.log("Navegador e Explorador de Arquivos acionados.");
}

(async () => {
  try {
    await listarProcessos();
    await informacoesDisco();
    await pingComSaidaContinua("8.8.8.8");
    // await abrirRecursos();  // descomentar para testar
  } catch (erro) {
    console.error("Falha:", erro.message);
  }
})();
```

**Advertência de segurança:** jamais concatenar entrada fornecida pelo usuário diretamente em uma chamada a `exec`. Um valor como `arquivo.txt & del /f /q C:\*` seria executado pelo interpretador. Quando os argumentos são variáveis, deve-se utilizar `spawn` com o vetor de argumentos, que não passa pelo interpretador.

### 6.4 Notificações e monitoramento

Notificação nativa do Windows via PowerShell:

```javascript
// notificacao.js — exibe uma notificação do Windows
const { exec } = require("node:child_process");

function notificar(titulo, mensagem) {
  const script = `
    [reflection.assembly]::loadwithpartialname('System.Windows.Forms') > $null
    [reflection.assembly]::loadwithpartialname('System.Drawing') > $null
    $icone = New-Object System.Windows.Forms.NotifyIcon
    $icone.Icon = [System.Drawing.SystemIcons]::Information
    $icone.BalloonTipTitle = '${titulo}'
    $icone.BalloonTipText = '${mensagem}'
    $icone.Visible = $true
    $icone.ShowBalloonTip(5000)
    Start-Sleep -Seconds 6
  `.replace(/\n\s*/g, "; ");

  exec(`powershell -NoProfile -Command "${script}"`);
}

notificar("Node.js", "Processamento concluido com exito");
```

Monitoramento de alterações em uma pasta:

```javascript
// monitor.js — observa modificações em tempo real
const fs = require("node:fs");
const path = require("node:path");

const PASTA = path.join(__dirname, "dados");

console.log(`Monitorando: ${PASTA}`);
console.log("Crie, altere ou exclua arquivos nessa pasta. Ctrl+C encerra.\n");

// Alguns editores geram múltiplos eventos para uma única alteração.
// O mapa abaixo aplica um intervalo mínimo entre notificações do mesmo arquivo.
const ultimoEvento = new Map();

fs.watch(PASTA, { recursive: true }, (tipo, arquivo) => {
  if (!arquivo) return;

  const agora = Date.now();
  if (agora - (ultimoEvento.get(arquivo) || 0) < 100) return;
  ultimoEvento.set(arquivo, agora);

  const horario = new Date().toLocaleTimeString("pt-BR");
  const descricao = tipo === "rename" ? "criado ou removido" : "alterado";
  console.log(`[${horario}] ${arquivo} — ${descricao}`);
});
```

Esse é exatamente o mecanismo empregado pelo **nodemon**, utilizado a partir do Tutorial 5.

---

## 7. Laboratório prático

### Laboratório 3.1 — Analisador de pasta

Implementar `analisador.js`, que receba o caminho de uma pasta pela linha de comando e produza um relatório contendo:

1. quantidade total de arquivos e de subpastas;
2. tamanho total ocupado, na unidade mais adequada;
3. distribuição por extensão, ordenada decrescentemente por quantidade;
4. os cinco maiores arquivos, com caminho relativo e tamanho;
5. os arquivos modificados nos últimos sete dias;
6. gravação do relatório completo em `relatorio.json`.

Uso previsto:

```powershell
node analisador.js "C:\Users\aluno\Documents"
```

O programa deve tratar os erros `ENOENT` (pasta inexistente) e `EACCES` (acesso negado), prosseguindo o percurso quando uma subpasta específica não puder ser lida.

### Laboratório 3.2 — Organizador da pasta Downloads

Implementar `organizador.js`, que classifique os arquivos de uma pasta em subpastas por categoria.

**Categorias exigidas:**

| Subpasta | Extensões |
|---|---|
| `Imagens` | `.jpg`, `.jpeg`, `.png`, `.gif`, `.bmp`, `.webp`, `.svg` |
| `Documentos` | `.pdf`, `.docx`, `.doc`, `.txt`, `.odt`, `.xlsx`, `.pptx` |
| `Compactados` | `.zip`, `.rar`, `.7z`, `.tar`, `.gz` |
| `Executaveis` | `.exe`, `.msi`, `.bat` |
| `Midia` | `.mp3`, `.mp4`, `.avi`, `.mkv`, `.wav` |
| `Codigo` | `.js`, `.py`, `.html`, `.css`, `.json`, `.java`, `.c` |
| `Outros` | demais extensões |

**Requisitos obrigatórios:**

1. **Modo de simulação por padrão.** Sem argumentos, o programa apenas imprime as movimentações que seriam realizadas. A execução efetiva exige a opção `--executar`.
2. **Tratamento de colisão de nomes.** Se `foto.jpg` já existir no destino, o novo arquivo deve ser gravado como `foto (1).jpg`, `foto (2).jpg`, e assim sucessivamente.
3. **Preservação de subpastas.** Apenas arquivos da raiz devem ser movidos; subpastas permanecem intactas.
4. **Registro de operações.** Cada movimentação deve ser acrescentada a `organizador.log`, com data, origem e destino.
5. **Relatório final.** Total de arquivos movidos por categoria e espaço reorganizado.

Esqueleto inicial:

```javascript
// organizador.js
const fs = require("node:fs/promises");
const path = require("node:path");

const CATEGORIAS = {
  Imagens: [".jpg", ".jpeg", ".png", ".gif", ".bmp", ".webp", ".svg"],
  Documentos: [".pdf", ".docx", ".doc", ".txt", ".odt", ".xlsx", ".pptx"],
  Compactados: [".zip", ".rar", ".7z", ".tar", ".gz"],
  Executaveis: [".exe", ".msi", ".bat"],
  Midia: [".mp3", ".mp4", ".avi", ".mkv", ".wav"],
  Codigo: [".js", ".py", ".html", ".css", ".json", ".java", ".c"],
};

// Determina a categoria de um arquivo a partir de sua extensão.
function categorizar(nomeArquivo) {
  const extensao = path.extname(nomeArquivo).toLowerCase();
  for (const [categoria, extensoes] of Object.entries(CATEGORIAS)) {
    if (extensoes.includes(extensao)) return categoria;
  }
  return "Outros";
}

// Devolve um caminho de destino livre, acrescentando um contador ao nome
// caso o arquivo já exista.
async function caminhoDisponivel(destino) {
  // TODO: implementar
}

async function organizar(pasta, executar) {
  // TODO: implementar
}

const [, , pastaInformada, ...opcoes] = process.argv;
const executar = opcoes.includes("--executar");

if (!pastaInformada) {
  console.error("Uso: node organizador.js <pasta> [--executar]");
  process.exit(1);
}

organizar(path.resolve(pastaInformada), executar).catch(console.error);
```

**Procedimento de teste seguro:** criar uma pasta `lab-arquivos\teste`, povoá-la com arquivos vazios de extensões variadas (comando `type nul > foto.jpg` no cmd, ou `New-Item foto.jpg` no PowerShell) e executar o organizador sobre ela. Somente após a validação completa em ambiente de teste o programa deve ser aplicado à pasta Downloads real.

### Laboratório 3.3 — Painel do sistema

Implementar `painel.js`, que atualize a tela a cada dois segundos exibindo:

1. data e hora corrente;
2. tempo de atividade do sistema, formatado em dias, horas e minutos;
3. uso de memória com barra de progresso;
4. quantidade de núcleos e modelo do processador;
5. os cinco processos que mais consomem memória, obtidos via `tasklist`;
6. espaço livre em cada unidade de disco.

A limpeza da tela é obtida com `console.clear()`. O encerramento com `Ctrl+C` deve ser tratado pelo evento `process.on("SIGINT", ...)`, exibindo mensagem de despedida antes de finalizar.

---

## 8. Síntese

1. O módulo `path` deve ser sempre utilizado na construção de caminhos, garantindo portabilidade entre sistemas.
2. `__dirname` refere-se à localização do código; `process.cwd()`, à pasta de invocação.
3. A interface `fs/promises` combinada a `async/await` é a forma recomendada de manipular arquivos.
4. A verificação de existência deve ser substituída pelo tratamento de exceções, notadamente do código `ENOENT`.
5. Arquivos grandes exigem *streams*, que mantêm o consumo de memória constante.
6. O módulo `os` expõe informações de hardware e do sistema; `child_process` permite executar comandos do Windows.
7. Operações destrutivas devem ser precedidas de modo de simulação e restritas a pastas de teste.

**Próximo tutorial:** [Website básico com HTML e CSS](./04-website-basico-html-css.md)
