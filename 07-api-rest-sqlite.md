# Tutorial 7 — Construção de uma API REST com Node.js e SQLite

> **Pré-requisitos:** Tutoriais 5 e 6 concluídos.
> **Duração estimada:** 4 aulas de 50 minutos.
> **Ao final deste tutorial, o estudante será capaz de:** explicar os princípios do estilo arquitetural REST, projetar rotas e códigos de status adequados, implementar operações completas de criação, leitura, atualização e exclusão, validar dados de entrada, paginar e filtrar resultados, configurar CORS, documentar a interface e testá-la com ferramentas apropriadas.

---

## 1. O que é uma API REST

Uma **API** (*Application Programming Interface*) é uma interface destinada ao consumo por programas, e não por pessoas. Enquanto um website devolve HTML pronto para exibição, uma API devolve **dados estruturados**, tipicamente em JSON, que o programa consumidor utiliza como julgar conveniente.

**REST** (*Representational State Transfer*) é um conjunto de convenções para o projeto dessas interfaces sobre HTTP.

### 1.1 Princípios

**Recursos identificados por URLs.** Cada substantivo do domínio corresponde a um caminho. Verbos não devem aparecer na URL:

```
CORRETO                          INCORRETO
GET    /tarefas                  GET  /listarTarefas
GET    /tarefas/42               GET  /obterTarefa?id=42
POST   /tarefas                  POST /criarTarefa
PUT    /tarefas/42               POST /atualizarTarefa
DELETE /tarefas/42               POST /excluirTarefa?id=42
```

A ação é expressa pelo **método HTTP**; a URL identifica apenas **o recurso**.

**Ausência de estado.** Cada requisição contém todas as informações necessárias ao seu processamento. O servidor não armazena o contexto da conversa entre requisições. A identificação do cliente, quando necessária, viaja em cada requisição — normalmente em um cabeçalho de autorização.

**Uso semântico dos códigos de status.** O resultado da operação é comunicado pelo código de status, não apenas pelo conteúdo do corpo.

**Representação uniforme.** As respostas seguem estrutura previsível, permitindo que o cliente as processe genericamente.

### 1.2 Correspondência entre métodos e operações

| Operação | Método | Rota | Status de sucesso |
|---|---|---|---|
| Listar | `GET` | `/tarefas` | `200 OK` |
| Obter uma | `GET` | `/tarefas/:id` | `200 OK` |
| Criar | `POST` | `/tarefas` | `201 Created` |
| Substituir | `PUT` | `/tarefas/:id` | `200 OK` |
| Alterar parcialmente | `PATCH` | `/tarefas/:id` | `200 OK` |
| Excluir | `DELETE` | `/tarefas/:id` | `204 No Content` |

**Propriedades relevantes:**

- **Idempotência:** `GET`, `PUT` e `DELETE` produzem o mesmo estado final quando repetidos. `POST` não: duas execuções criam dois registros.
- **Segurança:** `GET` não deve alterar estado. Uma rota `GET /tarefas/42/excluir` viola o princípio e pode ser acionada acidentalmente por mecanismos de pré-carregamento do navegador.

### 1.3 Códigos de status a empregar

| Código | Nome | Situação |
|---|---|---|
| `200` | OK | Consulta ou atualização bem-sucedida |
| `201` | Created | Recurso criado; incluir cabeçalho `Location` |
| `204` | No Content | Exclusão bem-sucedida; corpo vazio |
| `400` | Bad Request | Dados malformados ou inválidos |
| `401` | Unauthorized | Credencial ausente ou inválida |
| `403` | Forbidden | Autenticado, porém sem permissão |
| `404` | Not Found | Recurso inexistente |
| `409` | Conflict | Violação de restrição, como valor único duplicado |
| `422` | Unprocessable Entity | Sintaxe correta, semântica inválida |
| `429` | Too Many Requests | Limite de requisições excedido |
| `500` | Internal Server Error | Falha não prevista do servidor |

---

## 2. Estrutura do projeto

Será construída uma API completa de gerenciamento de tarefas.

```
api-tarefas/
├── .env
├── .env.exemplo
├── .gitignore
├── package.json
├── nodemon.json
├── servidor.js               ← inicialização
├── app.js                    ← configuração do Express
├── config/
│   └── banco.js
├── modelos/
│   ├── index.js
│   ├── Tarefa.js
│   └── Projeto.js
├── controladores/
│   └── tarefasControlador.js
├── servicos/
│   └── tarefasServico.js
├── middlewares/
│   ├── erros.js
│   ├── validacao.js
│   └── assincrono.js
├── rotas/
│   ├── index.js
│   └── tarefas.js
├── scripts/
│   └── inicializar.js
├── dados/
│   └── tarefas.sqlite
└── testes.http               ← requisições de teste
```

```powershell
mkdir api-tarefas
cd api-tarefas
npm init -y

npm install express sequelize sqlite3 dotenv cors helmet morgan
npm install --save-dev nodemon

mkdir config modelos controladores servicos middlewares rotas scripts dados
code .
```

**Arquivo `.env`:**

```env
PORT=3000
NODE_ENV=development
DB_STORAGE=./dados/tarefas.sqlite
ORIGENS_PERMITIDAS=http://localhost:5500,http://127.0.0.1:5500
```

**Arquivo `nodemon.json`:**

```json
{
  "watch": ["servidor.js", "app.js", "rotas/", "controladores/", "servicos/", "modelos/", "middlewares/"],
  "ext": "js,json",
  "ignore": ["node_modules/", "dados/", "*.test.js"],
  "delay": 400
}
```

**Arquivo `package.json` (scripts):**

```json
{
  "scripts": {
    "start": "node servidor.js",
    "dev": "nodemon servidor.js",
    "db:init": "node scripts/inicializar.js"
  }
}
```

---

## 3. Modelos

**Arquivo `config/banco.js`:**

