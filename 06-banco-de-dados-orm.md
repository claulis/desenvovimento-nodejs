# Tutorial 6 — Banco de Dados com SQLite, MySQL e ORM

> **Pré-requisitos:** Tutorial 5 concluído; noções de SQL (`SELECT`, `INSERT`, `UPDATE`, `DELETE`).
> **Duração estimada:** 4 aulas de 50 minutos.
> **Ao final deste tutorial, o estudante será capaz de:** explicar o que é um ORM e quando utilizá-lo, configurar o Sequelize com SQLite e com MySQL, definir modelos com validações, estabelecer relacionamentos entre entidades, executar operações de consulta e escrita, aplicar migrações e alternar entre bancos de dados sem alterar a lógica da aplicação.

---

## 1. Persistência de dados

Nos tutoriais anteriores, os dados residiam em vetores na memória do processo. Ao reiniciar o servidor — o que o nodemon faz a cada alteração —, todo o conteúdo é perdido. Uma aplicação real exige **persistência**: os dados devem sobreviver ao encerramento do programa.

### 1.1 Os bancos utilizados

**SQLite** armazena o banco inteiro em um único arquivo, sem servidor. Não requer instalação nem configuração de rede. É adequado a aplicações de escrita moderada, protótipos, aplicações de desktop e ao aprendizado. É, também, o banco mais amplamente implantado do mundo, presente em navegadores, sistemas móveis e equipamentos embarcados.

**MySQL** é um sistema cliente-servidor, com controle de acesso, replicação e suporte a alta concorrência de escrita. É o padrão em hospedagens compartilhadas e em aplicações web de médio e grande porte.

| Critério | SQLite | MySQL |
|---|---|---|
| Instalação | Nenhuma | Servidor dedicado |
| Armazenamento | Arquivo único `.sqlite` | Diretório gerenciado pelo servidor |
| Concorrência de escrita | Um escritor por vez | Múltiplos escritores |
| Acesso remoto | Não | Sim |
| Uso recomendado | Aprendizado, protótipos, desktop | Aplicações web em produção |

---

## 2. O que é um ORM

Um **ORM** (*Object-Relational Mapping*) traduz entre dois modelos de dados distintos: as tabelas e linhas do banco relacional e os objetos e classes da linguagem de programação.

Sem ORM, empregando SQL diretamente:

```javascript
const [linhas] = await conexao.execute(
  "SELECT l.*, a.nome AS autor_nome FROM livros l " +
  "JOIN autores a ON a.id = l.autor_id WHERE l.ano > ? ORDER BY l.titulo",
  [1900]
);
```

Com ORM:

```javascript
const livros = await Livro.findAll({
  where: { ano: { [Op.gt]: 1900 } },
  include: [{ model: Autor, as: "autor" }],
  order: [["titulo", "ASC"]],
});
```

**Vantagens do ORM:**
- proteção automática contra injeção de SQL, por meio de consultas parametrizadas;
- portabilidade: o mesmo código opera em SQLite, MySQL e PostgreSQL;
- validações declarativas antes da gravação;
- relacionamentos expressos como propriedades de objetos;
- migrações versionadas do esquema.

**Desvantagens do ORM:**
- consultas muito complexas resultam mais claras em SQL puro;
- existe custo de desempenho em operações de grande volume;
- a abstração pode gerar consultas ineficientes sem que o programador perceba — notadamente o problema N+1, tratado na Seção 8.

**Posição adotada:** o ORM é a escolha adequada para a maioria das operações. Consultas analíticas complexas podem ser escritas em SQL puro, recurso que o Sequelize também oferece.

---

## 3. Instalação e configuração

```powershell
mkdir catalogo-livros
cd catalogo-livros
npm init -y

npm install express ejs dotenv
npm install sequelize sqlite3 mysql2
npm install --save-dev nodemon sequelize-cli
```

| Pacote | Função |
|---|---|
| `sequelize` | O ORM propriamente dito |
| `sqlite3` | Driver de acesso ao SQLite |
| `mysql2` | Driver de acesso ao MySQL |
| `sequelize-cli` | Ferramenta de migrações e *seeders* |

**Arquivo `.env`:**

```env
PORT=3000
NODE_ENV=development

# Alternar entre "sqlite" e "mysql"
DB_DIALECT=sqlite

# Configuração do SQLite
DB_STORAGE=./dados/catalogo.sqlite

# Configuração do MySQL (utilizada apenas quando DB_DIALECT=mysql)
DB_HOST=localhost
DB_PORT=3306
DB_NAME=catalogo
DB_USER=root
DB_PASSWORD=
```

**Arquivo `.gitignore`:**

```gitignore
node_modules/
.env
dados/*.sqlite
*.log
```

**Arquivo `nodemon.json`:**

```json
{
  "watch": ["app.js", "modelos/", "rotas/", "controladores/", "servicos/", "views/"],
  "ext": "js,json,ejs",
  "ignore": ["node_modules/", "dados/"],
  "delay": 500
}
```

O item `dados/` em `ignore` é indispensável: sem ele, cada gravação no arquivo SQLite reinicia o servidor, gerando um ciclo perpétuo.

**Arquivo `config/banco.js`:**

