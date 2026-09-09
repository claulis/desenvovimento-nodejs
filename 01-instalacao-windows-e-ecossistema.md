# Tutorial 1 — Instalação do Node.js no Windows e o Ecossistema npm

> **Pré-requisitos:** Windows 10 ou 11 (64 bits), permissão para instalar programas e acesso à internet.
> **Duração estimada:** 2 aulas de 50 minutos.
> **Ao final deste tutorial, o estudante será capaz de:** instalar e verificar o Node.js, utilizar o REPL, executar arquivos JavaScript fora do navegador, criar um projeto com `package.json` e instalar bibliotecas de terceiros com o npm.

---

## 1. O que é o Node.js

O JavaScript foi criado, em 1995, para ser executado exclusivamente dentro do navegador. O navegador fornece ao JavaScript um **ambiente de execução** (*runtime*), composto pelo interpretador da linguagem e por um conjunto de funcionalidades externas à linguagem, tais como `document`, `alert` e `fetch`. Essas funcionalidades não pertencem ao JavaScript: pertencem ao navegador.

O **Node.js** é um ambiente de execução alternativo. Ele retira o motor JavaScript do navegador — o **V8**, desenvolvido pelo Google para o Chrome — e o acopla a um conjunto diferente de funcionalidades, voltadas ao sistema operacional: leitura e escrita de arquivos, abertura de portas de rede, execução de outros programas, acesso a variáveis de ambiente e criação de servidores HTTP.

A consequência prática é direta: com o Node.js, a mesma linguagem utilizada para programar a interface de uma página passa a ser utilizável para programar o servidor que entrega essa página, os scripts que automatizam tarefas do computador e as ferramentas de linha de comando.

| Característica | Navegador | Node.js |
|---|---|---|
| Motor JavaScript | V8 (Chrome), SpiderMonkey (Firefox) | V8 |
| Acesso a arquivos do disco | Não (por segurança) | Sim |
| Objeto global | `window` | `globalThis` / `global` |
| Manipulação de HTML (DOM) | Sim | Não |
| Criação de servidores de rede | Não | Sim |
| Gerenciador de pacotes padrão | — | npm |

**Observação importante:** o Node.js **não** possui `document`, `window` ou `alert`. Código que manipula elementos HTML não funciona no Node.js, e código que lê arquivos do disco não funciona no navegador. Confundir os dois ambientes é o erro mais frequente de quem inicia.

---

## 2. Versões LTS e Current

O projeto Node.js publica duas linhas de versões simultaneamente:

- **LTS** (*Long Term Support*): versões de numeração par (20, 22, 24). Recebem correções de segurança por aproximadamente 30 meses. É a linha recomendada para aprendizado, para produção e para uso em sala de aula.
- **Current**: versões de numeração ímpar. Contêm recursos experimentais e recebem suporte curto. Destinam-se a testes.

**Recomendação para este curso:** instalar a versão **LTS mais recente** disponível em <https://nodejs.org>. Todos os exemplos desta série foram verificados na linha 22 LTS e são compatíveis com versões LTS superiores.

---

## 3. Instalação — Método 1: instalador gráfico (recomendado para iniciantes)

### Passo 1 — Download

Acessar <https://nodejs.org> e clicar no botão da versão **LTS**. O site detecta o sistema operacional e oferece o arquivo `node-vXX.X.X-x64.msi`.

### Passo 2 — Execução do instalador

Executar o arquivo `.msi` baixado e avançar pelas telas:

