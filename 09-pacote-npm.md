# Tutorial 9 — Criação, Publicação e Uso de um Pacote npm

> **Pré-requisitos:** Tutoriais 1 a 8 concluídos; conta no GitHub.
> **Duração estimada:** 3 aulas de 50 minutos.
> **Ao final deste tutorial, o estudante será capaz de:** estruturar uma biblioteca reutilizável, redigir um `package.json` de publicação, escrever testes automatizados com o executor nativo, documentar a interface pública, testar o pacote localmente com `npm link`, publicar no registro público, aplicar versionamento semântico e consumir o pacote publicado em outro projeto.

---

## 1. O que será construído

Será desenvolvido, publicado e consumido o pacote **`utilitarios-br`**, uma biblioteca de funções para dados brasileiros:

- validação e formatação de CPF;
- validação e formatação de CEP;
- geração de identificadores de URL (*slugs*) a partir de texto acentuado;
- formatação de valores monetários e de datas no padrão brasileiro;
- interface de linha de comando para uso direto no terminal.

O objetivo não é a biblioteca em si, mas o domínio do processo: da estruturação do código à publicação e ao consumo por terceiros.

---

## 2. Anatomia de um pacote

Um pacote npm é um diretório contendo um `package.json` e o código correspondente. Três características o distinguem de um projeto comum:

1. **Interface pública explícita.** O campo `exports` determina o que pode ser importado; o restante permanece interno.
2. **Versionamento semântico.** Cada publicação recebe uma versão que comunica a natureza da mudança.
3. **Independência de contexto.** O pacote não pode depender de arquivos, variáveis de ambiente ou estruturas de pastas do projeto que o consome.

---

## 3. Estrutura do projeto

```
utilitarios-br/
├── package.json
├── README.md
├── LICENSE
├── CHANGELOG.md
├── .gitignore
├── .npmignore
├── src/
│   ├── index.js          ← ponto de entrada, reexporta os módulos
│   ├── documentos.js     ← CPF e CEP
│   ├── texto.js          ← slug e normalização
│   ├── formatacao.js     ← moeda e datas
│   └── cli.js            ← interface de linha de comando
└── testes/
    ├── documentos.test.js
    ├── texto.test.js
    └── formatacao.test.js
```

```powershell
mkdir utilitarios-br
cd utilitarios-br
npm init -y
mkdir src testes
code .
```

---

## 4. O arquivo `package.json`

```json
{
  "name": "utilitarios-br",
  "version": "0.1.0",
  "description": "Funções utilitárias para dados brasileiros: validação de CPF e CEP, geração de slugs e formatação de moeda e datas.",
  "type": "module",
  "main": "./src/index.js",
  "exports": {
    ".": "./src/index.js",
    "./documentos": "./src/documentos.js",
    "./texto": "./src/texto.js",
    "./formatacao": "./src/formatacao.js"
  },
  "bin": {
    "utilbr": "./src/cli.js"
  },
  "files": [
    "src",
    "README.md",
    "LICENSE"
  ],
  "scripts": {
    "test": "node --test testes/",
    "test:watch": "node --test --watch testes/",
    "verificar": "npm test && npm pack --dry-run"
  },
  "keywords": ["cpf", "cep", "slug", "brasil", "validacao", "formatacao"],
  "author": "Nome do Estudante <email@exemplo.com>",
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "git+https://github.com/USUARIO/utilitarios-br.git"
  },
  "bugs": {
    "url": "https://github.com/USUARIO/utilitarios-br/issues"
  },
  "homepage": "https://github.com/USUARIO/utilitarios-br#readme",
  "engines": {
    "node": ">=18.0.0"
  }
}
```

### 4.1 Campos relevantes para publicação

| Campo | Função |
|---|---|
| `name` | Identificador único no registro. Minúsculas, sem espaços |
| `version` | Versão SemVer. Cada publicação exige versão nova |
| `type` | `"module"` ativa o ESM, padrão atual para novos pacotes |
| `main` | Ponto de entrada para ferramentas antigas |
| `exports` | Interface pública. Arquivos não listados são inacessíveis |
| `bin` | Mapeia comandos de terminal a arquivos executáveis |
| `files` | Arquivos incluídos no pacote publicado |
| `engines` | Versão mínima do Node.js exigida |
| `keywords` | Termos de busca no registro |

**Distinção entre `files` e `.npmignore`:** o campo `files` opera por inclusão — apenas o listado é publicado. O `.npmignore` opera por exclusão. O primeiro é preferível, por ser explícito: um arquivo novo só é publicado se for deliberadamente incluído. Independentemente da configuração, `package.json`, `README` e `LICENSE` são sempre incluídos, e `node_modules` sempre excluído.