```javascript
// config/banco.js — conexão única, reutilizada por toda a aplicação
require("dotenv").config();

const { Sequelize } = require("sequelize");
const path = require("node:path");
const fs = require("node:fs");

const dialeto = process.env.DB_DIALECT || "sqlite";

let sequelize;

if (dialeto === "sqlite") {
  // Garante a existência da pasta de destino antes de criar o arquivo.
  const caminhoArquivo = path.resolve(
    process.env.DB_STORAGE || "./dados/catalogo.sqlite"
  );
  fs.mkdirSync(path.dirname(caminhoArquivo), { recursive: true });

  sequelize = new Sequelize({
    dialect: "sqlite",
    storage: caminhoArquivo,
    // logging recebe uma função ou false. Durante o desenvolvimento,
    // exibir o SQL gerado é valioso para compreender o ORM.
    logging:
      process.env.NODE_ENV === "development"
        ? (sql) => console.log(`  [SQL] ${sql}`)
        : false,
  });
} else {
  sequelize = new Sequelize(
    process.env.DB_NAME,
    process.env.DB_USER,
    process.env.DB_PASSWORD,
    {
      host: process.env.DB_HOST,
      port: Number(process.env.DB_PORT) || 3306,
      dialect: "mysql",
      logging: process.env.NODE_ENV === "development" ? console.log : false,
      timezone: "-03:00",
      // Pool de conexões: em vez de abrir e fechar conexões a cada
      // consulta, mantém-se um conjunto reutilizável.
      pool: {
        max: 10,      // conexões simultâneas máximas
        min: 0,
        acquire: 30000, // tempo máximo de espera por uma conexão (ms)
        idle: 10000,    // tempo até encerrar uma conexão ociosa (ms)
      },
      define: {
        charset: "utf8mb4",           // suporte completo a acentos e emojis
        collate: "utf8mb4_unicode_ci",
      },
    }
  );
}

/** Verifica a conectividade. Deve ser chamada na inicialização. */
async function testarConexao() {
  try {
    await sequelize.authenticate();
    console.log(`Conexão estabelecida (${dialeto}).`);
    return true;
  } catch (erro) {
    console.error("Falha na conexão com o banco:", erro.message);
    return false;
  }
}

module.exports = { sequelize, testarConexao, dialeto };
```

---

## 4. Modelos

Um **modelo** representa uma tabela. Cada instância do modelo corresponde a uma linha.

**Arquivo `modelos/Autor.js`:**

```javascript
// modelos/Autor.js
const { DataTypes } = require("sequelize");
const { sequelize } = require("../config/banco");

const Autor = sequelize.define(
  "Autor",
  {
    id: {
      type: DataTypes.INTEGER,
      primaryKey: true,
      autoIncrement: true,
    },
    nome: {
      type: DataTypes.STRING(120),
      allowNull: false,
      validate: {
        notEmpty: { msg: "O nome do autor é obrigatório" },
        len: { args: [2, 120], msg: "O nome deve ter entre 2 e 120 caracteres" },
      },
    },
    nacionalidade: {
      type: DataTypes.STRING(60),
      defaultValue: "Brasileira",
    },
    anoNascimento: {
      type: DataTypes.INTEGER,
      allowNull: true,
      validate: {
        min: { args: [1000], msg: "Ano de nascimento improvável" },
        max: { args: [new Date().getFullYear()], msg: "Ano no futuro" },
      },
    },
  },
  {
    tableName: "autores",
    timestamps: true, // cria automaticamente createdAt e updatedAt
  }
);

module.exports = Autor;
```

**Arquivo `modelos/Livro.js`:**

```javascript
// modelos/Livro.js
const { DataTypes } = require("sequelize");
const { sequelize } = require("../config/banco");

const Livro = sequelize.define(
  "Livro",
  {
    id: {
      type: DataTypes.INTEGER,
      primaryKey: true,
      autoIncrement: true,
    },
    titulo: {
      type: DataTypes.STRING(200),
      allowNull: false,
      validate: {
        notEmpty: { msg: "O título é obrigatório" },
        len: { args: [2, 200], msg: "O título deve ter entre 2 e 200 caracteres" },
      },
    },
    isbn: {
      type: DataTypes.STRING(20),
      allowNull: true,
      unique: true, // impede duplicidade
    },
    ano: {
      type: DataTypes.INTEGER,
      allowNull: false,
      validate: {
        min: { args: [1400], msg: "Ano anterior à imprensa" },
        max: {
          args: [new Date().getFullYear() + 1],
          msg: "Ano de publicação no futuro",
        },
      },
    },
    preco: {
      type: DataTypes.DECIMAL(10, 2),
      defaultValue: 0,
      validate: {
        min: { args: [0], msg: "O preço não pode ser negativo" },
      },
      // DECIMAL retorna texto em alguns dialetos; o getter normaliza
      // o valor para número, uniformizando o comportamento.
      get() {
        const valor = this.getDataValue("preco");
        return valor === null ? null : Number(valor);
      },
    },
    exemplares: {
      type: DataTypes.INTEGER,
      defaultValue: 1,
      validate: { min: 0 },
    },
    sinopse: {
      type: DataTypes.TEXT,
      allowNull: true,
    },
    disponivel: {
      type: DataTypes.BOOLEAN,
      defaultValue: true,
    },
  },
  {
    tableName: "livros",
    timestamps: true,

    // Índices aceleram consultas frequentes.
    indexes: [
      { fields: ["ano"] },
      { fields: ["titulo"] },
    ],

    // Escopos: filtros nomeados e reutilizáveis.
    scopes: {
      disponiveis: { where: { disponivel: true } },
      recentes: { where: { ano: { [require("sequelize").Op.gte]: 2000 } } },
    },

    // Hooks: funções executadas em momentos do ciclo de vida do registro.
    hooks: {
      beforeValidate(livro) {
        if (livro.titulo) livro.titulo = livro.titulo.trim();
        if (livro.isbn) livro.isbn = livro.isbn.replace(/[-\s]/g, "");
      },
      beforeSave(livro) {
        // Regra de negócio aplicada automaticamente.
        livro.disponivel = livro.exemplares > 0;
      },
    },
  }
);

// Método de instância: disponível em cada registro carregado.
Livro.prototype.descricaoCompleta = function () {
  return `${this.titulo} (${this.ano})`;
};

// Método de classe: invocado no próprio modelo.
Livro.buscarPorTitulo = function (termo) {
  const { Op } = require("sequelize");
  return this.findAll({
    where: { titulo: { [Op.like]: `%${termo}%` } },
    order: [["titulo", "ASC"]],
  });
};

module.exports = Livro;
```

