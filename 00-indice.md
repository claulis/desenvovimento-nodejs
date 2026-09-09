# Node.js do Zero ao Deploy — Série de Tutoriais

**Público-alvo:** estudantes de Ensino Médio Integrado em Informática para Internet, sem experiência prévia com Node.js.
**Pré-requisitos gerais:** lógica de programação, JavaScript básico (variáveis, funções, vetores, objetos), HTML e CSS.
**Carga horária estimada:** 28 aulas de 50 minutos, incluídos os laboratórios.

---

## Sequência dos tutoriais

| # | Tutorial | Conteúdo | Aulas |
|---|---|---|---|
| 1 | [Instalação no Windows e ecossistema npm](./01-instalacao-windows-e-ecossistema.md) | Instalação, REPL, `package.json`, npm, npx, SemVer | 2 |
| 2 | [Arquitetura do Node.js e suas aplicações](./02-arquitetura-e-aplicacoes.md) | V8, libuv, *event loop*, assincronismo, CommonJS e ESM | 2 |
| 3 | [Sistema de arquivos e recursos do Windows](./03-sistema-de-arquivos-e-windows.md) | `fs`, `path`, *streams*, `os`, `child_process` | 3 |
| 4 | [Website básico com HTML e CSS](./04-website-basico-html-css.md) | HTTP, módulo `http`, arquivos estáticos, MIME, 404 | 3 |
| 5 | [Aplicação web com Express e nodemon](./05-express-e-nodemon.md) | Express, *middlewares*, EJS, formulários, camadas | 4 |
| 6 | [Banco de dados com SQLite, MySQL e ORM](./06-banco-de-dados-orm.md) | Sequelize, modelos, relacionamentos, migrações | 4 |
| 7 | [API REST com Node.js e SQLite](./07-api-rest-sqlite.md) | REST, CRUD, validação, paginação, CORS, testes | 4 |
| 8 | [Implantação na Vercel](./08-deploy-vercel.md) | *Serverless*, Git, variáveis de ambiente, banco na nuvem | 3 |
| 9 | [Criação e publicação de um pacote npm](./09-pacote-npm.md) | Biblioteca, testes nativos, `npm link`, publicação, SemVer | 3 |

---

## Observações sobre a ordem

Os Tutoriais 4 e 5 devem ser estudados em sequência e sem intervalo. O Tutorial 4 constrói manualmente, com o módulo `http`, aquilo que o Tutorial 5 substitui por Express. O contraste entre as duas implementações é o objetivo pedagógico central desse par: sem a experiência do trabalho manual, as abstrações do *framework* tornam-se fórmulas memorizadas em vez de soluções compreendidas.

O Tutorial 7 pressupõe os Tutoriais 5 e 6. O Tutorial 8 exige uma API já construída — a do Tutorial 7. O Tutorial 9 é relativamente independente e pode ser antecipado, caso conveniente ao planejamento.

---

## Convenções adotadas

**Idioma do código.** Identificadores criados pelo autor do código estão em português (`buscarPorId`, `resposta`, `erros`). Palavras reservadas da linguagem, nomes de bibliotecas e parâmetros exigidos por bibliotecas permanecem em inglês (`async`, `express`, `req`, `res`, `next`). Essa é a convenção predominante em projetos brasileiros.

**Sistema de módulos.** Os Tutoriais 3 a 8 utilizam CommonJS (`require`), formato predominante na documentação existente. O Tutorial 9 utiliza ESM (`import`), padrão atual para publicação de novos pacotes. A distinção é explicada no Tutorial 2, Seção 6.

**Terminal.** Os comandos são apresentados para **PowerShell** no Windows. Diferenças relevantes em relação ao `cmd` são assinaladas quando existem.

**Segurança.** Cada tutorial trata as vulnerabilidades pertinentes ao seu conteúdo — XSS no Tutorial 4, *path traversal* no Tutorial 4, injeção de SQL nos Tutoriais 6 e 7, injeção de comandos no Tutorial 3, exposição de credenciais nos Tutoriais 5 e 8. A segurança é abordada no momento em que o problema surge, e não como tópico isolado.

---

## Projeto integrador

Os laboratórios formam uma progressão contínua. Ao final da série, o estudante terá construído e publicado:

1. um **website institucional**, dos Tutoriais 4 e 5;
2. um **sistema de biblioteca** com banco de dados, do Tutorial 6;
3. uma **API REST de tarefas**, com testes automatizados, do Tutorial 7;
4. a **API e a aplicação publicadas** em ambiente de produção, do Tutorial 8;
5. um **pacote npm autoral**, publicado no registro público, do Tutorial 9.

Esse conjunto constitui portfólio suficiente para demonstração em processo seletivo de estágio.

---

## Verificação do ambiente

Antes de iniciar o Tutorial 1, confirmar:

```powershell
node -v     # deve exibir v20.x.x ou superior
npm -v      # deve exibir 10.x.x ou superior
git --version
code -v
```

Caso algum comando não seja reconhecido, o Tutorial 1 apresenta o procedimento completo de instalação.

---

## Convenções tipográficas

- Blocos identificados como `powershell` contêm comandos de terminal.
- Blocos identificados como `javascript`, `html`, `css`, `json` ou `sql` contêm código a ser gravado em arquivo; o nome do arquivo é indicado imediatamente antes do bloco.
- Comentários dentro do código explicam **por que** determinada decisão foi tomada, e não apenas o que a linha executa.
- Advertências de segurança e erros frequentes aparecem em destaque no corpo do texto.