**Importância de `exports`:** sem ele, qualquer arquivo interno pode ser importado por terceiros. Se `src/interno/auxiliar.js` for importado por um consumidor, sua alteração passa a constituir mudança incompatível, ainda que se trate de código interno. O campo `exports` delimita o compromisso público.

### 4.2 Verificação de disponibilidade do nome

```powershell
npm view utilitarios-br
```

Se o pacote existir, seus dados são exibidos; caso contrário, retorna-se erro `E404`, indicando nome disponível.

Nomes genéricos estão, em sua maioria, ocupados. A alternativa é o **pacote com escopo**, vinculado ao nome de usuário:

```json
{ "name": "@seu-usuario/utilitarios-br" }
```

Pacotes com escopo são privados por padrão. A publicação pública exige a opção `--access public`.

---

## 5. Implementação

**Arquivo `src/documentos.js`:**

```javascript
// src/documentos.js — validação e formatação de documentos brasileiros

/**
 * Remove todos os caracteres não numéricos de um valor.
 * @param {string|number} valor
 * @returns {string} apenas os dígitos
 */
export function apenasDigitos(valor) {
  return String(valor ?? "").replace(/\D/g, "");
}

/**
 * Valida um CPF pelo algoritmo dos dígitos verificadores.
 *
 * O CPF possui onze dígitos: nove de base e dois verificadores.
 * Cada verificador resulta da soma ponderada dos dígitos anteriores,
 * com pesos decrescentes, seguida de operação modular.
 *
 * @param {string} cpf com ou sem pontuação
 * @returns {boolean}
 *
 * @example
 * validarCPF("529.982.247-25"); // true
 * validarCPF("111.111.111-11"); // false
 */
export function validarCPF(cpf) {
  const numeros = apenasDigitos(cpf);

  if (numeros.length !== 11) return false;

  // Sequências de dígitos idênticos satisfazem o cálculo matemático,
  // porém não constituem CPFs válidos e são rejeitadas explicitamente.
  if (/^(\d)\1{10}$/.test(numeros)) return false;

  /**
   * Calcula um dígito verificador.
   * @param {string} base dígitos considerados no cálculo
   * @param {number} pesoInicial peso do primeiro dígito
   */
  const calcularDigito = (base, pesoInicial) => {
    let soma = 0;
    for (let i = 0; i < base.length; i++) {
      soma += Number(base[i]) * (pesoInicial - i);
    }
    const resto = (soma * 10) % 11;
    return resto === 10 ? 0 : resto;
  };

  const primeiro = calcularDigito(numeros.slice(0, 9), 10);
  if (primeiro !== Number(numeros[9])) return false;

  const segundo = calcularDigito(numeros.slice(0, 10), 11);
  return segundo === Number(numeros[10]);
}

/**
 * Formata um CPF no padrão 000.000.000-00.
 * @param {string} cpf
 * @returns {string}
 * @throws {Error} quando o valor não possui onze dígitos
 */
export function formatarCPF(cpf) {
  const n = apenasDigitos(cpf);

  if (n.length !== 11) {
    throw new Error(`CPF deve conter 11 dígitos; foram recebidos ${n.length}`);
  }

  return `${n.slice(0, 3)}.${n.slice(3, 6)}.${n.slice(6, 9)}-${n.slice(9)}`;
}

/**
 * Verifica se um CEP possui oito dígitos.
 * @param {string} cep
 * @returns {boolean}
 */
export function validarCEP(cep) {
  const n = apenasDigitos(cep);
  return n.length === 8 && !/^(\d)\1{7}$/.test(n);
}

/**
 * Formata um CEP no padrão 00000-000.
 * @param {string} cep
 * @returns {string}
 * @throws {Error} quando o valor não possui oito dígitos
 */
export function formatarCEP(cep) {
  const n = apenasDigitos(cep);

  if (n.length !== 8) {
    throw new Error(`CEP deve conter 8 dígitos; foram recebidos ${n.length}`);
  }

  return `${n.slice(0, 5)}-${n.slice(5)}`;
}
```

**Arquivo `src/texto.js`:**