### 4.1 Tipos de dados principais

| Tipo Sequelize | SQLite | MySQL | Uso |
|---|---|---|---|
| `STRING(n)` | `VARCHAR(n)` | `VARCHAR(n)` | Texto curto |
| `TEXT` | `TEXT` | `TEXT` | Texto longo |
| `INTEGER` | `INTEGER` | `INT` | Número inteiro |
| `BIGINT` | `INTEGER` | `BIGINT` | Inteiro grande |
| `FLOAT` / `DOUBLE` | `REAL` | `FLOAT` / `DOUBLE` | Número real aproximado |
| `DECIMAL(p, e)` | `DECIMAL` | `DECIMAL(p, e)` | Valores monetários |
| `BOOLEAN` | `TINYINT(1)` | `TINYINT(1)` | Verdadeiro ou falso |
| `DATE` | `DATETIME` | `DATETIME` | Data e hora |
| `DATEONLY` | `DATE` | `DATE` | Somente data |
| `ENUM(...)` | `TEXT` + `CHECK` | `ENUM` | Conjunto fechado de valores |
| `JSON` | `TEXT` | `JSON` | Estrutura serializada |

**Recomendação sobre valores monetários:** utilizar `DECIMAL`, e não `FLOAT`. A aritmética de ponto flutuante binária não representa exatamente valores decimais — em JavaScript, `0.1 + 0.2` resulta em `0.30000000000000004`. Em cálculos financeiros acumulados, esse desvio é inaceitável.

---

## 5. Relacionamentos

**Arquivo `modelos/index.js`:**

```javascript
// modelos/index.js — define as associações e exporta os modelos
const { sequelize } = require("../config/banco");

const Autor = require("./Autor");
const Livro = require("./Livro");
const Categoria = require("./Categoria");
const Emprestimo = require("./Emprestimo");

// ---------- UM PARA MUITOS ----------
// Um autor possui vários livros; cada livro pertence a um autor.
// A chave estrangeira autorId é criada na tabela "livros".
Autor.hasMany(Livro, {
  foreignKey: "autorId",
  as: "livros",
  onDelete: "CASCADE",  // excluir o autor remove seus livros
  onUpdate: "CASCADE",
});

Livro.belongsTo(Autor, {
  foreignKey: "autorId",
  as: "autor",
});

// ---------- MUITOS PARA MUITOS ----------
// Um livro pode pertencer a várias categorias e uma categoria pode
// conter vários livros. É criada a tabela intermediária livro_categorias.
Livro.belongsToMany(Categoria, {
  through: "livro_categorias",
  foreignKey: "livroId",
  otherKey: "categoriaId",
  as: "categorias",
  timestamps: false,
});

Categoria.belongsToMany(Livro, {
  through: "livro_categorias",
  foreignKey: "categoriaId",
  otherKey: "livroId",
  as: "livros",
  timestamps: false,
});

// ---------- UM PARA MUITOS (empréstimos) ----------
Livro.hasMany(Emprestimo, { foreignKey: "livroId", as: "emprestimos" });
Emprestimo.belongsTo(Livro, { foreignKey: "livroId", as: "livro" });

module.exports = { sequelize, Autor, Livro, Categoria, Emprestimo };
```

**Arquivo `modelos/Categoria.js`:**

```javascript
const { DataTypes } = require("sequelize");
const { sequelize } = require("../config/banco");

const Categoria = sequelize.define(
  "Categoria",
  {
    id: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
    nome: {
      type: DataTypes.STRING(60),
      allowNull: false,
      unique: true,
      validate: { notEmpty: { msg: "O nome da categoria é obrigatório" } },
    },
    cor: {
      type: DataTypes.STRING(7),
      defaultValue: "#2d6a4f",
      validate: {
        is: { args: /^#[0-9a-fA-F]{6}$/, msg: "Cor deve estar no formato #rrggbb" },
      },
    },
  },
  { tableName: "categorias", timestamps: false }
);

module.exports = Categoria;
```

**Arquivo `modelos/Emprestimo.js`:**

```javascript
const { DataTypes } = require("sequelize");
const { sequelize } = require("../config/banco");

const Emprestimo = sequelize.define(
  "Emprestimo",
  {
    id: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
    nomeLeitor: {
      type: DataTypes.STRING(120),
      allowNull: false,
      validate: { notEmpty: { msg: "O nome do leitor é obrigatório" } },
    },
    dataEmprestimo: {
      type: DataTypes.DATEONLY,
      allowNull: false,
      defaultValue: DataTypes.NOW,
    },
    dataPrevistaDevolucao: { type: DataTypes.DATEONLY, allowNull: false },
    dataDevolucao: { type: DataTypes.DATEONLY, allowNull: true },
    situacao: {
      type: DataTypes.ENUM("ativo", "devolvido", "atrasado"),
      defaultValue: "ativo",
    },
  },
  { tableName: "emprestimos", timestamps: true }
);

module.exports = Emprestimo;
```