```javascript
require("dotenv").config();

const { Sequelize } = require("sequelize");
const path = require("node:path");
const fs = require("node:fs");

const caminho = path.resolve(process.env.DB_STORAGE || "./dados/tarefas.sqlite");
fs.mkdirSync(path.dirname(caminho), { recursive: true });

const sequelize = new Sequelize({
  dialect: "sqlite",
  storage: caminho,
  logging: process.env.NODE_ENV === "development" ? console.log : false,
});

module.exports = { sequelize };
```

**Arquivo `modelos/Projeto.js`:**

```javascript
const { DataTypes } = require("sequelize");
const { sequelize } = require("../config/banco");

const Projeto = sequelize.define(
  "Projeto",
  {
    id: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
    nome: {
      type: DataTypes.STRING(100),
      allowNull: false,
      unique: true,
      validate: { notEmpty: { msg: "O nome do projeto é obrigatório" } },
    },
    cor: {
      type: DataTypes.STRING(7),
      defaultValue: "#2d6a4f",
      validate: {
        is: { args: /^#[0-9a-fA-F]{6}$/, msg: "A cor deve seguir o formato #rrggbb" },
      },
    },
  },
  { tableName: "projetos", timestamps: true }
);

module.exports = Projeto;
```

**Arquivo `modelos/Tarefa.js`:**

```javascript
const { DataTypes } = require("sequelize");
const { sequelize } = require("../config/banco");

const Tarefa = sequelize.define(
  "Tarefa",
  {
    id: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },

    titulo: {
      type: DataTypes.STRING(150),
      allowNull: false,
      validate: {
        notEmpty: { msg: "O título é obrigatório" },
        len: { args: [3, 150], msg: "O título deve ter entre 3 e 150 caracteres" },
      },
    },

    descricao: { type: DataTypes.TEXT, allowNull: true },

    prioridade: {
      type: DataTypes.ENUM("baixa", "media", "alta", "urgente"),
      defaultValue: "media",
      validate: {
        isIn: {
          args: [["baixa", "media", "alta", "urgente"]],
          msg: "Prioridade deve ser: baixa, media, alta ou urgente",
        },
      },
    },

    situacao: {
      type: DataTypes.ENUM("pendente", "em_andamento", "concluida", "cancelada"),
      defaultValue: "pendente",
    },

    prazo: {
      type: DataTypes.DATEONLY,
      allowNull: true,
      validate: {
        isDate: { msg: "O prazo deve ser uma data válida (AAAA-MM-DD)" },
      },
    },

    concluidaEm: { type: DataTypes.DATE, allowNull: true },
  },
  {
    tableName: "tarefas",
    timestamps: true,
    indexes: [{ fields: ["situacao"] }, { fields: ["prioridade"] }, { fields: ["prazo"] }],

    hooks: {
      // Registra automaticamente o instante de conclusão quando a
      // situação muda para "concluida", e o limpa em caso de reabertura.
      beforeSave(tarefa) {
        if (tarefa.changed("situacao")) {
          tarefa.concluidaEm = tarefa.situacao === "concluida" ? new Date() : null;
        }
      },
    },
  }
);

// Propriedade calculada: não existe no banco, é derivada dos dados.
Object.defineProperty(Tarefa.prototype, "atrasada", {
  get() {
    if (!this.prazo || this.situacao === "concluida" || this.situacao === "cancelada") {
      return false;
    }
    return new Date(this.prazo) < new Date(new Date().toDateString());
  },
});

// Personaliza o JSON devolvido pela API, acrescentando o campo calculado.
Tarefa.prototype.toJSON = function () {
  const valores = { ...this.get() };
  valores.atrasada = this.atrasada;
  return valores;
};

module.exports = Tarefa;
```

**Arquivo `modelos/index.js`:**

```javascript
const { sequelize } = require("../config/banco");
const Tarefa = require("./Tarefa");
const Projeto = require("./Projeto");

Projeto.hasMany(Tarefa, { foreignKey: "projetoId", as: "tarefas", onDelete: "SET NULL" });
Tarefa.belongsTo(Projeto, { foreignKey: "projetoId", as: "projeto" });

module.exports = { sequelize, Tarefa, Projeto };
```

---

## 4. Middlewares de apoio

**Arquivo `middlewares/assincrono.js`:**

```javascript
// middlewares/assincrono.js
// Encapsula funções assíncronas de rota, encaminhando qualquer rejeição
// ao middleware de erro. Evita a repetição de try/catch em cada rota.
module.exports = function assincrono(manipulador) {
  return (req, res, next) => {
    Promise.resolve(manipulador(req, res, next)).catch(next);
  };
};
```

**Arquivo `middlewares/erros.js`:**

```javascript
// middlewares/erros.js

/** Erro da aplicação com código de status associado. */
class ErroAPI extends Error {
  constructor(mensagem, status = 500, detalhes = null) {
    super(mensagem);
    this.status = status;
    this.detalhes = detalhes;
    this.name = "ErroAPI";
  }
}

/** Middleware de rota inexistente. Registrado após todas as rotas. */
function naoEncontrado(req, res) {
  res.status(404).json({
    sucesso: false,
    erro: {
      codigo: 404,
      mensagem: `A rota ${req.method} ${req.originalUrl} não existe nesta API`,
    },
  });
}

/** Middleware central de erros. Os quatro parâmetros são obrigatórios. */
function tratarErros(erro, req, res, next) {
  // Erros de validação declarados nos modelos.
  if (erro.name === "SequelizeValidationError") {
    return res.status(400).json({
      sucesso: false,
      erro: {
        codigo: 400,
        mensagem: "Os dados enviados são inválidos",
        campos: erro.errors.map((e) => ({ campo: e.path, mensagem: e.message })),
      },
    });
  }

  // Violação de restrição de unicidade.
  if (erro.name === "SequelizeUniqueConstraintError") {
    return res.status(409).json({
      sucesso: false,
      erro: {
        codigo: 409,
        mensagem: "Já existe um registro com esse valor",
        campos: erro.errors.map((e) => ({ campo: e.path, mensagem: e.message })),
      },
    });
  }

  // Chave estrangeira inexistente.
  if (erro.name === "SequelizeForeignKeyConstraintError") {
    return res.status(400).json({
      sucesso: false,
      erro: { codigo: 400, mensagem: "Referência a um registro inexistente" },
    });
  }

  // JSON malformado no corpo da requisição.
  if (erro.type === "entity.parse.failed") {
    return res.status(400).json({
      sucesso: false,
      erro: { codigo: 400, mensagem: "O corpo da requisição não é um JSON válido" },
    });
  }

  // Erros previstos pela aplicação.
  if (erro instanceof ErroAPI) {
    return res.status(erro.status).json({
      sucesso: false,
      erro: { codigo: erro.status, mensagem: erro.message, detalhes: erro.detalhes },
    });
  }

  // Falha imprevista: registrar internamente e devolver mensagem genérica.
  // Detalhes de infraestrutura jamais devem ser expostos ao cliente.
  console.error(`[${new Date().toISOString()}] ERRO NÃO TRATADO`);
  console.error(erro.stack);

  res.status(500).json({
    sucesso: false,
    erro: {
      codigo: 500,
      mensagem: "Erro interno do servidor",
      ...(process.env.NODE_ENV === "development" && { detalhe: erro.message }),
    },
  });
}

module.exports = { ErroAPI, naoEncontrado, tratarErros };
```