```javascript
// src/texto.js — manipulação de texto

/**
 * Remove os sinais diacríticos de um texto.
 *
 * A normalização NFD decompõe caracteres acentuados em duas partes:
 * a letra base e o sinal. Os sinais ocupam a faixa Unicode
 * U+0300 a U+036F e podem então ser removidos.
 *
 * @param {string} texto
 * @returns {string}
 *
 * @example
 * removerAcentos("informação"); // "informacao"
 */
export function removerAcentos(texto) {
  return String(texto ?? "")
    .normalize("NFD")
    .replace(/[\u0300-\u036f]/g, "");
}

/**
 * Converte um texto em identificador adequado a URLs.
 *
 * @param {string} texto
 * @returns {string}
 *
 * @example
 * gerarSlug("Instituto Federal de Brasília!"); // "instituto-federal-de-brasilia"
 */
export function gerarSlug(texto) {
  return removerAcentos(texto)
    .toLowerCase()
    .replace(/[^a-z0-9\s-]/g, "")  // descarta pontuação e símbolos
    .trim()
    .replace(/\s+/g, "-")          // espaços tornam-se hifens
    .replace(/-+/g, "-");          // hifens consecutivos são condensados
}

/**
 * Aplica capitalização de nome próprio, preservando em minúsculas as
 * preposições de uso corrente na língua portuguesa.
 *
 * @param {string} nome
 * @returns {string}
 *
 * @example
 * capitalizarNome("MARIA DA SILVA E SOUZA"); // "Maria da Silva e Souza"
 */
export function capitalizarNome(nome) {
  const excecoes = new Set(["da", "de", "do", "das", "dos", "e", "di", "du"]);

  return String(nome ?? "")
    .toLowerCase()
    .split(/\s+/)
    .filter(Boolean)
    .map((palavra, indice) => {
      if (indice > 0 && excecoes.has(palavra)) return palavra;
      return palavra.charAt(0).toUpperCase() + palavra.slice(1);
    })
    .join(" ");
}

/**
 * Trunca um texto, acrescentando reticências quando necessário.
 * O corte ocorre no último espaço anterior ao limite, evitando
 * partir palavras ao meio.
 *
 * @param {string} texto
 * @param {number} limite quantidade máxima de caracteres
 * @returns {string}
 */
export function truncar(texto, limite = 100) {
  const t = String(texto ?? "");

  if (t.length <= limite) return t;

  const cortado = t.slice(0, limite);
  const ultimoEspaco = cortado.lastIndexOf(" ");

  return (ultimoEspaco > 0 ? cortado.slice(0, ultimoEspaco) : cortado) + "…";
}
```

**Arquivo `src/formatacao.js`:**

```javascript
// src/formatacao.js — formatação de valores e datas no padrão brasileiro

/**
 * Formata um número como valor monetário em reais.
 * @param {number} valor
 * @returns {string}
 *
 * @example
 * formatarMoeda(1234.5); // "R$ 1.234,50"
 */
export function formatarMoeda(valor) {
  const numero = Number(valor);

  if (!Number.isFinite(numero)) {
    throw new TypeError("O valor informado não é um número finito");
  }

  return new Intl.NumberFormat("pt-BR", {
    style: "currency",
    currency: "BRL",
  }).format(numero);
}

/**
 * Formata uma data no padrão dd/mm/aaaa.
 * @param {Date|string|number} data
 * @returns {string}
 */
export function formatarData(data) {
  const d = data instanceof Date ? data : new Date(data);

  if (Number.isNaN(d.getTime())) {
    throw new TypeError("Data inválida");
  }

  return new Intl.DateTimeFormat("pt-BR").format(d);
}

/**
 * Descreve um instante em relação ao momento presente.
 * @param {Date|string|number} data
 * @returns {string}
 *
 * @example
 * tempoRelativo(new Date(Date.now() - 3600_000)); // "há 1 hora"
 */
export function tempoRelativo(data) {
  const d = data instanceof Date ? data : new Date(data);

  if (Number.isNaN(d.getTime())) {
    throw new TypeError("Data inválida");
  }

  const segundos = Math.round((d.getTime() - Date.now()) / 1000);
  const formatador = new Intl.RelativeTimeFormat("pt-BR", { numeric: "auto" });

  const unidades = [
    ["year", 31_536_000],
    ["month", 2_592_000],
    ["day", 86_400],
    ["hour", 3_600],
    ["minute", 60],
    ["second", 1],
  ];

  for (const [unidade, fator] of unidades) {
    if (Math.abs(segundos) >= fator || unidade === "second") {
      return formatador.format(Math.round(segundos / fator), unidade);
    }
  }
}
```

**Arquivo `src/index.js`:**

```javascript
// src/index.js — ponto de entrada do pacote.
// Reexporta a interface pública, permitindo importar tudo de um só lugar.

export {
  apenasDigitos,
  validarCPF,
  formatarCPF,
  validarCEP,
  formatarCEP,
} from "./documentos.js";

export {
  removerAcentos,
  gerarSlug,
  capitalizarNome,
  truncar,
} from "./texto.js";

export {
  formatarMoeda,
  formatarData,
  tempoRelativo,
} from "./formatacao.js";
```