---

## 6. Criação das tabelas e carga inicial

**Arquivo `scripts/inicializar.js`:**

```javascript
// scripts/inicializar.js — cria as tabelas e insere dados de exemplo
// Execução: node scripts/inicializar.js
const { sequelize, Autor, Livro, Categoria } = require("../modelos");

async function inicializar() {
  console.log("Verificando a conexão...");
  await sequelize.authenticate();

  // force: true APAGA e recria todas as tabelas.
  // Adequado apenas em desenvolvimento; em produção, utilizam-se migrações.
  console.log("Recriando as tabelas...");
  await sequelize.sync({ force: true });

  console.log("Inserindo categorias...");
  const categorias = await Categoria.bulkCreate([
    { nome: "Romance", cor: "#c1121f" },
    { nome: "Ficção científica", cor: "#003049" },
    { nome: "Técnico", cor: "#2d6a4f" },
    { nome: "Poesia", cor: "#7209b7" },
  ]);

  console.log("Inserindo autores...");
  const machado = await Autor.create({
    nome: "Machado de Assis",
    nacionalidade: "Brasileira",
    anoNascimento: 1839,
  });

  const clarice = await Autor.create({
    nome: "Clarice Lispector",
    nacionalidade: "Brasileira",
    anoNascimento: 1920,
  });

  const asimov = await Autor.create({
    nome: "Isaac Asimov",
    nacionalidade: "Norte-americana",
    anoNascimento: 1920,
  });

  console.log("Inserindo livros...");

  // create devolve a instância criada, permitindo encadear operações.
  const domCasmurro = await Livro.create({
    titulo: "Dom Casmurro",
    ano: 1899,
    preco: 39.9,
    exemplares: 5,
    isbn: "9788535910663",
    sinopse: "Bentinho narra sua suspeita a respeito de Capitu.",
    autorId: machado.id,
  });

  // setCategorias é gerado automaticamente pela associação
  // belongsToMany: preenche a tabela intermediária.
  await domCasmurro.setCategorias([categorias[0].id]);

  const horaDaEstrela = await Livro.create({
    titulo: "A Hora da Estrela",
    ano: 1977,
    preco: 34.5,
    exemplares: 3,
    autorId: clarice.id,
  });
  await horaDaEstrela.setCategorias([categorias[0].id]);

  const fundacao = await Livro.create({
    titulo: "Fundação",
    ano: 1951,
    preco: 52.0,
    exemplares: 2,
    autorId: asimov.id,
  });
  await fundacao.setCategorias([categorias[1].id]);

  // bulkCreate insere vários registros em uma única operação.
  // validate: true assegura a aplicação das validações do modelo.
  await Livro.bulkCreate(
    [
      { titulo: "Memórias Póstumas de Brás Cubas", ano: 1881, preco: 42.0,
        exemplares: 4, autorId: machado.id },
      { titulo: "Quincas Borba", ano: 1891, preco: 38.0,
        exemplares: 0, autorId: machado.id },
      { titulo: "Eu, Robô", ano: 1950, preco: 49.9,
        exemplares: 3, autorId: asimov.id },
    ],
    { validate: true }
  );

  const totalAutores = await Autor.count();
  const totalLivros = await Livro.count();

  console.log(`\nCarga concluída: ${totalAutores} autores, ${totalLivros} livros.`);
  await sequelize.close();
}

inicializar().catch((erro) => {
  console.error("Falha na inicialização:", erro.message);
  process.exit(1);
});
```

```powershell
node scripts/inicializar.js
```

### 6.1 Modos de sincronização

| Chamada | Efeito | Contexto |
|---|---|---|
| `sync()` | Cria as tabelas ausentes | Primeira execução |
| `sync({ alter: true })` | Ajusta as tabelas ao modelo | Desenvolvimento |
| `sync({ force: true })` | **Apaga** e recria as tabelas | Desenvolvimento, com perda de dados |
| Migrações | Alterações versionadas e reversíveis | **Produção** |

**Advertência:** `sync({ force: true })` executado em produção destrói todos os dados. Em ambiente produtivo, utilizam-se exclusivamente migrações (Seção 9).

---

## 7. Operações de consulta e escrita

**Arquivo `scripts/consultas.js`:**