1. **Welcome** → *Next*.
2. **End-User License Agreement** → marcar *I accept the terms* → *Next*.
3. **Destination Folder** → manter `C:\Program Files\nodejs\` → *Next*.
4. **Custom Setup** → **manter todos os componentes marcados**. É essencial que `Add to PATH` esteja selecionado; sem isso, o comando `node` não será reconhecido no terminal.
5. **Tools for Native Modules** → esta tela oferece a instalação automática de Python e das ferramentas de compilação do Visual Studio. Para os tutoriais desta série, **não é necessário marcar** essa opção. Ela é exigida apenas por bibliotecas que compilam código C++ durante a instalação.
6. **Install** → confirmar o aviso do Controle de Conta de Usuário (UAC) → aguardar → *Finish*.

### Passo 3 — Verificação

Abrir o **PowerShell** (tecla `Windows`, digitar `powershell`, `Enter`) e executar:

```powershell
node -v
npm -v
```

A saída esperada é semelhante a:

```
v22.22.2
10.9.7
```

Caso o terminal responda `'node' não é reconhecido como um comando interno ou externo`, o PATH não foi atualizado. A solução é **fechar todas as janelas de terminal e abrir uma nova** — o PATH é lido apenas na abertura do terminal. Se o erro persistir, reiniciar o computador e, em último caso, reinstalar marcando `Add to PATH`.

---

## 4. Instalação — Método 2: winget (linha de comando)

O Windows 10 e 11 incluem o gerenciador de pacotes `winget`. Em um PowerShell:

```powershell
winget install OpenJS.NodeJS.LTS
```

Após a conclusão, é necessário **abrir um novo terminal** antes de executar `node -v`.

Para atualizar posteriormente:

```powershell
winget upgrade OpenJS.NodeJS.LTS
```

---

## 5. Instalação — Método 3: nvm-windows (múltiplas versões)

Projetos diferentes podem exigir versões diferentes do Node.js. O **nvm-windows** permite instalar várias versões e alternar entre elas.

1. Baixar `nvm-setup.exe` em <https://github.com/coreybutler/nvm-windows/releases>.
2. **Desinstalar previamente** qualquer Node.js instalado pelo método 1 ou 2 (Configurações → Aplicativos), pois as instalações conflitam.
3. Executar o instalador e abrir um **novo PowerShell como Administrador**.

Comandos principais:

```powershell
nvm list available        # lista as versões disponíveis para download
nvm install lts           # instala a LTS mais recente
nvm install 20.19.0       # instala uma versão específica
nvm list                  # lista as versões instaladas
nvm use 22.22.2           # ativa uma versão (requer terminal como Administrador)
```

**Observação:** o `nvm use` deve ser executado em terminal com privilégios de administrador, pois altera um link simbólico em `C:\Program Files\nodejs`.

---

## 6. Configuração do PowerShell para scripts npm

Por padrão, o PowerShell bloqueia a execução de scripts `.ps1`. Como o npm é distribuído no Windows na forma de um script `npm.ps1`, esse bloqueio produz o seguinte erro:

```
npm : O arquivo C:\Program Files\nodejs\npm.ps1 não pode ser carregado
porque a execução de scripts foi desabilitada neste sistema.
```

A correção é executada **uma única vez**, em um PowerShell aberto como Administrador:

```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

A política `RemoteSigned` permite a execução de scripts criados localmente e exige assinatura digital apenas para scripts baixados da internet. Trata-se da configuração recomendada para estações de desenvolvimento.

---

## 7. Ambiente de edição: Visual Studio Code