**Arquivo `middlewares/validacao.js`:**

```javascript
// middlewares/validacao.js — validação da entrada antes do controlador
const { ErroAPI } = require("./erros");

const PRIORIDADES = ["baixa", "media", "alta", "urgente"];
const SITUACOES = ["pendente", "em_andamento", "concluida", "cancelada"];

/** Verifica que o parâmetro :id é um inteiro positivo. */
function validarId(req, res, next) {
  const id = Number(req.params.id);

  if (!Number.isInteger(id) || id < 1) {
    return next(new ErroAPI("O identificador deve ser um número inteiro positivo", 400));
  }

  req.idValidado = id;
  next();
}

/** Valida o corpo na criação de uma tarefa. */
function validarCriacao(req, res, next) {
  const { titulo, prioridade, situacao, prazo } = req.body;
  const erros = [];

  if (typeof titulo !== "string" || titulo.trim().length < 3) {
    erros.push({ campo: "titulo", mensagem: "O título deve ter ao menos 3 caracteres" });
  }

  if (prioridade !== undefined && !PRIORIDADES.includes(prioridade)) {
    erros.push({
      campo: "prioridade",
      mensagem: `Valor inválido. Aceitos: ${PRIORIDADES.join(", ")}`,
    });
  }

  if (situacao !== undefined && !SITUACOES.includes(situacao)) {
    erros.push({
      campo: "situacao",
      mensagem: `Valor inválido. Aceitos: ${SITUACOES.join(", ")}`,
    });
  }

  if (prazo !== undefined && prazo !== null && !/^\d{4}-\d{2}-\d{2}$/.test(prazo)) {
    erros.push({ campo: "prazo", mensagem: "O prazo deve seguir o formato AAAA-MM-DD" });
  }

  if (erros.length > 0) {
    return next(new ErroAPI("Os dados enviados são inválidos", 400, erros));
  }

  // Normalização: remove espaços supérfluos e descarta campos não previstos,
  // impedindo que o cliente grave propriedades arbitrárias.
  req.body = {
    titulo: titulo.trim(),
    descricao: req.body.descricao?.trim() || null,
    prioridade: prioridade || "media",
    situacao: situacao || "pendente",
    prazo: prazo || null,
    projetoId: req.body.projetoId ? Number(req.body.projetoId) : null,
  };

  next();
}

/** Valida o corpo na atualização parcial: todos os campos são opcionais. */
function validarAtualizacao(req, res, next) {
  const erros = [];
  const permitidos = ["titulo", "descricao", "prioridade", "situacao", "prazo", "projetoId"];

  const enviados = Object.keys(req.body).filter((c) => permitidos.includes(c));

  if (enviados.length === 0) {
    return next(
      new ErroAPI(`Informe ao menos um campo. Permitidos: ${permitidos.join(", ")}`, 400)
    );
  }

  if (req.body.titulo !== undefined) {
    if (typeof req.body.titulo !== "string" || req.body.titulo.trim().length < 3) {
      erros.push({ campo: "titulo", mensagem: "O título deve ter ao menos 3 caracteres" });
    }
  }

  if (req.body.prioridade !== undefined && !PRIORIDADES.includes(req.body.prioridade)) {
    erros.push({ campo: "prioridade", mensagem: `Aceitos: ${PRIORIDADES.join(", ")}` });
  }

  if (req.body.situacao !== undefined && !SITUACOES.includes(req.body.situacao)) {
    erros.push({ campo: "situacao", mensagem: `Aceitos: ${SITUACOES.join(", ")}` });
  }

  if (erros.length > 0) {
    return next(new ErroAPI("Os dados enviados são inválidos", 400, erros));
  }

  // Mantém apenas os campos permitidos que foram efetivamente enviados.
  const limpo = {};
  for (const campo of enviados) limpo[campo] = req.body[campo];
  req.body = limpo;

  next();
}

module.exports = { validarId, validarCriacao, validarAtualizacao, PRIORIDADES, SITUACOES };
```

---

## 5. Serviço e controlador

**Arquivo `servicos/tarefasServico.js`:**