```javascript
// scripts/consultas.js — demonstração das operações do Sequelize
const { Op } = require("sequelize");
const { sequelize, Autor, Livro, Categoria } = require("../modelos");

async function demonstrar() {
  console.log("\n===== LEITURA =====");

  // Todos os registros
  const todos = await Livro.findAll();
  console.log(`Total de livros: ${todos.length}`);

  // Busca pela chave primária
  const porId = await Livro.findByPk(1);
  console.log("findByPk(1):", porId?.titulo);

  // Primeiro registro que satisfaz a condição
  const primeiro = await Livro.findOne({ where: { ano: 1899 } });
  console.log("findOne(ano=1899):", primeiro?.titulo);

  // Seleção de colunas específicas, ordenação e limite
  const selecionados = await Livro.findAll({
    attributes: ["id", "titulo", "ano", "preco"],
    order: [["ano", "DESC"]],
    limit: 3,
  });
  console.table(selecionados.map((l) => l.toJSON()));

  console.log("\n===== OPERADORES =====");

  // Op.gte: maior ou igual
  const recentes = await Livro.findAll({ where: { ano: { [Op.gte]: 1950 } } });
  console.log(`Publicados a partir de 1950: ${recentes.length}`);

  // Op.between: intervalo fechado
  const faixaPreco = await Livro.findAll({
    where: { preco: { [Op.between]: [30, 45] } },
  });
  console.log(`Preço entre 30 e 45: ${faixaPreco.length}`);

  // Op.like: correspondência parcial
  const comMemoria = await Livro.findAll({
    where: { titulo: { [Op.like]: "%Memórias%" } },
  });
  console.log("Título contendo 'Memórias':", comMemoria.map((l) => l.titulo));

  // Op.or e Op.and: combinação de condições
  const combinado = await Livro.findAll({
    where: {
      [Op.and]: [
        { exemplares: { [Op.gt]: 0 } },
        { [Op.or]: [{ ano: { [Op.lt]: 1900 } }, { preco: { [Op.lt]: 40 } }] },
      ],
    },
  });
  console.log(`Consulta combinada: ${combinado.length} resultado(s)`);

  // Op.in: pertencimento a um conjunto
  const especificos = await Livro.findAll({ where: { id: { [Op.in]: [1, 3, 5] } } });
  console.log(`IDs 1, 3 e 5: ${especificos.length} encontrado(s)`);

  console.log("\n===== RELACIONAMENTOS (include) =====");

  // include gera um JOIN: carrega os dados relacionados na mesma consulta.
  const livrosComAutor = await Livro.findAll({
    include: [{ model: Autor, as: "autor", attributes: ["nome", "nacionalidade"] }],
    limit: 3,
  });

  for (const livro of livrosComAutor) {
    console.log(`  ${livro.titulo} — ${livro.autor.nome}`);
  }

  // Direção inversa: autores com seus livros
  const autoresComLivros = await Autor.findAll({
    include: [{ model: Livro, as: "livros", attributes: ["titulo", "ano"] }],
  });

  for (const autor of autoresComLivros) {
    console.log(`  ${autor.nome}: ${autor.livros.length} livro(s)`);
  }

  // Filtro aplicado à tabela relacionada.
  // required: true transforma o LEFT JOIN em INNER JOIN, retornando
  // apenas os autores que possuem livros correspondentes ao filtro.
  const autoresSeculoXIX = await Autor.findAll({
    include: [
      {
        model: Livro,
        as: "livros",
        where: { ano: { [Op.lt]: 1900 } },
        required: true,
      },
    ],
  });
  console.log(`Autores com obras anteriores a 1900: ${autoresSeculoXIX.length}`);

  console.log("\n===== AGREGAÇÕES =====");

  console.log("Quantidade de livros:", await Livro.count());
  console.log("Disponíveis:", await Livro.count({ where: { disponivel: true } }));
  console.log("Soma dos preços:", await Livro.sum("preco"));
  console.log("Preço médio:", (await Livro.findAll({
    attributes: [[sequelize.fn("AVG", sequelize.col("preco")), "media"]],
    raw: true,
  }))[0].media);
  console.log("Ano mais antigo:", await Livro.min("ano"));
  console.log("Ano mais recente:", await Livro.max("ano"));

  // Agrupamento: quantidade de livros por autor
  const porAutor = await Livro.findAll({
    attributes: [
      "autorId",
      [sequelize.fn("COUNT", sequelize.col("Livro.id")), "quantidade"],
    ],
    include: [{ model: Autor, as: "autor", attributes: ["nome"] }],
    group: ["autorId", "autor.id"],
    raw: true,
    nest: true,
  });
  console.table(porAutor);

  console.log("\n===== PAGINAÇÃO =====");

  const pagina = 1;
  const porPagina = 3;

  // findAndCountAll devolve os registros da página e o total geral,
  // valor necessário para calcular a quantidade de páginas.
  const { count, rows } = await Livro.findAndCountAll({
    limit: porPagina,
    offset: (pagina - 1) * porPagina,
    order: [["titulo", "ASC"]],
  });

  console.log(`Página ${pagina} de ${Math.ceil(count / porPagina)} — ${count} registros`);
  rows.forEach((l) => console.log(`  ${l.titulo}`));

  console.log("\n===== ESCRITA =====");

  const novo = await Livro.create({
    titulo: "O Cortiço",
    ano: 1890,
    preco: 29.9,
    exemplares: 2,
    autorId: 1,
  });
  console.log("Criado com id:", novo.id);

  // Atualização de uma instância já carregada
  novo.preco = 31.5;
  novo.exemplares = 4;
  await novo.save();
  console.log("Atualizado. Preço:", novo.preco);

  // Atualização em massa; devolve a quantidade de linhas afetadas
  const [afetadas] = await Livro.update(
    { disponivel: false },
    { where: { exemplares: 0 } }
  );
  console.log(`Marcados como indisponíveis: ${afetadas}`);

  // findOrCreate: busca e, se não encontrar, cria
  const [categoria, foiCriada] = await Categoria.findOrCreate({
    where: { nome: "Biografia" },
    defaults: { cor: "#8b5cf6" },
  });
  console.log(`Categoria "${categoria.nome}" ${foiCriada ? "criada" : "já existia"}`);

  // Exclusão
  await novo.destroy();
  console.log("Registro removido.");

  console.log("\n===== TRANSAÇÕES =====");

  // Uma transação garante que todas as operações sejam efetivadas
  // em conjunto ou nenhuma delas o seja.
  const transacao = await sequelize.transaction();

  try {
    const autor = await Autor.create(
      { nome: "Autor de Teste", nacionalidade: "Brasileira" },
      { transaction: transacao }
    );

    await Livro.create(
      { titulo: "Obra de Teste", ano: 2026, preco: 10, autorId: autor.id },
      { transaction: transacao }
    );

    // Falha proposital: ano inválido viola a validação do modelo.
    await Livro.create(
      { titulo: "Obra Inválida", ano: 3000, preco: 10, autorId: autor.id },
      { transaction: transacao }
    );

    await transacao.commit();
    console.log("Transação efetivada.");
  } catch (erro) {
    await transacao.rollback();
    console.log("Transação desfeita:", erro.message);
    console.log("Nenhum dos registros foi gravado, inclusive o autor.");
  }

  console.log("\n===== SQL PURO =====");

  // Recurso reservado a consultas que o ORM expressa com dificuldade.
  // Os parâmetros DEVEM ser passados em replacements, jamais concatenados,
  // sob pena de injeção de SQL.
  const [resultado] = await sequelize.query(
    `SELECT a.nome AS autor, COUNT(l.id) AS total, AVG(l.preco) AS media
     FROM autores a
     LEFT JOIN livros l ON l.autorId = a.id
     GROUP BY a.id, a.nome
     HAVING COUNT(l.id) > :minimo
     ORDER BY total DESC`,
    { replacements: { minimo: 0 } }
  );
  console.table(resultado);

  await sequelize.close();
}

demonstrar().catch((erro) => {
  console.error("Erro:", erro);
  process.exit(1);
});
```