---

## 6. Interface de linha de comando

**Arquivo `src/cli.js`:**

```javascript
#!/usr/bin/env node
// A linha acima (shebang) indica ao sistema qual interpretador utilizar.
// É obrigatória em arquivos declarados no campo "bin" do package.json.

import {
  validarCPF, formatarCPF, validarCEP, formatarCEP,
  gerarSlug, capitalizarNome, formatarMoeda,
} from "./index.js";

const [, , comando, ...argumentos] = process.argv;

const AJUDA = `
utilitarios-br — utilitários para dados brasileiros

Uso:
  utilbr <comando> <valor>

Comandos:
  cpf <numero>        Valida e formata um CPF
  cep <numero>        Valida e formata um CEP
  slug <texto>        Gera um identificador de URL
  nome <texto>        Aplica capitalização de nome próprio
  moeda <numero>      Formata um valor em reais
  ajuda               Exibe esta mensagem

Exemplos:
  utilbr cpf 52998224725
  utilbr slug "Instituto Federal de Brasília"
  utilbr moeda 1234.5
`;

// Os argumentos são reunidos para aceitar textos com espaços sem aspas.
const valor = argumentos.join(" ");

switch (comando) {
  case "cpf": {
    if (!valor) {
      console.error("Informe um CPF.");
      process.exit(1);
    }

    const valido = validarCPF(valor);
    console.log(`CPF     : ${valor}`);
    console.log(`Situação: ${valido ? "válido" : "inválido"}`);

    if (valido) console.log(`Formatado: ${formatarCPF(valor)}`);

    // Código de saída diferente de zero permite encadear o comando
    // em scripts: "utilbr cpf X && echo ok".
    process.exit(valido ? 0 : 1);
  }

  case "cep": {
    const valido = validarCEP(valor);
    console.log(`Situação: ${valido ? "válido" : "inválido"}`);
    if (valido) console.log(`Formatado: ${formatarCEP(valor)}`);
    process.exit(valido ? 0 : 1);
  }

  case "slug":
    console.log(gerarSlug(valor));
    break;

  case "nome":
    console.log(capitalizarNome(valor));
    break;

  case "moeda":
    try {
      console.log(formatarMoeda(valor));
    } catch (erro) {
      console.error(erro.message);
      process.exit(1);
    }
    break;

  case "ajuda":
  case "--help":
  case "-h":
  case undefined:
    console.log(AJUDA);
    break;

  default:
    console.error(`Comando desconhecido: ${comando}`);
    console.log(AJUDA);
    process.exit(1);
}
```

Teste local:

```powershell
node src/cli.js cpf 52998224725
node src/cli.js slug "Instituto Federal de Brasília"
node src/cli.js moeda 1234.5
```

---

## 7. Testes automatizados

O Node.js dispõe de executor de testes nativo desde a versão 18, dispensando bibliotecas externas.

**Arquivo `testes/documentos.test.js`:**

```javascript
import { test, describe } from "node:test";
import assert from "node:assert/strict";

import {
  validarCPF, formatarCPF, validarCEP, formatarCEP, apenasDigitos,
} from "../src/documentos.js";

describe("validarCPF", () => {
  test("aceita CPF válido com pontuação", () => {
    assert.equal(validarCPF("529.982.247-25"), true);
  });

  test("aceita CPF válido sem pontuação", () => {
    assert.equal(validarCPF("52998224725"), true);
  });

  test("rejeita dígito verificador incorreto", () => {
    assert.equal(validarCPF("529.982.247-24"), false);
  });

  test("rejeita sequências de dígitos idênticos", () => {
    for (let d = 0; d <= 9; d++) {
      assert.equal(validarCPF(String(d).repeat(11)), false, `falhou para ${d}`);
    }
  });

  test("rejeita quantidade incorreta de dígitos", () => {
    assert.equal(validarCPF("1234567890"), false);
    assert.equal(validarCPF("123456789012"), false);
  });

  test("rejeita valores nulos, indefinidos e vazios", () => {
    assert.equal(validarCPF(null), false);
    assert.equal(validarCPF(undefined), false);
    assert.equal(validarCPF(""), false);
  });
});

describe("formatarCPF", () => {
  test("aplica a máscara corretamente", () => {
    assert.equal(formatarCPF("52998224725"), "529.982.247-25");
  });

  test("preserva a formatação de um valor já formatado", () => {
    assert.equal(formatarCPF("529.982.247-25"), "529.982.247-25");
  });

  test("lança erro quando faltam dígitos", () => {
    assert.throws(() => formatarCPF("123"), /11 dígitos/);
  });
});

describe("CEP", () => {
  test("valida CEP de oito dígitos", () => {
    assert.equal(validarCEP("72870-000"), true);
    assert.equal(validarCEP("72870000"), true);
  });

  test("rejeita CEP com quantidade incorreta de dígitos", () => {
    assert.equal(validarCEP("7287000"), false);
  });

  test("formata corretamente", () => {
    assert.equal(formatarCEP("72870000"), "72870-000");
  });
});

describe("apenasDigitos", () => {
  test("remove caracteres não numéricos", () => {
    assert.equal(apenasDigitos("(61) 3333-4444"), "6133334444");
  });

  test("trata valores ausentes sem lançar exceção", () => {
    assert.equal(apenasDigitos(null), "");
    assert.equal(apenasDigitos(undefined), "");
  });
});
```