Embora o Node.js funcione com qualquer editor de texto, recomenda-se o **Visual Studio Code** (<https://code.visualstudio.com>), gratuito e com integração nativa ao terminal.

Instalação por linha de comando:

```powershell
winget install Microsoft.VisualStudioCode
```

Extensões recomendadas (menu lateral *Extensions*, atalho `Ctrl+Shift+X`):

| Extensão | Finalidade |
|---|---|
| **ESLint** | Aponta erros e más práticas enquanto o código é escrito |
| **Prettier** | Formata o código automaticamente ao salvar |
| **REST Client** ou **Thunder Client** | Testa APIs sem sair do editor (utilizado no Tutorial 7) |
| **SQLite Viewer** | Visualiza o conteúdo de bancos SQLite (utilizado no Tutorial 6) |
| **Portuguese (Brazil) Language Pack** | Traduz a interface do editor |

O terminal integrado é aberto com `` Ctrl+` `` e já se posiciona na pasta do projeto aberto, o que dispensa navegação manual com `cd`.

---

## 8. Primeiro contato: o REPL

O **REPL** (*Read-Eval-Print Loop*) é um interpretador interativo. Serve para testar trechos curtos de código sem criar arquivos. Para iniciá-lo, digitar `node` sem argumentos:

```powershell
node
```

```
Welcome to Node.js v22.22.2.
Type ".help" for more information.
> 2 + 3
5
> const nome = "Instituto Federal de Brasília";
undefined
> nome.toUpperCase()
'INSTITUTO FEDERAL DE BRASÍLIA'
> [1, 2, 3, 4].filter(n => n % 2 === 0)
[ 2, 4 ]
> process.platform
'win32'
> .exit
```

Comandos úteis do REPL:

- `.help` — lista os comandos disponíveis;
- `.editor` — abre modo de múltiplas linhas (`Ctrl+D` finaliza);
- `.exit` ou `Ctrl+C` duas vezes — encerra a sessão.

O REPL é adequado para experimentação. Programas reais são escritos em arquivos.

---

## 9. Primeiro programa em arquivo

Criar uma pasta de trabalho e um arquivo `ola.js`:

```powershell
mkdir C:\Users\%USERNAME%\projetos\aula01
cd C:\Users\%USERNAME%\projetos\aula01
code .
```

**Arquivo `ola.js`:**

```javascript
// ola.js — primeiro programa executado fora do navegador

// 1) Saída no terminal
console.log("Olá, mundo! Este código está sendo executado pelo Node.js.");

// 2) Informações fornecidas pelo ambiente de execução, e não pela linguagem.
//    O objeto global `process` representa o processo em execução no sistema.
console.log("Versão do Node.js:", process.version);
console.log("Sistema operacional:", process.platform);
console.log("Arquitetura do processador:", process.arch);
console.log("Pasta de trabalho atual:", process.cwd());

// 3) Argumentos recebidos pela linha de comando.
//    process.argv[0] = caminho do executável node
//    process.argv[1] = caminho do arquivo executado
//    process.argv[2] em diante = argumentos fornecidos pelo usuário
const argumentos = process.argv.slice(2);

if (argumentos.length === 0) {
  console.log("\nNenhum argumento foi informado.");
  console.log("Experimente executar: node ola.js Ana Bruno Carla");
} else {
  console.log(`\n${argumentos.length} argumento(s) recebido(s):`);
  argumentos.forEach((valor, indice) => {
    console.log(`  [${indice}] ${valor}`);
  });
}
```

Execução:

```powershell
node ola.js
node ola.js Ana Bruno Carla
```

Saída esperada da segunda execução:

```
Olá, mundo! Este código está sendo executado pelo Node.js.
Versão do Node.js: v22.22.2
Sistema operacional: win32
Arquitetura do processador: x64
Pasta de trabalho atual: C:\Users\aluno\projetos\aula01

3 argumento(s) recebido(s):
  [0] Ana
  [1] Bruno
  [2] Carla
```

**Análise do resultado:** nenhuma das informações impressas — versão, plataforma, argumentos de linha de comando — está disponível no JavaScript de navegador. Elas provêm do objeto `process`, injetado pelo Node.js no escopo global. Esse é o primeiro indício concreto de que o ambiente de execução, e não a linguagem, define o que um programa consegue fazer.

---

## 10. O npm e o conceito de pacote

O **npm** (*Node Package Manager*) é instalado automaticamente junto com o Node.js. Ele desempenha três papéis:

1. **Cliente de instalação:** baixa bibliotecas do registro público <https://www.npmjs.com>, que hospeda mais de três milhões de pacotes.
2. **Gerenciador de metadados:** mantém o arquivo `package.json`, que descreve o projeto e suas dependências.
3. **Executor de tarefas:** roda comandos declarados na seção `scripts`.

### 10.1 Criação de um projeto

```powershell
mkdir C:\Users\%USERNAME%\projetos\meu-projeto
cd C:\Users\%USERNAME%\projetos\meu-projeto
npm init -y
```

A opção `-y` aceita todos os valores padrão. Sem ela, o npm formula perguntas interativas sobre nome, versão, descrição, autor e licença.

O comando gera o arquivo **`package.json`**:

```json
{
  "name": "meu-projeto",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

Significado dos campos principais:

| Campo | Descrição |
|---|---|
| `name` | Identificador do projeto. Somente minúsculas, sem espaços |
| `version` | Versão no formato **SemVer** (`MAIOR.MENOR.CORREÇÃO`) |
| `main` | Arquivo carregado quando o pacote é importado por outro projeto |
| `scripts` | Atalhos de comandos executáveis por `npm run <nome>` |
| `dependencies` | Bibliotecas necessárias à execução do programa |
| `devDependencies` | Bibliotecas necessárias apenas ao desenvolvimento |
| `type` | Define o sistema de módulos: ausente = CommonJS; `"module"` = ESM |

### 10.2 Instalação de uma biblioteca

Será instalada a biblioteca **chalk**, que colore a saída do terminal:

```powershell
npm install chalk
```

Três alterações ocorrem no projeto:

1. A pasta **`node_modules/`** é criada, contendo o código-fonte de `chalk` e de todas as suas dependências indiretas.
2. O campo `dependencies` é acrescentado ao `package.json`.
3. O arquivo **`package-lock.json`** é gerado, registrando a versão exata de cada pacote da árvore de dependências.

**Arquivo `cores.js`:**

```javascript
// cores.js — utilização de uma biblioteca instalada pelo npm
// A versão 5 do chalk é distribuída como módulo ESM. Para importá-la em um
// arquivo CommonJS, utiliza-se a função import() dinâmica, que devolve
// uma Promise. O await de nível superior está disponível em módulos ESM;
// aqui é usada uma função assíncrona autoinvocada.

(async () => {
  const { default: chalk } = await import("chalk");

  console.log(chalk.green("✔ Operação concluída com êxito"));
  console.log(chalk.yellow("⚠ Atenção: espaço em disco reduzido"));
  console.log(chalk.red.bold("✖ Falha na conexão com o banco de dados"));
  console.log(chalk.bgBlue.white(" INFORMAÇÃO "), "Servidor iniciado na porta 3000");

  const nota = 8.5;
  const cor = nota >= 6 ? chalk.green : chalk.red;
  console.log(`Nota final: ${cor(nota.toFixed(1))}`);
})();
```

```powershell
node cores.js
```

Alternativa sem `import()` dinâmico: instalar a versão 4, distribuída em CommonJS.

```powershell
npm install chalk@4
```

```javascript
const chalk = require("chalk");
console.log(chalk.green("Funciona com require()"));
```

Essa diferença entre CommonJS e ESM é detalhada no Tutorial 2.

### 10.3 Dependências de desenvolvimento

```powershell
npm install --save-dev nodemon
```

A opção `--save-dev` (abreviada `-D`) registra o pacote em `devDependencies`. Ferramentas de teste, de formatação e de reinício automático pertencem a essa categoria: são necessárias para desenvolver o projeto, mas não para executá-lo em um servidor de produção.

O `package.json` passa a apresentar:

```json
{
  "dependencies": {
    "chalk": "^5.3.0"
  },
  "devDependencies": {
    "nodemon": "^3.1.0"
  }
}
```

### 10.4 O prefixo `^` e o versionamento semântico

O **SemVer** atribui significado a cada número da versão `MAIOR.MENOR.CORREÇÃO`:

- **CORREÇÃO** (`5.3.0` → `5.3.1`): correção de defeito, sem alteração de comportamento;
- **MENOR** (`5.3.0` → `5.4.0`): novo recurso compatível com o código existente;
- **MAIOR** (`5.3.0` → `6.0.0`): alteração incompatível, capaz de quebrar o código existente.

Os prefixos determinam quais atualizações o npm pode aplicar:

| Notação | Aceita | Exemplo |
|---|---|---|
| `^5.3.0` | Atualizações menores e de correção | `5.3.1`, `5.9.2` — nunca `6.0.0` |
| `~5.3.0` | Somente correções | `5.3.4` — nunca `5.4.0` |
| `5.3.0` | Somente a versão exata | apenas `5.3.0` |

### 10.5 A pasta `node_modules` e o controle de versão

A pasta `node_modules` frequentemente ocupa centenas de megabytes e contém milhares de arquivos. Ela **nunca** deve ser enviada a um repositório Git, pois pode ser integralmente reconstruída a partir do `package.json` e do `package-lock.json`.

**Arquivo `.gitignore`:**

```gitignore
node_modules/
.env
*.log
.DS_Store
```

Ao clonar um projeto que já possua `package.json`, o comando abaixo reconstrói a pasta:

```powershell
npm install
```

Em ambientes de integração contínua e de implantação, prefere-se:

```powershell
npm ci
```

O comando `npm ci` instala exatamente as versões registradas no `package-lock.json`, sem consultar intervalos de versão, o que garante instalações reprodutíveis.

### 10.6 Scripts do npm

A seção `scripts` do `package.json` define atalhos:

```json
{
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js",
    "cores": "node cores.js"
  }
}
```

Execução:

```powershell
npm start        # os scripts start, test, stop e restart dispensam "run"
npm run dev
npm run cores
```

Vantagem prática: qualquer pessoa que receba o projeto descobre como executá-lo lendo o `package.json`, sem depender de instruções externas.

### 10.7 O comando `npx`

O `npx` executa um pacote **sem instalá-lo permanentemente**. É útil para ferramentas de uso esporádico:

```powershell
npx cowsay "Aula de Node.js"
npx create-react-app minha-aplicacao
npx http-server .
```

O pacote é baixado para um cache temporário, executado e descartado. Isso evita poluir o sistema com dezenas de instalações globais.

### 10.8 Comandos npm de referência

```powershell
npm install <pacote>              # instala e registra em dependencies
npm install -D <pacote>           # instala e registra em devDependencies
npm install -g <pacote>           # instala globalmente (disponível em todo o sistema)
npm uninstall <pacote>            # remove o pacote
npm list --depth=0                # lista as dependências diretas
npm outdated                      # exibe pacotes com versões mais recentes disponíveis
npm update                        # atualiza respeitando os intervalos do package.json
npm audit                         # relata vulnerabilidades conhecidas
npm audit fix                     # corrige as vulnerabilidades quando possível
npm run                           # lista todos os scripts disponíveis
npm docs <pacote>                 # abre a documentação do pacote no navegador
```

---

## 11. Panorama do ecossistema

O quadro a seguir apresenta bibliotecas de uso corrente, organizadas por finalidade. Não é necessário memorizá-lo; ele serve como mapa de referência para os próximos tutoriais.

**Servidores web e APIs**
- `express` — o *framework* web mais adotado; utilizado nos Tutoriais 5, 6, 7 e 8;
- `fastify` — alternativa com desempenho superior e validação embutida;
- `cors`, `helmet`, `morgan` — *middlewares* de compartilhamento entre origens, cabeçalhos de segurança e registro de requisições.

**Bancos de dados**
- `sequelize` — ORM compatível com SQLite, MySQL, PostgreSQL e SQL Server; utilizado no Tutorial 6;
- `prisma` — ORM moderno, com esquema declarativo e tipagem gerada automaticamente;
- `better-sqlite3`, `mysql2`, `pg` — *drivers* de acesso direto, sem ORM.

**Autenticação e segurança**
- `bcrypt` — geração de *hash* de senhas;
- `jsonwebtoken` — emissão e validação de tokens JWT;
- `zod`, `joi` — validação de dados de entrada.

**Qualidade de código e testes**
- `eslint` — análise estática;
- `prettier` — formatação automática;
- `jest`, `vitest` — estruturas de teste;
- `node:test` — executor de testes nativo, dispensa instalação.

**Ferramentas de desenvolvimento**
- `nodemon` — reinicia o servidor a cada alteração de arquivo;
- `dotenv` — carrega variáveis de ambiente de um arquivo `.env`;
- `typescript` — superconjunto tipado do JavaScript.

**Aplicações além do servidor web**
- `electron` — aplicações de desktop (Visual Studio Code, Discord e Slack são construídos com Electron);
- `commander`, `inquirer` — construção de ferramentas de linha de comando;
- `puppeteer`, `playwright` — automação de navegador e extração de dados;
- `vite`, `webpack` — empacotamento de aplicações front-end.

---

## 12. Laboratório prático

### Laboratório 1.1 — Diagnóstico do ambiente

Criar o projeto `diagnostico` e implementar `diagnostico.js`, que apresente um relatório do ambiente utilizando exclusivamente os módulos nativos `os` e `process`.

```powershell
mkdir diagnostico
cd diagnostico
npm init -y
```

**Arquivo `diagnostico.js`:**

```javascript
// diagnostico.js — relatório do ambiente de execução
// O prefixo "node:" identifica explicitamente um módulo nativo do Node.js e
// evita ambiguidade com pacotes de terceiros de mesmo nome.
const os = require("node:os");

// Converte bytes em gigabytes com duas casas decimais.
function paraGigabytes(bytes) {
  return (bytes / 1024 ** 3).toFixed(2);
}

// Converte segundos em uma representação legível de horas e minutos.
function formatarTempoAtivo(segundos) {
  const horas = Math.floor(segundos / 3600);
  const minutos = Math.floor((segundos % 3600) / 60);
  return `${horas}h ${minutos}min`;
}

// Imprime uma linha do relatório com rótulo alinhado à esquerda.
function linha(rotulo, valor) {
  console.log(`  ${rotulo.padEnd(26, ".")} ${valor}`);
}

console.log("=".repeat(58));
console.log("  RELATÓRIO DO AMBIENTE DE EXECUÇÃO");
console.log("=".repeat(58));

console.log("\n[ NODE.JS ]");
linha("Versão do Node.js", process.version);
linha("Versão do motor V8", process.versions.v8);
linha("Identificador do processo", process.pid);

console.log("\n[ SISTEMA OPERACIONAL ]");
linha("Plataforma", os.platform());
linha("Versão", os.release());
linha("Arquitetura", os.arch());
linha("Nome da máquina", os.hostname());
linha("Tempo ligado", formatarTempoAtivo(os.uptime()));

console.log("\n[ HARDWARE ]");
linha("Modelo do processador", os.cpus()[0].model.trim());
linha("Núcleos lógicos", os.cpus().length);
linha("Memória total", `${paraGigabytes(os.totalmem())} GB`);
linha("Memória livre", `${paraGigabytes(os.freemem())} GB`);

const percentualUso = (1 - os.freemem() / os.totalmem()) * 100;
const barrasPreenchidas = Math.round(percentualUso / 5);
const barra = "█".repeat(barrasPreenchidas) + "░".repeat(20 - barrasPreenchidas);
linha("Uso de memória", `${barra} ${percentualUso.toFixed(1)}%`);

console.log("\n[ USUÁRIO ]");
const usuario = os.userInfo();
linha("Nome de usuário", usuario.username);
linha("Pasta pessoal", usuario.homedir);
linha("Pasta temporária", os.tmpdir());

console.log("\n" + "=".repeat(58));
```

```powershell
node diagnostico.js
```

**Questões de verificação:**
1. Qual valor `os.platform()` retornaria em um computador com Linux? E com macOS?
2. Por que `os.freemem()` produz resultados diferentes a cada execução?
3. O que ocorre se a chamada `os.cpus()[0]` for substituída por `os.cpus()[99]`? Justificar a resposta com base na estrutura de dados retornada.

### Laboratório 1.2 — Projeto com dependências

1. Criar o projeto `saudacao-colorida` com `npm init -y`.
2. Instalar `chalk@4` como dependência de execução.
3. Instalar `nodemon` como dependência de desenvolvimento.
4. Implementar `index.js`, que receba um nome pela linha de comando e exiba uma saudação colorida; caso nenhum nome seja informado, o programa deve utilizar `os.userInfo().username`.
5. Registrar em `package.json` os scripts `start` (`node index.js`) e `dev` (`nodemon index.js`).
6. Criar `.gitignore` contendo `node_modules/`.
7. Executar `npm run dev`, alterar o texto da saudação e observar o reinício automático.

**Entrega:** a pasta do projeto compactada **sem** a pasta `node_modules`, acompanhada de uma captura de tela do terminal demonstrando o reinício automático promovido pelo nodemon.

---

## 13. Erros frequentes e respectivas soluções

| Mensagem | Causa | Solução |
|---|---|---|
| `'node' não é reconhecido...` | PATH não atualizado | Fechar e reabrir o terminal; reinstalar marcando *Add to PATH* |
| `npm.ps1 não pode ser carregado` | Política de execução do PowerShell | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` |
| `Cannot find module 'chalk'` | Dependência não instalada ou terminal em pasta incorreta | Executar `npm install` na pasta que contém o `package.json` |
| `EADDRINUSE: address already in use` | Porta ocupada por outro processo | Alterar a porta ou encerrar o processo (Tutorial 3) |
| `ERR_REQUIRE_ESM` | Uso de `require()` em pacote ESM | Instalar versão CommonJS ou utilizar `import()` dinâmico |
| `EACCES` / `EPERM` em `npm install -g` | Falta de privilégios | Abrir o terminal como Administrador ou preferir `npx` |

---

## 14. Síntese

1. O Node.js é um ambiente de execução que permite executar JavaScript fora do navegador, com acesso ao sistema operacional.
2. A linha LTS é a indicada para aprendizado e produção.
3. O `package.json` descreve o projeto; o `package-lock.json` fixa as versões exatas; a pasta `node_modules` armazena o código instalado e não é versionada.
4. O npm instala bibliotecas, gerencia dependências e executa scripts; o `npx` executa pacotes sem instalação permanente.
5. O objeto global `process` e o módulo `os` expõem informações do processo e da máquina, indisponíveis no JavaScript de navegador.

**Próximo tutorial:** [Arquitetura do Node.js e suas aplicações](./02-arquitetura-e-aplicacoes.md)