### 7.1 Operadores de consulta

| Operador | SQL equivalente | Exemplo |
|---|---|---|
| `Op.eq` | `=` | `{ ano: { [Op.eq]: 1899 } }` |
| `Op.ne` | `<>` | `{ ano: { [Op.ne]: 1899 } }` |
| `Op.gt` / `Op.gte` | `>` / `>=` | `{ preco: { [Op.gte]: 30 } }` |
| `Op.lt` / `Op.lte` | `<` / `<=` | `{ ano: { [Op.lt]: 1900 } }` |
| `Op.between` | `BETWEEN` | `{ preco: { [Op.between]: [10, 50] } }` |
| `Op.in` / `Op.notIn` | `IN` / `NOT IN` | `{ id: { [Op.in]: [1, 2, 3] } }` |
| `Op.like` | `LIKE` | `{ titulo: { [Op.like]: "%Dom%" } }` |
| `Op.is` | `IS NULL` | `{ dataDevolucao: { [Op.is]: null } }` |
| `Op.not` | `NOT` | `{ disponivel: { [Op.not]: true } }` |
| `Op.and` / `Op.or` | `AND` / `OR` | `{ [Op.or]: [{ a: 1 }, { b: 2 }] }` |

---

## 8. O problema N+1

Trata-se do erro de desempenho mais frequente no uso de ORMs.

```javascript
// INCORRETO: uma consulta para os livros e uma consulta adicional
// por livro para obter o autor. Com 100 livros, são 101 consultas.
const livros = await Livro.findAll();
for (const livro of livros) {
  const autor = await Autor.findByPk(livro.autorId); // consulta em laço
  console.log(`${livro.titulo} — ${autor.nome}`);
}

// CORRETO: uma única consulta com JOIN.
const livros = await Livro.findAll({
  include: [{ model: Autor, as: "autor" }],
});
for (const livro of livros) {
  console.log(`${livro.titulo} — ${livro.autor.nome}`);
}
```

**Método de detecção:** manter `logging` ativo durante o desenvolvimento. A repetição de consultas idênticas variando apenas o parâmetro caracteriza o problema. A correção consiste em acrescentar `include`.

---

## 9. Migrações

O uso de `sync({ force: true })` é inadmissível em produção, pois apaga os dados. As **migrações** registram as alterações do esquema em arquivos versionados, aplicáveis e reversíveis.

**Arquivo `.sequelizerc`:**

```javascript
const path = require("path");

module.exports = {
  config: path.resolve("config", "config.js"),
  "models-path": path.resolve("modelos"),
  "migrations-path": path.resolve("migracoes"),
  "seeders-path": path.resolve("seeders"),
};
```

**Arquivo `config/config.js`:**

```javascript
require("dotenv").config();

module.exports = {
  development: {
    dialect: process.env.DB_DIALECT || "sqlite",
    storage: process.env.DB_STORAGE || "./dados/catalogo.sqlite",
    host: process.env.DB_HOST,
    database: process.env.DB_NAME,
    username: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
  },
  production: {
    dialect: "mysql",
    host: process.env.DB_HOST,
    port: Number(process.env.DB_PORT) || 3306,
    database: process.env.DB_NAME,
    username: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    dialectOptions: { ssl: { rejectUnauthorized: true } },
  },
};
```

Criação de uma migração:

```powershell
npx sequelize-cli migration:generate --name criar-tabela-editoras
```

**Arquivo `migracoes/XXXXXXXX-criar-tabela-editoras.js`:**

```javascript
"use strict";

module.exports = {
  // up: aplica a alteração
  async up(queryInterface, Sequelize) {
    await queryInterface.createTable("editoras", {
      id: {
        type: Sequelize.INTEGER,
        primaryKey: true,
        autoIncrement: true,
        allowNull: false,
      },
      nome: { type: Sequelize.STRING(120), allowNull: false },
      cidade: { type: Sequelize.STRING(80) },
      createdAt: { type: Sequelize.DATE, allowNull: false },
      updatedAt: { type: Sequelize.DATE, allowNull: false },
    });

    // Acréscimo da chave estrangeira na tabela existente
    await queryInterface.addColumn("livros", "editoraId", {
      type: Sequelize.INTEGER,
      allowNull: true,
      references: { model: "editoras", key: "id" },
      onDelete: "SET NULL",
      onUpdate: "CASCADE",
    });

    await queryInterface.addIndex("livros", ["editoraId"]);
  },

  // down: desfaz a alteração, na ordem inversa
  async down(queryInterface) {
    await queryInterface.removeIndex("livros", ["editoraId"]);
    await queryInterface.removeColumn("livros", "editoraId");
    await queryInterface.dropTable("editoras");
  },
};
```