```javascript
// servicos/tarefasServico.js — regras de negócio e acesso a dados
const { Op } = require("sequelize");
const { Tarefa, Projeto, sequelize } = require("../modelos");

const CAMPOS_ORDENAVEIS = ["id", "titulo", "prioridade", "situacao", "prazo", "createdAt"];

async function listar(parametros = {}) {
  const {
    situacao,
    prioridade,
    projetoId,
    busca,
    atrasadas,
    pagina = 1,
    porPagina = 10,
    ordenarPor = "createdAt",
    ordem = "DESC",
  } = parametros;

  const where = {};

  if (situacao) where.situacao = situacao;
  if (prioridade) where.prioridade = prioridade;
  if (projetoId) where.projetoId = Number(projetoId);

  // Busca textual em título ou descrição.
  if (busca) {
    where[Op.or] = [
      { titulo: { [Op.like]: `%${busca}%` } },
      { descricao: { [Op.like]: `%${busca}%` } },
    ];
  }

  // Tarefas com prazo vencido e ainda não finalizadas.
  if (atrasadas === "true" || atrasadas === true) {
    where.prazo = { [Op.lt]: new Date().toISOString().slice(0, 10) };
    where.situacao = { [Op.notIn]: ["concluida", "cancelada"] };
  }

  // Limites protegem o servidor contra requisições abusivas.
  const limite = Math.min(Math.max(Number(porPagina) || 10, 1), 100);
  const paginaAtual = Math.max(Number(pagina) || 1, 1);

  // O campo de ordenação provém do cliente: deve ser validado contra uma
  // lista fechada, sob pena de injeção de SQL na cláusula ORDER BY.
  const campo = CAMPOS_ORDENAVEIS.includes(ordenarPor) ? ordenarPor : "createdAt";
  const direcao = String(ordem).toUpperCase() === "ASC" ? "ASC" : "DESC";

  const { count, rows } = await Tarefa.findAndCountAll({
    where,
    include: [{ model: Projeto, as: "projeto", attributes: ["id", "nome", "cor"] }],
    limit: limite,
    offset: (paginaAtual - 1) * limite,
    order: [[campo, direcao]],
  });

  return {
    dados: rows,
    paginacao: {
      total: count,
      pagina: paginaAtual,
      porPagina: limite,
      totalPaginas: Math.ceil(count / limite),
      temProxima: paginaAtual * limite < count,
      temAnterior: paginaAtual > 1,
    },
  };
}

async function buscarPorId(id) {
  return Tarefa.findByPk(id, {
    include: [{ model: Projeto, as: "projeto", attributes: ["id", "nome", "cor"] }],
  });
}

async function criar(dados) {
  const tarefa = await Tarefa.create(dados);
  return buscarPorId(tarefa.id);
}

async function atualizar(id, dados) {
  const tarefa = await Tarefa.findByPk(id);
  if (!tarefa) return null;

  await tarefa.update(dados);
  return buscarPorId(id);
}

async function remover(id) {
  const linhas = await Tarefa.destroy({ where: { id } });
  return linhas > 0;
}

async function alternarConclusao(id) {
  const tarefa = await Tarefa.findByPk(id);
  if (!tarefa) return null;

  tarefa.situacao = tarefa.situacao === "concluida" ? "pendente" : "concluida";
  await tarefa.save(); // o hook beforeSave ajusta concluidaEm

  return buscarPorId(id);
}

async function estatisticas() {
  // Uma única consulta agrupada, em lugar de uma contagem por situação.
  const porSituacao = await Tarefa.findAll({
    attributes: ["situacao", [sequelize.fn("COUNT", sequelize.col("id")), "total"]],
    group: ["situacao"],
    raw: true,
  });

  const porPrioridade = await Tarefa.findAll({
    attributes: ["prioridade", [sequelize.fn("COUNT", sequelize.col("id")), "total"]],
    group: ["prioridade"],
    raw: true,
  });

  const atrasadas = await Tarefa.count({
    where: {
      prazo: { [Op.lt]: new Date().toISOString().slice(0, 10) },
      situacao: { [Op.notIn]: ["concluida", "cancelada"] },
    },
  });

  const total = await Tarefa.count();
  const concluidas = Number(
    porSituacao.find((s) => s.situacao === "concluida")?.total || 0
  );

  return {
    total,
    atrasadas,
    percentualConcluido: total > 0 ? Number(((concluidas / total) * 100).toFixed(1)) : 0,
    porSituacao: Object.fromEntries(porSituacao.map((s) => [s.situacao, Number(s.total)])),
    porPrioridade: Object.fromEntries(
      porPrioridade.map((p) => [p.prioridade, Number(p.total)])
    ),
  };
}

module.exports = {
  listar, buscarPorId, criar, atualizar, remover, alternarConclusao, estatisticas,
};
```

**Arquivo `controladores/tarefasControlador.js`:**

```javascript
// controladores/tarefasControlador.js
// Responsável por traduzir entre HTTP e o serviço. Não contém regras
// de negócio nem acesso direto ao banco.
const servico = require("../servicos/tarefasServico");
const { ErroAPI } = require("../middlewares/erros");

async function listar(req, res) {
  const resultado = await servico.listar(req.query);

  res.json({
    sucesso: true,
    ...resultado,
  });
}

async function obter(req, res) {
  const tarefa = await servico.buscarPorId(req.idValidado);

  if (!tarefa) {
    throw new ErroAPI(`Tarefa ${req.idValidado} não encontrada`, 404);
  }

  res.json({ sucesso: true, dados: tarefa });
}

async function criar(req, res) {
  const tarefa = await servico.criar(req.body);

  // 201 Created com o cabeçalho Location apontando para o novo recurso.
  res
    .status(201)
    .location(`/api/tarefas/${tarefa.id}`)
    .json({ sucesso: true, mensagem: "Tarefa criada", dados: tarefa });
}

async function atualizar(req, res) {
  const tarefa = await servico.atualizar(req.idValidado, req.body);

  if (!tarefa) {
    throw new ErroAPI(`Tarefa ${req.idValidado} não encontrada`, 404);
  }

  res.json({ sucesso: true, mensagem: "Tarefa atualizada", dados: tarefa });
}

async function remover(req, res) {
  const removida = await servico.remover(req.idValidado);

  if (!removida) {
    throw new ErroAPI(`Tarefa ${req.idValidado} não encontrada`, 404);
  }

  // 204 No Content: exclusão bem-sucedida, sem corpo de resposta.
  res.status(204).end();
}

async function alternar(req, res) {
  const tarefa = await servico.alternarConclusao(req.idValidado);

  if (!tarefa) {
    throw new ErroAPI(`Tarefa ${req.idValidado} não encontrada`, 404);
  }

  res.json({ sucesso: true, dados: tarefa });
}

async function estatisticas(req, res) {
  res.json({ sucesso: true, dados: await servico.estatisticas() });
}

module.exports = { listar, obter, criar, atualizar, remover, alternar, estatisticas };
```

