# Criação de uma API com Next.js

## 1. Bibliotecas necessárias e instalação
- **MySQL2**: Para a criação da conexão e manipulação do banco de dados;

```bash
# Instalação da biblioteca
npm install mysql2
```

## 2. Exemplos de códigos e diretórios
### 2.1. Criando conexão com o banco de dados

```js
import mysql from 'mysql2/promise';

export const database = await mysql.createPool({
	host: "localhost",
  user: "root",
  password: "root",
  database: "psifinance",
  port: 3306
  waitForConnections: true,
  connectionLimit: 10,
});
```

> **IMPORTANTE!** Criar um módulo separado para colocar o código acima _(geralmente ``"src/lib/database.js"``)_. Lembre-se de criar a const de conexão com o mesmo nome do seu arquivo. **EX:** Se o diretório da sua conexão for ``"src/lib/database.js"``, seu "export const" deve ter o nome ``"database"``.

É **EXTREMAMENTE** importante manter as credenciais do banco de dados fora dos commits e do código-fonte. Em ambientes fora do desenvolvimento, sempre utilize variáveis de ambiente para proteger essas informações, seguindo o exemplo abaixo:

```js
import mysql from 'mysql2/promise';

export const Connection = mysql.createPool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  port: process.env.DB_PORT ? Number(process.env.DB_PORT) : 3306,
  waitForConnections: true,
  connectionLimit: 10,
});
```
> **OBS:** Pra criar esse .env.local vc pode só criar um arquivo com o nome ".env.local" sem nenhuma extensão na raiz do seu projeto. Ele terá a estrutura mais ou menos assim:

```
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=root
DB_NAME=psifinance
DB_PORT=3306
```
### 2.2. Criando rotas
#### Rota GET (Solicitar dados) 
``src/app/api/clientes/route.js``

```js
import { NextResponse } from "next/server";
import { database } from "@/lib/database";

export async function GET() {
  try {
    const [rows] = await database.query("SELECT * FROM clientes");
    return NextResponse.json(rows);
  } catch (error) {
    return NextResponse.json(
      { error: "Erro ao buscar clientes." },
      { status: 500 }
    );
  }
}
```

#### Rota GET com busca específica (Solicitar dados específicos)
``src/app/api/clientes/[id]/route.js``

```js
import { NextResponse } from "next/server";
import { database } from "@/lib/database";

export async function GET(req, { params }) {
  try {
    const { id } = params;

    const [rows] = await database.query(
      "SELECT * FROM clientes WHERE cliente_id = ?",
      [id]
    );

    if (rows.length === 0) {
      return NextResponse.json(
        { error: "Cliente não encontrado." },
        { status: 404 }
      );
    }

    return NextResponse.json(rows[0]);
  } catch (error) {
    return NextResponse.json(
      { error: "Erro ao buscar cliente." },
      { status: 500 }
    );
  }
}
```

#### Rota POST (Enviar dados)
``src/app/api/clientes/route.js``

```js
import { NextResponse } from "next/server";
import { database } from "@/lib/database";

export async function POST(req) {
  try {
    const body = await req.json();
    const { nome, email } = body;

    if (!nome || !email) {
      return NextResponse.json(
        { error: "Todos os campos são obrigatórios." },
        { status: 400 }
      );
    }

    const query = `INSERT INTO clientes (nome, email) VALUES (?, ?)`;
    const [result] = await database.query(query, [nome, email]);

    return NextResponse.json(
      {
        message: "Cliente criado com sucesso!",
        insertId: result.insertId,
      },
      { status: 201 }
    );
  } catch (error) {
    return NextResponse.json(
      { error: "Erro ao criar cliente." },
      { status: 500 }
    );
  }
}
```

#### Rota PUT (Atualizar TODOS os dados)
``src/app/api/clientes/[id]/route.js``