Comandos:

```powershell
npx sequelize-cli db:migrate           # aplica as migrações pendentes
npx sequelize-cli db:migrate:status    # exibe a situação de cada migração
npx sequelize-cli db:migrate:undo      # desfaz a última
npx sequelize-cli db:migrate:undo:all  # desfaz todas
```

**Regra operacional:** toda migração deve possuir um `down` funcional. Uma migração irreversível impede o retorno a uma versão anterior em caso de falha na implantação.

---

## 10. Integração com o Express

**Arquivo `servicos/livrosServico.js`:**

```javascript
// servicos/livrosServico.js
// Compare-se com a versão em memória do Tutorial 5: a interface das
// funções permanece idêntica; apenas a implementação foi substituída.
const { Op } = require("sequelize");
const { Livro, Autor, Categoria } = require("../modelos");

async function listar({ termo, ano, autorId, pagina = 1, porPagina = 10 } = {}) {
  const where = {};

  if (termo) where.titulo = { [Op.like]: `%${termo}%` };
  if (ano) where.ano = Number(ano);
  if (autorId) where.autorId = Number(autorId);

  const { count, rows } = await Livro.findAndCountAll({
    where,
    include: [
      { model: Autor, as: "autor", attributes: ["id", "nome"] },
      { model: Categoria, as: "categorias", attributes: ["id", "nome", "cor"],
        through: { attributes: [] } }, // omite as colunas da tabela intermediária
    ],
    limit: Number(porPagina),
    offset: (Number(pagina) - 1) * Number(porPagina),
    order: [["titulo", "ASC"]],
    distinct: true, // necessário para contagem correta com belongsToMany
  });

  return {
    dados: rows,
    total: count,
    pagina: Number(pagina),
    totalPaginas: Math.ceil(count / Number(porPagina)),
  };
}

async function buscarPorId(id) {
  return Livro.findByPk(id, {
    include: [
      { model: Autor, as: "autor" },
      { model: Categoria, as: "categorias", through: { attributes: [] } },
    ],
  });
}

async function criar(dados) {
  const livro = await Livro.create(dados);
  if (Array.isArray(dados.categorias)) {
    await livro.setCategorias(dados.categorias);
  }
  return buscarPorId(livro.id);
}

async function atualizar(id, dados) {
  const livro = await Livro.findByPk(id);
  if (!livro) return null;

  await livro.update(dados);
  if (Array.isArray(dados.categorias)) {
    await livro.setCategorias(dados.categorias);
  }
  return buscarPorId(id);
}

async function remover(id) {
  const livro = await Livro.findByPk(id);
  if (!livro) return false;
  await livro.destroy();
  return true;
}

async function estatisticas() {
  return {
    totalLivros: await Livro.count(),
    disponiveis: await Livro.count({ where: { disponivel: true } }),
    totalAutores: await Autor.count(),
    valorAcervo: (await Livro.sum("preco")) || 0,
  };
}

module.exports = { listar, buscarPorId, criar, atualizar, remover, estatisticas };
```

**Arquivo `app.js`:**

```javascript
require("dotenv").config();

const express = require("express");
const path = require("node:path");

const { sequelize, testarConexao } = require("./config/banco");
require("./modelos"); // registra os modelos e as associações

const app = express();
const PORTA = process.env.PORT || 3000;

app.set("view engine", "ejs");
app.set("views", path.join(__dirname, "views"));
app.use(express.static(path.join(__dirname, "publico")));
app.use(express.urlencoded({ extended: true }));
app.use(express.json());

app.use("/livros", require("./rotas/livros"));

app.get("/", async (req, res, next) => {
  try {
    const servico = require("./servicos/livrosServico");
    res.render("index", {
      titulo: "Catálogo",
      estatisticas: await servico.estatisticas(),
    });
  } catch (erro) {
    next(erro);
  }
});

app.use((req, res) => {
  res.status(404).render("erro", { titulo: "Não encontrado", codigo: 404,
    mensagem: "Página inexistente." });
});

app.use((erro, req, res, next) => {
  console.error(erro);

  // Erros de validação do Sequelize recebem tratamento específico,
  // permitindo devolver ao usuário as mensagens declaradas nos modelos.
  if (erro.name === "SequelizeValidationError") {
    return res.status(400).render("erro", {
      titulo: "Dados inválidos",
      codigo: 400,
      mensagem: erro.errors.map((e) => e.message).join("; "),
    });
  }

  if (erro.name === "SequelizeUniqueConstraintError") {
    return res.status(409).render("erro", {
      titulo: "Registro duplicado",
      codigo: 409,
      mensagem: "Já existe um registro com esse valor único.",
    });
  }

  res.status(500).render("erro", { titulo: "Erro", codigo: 500,
    mensagem: "Falha no processamento." });
});

// Inicialização: a conexão é verificada antes de aceitar requisições.
async function iniciar() {
  if (!(await testarConexao())) {
    console.error("Encerrando: banco de dados indisponível.");
    process.exit(1);
  }

  // Em desenvolvimento, alter mantém as tabelas alinhadas aos modelos.
  if (process.env.NODE_ENV === "development") {
    await sequelize.sync({ alter: true });
  }

  app.listen(PORTA, () => console.log(`http://localhost:${PORTA}`));
}