---

## 6. Rotas

**Arquivo `rotas/tarefas.js`:**

```javascript
// rotas/tarefas.js
const express = require("express");
const controlador = require("../controladores/tarefasControlador");
const assincrono = require("../middlewares/assincrono");
const { validarId, validarCriacao, validarAtualizacao } =
  require("../middlewares/validacao");

const router = express.Router();

// A rota /estatisticas precede /:id — na ordem inversa, "estatisticas"
// seria interpretado como um identificador.
router.get("/estatisticas", assincrono(controlador.estatisticas));

router.get("/", assincrono(controlador.listar));
router.post("/", validarCriacao, assincrono(controlador.criar));

router.get("/:id", validarId, assincrono(controlador.obter));
router.patch("/:id", validarId, validarAtualizacao, assincrono(controlador.atualizar));
router.put("/:id", validarId, validarCriacao, assincrono(controlador.atualizar));
router.delete("/:id", validarId, assincrono(controlador.remover));

router.post("/:id/alternar", validarId, assincrono(controlador.alternar));

module.exports = router;
```

**Arquivo `rotas/index.js`:**

```javascript
const express = require("express");
const router = express.Router();

router.use("/tarefas", require("./tarefas"));

// Rota de verificação de disponibilidade, utilizada por ferramentas
// de monitoramento e pelas plataformas de hospedagem.
router.get("/saude", (req, res) => {
  res.json({
    sucesso: true,
    servico: "API de Tarefas",
    versao: "1.0.0",
    tempoAtivoSegundos: Math.floor(process.uptime()),
    horario: new Date().toISOString(),
  });
});

module.exports = router;
```

**Arquivo `app.js`:**

```javascript
require("dotenv").config();

const express = require("express");
const cors = require("cors");
const helmet = require("helmet");
const morgan = require("morgan");

const rotas = require("./rotas");
const { naoEncontrado, tratarErros } = require("./middlewares/erros");

const app = express();

// ---------- SEGURANÇA ----------
app.use(helmet());

// ---------- CORS ----------
// Por padrão, o navegador impede que uma página em um domínio faça
// requisições a outro domínio. O CORS autoriza explicitamente as
// origens permitidas.
const origensPermitidas = (process.env.ORIGENS_PERMITIDAS || "")
  .split(",")
  .map((o) => o.trim())
  .filter(Boolean);

app.use(
  cors({
    origin(origem, callback) {
      // Requisições sem origem (Postman, curl, aplicações servidoras)
      // são aceitas.
      if (!origem) return callback(null, true);

      // Em desenvolvimento, todas as origens são liberadas.
      if (process.env.NODE_ENV === "development") return callback(null, true);

      if (origensPermitidas.includes(origem)) return callback(null, true);

      callback(new Error(`Origem não autorizada: ${origem}`));
    },
    methods: ["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
  })
);

// ---------- REGISTRO E INTERPRETAÇÃO DO CORPO ----------
app.use(morgan(process.env.NODE_ENV === "development" ? "dev" : "combined"));
app.use(express.json({ limit: "1mb" }));
app.use(express.urlencoded({ extended: true }));

// ---------- ROTAS ----------
app.get("/", (req, res) => {
  res.json({
    api: "API de Tarefas",
    versao: "1.0.0",
    documentacao: "/api/documentacao",
    recursos: {
      tarefas: "/api/tarefas",
      estatisticas: "/api/tarefas/estatisticas",
      saude: "/api/saude",
    },
  });
});

app.use("/api", rotas);

// ---------- TRATAMENTO DE ERROS: sempre por último ----------
app.use(naoEncontrado);
app.use(tratarErros);

module.exports = app;
```

**Arquivo `servidor.js`:**

```javascript
// servidor.js — inicialização
// A separação entre app.js e servidor.js permite importar a aplicação
// em testes automatizados sem abrir uma porta de rede, e é também o
// formato exigido pela implantação em ambiente serverless (Tutorial 8).
const app = require("./app");
const { sequelize } = require("./modelos");

const PORTA = process.env.PORT || 3000;

async function iniciar() {
  try {
    await sequelize.authenticate();
    console.log("Conexão com o banco estabelecida.");

    if (process.env.NODE_ENV === "development") {
      await sequelize.sync({ alter: true });
      console.log("Estrutura das tabelas sincronizada.");
    }

    const servidor = app.listen(PORTA, () => {
      console.log(`\n  API de Tarefas`);
      console.log(`  http://localhost:${PORTA}`);
      console.log(`  Ambiente: ${process.env.NODE_ENV || "development"}\n`);
    });

    // Encerramento controlado: interrompe a aceitação de novas conexões,
    // aguarda a conclusão das requisições em curso e fecha o banco.
    const encerrar = async (sinal) => {
      console.log(`\n${sinal} recebido. Encerrando...`);
      servidor.close(async () => {
        await sequelize.close();
        console.log("Recursos liberados.");
        process.exit(0);
      });

      // Limite de tolerância para requisições pendentes.
      setTimeout(() => process.exit(1), 10_000);
    };

    process.on("SIGINT", () => encerrar("SIGINT"));
    process.on("SIGTERM", () => encerrar("SIGTERM"));
  } catch (erro) {
    console.error("Falha na inicialização:", erro.message);
    process.exit(1);
  }
}

iniciar();
```

**Arquivo `scripts/inicializar.js`:**

```javascript
const { sequelize, Tarefa, Projeto } = require("../modelos");