**Arquivo `testes/texto.test.js`:**

```javascript
import { test, describe } from "node:test";
import assert from "node:assert/strict";

import { removerAcentos, gerarSlug, capitalizarNome, truncar }
  from "../src/texto.js";

describe("removerAcentos", () => {
  test("remove diacríticos preservando as letras", () => {
    assert.equal(removerAcentos("informação"), "informacao");
    assert.equal(removerAcentos("Brasília"), "Brasilia");
    assert.equal(removerAcentos("ÁÉÍÓÚÂÊÔÃÕÇ"), "AEIOUAEOAOC");
  });
});

describe("gerarSlug", () => {
  test("converte texto acentuado em identificador de URL", () => {
    assert.equal(
      gerarSlug("Instituto Federal de Brasília"),
      "instituto-federal-de-brasilia"
    );
  });

  test("descarta pontuação e símbolos", () => {
    assert.equal(gerarSlug("Node.js: guia prático!"), "nodejs-guia-pratico");
  });

  test("condensa espaços e hifens consecutivos", () => {
    assert.equal(gerarSlug("  muitos    espaços  "), "muitos-espacos");
  });
});

describe("capitalizarNome", () => {
  test("preserva preposições em minúsculas", () => {
    assert.equal(capitalizarNome("MARIA DA SILVA E SOUZA"), "Maria da Silva e Souza");
  });

  test("capitaliza a primeira palavra ainda que seja preposição", () => {
    assert.equal(capitalizarNome("do carmo"), "Do Carmo");
  });
});

describe("truncar", () => {
  test("preserva textos dentro do limite", () => {
    assert.equal(truncar("texto curto", 50), "texto curto");
  });

  test("corta no último espaço anterior ao limite", () => {
    assert.equal(truncar("um texto razoavelmente longo aqui", 15), "um texto…");
  });
});
```

Execução:

```powershell
npm test
```

```
✔ validarCPF (6 subtests)
✔ formatarCPF (3 subtests)
✔ CEP (3 subtests)
✔ apenasDigitos (2 subtests)
...
ℹ tests 24
ℹ pass 24
ℹ fail 0
```

Modo de observação, que reexecuta os testes a cada alteração:

```powershell
npm run test:watch
```

Relatório de cobertura:

```powershell
node --test --experimental-test-coverage testes/
```

---

## 8. Documentação

O `README.md` é exibido na página do pacote no npm e constitui o principal fator de adoção.

**Arquivo `README.md`:**

````markdown
# utilitarios-br

Funções utilitárias para dados brasileiros: validação de CPF e CEP,
geração de slugs e formatação de moeda e datas.

Sem dependências externas. Compatível com Node.js 18 ou superior.

## Instalação

```bash
npm install utilitarios-br
```

## Uso

```javascript
import { validarCPF, formatarCPF, gerarSlug, formatarMoeda }
  from "utilitarios-br";

validarCPF("529.982.247-25");        // true
formatarCPF("52998224725");          // "529.982.247-25"
gerarSlug("Olá, Mundo!");            // "ola-mundo"
formatarMoeda(1234.5);               // "R$ 1.234,50"
```

Importação por módulo, quando apenas parte da biblioteca é necessária:

```javascript
import { validarCPF } from "utilitarios-br/documentos";
import { gerarSlug } from "utilitarios-br/texto";
```

## Referência

### Documentos

| Função | Parâmetros | Retorno | Descrição |
|---|---|---|---|
| `validarCPF(cpf)` | `string` | `boolean` | Valida os dígitos verificadores |
| `formatarCPF(cpf)` | `string` | `string` | Aplica a máscara `000.000.000-00` |
| `validarCEP(cep)` | `string` | `boolean` | Verifica a quantidade de dígitos |
| `formatarCEP(cep)` | `string` | `string` | Aplica a máscara `00000-000` |
| `apenasDigitos(valor)` | `string \| number` | `string` | Remove caracteres não numéricos |