iniciar();

// Encerramento controlado: fecha a conexão com o banco.
process.on("SIGINT", async () => {
  console.log("\nEncerrando...");
  await sequelize.close();
  process.exit(0);
});
```

---

## 11. Migração para o MySQL

### 11.1 Instalação

```powershell
winget install Oracle.MySQL
```

Alternativa: XAMPP (<https://www.apachefriends.org>), que inclui MySQL e phpMyAdmin, com interface gráfica adequada ao ambiente escolar.

### 11.2 Criação do banco

```sql
CREATE DATABASE catalogo
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE USER 'app_catalogo'@'localhost' IDENTIFIED BY 'senha_forte_aqui';
GRANT ALL PRIVILEGES ON catalogo.* TO 'app_catalogo'@'localhost';
FLUSH PRIVILEGES;
```

**Boa prática:** criar um usuário específico para a aplicação, com privilégios restritos ao banco necessário. O usuário `root` não deve ser utilizado por aplicações.

### 11.3 Alternância

Alterar apenas o arquivo `.env`:

```env
DB_DIALECT=mysql
DB_HOST=localhost
DB_PORT=3306
DB_NAME=catalogo
DB_USER=app_catalogo
DB_PASSWORD=senha_forte_aqui
```

```powershell
node scripts/inicializar.js
npm run dev
```

**Nenhuma linha de código da aplicação é modificada.** Esse é o benefício central do ORM: a portabilidade entre sistemas de banco de dados.

### 11.4 Diferenças remanescentes

| Aspecto | SQLite | MySQL |
|---|---|---|
| `Op.like` | Insensível a maiúsculas em ASCII | Depende do *collation* |
| `ENUM` | Emulado por `TEXT` com restrição | Tipo nativo |
| Chaves estrangeiras | Exigem `PRAGMA foreign_keys = ON` | Ativas por padrão no InnoDB |
| `ALTER TABLE` | Suporte limitado | Suporte completo |
| Escrita concorrente | Um escritor por vez | Múltiplos escritores |

Ativação das chaves estrangeiras no SQLite:

```javascript
sequelize = new Sequelize({
  dialect: "sqlite",
  storage: caminhoArquivo,
  dialectOptions: {
    // Executado a cada nova conexão
  },
});

// Após a conexão:
await sequelize.query("PRAGMA foreign_keys = ON");
```

---

## 12. Laboratório prático

### Laboratório 6.1 — Sistema de biblioteca

Completar o sistema iniciado neste tutorial, incorporando empréstimos.

**Funcionalidades exigidas:**

1. CRUD completo de autores, livros e categorias;
2. registro de empréstimo, que decremente o número de exemplares disponíveis;
3. registro de devolução, que restaure o exemplar e defina `dataDevolucao`;
4. listagem de empréstimos em atraso — `situacao = "ativo"` e `dataPrevistaDevolucao` anterior à data corrente;
5. impedimento do empréstimo quando `exemplares` for zero;
6. busca de livros por título, autor e categoria, com filtros combináveis;
7. paginação de dez registros por página, com navegação;
8. painel com: total de livros, exemplares emprestados, empréstimos em atraso e cinco livros mais emprestados.

**Requisitos técnicos:**
- empréstimo e devolução implementados dentro de **transação**;
- validações declaradas nos modelos, não nos controladores;
- `include` utilizado em todas as listagens que exibam dados relacionados — nenhuma consulta dentro de laço;
- exclusão de autor com livros deve ser bloqueada com mensagem apropriada, ou remover os livros em cascata, conforme decisão documentada.

### Laboratório 6.2 — Comparação entre SQLite e MySQL

1. Executar o sistema com `DB_DIALECT=sqlite` e povoá-lo com quinhentos livros gerados por script.
2. Medir, com `console.time`, o tempo de: inserção dos quinhentos registros; listagem paginada; busca por título com `LIKE`; agregação por autor.
3. Repetir integralmente com `DB_DIALECT=mysql`.
4. Elaborar tabela comparativa e gráfico.
5. Redigir análise de uma página respondendo: em que situações cada banco é preferível? Quais operações apresentaram maior diferença? A que se atribui a diferença?

### Laboratório 6.3 — Investigação do problema N+1

1. Implementar deliberadamente uma listagem de livros que consulte o autor dentro do laço.
2. Ativar `logging` e registrar a quantidade de consultas geradas.
3. Medir o tempo total com cem livros.
4. Corrigir com `include`.
5. Repetir as medições e apresentar tabela comparativa de quantidade de consultas e tempo.
6. Explicar por que a diferença se acentua conforme o volume de dados cresce.

---

## 13. Síntese

1. Um ORM traduz entre tabelas relacionais e objetos, oferecendo portabilidade, validações e proteção contra injeção de SQL.
2. Os modelos definem estrutura, validações, índices, escopos e *hooks*.
3. Relacionamentos são declarados por `hasMany`, `belongsTo` e `belongsToMany`, e consultados com `include`.
4. `sync({ force: true })` destina-se exclusivamente ao desenvolvimento; produção exige migrações versionadas com `up` e `down`.
5. Operações que devem ser efetivadas em conjunto exigem transações.
6. Valores monetários devem utilizar `DECIMAL`, jamais `FLOAT`.
7. O problema N+1 é detectado pelo registro de SQL e corrigido com `include`.
8. A separação em camadas permite alternar de SQLite para MySQL alterando apenas o arquivo `.env`.

**Próximo tutorial:** [API REST com Node.js e SQLite](./07-api-rest-sqlite.md)