async function inicializar() {
  await sequelize.sync({ force: true });

  const projetos = await Projeto.bulkCreate([
    { nome: "Trabalho de conclusão", cor: "#c1121f" },
    { nome: "Estudos", cor: "#2d6a4f" },
    { nome: "Pessoal", cor: "#003049" },
  ]);

  const hoje = new Date();
  const emDias = (n) =>
    new Date(hoje.getTime() + n * 86_400_000).toISOString().slice(0, 10);

  await Tarefa.bulkCreate([
    { titulo: "Definir o tema do projeto integrador", prioridade: "urgente",
      situacao: "concluida", prazo: emDias(-10), projetoId: projetos[0].id },
    { titulo: "Levantar requisitos com o cliente", prioridade: "alta",
      situacao: "em_andamento", prazo: emDias(3), projetoId: projetos[0].id },
    { titulo: "Modelar o banco de dados", descricao: "Diagrama entidade-relacionamento",
      prioridade: "alta", situacao: "pendente", prazo: emDias(7), projetoId: projetos[0].id },
    { titulo: "Revisar o event loop do Node.js", prioridade: "media",
      situacao: "pendente", prazo: emDias(-2), projetoId: projetos[1].id },
    { titulo: "Praticar consultas com Sequelize", prioridade: "media",
      situacao: "em_andamento", prazo: emDias(5), projetoId: projetos[1].id },
    { titulo: "Organizar a pasta de downloads", prioridade: "baixa",
      situacao: "pendente", projetoId: projetos[2].id },
    { titulo: "Renovar empréstimo na biblioteca", prioridade: "alta",
      situacao: "pendente", prazo: emDias(-1), projetoId: projetos[2].id },
  ]);

  console.log(`Base preparada: ${await Tarefa.count()} tarefas.`);
  await sequelize.close();
}

inicializar().catch((erro) => {
  console.error(erro);
  process.exit(1);
});
```

Execução:

```powershell
npm run db:init
npm run dev
```

---

## 7. Teste da API

### 7.1 Extensão REST Client

Instalar a extensão **REST Client** no Visual Studio Code e criar o arquivo abaixo. Cada bloco exibirá o botão *Send Request*.

**Arquivo `testes.http`:**

```http
@base = http://localhost:3000/api

### Verificação de disponibilidade
GET {{base}}/saude

### Listar todas as tarefas
GET {{base}}/tarefas

### Listar com paginação
GET {{base}}/tarefas?pagina=1&porPagina=3

### Filtrar por situação
GET {{base}}/tarefas?situacao=pendente

### Filtrar por prioridade e ordenar por prazo
GET {{base}}/tarefas?prioridade=alta&ordenarPor=prazo&ordem=ASC

### Somente tarefas atrasadas
GET {{base}}/tarefas?atrasadas=true

### Busca textual
GET {{base}}/tarefas?busca=banco

### Obter uma tarefa específica
GET {{base}}/tarefas/1

### Identificador inexistente (404 esperado)
GET {{base}}/tarefas/9999

### Identificador inválido (400 esperado)
GET {{base}}/tarefas/abc

### Estatísticas
GET {{base}}/tarefas/estatisticas

### Criar tarefa (201 esperado)
POST {{base}}/tarefas
Content-Type: application/json

{
  "titulo": "Escrever a documentação da API",
  "descricao": "Descrever todas as rotas e os códigos de status",
  "prioridade": "alta",
  "prazo": "2026-10-15",
  "projetoId": 1
}

### Criar com dados inválidos (400 esperado)
POST {{base}}/tarefas
Content-Type: application/json

{
  "titulo": "ab",
  "prioridade": "urgentissima"
}

### Atualização parcial
PATCH {{base}}/tarefas/1
Content-Type: application/json

{
  "situacao": "em_andamento"
}

### Alternar conclusão
POST {{base}}/tarefas/2/alternar

### Excluir (204 esperado)
DELETE {{base}}/tarefas/7

### Rota inexistente (404 esperado)
GET {{base}}/inexistente
```

### 7.2 Testes pelo PowerShell

```powershell
# Listagem
curl.exe http://localhost:3000/api/tarefas

# Criação
curl.exe -X POST http://localhost:3000/api/tarefas `
  -H "Content-Type: application/json" `
  -d '{\"titulo\":\"Tarefa criada pelo terminal\",\"prioridade\":\"alta\"}'

# Atualização parcial
curl.exe -X PATCH http://localhost:3000/api/tarefas/1 `
  -H "Content-Type: application/json" `
  -d '{\"situacao\":\"concluida\"}'

# Exclusão
curl.exe -X DELETE http://localhost:3000/api/tarefas/5

# Exibição apenas do código de status
curl.exe -o NUL -w "%{http_code}\n" -s http://localhost:3000/api/tarefas/9999
```

### 7.3 Exemplos de resposta

Listagem bem-sucedida:

```json
{
  "sucesso": true,
  "dados": [
    {
      "id": 3,
      "titulo": "Modelar o banco de dados",
      "descricao": "Diagrama entidade-relacionamento",
      "prioridade": "alta",
      "situacao": "pendente",
      "prazo": "2026-09-14",
      "concluidaEm": null,
      "projetoId": 1,
      "createdAt": "2026-09-07T13:20:11.482Z",
      "updatedAt": "2026-09-07T13:20:11.482Z",
      "atrasada": false,
      "projeto": { "id": 1, "nome": "Trabalho de conclusão", "cor": "#c1121f" }
    }
  ],
  "paginacao": {
    "total": 7,
    "pagina": 1,
    "porPagina": 10,
    "totalPaginas": 1,
    "temProxima": false,
    "temAnterior": false
  }
}
```

Erro de validação:

```json
{
  "sucesso": false,
  "erro": {
    "codigo": 400,
    "mensagem": "Os dados enviados são inválidos",
    "detalhes": [
      { "campo": "titulo", "mensagem": "O título deve ter ao menos 3 caracteres" },
      { "campo": "prioridade", "mensagem": "Valor inválido. Aceitos: baixa, media, alta, urgente" }
    ]
  }
}
```

---

## 8. Cliente de demonstração