`formatarCPF` e `formatarCEP` lançam `Error` quando a quantidade de
dígitos é incorreta.

### Texto

| Função | Parâmetros | Retorno |
|---|---|---|
| `removerAcentos(texto)` | `string` | `string` |
| `gerarSlug(texto)` | `string` | `string` |
| `capitalizarNome(nome)` | `string` | `string` |
| `truncar(texto, limite)` | `string`, `number` | `string` |

### Formatação

| Função | Parâmetros | Retorno |
|---|---|---|
| `formatarMoeda(valor)` | `number` | `string` |
| `formatarData(data)` | `Date \| string \| number` | `string` |
| `tempoRelativo(data)` | `Date \| string \| number` | `string` |

## Linha de comando

```bash
npx utilitarios-br cpf 52998224725
npx utilitarios-br slug "Instituto Federal de Brasília"
npx utilitarios-br moeda 1234.5
```

## Licença

MIT
````

**Arquivo `LICENSE`** (MIT):

```
MIT License

Copyright (c) 2026 Nome do Estudante

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

**Observação sobre licenças:** um pacote sem licença não pode ser legalmente utilizado por terceiros, ainda que o código-fonte esteja publicamente acessível. A MIT é permissiva e de uso corrente. Outras opções encontram-se em <https://choosealicense.com>.

---

## 9. Teste local antes da publicação

### 9.1 Inspeção do conteúdo

```powershell
npm pack --dry-run
```

```
npm notice 📦  utilitarios-br@0.1.0
npm notice === Tarball Contents ===
npm notice 1.1kB LICENSE
npm notice 2.4kB README.md
npm notice 1.2kB package.json
npm notice 2.8kB src/cli.js
npm notice 2.1kB src/documentos.js
npm notice 1.6kB src/formatacao.js
npm notice  412B src/index.js
npm notice 1.9kB src/texto.js
npm notice === Tarball Details ===
npm notice total files: 8
```

**Verificação obrigatória:** conferir que nenhum arquivo indevido — testes, `.env`, arquivos temporários — consta da relação. Cada arquivo publicado será baixado por todos os consumidores do pacote.

### 9.2 Instalação local com `npm link`

O comando `npm link` cria um vínculo simbólico entre o pacote em desenvolvimento e um projeto consumidor, permitindo testá-lo antes da publicação.

```powershell
# Na pasta do pacote
cd utilitarios-br
npm link

# Em outra pasta, um projeto de teste
cd ..
mkdir testar-pacote
cd testar-pacote
npm init -y
npm pkg set type=module
npm link utilitarios-br
```

**Arquivo `testar-pacote/teste.js`:**

```javascript
import {
  validarCPF, formatarCPF, gerarSlug, capitalizarNome,
  formatarMoeda, formatarData, truncar,
} from "utilitarios-br";

console.log("--- Documentos ---");
console.log("CPF válido  :", validarCPF("529.982.247-25"));
console.log("CPF inválido:", validarCPF("111.111.111-11"));
console.log("Formatado   :", formatarCPF("52998224725"));

console.log("\n--- Texto ---");
console.log("Slug        :", gerarSlug("Instituto Federal de Brasília — Campus Valparaíso"));
console.log("Nome        :", capitalizarNome("JOAO DA SILVA E SOUZA"));
console.log("Truncado    :", truncar("Um texto razoavelmente longo para demonstrar", 20));

console.log("\n--- Formatação ---");
console.log("Moeda       :", formatarMoeda(1234.5));
console.log("Data        :", formatarData(new Date("2026-09-07")));