```js
import { NextResponse } from "next/server";
import { database } from "@/lib/database";

export async function PUT(req, { params }) {
  try {
    const { id } = params;
    const body = await req.json();

    const { nome, email } = body;

    if (!nome || !email) {
      return NextResponse.json(
        { error: "Todos os campos são obrigatórios." },
        { status: 400 }
      );
    }

    const query = `
      UPDATE clientes SET nome = ?, email = ?
      WHERE cliente_id = ?
    `;

    const [result] = await database.query(query, [nome, email, id]);

    if (result.affectedRows === 0) {
      return NextResponse.json(
        { error: "Cliente não encontrado." },
        { status: 404 }
      );
    }

    return NextResponse.json({ message: "Cliente atualizado com sucesso!" });
  } catch (error) {
    return NextResponse.json(
      { error: "Erro ao atualizar cliente." },
      { status: 500 }
    );
  }
}
```

#### Rota PATCH (Atualizar dados PARCIALMENTE)
``src/app/api/clientes/[id]/route.js``

```js
import { NextResponse } from "next/server";
import { database } from "@/lib/database";

export async function PATCH(req, { params }) {
  try {
    const { id } = params;
    const body = await req.json();

    const campos = [];
    const valores = [];

    Object.entries(body).forEach(([chave, valor]) => {
      campos.push(`${chave} = ?`);
      valores.push(valor);
    });

    if (campos.length === 0) {
      return NextResponse.json(
        { error: "Nenhum campo fornecido." },
        { status: 400 }
      );
    }

    valores.push(id);

    const query = `
      UPDATE clientes
      SET ${campos.join(", ")}
      WHERE cliente_id = ?
    `;

    const [result] = await database.query(query, valores);

    if (result.affectedRows === 0) {
      return NextResponse.json(
        { error: "Cliente não encontrado." },
        { status: 404 }
      );
    }

    return NextResponse.json({ message: "Cliente atualizado parcialmente!" });
  } catch (error) {
    return NextResponse.json(
      { error: "Erro no PATCH." },
      { status: 500 }
    );
  }
}
```

#### Rota DELETE (Excluir dados)
``src/app/api/clientes/[id]/route.js``

```js
import { NextResponse } from "next/server";
import { database } from "@/lib/database";

export async function DELETE(req, { params }) {
  try {
    const { id } = params;

    const [result] = await database.query(
      "DELETE FROM clientes WHERE cliente_id = ?",
      [id]
    );

    if (result.affectedRows === 0) {
      return NextResponse.json(
        { error: "Cliente não encontrado." },
        { status: 404 }
      );
    }

    return NextResponse.json({ message: "Cliente removido com sucesso!" });
  } catch (error) {
    return NextResponse.json(
      { error: "Erro ao excluir cliente." },
      { status: 500 }
    );
  }
}
```
### 2.3. Dicas

- É possível testar as rotas criadas com o [Insomnia](https://insomnia.rest/download), colocando a URL da API e lançando um body JSON com as informações necessárias.
- Utilize o [repositório](https://github.com/marcos-ywb/oficina-autoeletrica) do meu projeto como base, lá tem várias rotas com métodos HTTP diferentes e funcionais, assim como a estrutura de diretórios e arquivos adequada.
- Utilize a estrutura try/catch para capturar erros durante a execução da rota. **EX:**

```js
try {
  // Tente executar um comando
} catch (err) {
  // Capture o erro (se houver) e retorne de forma amigável
}
```
- Evite retorno não amigável de erros ao usuário final. **EX:**
```js
// Evite isso
Error: ER_PARSE_ERROR (SQL syntax error near “+”) 

// Prefira isso
{ "error": "Erro interno ao processar a requisição." }
```
- Mantenha uma estrutura de retorno de informações consistente em todas as rotas. **EX:**
```js
return NextResponse.json({
  success: true,
  data: rows,
});

//ou

return NextResponse.json({
  success: false,
  error: "Mensagem..."
});

// Isso ajuda a manter a API e previsivel no momento de consumi-la
```