**Arquivo `publico/index.html`** (a ser servido separadamente, por exemplo com a extensão Live Server, para exercitar o CORS):

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Cliente da API de Tarefas</title>
  <style>
    body { font-family: "Segoe UI", sans-serif; max-width: 780px;
           margin: 2rem auto; padding: 0 1rem; }
    h1 { color: #2d6a4f; }
    .tarefa { border: 1px solid #e0e0e0; border-left: 4px solid #2d6a4f;
              border-radius: 8px; padding: .9rem; margin-bottom: .6rem;
              display: flex; justify-content: space-between; align-items: center; }
    .tarefa.concluida { opacity: .55; border-left-color: #999; }
    .tarefa.concluida h3 { text-decoration: line-through; }
    .tarefa.atrasada { border-left-color: #c1121f; background: #fff5f5; }
    .tarefa h3 { margin: 0 0 .25rem; font-size: 1rem; }
    .meta { font-size: .82rem; color: #666; }
    button { cursor: pointer; border: none; border-radius: 6px;
             padding: .4rem .8rem; margin-left: .3rem; }
    .concluir { background: #2d6a4f; color: #fff; }
    .excluir { background: #c1121f; color: #fff; }
    form { display: flex; gap: .5rem; margin-bottom: 1.5rem; flex-wrap: wrap; }
    input, select { padding: .55rem; border: 1px solid #ccc; border-radius: 6px; }
    input[type=text] { flex: 1; min-width: 200px; }
    #mensagem { padding: .7rem; border-radius: 6px; margin-bottom: 1rem;
                display: none; }
  </style>
</head>
<body>
  <h1>Tarefas</h1>
  <div id="mensagem"></div>

  <form id="formulario">
    <input type="text" id="titulo" placeholder="Título da nova tarefa" required>
    <select id="prioridade">
      <option value="baixa">Baixa</option>
      <option value="media" selected>Média</option>
      <option value="alta">Alta</option>
      <option value="urgente">Urgente</option>
    </select>
    <input type="date" id="prazo">
    <button type="submit" class="concluir">Adicionar</button>
  </form>

  <div id="lista">Carregando...</div>

  <script>
    const API = "http://localhost:3000/api";

    function avisar(texto, sucesso = true) {
      const el = document.getElementById("mensagem");
      el.textContent = texto;
      el.style.display = "block";
      el.style.background = sucesso ? "#d8f3dc" : "#ffe5e5";
      el.style.color = sucesso ? "#1b4332" : "#8b0000";
      setTimeout(() => (el.style.display = "none"), 3500);
    }

    async function carregar() {
      const lista = document.getElementById("lista");

      try {
        const resposta = await fetch(`${API}/tarefas?porPagina=50`);
        if (!resposta.ok) throw new Error(`Status ${resposta.status}`);

        const { dados } = await resposta.json();

        if (dados.length === 0) {
          lista.innerHTML = "<p>Nenhuma tarefa cadastrada.</p>";
          return;
        }

        lista.innerHTML = dados
          .map((t) => `
            <div class="tarefa ${t.situacao === "concluida" ? "concluida" : ""}
                                ${t.atrasada ? "atrasada" : ""}">
              <div>
                <h3>${t.titulo}</h3>
                <div class="meta">
                  ${t.prioridade} &middot; ${t.situacao}
                  ${t.prazo ? " &middot; prazo " + t.prazo : ""}
                  ${t.atrasada ? " &middot; ATRASADA" : ""}
                </div>
              </div>
              <div>
                <button class="concluir" onclick="alternar(${t.id})">
                  ${t.situacao === "concluida" ? "Reabrir" : "Concluir"}
                </button>
                <button class="excluir" onclick="excluir(${t.id})">Excluir</button>
              </div>
            </div>`)
          .join("");
      } catch (erro) {
        lista.innerHTML = `<p>Falha ao carregar: ${erro.message}.
          Verifique se a API está em execução.</p>`;
      }
    }

    document.getElementById("formulario").addEventListener("submit", async (evento) => {
      evento.preventDefault();

      const corpo = {
        titulo: document.getElementById("titulo").value,
        prioridade: document.getElementById("prioridade").value,
        prazo: document.getElementById("prazo").value || null,
      };

      const resposta = await fetch(`${API}/tarefas`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(corpo),
      });

      const resultado = await resposta.json();

      if (!resposta.ok) {
        avisar(resultado.erro.mensagem, false);
        return;
      }

      document.getElementById("formulario").reset();
      avisar("Tarefa criada.");
      carregar();
    });

    async function alternar(id) {
      await fetch(`${API}/tarefas/${id}/alternar`, { method: "POST" });
      carregar();
    }

    async function excluir(id) {
      if (!confirm("Confirma a exclusão?")) return;
      await fetch(`${API}/tarefas/${id}`, { method: "DELETE" });
      avisar("Tarefa excluída.");
      carregar();
    }

    carregar();
  </script>
</body>
</html>
```

**Experimento sobre CORS:** abrir a página com a extensão Live Server (porta 5500) e, com `NODE_ENV=production` e a origem ausente de `ORIGENS_PERMITIDAS`, observar o bloqueio no console do navegador. Em seguida, incluir a origem na variável, reiniciar a API e confirmar o funcionamento.

---

## 9. Documentação da interface

Uma API sem documentação é inutilizável por terceiros. A forma mais simples consiste em uma rota que descreva a própria interface.

```javascript
// rotas/index.js — acréscimo
router.get("/documentacao", (req, res) => {
  res.json({
    api: "API de Tarefas",
    versao: "1.0.0",
    baseUrl: "/api",
    recursos: [
      {
        metodo: "GET",
        rota: "/tarefas",
        descricao: "Lista tarefas com filtros e paginação",
        parametrosQuery: {
          situacao: "pendente | em_andamento | concluida | cancelada",
          prioridade: "baixa | media | alta | urgente",
          projetoId: "número inteiro",
          busca: "texto pesquisado em título e descrição",
          atrasadas: "true para exibir apenas as vencidas",
          pagina: "padrão 1",
          porPagina: "padrão 10, máximo 100",
          ordenarPor: "id | titulo | prioridade | situacao | prazo | createdAt",
          ordem: "ASC | DESC",
        },
        respostas: { 200: "Lista paginada" },
      },
      {
        metodo: "POST",
        rota: "/tarefas",
        descricao: "Cria uma tarefa",
        corpo: {
          titulo: "obrigatório, 3 a 150 caracteres",
          descricao: "opcional",
          prioridade: "opcional, padrão media",
          prazo: "opcional, formato AAAA-MM-DD",
          projetoId: "opcional",
        },
        respostas: { 201: "Tarefa criada", 400: "Dados inválidos" },
      },
      {
        metodo: "GET",
        rota: "/tarefas/:id",
        respostas: { 200: "Tarefa", 400: "ID inválido", 404: "Não encontrada" },
      },
      {
        metodo: "PATCH",
        rota: "/tarefas/:id",
        descricao: "Atualiza parcialmente",
        respostas: { 200: "Atualizada", 400: "Inválido", 404: "Não encontrada" },
      },
      {
        metodo: "DELETE",
        rota: "/tarefas/:id",
        respostas: { 204: "Excluída", 404: "Não encontrada" },
      },
      {
        metodo: "POST",
        rota: "/tarefas/:id/alternar",
        descricao: "Alterna entre concluída e pendente",
      },
      { metodo: "GET", rota: "/tarefas/estatisticas" },
      { metodo: "GET", rota: "/saude" },
    ],
  });
});
```

Em projetos profissionais, adota-se o padrão **OpenAPI** com a interface Swagger:

```powershell
npm install swagger-ui-express swagger-jsdoc
```

---

## 10. Laboratório prático

### Laboratório 10.1 — Recurso de projetos

Implementar o CRUD completo do recurso `Projeto`, seguindo a mesma arquitetura das tarefas.

**Rotas exigidas:**

| Método | Rota | Comportamento |
|---|---|---|
| GET | `/api/projetos` | Lista, incluindo a contagem de tarefas de cada projeto |
| GET | `/api/projetos/:id` | Detalhe com as tarefas associadas |
| POST | `/api/projetos` | Cria |
| PATCH | `/api/projetos/:id` | Atualiza |
| DELETE | `/api/projetos/:id` | Exclui |
| GET | `/api/projetos/:id/tarefas` | Tarefas do projeto, com paginação |

**Requisitos:**
1. a contagem de tarefas deve ser obtida por agregação em uma única consulta, não por laço;
2. a exclusão de projeto com tarefas associadas deve responder `409 Conflict`, salvo quando a *query string* `?forcar=true` for informada;
3. a duplicidade de nome deve resultar em `409`;
4. validação de cor no formato `#rrggbb`.

### Laboratório 10.2 — Autenticação por chave

Implementar um *middleware* que exija o cabeçalho `X-API-Key` nas operações de escrita.

**Especificação:**
1. `GET` permanece público; `POST`, `PUT`, `PATCH` e `DELETE` exigem a chave;
2. a chave válida é definida na variável de ambiente `API_KEY`;
3. chave ausente resulta em `401`, com o cabeçalho `WWW-Authenticate`;
4. chave incorreta resulta em `403`;
5. as tentativas rejeitadas devem ser registradas no terminal com data e endereço IP;
6. a rota `/api/saude` permanece livre.

**Questão:** por que uma chave estática é inadequada em uma aplicação real com múltiplos usuários? Pesquisar e descrever, em um parágrafo, como o padrão JWT resolve essa limitação.

### Laboratório 10.3 — Testes automatizados

Implementar uma suíte de testes com o executor nativo `node:test`, sem bibliotecas externas.

```javascript
// testes/tarefas.test.js
const { test, describe, before, after } = require("node:test");
const assert = require("node:assert");

const app = require("../app");
const { sequelize } = require("../modelos");

let servidor;
let base;

before(async () => {
  await sequelize.sync({ force: true });
  // A porta 0 faz o sistema operacional atribuir uma porta livre,
  // evitando conflito com o servidor de desenvolvimento.
  servidor = app.listen(0);
  base = `http://localhost:${servidor.address().port}/api`;
});

after(async () => {
  servidor.close();
  await sequelize.close();
});

describe("API de Tarefas", () => {
  let idCriado;

  test("POST /tarefas cria uma tarefa e devolve 201", async () => {
    const resposta = await fetch(`${base}/tarefas`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ titulo: "Tarefa de teste", prioridade: "alta" }),
    });

    assert.strictEqual(resposta.status, 201);

    const corpo = await resposta.json();
    assert.strictEqual(corpo.sucesso, true);
    assert.strictEqual(corpo.dados.titulo, "Tarefa de teste");

    idCriado = corpo.dados.id;
  });

  test("POST /tarefas rejeita título curto com 400", async () => {
    const resposta = await fetch(`${base}/tarefas`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ titulo: "ab" }),
    });

    assert.strictEqual(resposta.status, 400);
  });

  test("GET /tarefas/:id devolve a tarefa criada", async () => {
    const resposta = await fetch(`${base}/tarefas/${idCriado}`);
    assert.strictEqual(resposta.status, 200);
  });

  test("GET /tarefas/:id inexistente devolve 404", async () => {
    const resposta = await fetch(`${base}/tarefas/99999`);
    assert.strictEqual(resposta.status, 404);
  });

  test("DELETE /tarefas/:id devolve 204", async () => {
    const resposta = await fetch(`${base}/tarefas/${idCriado}`, { method: "DELETE" });
    assert.strictEqual(resposta.status, 204);
  });
});
```

```powershell
node --test testes/
```

**Casos adicionais exigidos:** identificador não numérico (400); paginação com `porPagina` acima de 100 (limitado a 100); filtro por situação; alternância de conclusão; `PATCH` sem nenhum campo (400).

---

## 11. Síntese

1. Uma API REST identifica recursos por URLs compostas de substantivos; a ação é expressa pelo método HTTP.
2. Os códigos de status devem refletir com precisão o resultado da operação.
3. A validação da entrada precede o controlador e deve rejeitar campos não previstos.
4. Campos de ordenação provenientes do cliente devem ser confrontados com uma lista fechada.
5. A paginação é obrigatória em listagens que possam crescer indefinidamente, com limite superior de itens por página.
6. O tratamento de erros centralizado converte exceções em respostas padronizadas, sem expor detalhes de infraestrutura.
7. O CORS controla quais origens podem consumir a API a partir de um navegador.
8. A separação entre `app.js` e `servidor.js` viabiliza testes automatizados e implantação em ambiente *serverless*.

**Próximo tutorial:** [Implantação na Vercel](./08-deploy-vercel.md)