console.log("\n--- Importação por módulo ---");
const { validarCEP } = await import("utilitarios-br/documentos");
console.log("CEP         :", validarCEP("72870-000"));
```

```powershell
node teste.js
```

Teste do comando de terminal:

```powershell
utilbr cpf 52998224725
utilbr slug "Teste do comando de terminal"
```

Desfazer o vínculo após os testes:

```powershell
npm unlink utilitarios-br        # no projeto consumidor
cd ../utilitarios-br
npm unlink -g utilitarios-br     # remove o vínculo global
```

**Alternativa a `npm link`:** instalar diretamente o arquivo empacotado, o que reproduz com maior fidelidade a instalação real:

```powershell
cd utilitarios-br
npm pack                          # gera utilitarios-br-0.1.0.tgz
cd ../testar-pacote
npm install ../utilitarios-br/utilitarios-br-0.1.0.tgz
```

---

## 10. Publicação

### 10.1 Conta no npm

1. Criar conta em <https://www.npmjs.com/signup>.
2. Confirmar o endereço de correio eletrônico. **Contas não confirmadas não podem publicar.**
3. Ativar a autenticação em dois fatores em *Account Settings → Two-Factor Authentication*. É exigida para publicação.

### 10.2 Autenticação

```powershell
npm login
```

```
Login at:
https://www.npmjs.com/login?next=/login/cli/xxxxx
Press ENTER to open in the browser...
```

Verificação:

```powershell
npm whoami
```

### 10.3 Lista de verificação antes de publicar

| Item | Comando ou verificação |
|---|---|
| Testes aprovados | `npm test` |
| Conteúdo do pacote conferido | `npm pack --dry-run` |
| Nome disponível | `npm view <nome>` |
| Versão ainda não publicada | `npm view <nome> versions` |
| README com exemplos funcionais | leitura |
| Arquivo LICENSE presente | verificação |
| `repository` apontando ao repositório correto | verificação |
| `.env` e credenciais ausentes | `npm pack --dry-run` |
| Pacote testado com `npm link` | execução |

### 10.4 Publicação

```powershell
npm publish
```

Para pacotes com escopo:

```powershell
npm publish --access public
```

Ao final, o pacote torna-se acessível em `https://www.npmjs.com/package/utilitarios-br`.

**Advertências importantes:**

1. **A versão publicada é definitiva.** Não é possível republicar a mesma versão, mesmo após remoção.
2. **A remoção é restrita.** O comando `npm unpublish` só é permitido nas primeiras 72 horas e apenas quando nenhum outro pacote depende do publicado. A política existe para impedir a quebra de projetos alheios.
3. **Publicação é irreversível na prática.** Credenciais ou dados sensíveis incluídos por engano devem ser considerados comprometidos e imediatamente revogados.

Publicação de versão de teste, que não é instalada por padrão:

```powershell
npm publish --tag beta
# instalação: npm install utilitarios-br@beta
```

---

## 11. Versionamento

### 11.1 Regras do SemVer

Dada a versão `MAIOR.MENOR.CORREÇÃO`:

| Alteração | Incremento | Exemplo |
|---|---|---|
| Correção de defeito, sem mudança de comportamento | CORREÇÃO | `1.2.3` → `1.2.4` |
| Nova função compatível com o existente | MENOR | `1.2.3` → `1.3.0` |
| Remoção ou alteração incompatível | MAIOR | `1.2.3` → `2.0.0` |

Versões `0.x.y` indicam interface ainda instável; nesse estágio, alterações incompatíveis são admitidas em incrementos menores.

**Exemplos aplicados a este pacote:**

- corrigir `truncar`, que cortava incorretamente textos sem espaços → `0.1.0` → `0.1.1`;
- acrescentar `validarCNPJ` → `0.1.1` → `0.2.0`;
- alterar `formatarCPF` para devolver `null` em vez de lançar exceção → `0.2.0` → `1.0.0`, pois todo código que dependia da exceção deixaria de funcionar.

### 11.2 Comandos de versionamento

```powershell
npm version patch     # 0.1.0 → 0.1.1
npm version minor     # 0.1.1 → 0.2.0
npm version major     # 0.2.0 → 1.0.0
```

Cada comando atualiza o `package.json`, cria um *commit* e uma etiqueta Git. Após a execução:

```powershell
git push && git push --tags
npm publish
```

### 11.3 Registro de alterações

**Arquivo `CHANGELOG.md`:**

```markdown
# Registro de alterações

O formato segue [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/)
e o versionamento adota [SemVer](https://semver.org/lang/pt-BR/).

## [0.2.0] — 2026-09-20

### Adicionado
- Função `validarCNPJ` para validação de CNPJ.
- Comando `utilbr cnpj` na interface de linha de comando.

### Corrigido
- `truncar` cortava incorretamente textos sem espaços.

## [0.1.0] — 2026-09-07

### Adicionado
- Validação e formatação de CPF e CEP.
- Geração de slugs, remoção de acentos e capitalização de nomes.
- Formatação de moeda, data e tempo relativo.
- Interface de linha de comando `utilbr`.
```

---

## 12. Consumo do pacote publicado

Em um projeto novo — por exemplo, a API do Tutorial 7:

```powershell
npm install utilitarios-br
```

Uso em um *middleware* de validação:

```javascript
// middlewares/validacao.js — acréscimo em projeto CommonJS.
// O pacote é ESM: import() dinâmico devolve uma Promise, resolvida uma
// única vez e reaproveitada nas requisições seguintes.
let utilitarios = null;

async function carregarUtilitarios() {
  if (!utilitarios) utilitarios = await import("utilitarios-br");
  return utilitarios;
}

async function validarCadastro(req, res, next) {
  const { validarCPF, formatarCPF } = await carregarUtilitarios();
  const { cpf } = req.body;

  if (!validarCPF(cpf)) {
    return res.status(400).json({
      sucesso: false,
      erro: { codigo: 400, mensagem: "CPF inválido" },
    });
  }

  // Normalização: o CPF é armazenado sempre no mesmo formato.
  req.body.cpf = formatarCPF(cpf);
  next();
}
```

**Observação sobre a importação:** em um projeto CommonJS, um pacote ESM não pode ser carregado com `require()`. As alternativas são `import()` dinâmico, como acima, ou a conversão do projeto para ESM mediante `"type": "module"` no `package.json`.

Atualização em projetos consumidores:

```powershell
npm outdated                      # exibe as versões disponíveis
npm update utilitarios-br         # atualiza respeitando o intervalo
npm install utilitarios-br@latest # força a versão mais recente
```

---

## 13. Laboratório prático

### Laboratório 13.1 — Publicação do pacote

1. Implementar integralmente o pacote `utilitarios-br`, com nome próprio ou escopo pessoal.
2. Alcançar, no mínimo, vinte e cinco casos de teste aprovados.
3. Redigir README completo, com exemplos executáveis.
4. Testar com `npm link` em projeto separado.
5. Publicar no registro público.
6. Instalar o pacote publicado em um projeto novo e comprovar seu funcionamento.

**Entrega:** endereço do pacote no npm, endereço do repositório e captura de tela da instalação e do uso em projeto distinto.

### Laboratório 13.2 — Evolução e versionamento

Sobre o pacote publicado, executar três ciclos completos de alteração:

**Ciclo 1 — CORREÇÃO (`0.1.0` → `0.1.1`)**
Corrigir `truncar` quando o texto não contém espaços dentro do limite. Acrescentar teste que reproduza o defeito antes da correção.

**Ciclo 2 — MENOR (`0.1.1` → `0.2.0`)**
Acrescentar `validarCNPJ` e `formatarCNPJ`, com o respectivo comando na interface de linha de comando e ao menos seis casos de teste.

**Ciclo 3 — MAIOR (`0.2.0` → `1.0.0`)**
Substituir o lançamento de exceção em `formatarCPF` pelo retorno de `null` em caso de valor inválido. Documentar a mudança incompatível no `CHANGELOG.md` e descrever, no README, o procedimento de migração.

**Entrega:** três versões publicadas, `CHANGELOG.md` completo e relatório justificando o incremento adotado em cada ciclo.

### Laboratório 13.3 — Pacote autoral

Conceber, implementar e publicar um pacote original, de utilidade real, que **não duplique** biblioteca existente — verificação obrigatória por busca no registro npm.

**Sugestões:** utilitários de cálculo escolar (média ponderada, conversão de conceitos, cálculo de frequência); manipulação de datas letivas com feriados nacionais; validação de dados acadêmicos; formatação de referências bibliográficas conforme a NBR 6023.

**Requisitos mínimos:**
1. pelo menos cinco funções públicas documentadas com JSDoc;
2. cobertura de testes superior a 80%;
3. README com instalação, referência completa e três exemplos práticos;
4. interface de linha de comando funcional;
5. licença definida;
6. zero dependências de execução (`dependencies` vazio);
7. `CHANGELOG.md`;
8. repositório público no GitHub vinculado ao pacote.

**Apresentação (10 minutos):** problema que o pacote resolve; decisões de projeto da interface pública; demonstração ao vivo da instalação e do uso; relato de uma dificuldade enfrentada e da solução adotada.

---

## 14. Síntese

1. Um pacote npm distingue-se de um projeto comum pela interface pública explícita, pelo versionamento semântico e pela independência de contexto.
2. O campo `exports` delimita o que terceiros podem importar; `files` controla o que é publicado.
3. Arquivos declarados em `bin` requerem a linha *shebang* e tornam-se comandos de terminal.
4. O executor `node --test` dispensa bibliotecas externas para testes.
5. `npm pack --dry-run` deve ser executado antes de toda publicação, para conferir o conteúdo.
6. `npm link` permite testar o pacote em um projeto consumidor antes da publicação.
7. Versões publicadas são definitivas; a remoção é limitada às primeiras 72 horas.
8. O incremento da versão deve corresponder à natureza da alteração, e mudanças incompatíveis exigem incremento maior.
9. Um pacote sem licença não pode ser legalmente utilizado por terceiros.

**Retorno ao índice:** [Índice da série](./00-indice.md)
