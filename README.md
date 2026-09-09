# Node.js do Zero ao Deploy — Série de Tutoriais

[![Node.js](https://img.shields.io/badge/Node.js-20_LTS-339933?style=flat-square&logo=nodedotjs&logoColor=white)](https://nodejs.org/)
[![npm](https://img.shields.io/badge/npm-10.x-CB3837?style=flat-square&logo=npm&logoColor=white)](https://docs.npmjs.com/)
[![JavaScript](https://img.shields.io/badge/JavaScript-ES2022-F7DF1E?style=flat-square&logo=javascript&logoColor=black)](https://developer.mozilla.org/pt-BR/docs/Web/JavaScript)
[![Licença](https://img.shields.io/badge/Licen%C3%A7a-CC0_1.0-lightgrey?style=flat-square)](./LICENSE)
[![Tutoriais](https://img.shields.io/badge/Tutoriais-9-blue?style=flat-square)](#sequência-dos-tutoriais)

---

## Tecnologias utilizadas

### Plataforma e ambiente

[![Node.js](https://img.shields.io/badge/Node.js-%E2%89%A5_20.0.0-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)](https://nodejs.org/pt)
[![npm](https://img.shields.io/badge/npm-10.9-CB3837?style=for-the-badge&logo=npm&logoColor=white)](https://www.npmjs.com/)
[![Windows](https://img.shields.io/badge/Windows-10_|_11-0078D4?style=for-the-badge&logo=windows&logoColor=white)](https://www.microsoft.com/windows)
[![PowerShell](https://img.shields.io/badge/PowerShell-5.1_|_7-5391FE?style=for-the-badge&logo=powershell&logoColor=white)](https://learn.microsoft.com/pt-br/powershell/)
[![VS Code](https://img.shields.io/badge/VS_Code-1.9x-007ACC?style=for-the-badge&logo=visualstudiocode&logoColor=white)](https://code.visualstudio.com/)
[![Git](https://img.shields.io/badge/Git-2.4x-F05032?style=for-the-badge&logo=git&logoColor=white)](https://git-scm.com/)
[![GitHub](https://img.shields.io/badge/GitHub-repositório-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/)

### Bibliotecas Node.js

| Badge | Pacote | Versão | Tutorial | Finalidade |
|---|---|---|---|---|
| [![Express](https://img.shields.io/badge/Express-4.19.2-000000?style=flat-square&logo=express&logoColor=white)](https://www.npmjs.com/package/express) | `express` | `^4.19.2` | 5, 7, 8 | Framework web e roteamento |
| [![nodemon](https://img.shields.io/badge/nodemon-3.1.0-76D04B?style=flat-square&logo=nodemon&logoColor=white)](https://www.npmjs.com/package/nodemon) | `nodemon` | `^3.1.0` | 5, 6, 7 | Reinício automático em desenvolvimento |
| [![EJS](https://img.shields.io/badge/EJS-3.1-B4CA65?style=flat-square&logo=ejs&logoColor=black)](https://www.npmjs.com/package/ejs) | `ejs` | `^3.1.10` | 5, 6, 8 | Motor de templates |
| [![Sequelize](https://img.shields.io/badge/Sequelize-6.37.3-52B0E7?style=flat-square&logo=sequelize&logoColor=white)](https://www.npmjs.com/package/sequelize) | `sequelize` | `^6.37.3` | 6, 7, 8 | ORM |
| [![sequelize-cli](https://img.shields.io/badge/sequelize--cli-6.6-52B0E7?style=flat-square&logo=sequelize&logoColor=white)](https://www.npmjs.com/package/sequelize-cli) | `sequelize-cli` | `^6.6.2` | 6 | Migrações e *seeders* |
| [![sqlite3](https://img.shields.io/badge/sqlite3-5.1-003B57?style=flat-square&logo=sqlite&logoColor=white)](https://www.npmjs.com/package/sqlite3) | `sqlite3` | `^5.1.7` | 6, 7 | Driver SQLite |
| [![mysql2](https://img.shields.io/badge/mysql2-3.9-4479A1?style=flat-square&logo=mysql&logoColor=white)](https://www.npmjs.com/package/mysql2) | `mysql2` | `^3.9.7` | 6 | Driver MySQL |
| [![pg](https://img.shields.io/badge/pg-8.11.5-4169E1?style=flat-square&logo=postgresql&logoColor=white)](https://www.npmjs.com/package/pg) | `pg` + `pg-hstore` | `^8.11.5` | 8 | Driver PostgreSQL |
| [![dotenv](https://img.shields.io/badge/dotenv-16.4.5-ECD53F?style=flat-square&logo=dotenv&logoColor=black)](https://www.npmjs.com/package/dotenv) | `dotenv` | `^16.4.5` | 5, 6, 7, 8 | Variáveis de ambiente |
| [![CORS](https://img.shields.io/badge/cors-2.8.5-FF6C37?style=flat-square)](https://www.npmjs.com/package/cors) | `cors` | `^2.8.5` | 7, 8 | Política de origem cruzada |
| [![Helmet](https://img.shields.io/badge/helmet-7.1.0-0C4B33?style=flat-square)](https://www.npmjs.com/package/helmet) | `helmet` | `^7.1.0` | 7, 8 | Cabeçalhos de segurança |
| [![morgan](https://img.shields.io/badge/morgan-1.10.0-8A2BE2?style=flat-square)](https://www.npmjs.com/package/morgan) | `morgan` | `^1.10.0` | 5, 7 | Registro de requisições |
| [![compression](https://img.shields.io/badge/compression-1.7-6E4C13?style=flat-square)](https://www.npmjs.com/package/compression) | `compression` | `^1.7.4` | 5 | Compressão gzip |
| [![chalk](https://img.shields.io/badge/chalk-5.3.0-FF6188?style=flat-square)](https://www.npmjs.com/package/chalk) | `chalk` | `^5.3.0` | 1 | Cores no terminal |

### Módulos nativos do Node.js

[![fs](https://img.shields.io/badge/node:fs-nativo-339933?style=flat-square&logo=nodedotjs&logoColor=white)](https://nodejs.org/api/fs.html)
[![path](https://img.shields.io/badge/node:path-nativo-339933?style=flat-square&logo=nodedotjs&logoColor=white)](https://nodejs.org/api/path.html)
[![http](https://img.shields.io/badge/node:http-nativo-339933?style=flat-square&logo=nodedotjs&logoColor=white)](https://nodejs.org/api/http.html)
[![os](https://img.shields.io/badge/node:os-nativo-339933?style=flat-square&logo=nodedotjs&logoColor=white)](https://nodejs.org/api/os.html)
[![child_process](https://img.shields.io/badge/node:child__process-nativo-339933?style=flat-square&logo=nodedotjs&logoColor=white)](https://nodejs.org/api/child_process.html)
[![stream](https://img.shields.io/badge/node:stream-nativo-339933?style=flat-square&logo=nodedotjs&logoColor=white)](https://nodejs.org/api/stream.html)
[![worker_threads](https://img.shields.io/badge/node:worker__threads-nativo-339933?style=flat-square&logo=nodedotjs&logoColor=white)](https://nodejs.org/api/worker_threads.html)
[![test](https://img.shields.io/badge/node:test-nativo-339933?style=flat-square&logo=nodedotjs&logoColor=white)](https://nodejs.org/api/test.html)

### Bancos de dados

[![SQLite](https://img.shields.io/badge/SQLite-3.4x-003B57?style=for-the-badge&logo=sqlite&logoColor=white)](https://www.sqlite.org/)
[![MySQL](https://img.shields.io/badge/MySQL-8.0-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](https://dev.mysql.com/doc/)

### Frontend e implantação

[![HTML5](https://img.shields.io/badge/HTML5-padrão-E34F26?style=for-the-badge&logo=html5&logoColor=white)](https://developer.mozilla.org/pt-BR/docs/Web/HTML)
[![CSS3](https://img.shields.io/badge/CSS3-Grid_|_Flexbox-1572B6?style=for-the-badge&logo=css3&logoColor=white)](https://developer.mozilla.org/pt-BR/docs/Web/CSS)
[![Vercel](https://img.shields.io/badge/Vercel-CLI_37+-000000?style=for-the-badge&logo=vercel&logoColor=white)](https://vercel.com/docs)
[![REST](https://img.shields.io/badge/API-REST-005571?style=for-the-badge)](https://developer.mozilla.org/en-US/docs/Glossary/REST)
[![Markdown](https://img.shields.io/badge/Markdown-CommonMark-000000?style=for-the-badge&logo=markdown&logoColor=white)](https://commonmark.org/)

---

## Sequência dos tutoriais

| # | Tutorial | Conteúdo | Tecnologias |
|---|---|---|---|
| 1 | [Instalação no Windows e ecossistema npm](./01-instalacao-windows-e-ecossistema.md) | Instalação, REPL, `package.json`, npm, npx, SemVer | ![Node](https://img.shields.io/badge/-Node.js-339933?style=flat-square&logo=nodedotjs&logoColor=white) ![npm](https://img.shields.io/badge/-npm-CB3837?style=flat-square&logo=npm&logoColor=white) ![Windows](https://img.shields.io/badge/-Windows-0078D4?style=flat-square&logo=windows&logoColor=white) |
| 2 | [Arquitetura do Node.js e suas aplicações](./02-arquitetura-e-aplicacoes.md) | V8, libuv, *event loop*, assincronismo, CommonJS e ESM | ![V8](https://img.shields.io/badge/-V8-4B8BF5?style=flat-square&logo=v8&logoColor=white) ![Node](https://img.shields.io/badge/-worker__threads-339933?style=flat-square&logo=nodedotjs&logoColor=white) |
| 3 | [Sistema de arquivos e recursos do Windows](./03-sistema-de-arquivos-e-windows.md) | `fs`, `path`, *streams*, `os`, `child_process` | ![fs](https://img.shields.io/badge/-node:fs-339933?style=flat-square&logo=nodedotjs&logoColor=white) ![PowerShell](https://img.shields.io/badge/-PowerShell-5391FE?style=flat-square&logo=powershell&logoColor=white) |
| 4 | [Website básico com HTML e CSS](./04-website-basico-html-css.md) | HTTP, módulo `http`, arquivos estáticos, MIME, 404 | ![HTML5](https://img.shields.io/badge/-HTML5-E34F26?style=flat-square&logo=html5&logoColor=white) ![CSS3](https://img.shields.io/badge/-CSS3-1572B6?style=flat-square&logo=css3&logoColor=white) |
| 5 | [Aplicação web com Express e nodemon](./05-express-e-nodemon.md) | Express, *middlewares*, EJS, formulários, camadas | ![Express](https://img.shields.io/badge/-Express-000000?style=flat-square&logo=express&logoColor=white) ![nodemon](https://img.shields.io/badge/-nodemon-76D04B?style=flat-square&logo=nodemon&logoColor=white) ![EJS](https://img.shields.io/badge/-EJS-B4CA65?style=flat-square&logo=ejs&logoColor=black) |
| 6 | [Banco de dados com SQLite, MySQL e ORM](./06-banco-de-dados-orm.md) | Sequelize, modelos, relacionamentos, migrações | ![Sequelize](https://img.shields.io/badge/-Sequelize-52B0E7?style=flat-square&logo=sequelize&logoColor=white) ![SQLite](https://img.shields.io/badge/-SQLite-003B57?style=flat-square&logo=sqlite&logoColor=white) ![MySQL](https://img.shields.io/badge/-MySQL-4479A1?style=flat-square&logo=mysql&logoColor=white) |
| 7 | [API REST com Node.js e SQLite](./07-api-rest-sqlite.md) | REST, CRUD, validação, paginação, CORS, testes | ![Express](https://img.shields.io/badge/-Express-000000?style=flat-square&logo=express&logoColor=white) ![SQLite](https://img.shields.io/badge/-SQLite-003B57?style=flat-square&logo=sqlite&logoColor=white) ![Helmet](https://img.shields.io/badge/-helmet-0C4B33?style=flat-square) |
| 8 | [Implantação na Vercel](./08-deploy-vercel.md) | *Serverless*, Git, variáveis de ambiente, banco na nuvem | ![Vercel](https://img.shields.io/badge/-Vercel-000000?style=flat-square&logo=vercel&logoColor=white) ![PostgreSQL](https://img.shields.io/badge/-PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white) ![Git](https://img.shields.io/badge/-Git-F05032?style=flat-square&logo=git&logoColor=white) |
| 9 | [Criação e publicação de um pacote npm](./09-pacote-npm.md) | Biblioteca, testes nativos, `npm link`, publicação, SemVer | ![npm](https://img.shields.io/badge/-npm-CB3837?style=flat-square&logo=npm&logoColor=white) ![ESM](https://img.shields.io/badge/-ESM-F7DF1E?style=flat-square&logo=javascript&logoColor=black) |

---

## Licença

Distribuído sob [CC0 1.0 Universal](./LICENSE).

---
