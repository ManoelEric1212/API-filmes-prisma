# CRIANDO API REST SIMPLES COM PRISMA (FINS DIDÁTICOS)
Criação de uma API com prisma afim de aprender um pouco mais sobre a lib e aprofundar os conhecimentos no Back-End com NodeJS, utilizando um fluxo de trabalho baseado no GItFlow.

# Caso de uso simples de aluguel de filmes 

A regra de negócio para esse projeto é bem simples, apenas duas tabelas uma para Filmes e Outra para usuários, onde um usuário pode está associado a vários filmes, assim como um filme pode estar associado a outros n usuários, resultando assim em uma terceira tabela de relacionamento denominada aluguel. Validações simples, apenas na criação de usuários e filmes, bem como também na associação de um usuário a um filme.

![DataBaseApiFilmes](https://user-images.githubusercontent.com/35776840/229021232-25250fbc-bc2a-4fbe-8553-bd84ac9e27aa.png)

# Fluxo de trabalho 

O fluxo desse projeto, também serviu para testar o fluxo de GitFlow. Com suas Branches e padronização de commits.

![gitFlow](https://user-images.githubusercontent.com/35776840/229412233-e333dd38-6dd5-4277-a052-3e79c834b250.png)
bf-6e20-40b3-bad4-090fc62de155.png)

# Visualização da API Rest

Para a visualização da API construida nesse projeto, utilizou-se do Insomnia com um padrão também para melhor organização.

![insominia](https://user-images.githubusercontent.com/35776840/229022529-674372bf-6e20-40b3-bad4-090fc62de155.png)

# Documentação da API Rest

![fig2](https://user-images.githubusercontent.com/35776840/230360600-5e7db4d4-8f71-4fdc-946e-0336cdd47928.png)

# Arquitetura do projeto 

![CleanArchitecture](https://user-images.githubusercontent.com/35776840/230363603-53dab8bc-b12a-43e7-9265-57611c7b9b8a.jpg)

## Setup inicial de configurações
- Instalar dependências:

```
npm i prisma ts-node-dev typescript @types/express -D
npm i @prisma/client express
```

- Adicionar scripts de "dev"
```js

  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "dev": "ts-node-dev --exit-child --transpile-only --ignore-watch node_modules src/server.ts"
  }
  
```

- Criar o arquivo tsconfig.json
Consultando a documentação do Prisma em [Setup projeto prisma](https://www.prisma.io/docs/getting-started/setup-prisma/start-from-scratch/relational-databases-typescript-postgres),
observa-se que é necessário apenas rodar o comando 
```
npx tsc init
```


-Adicionar o prisma

```
npx prisma init
```
Next steps:
1. Set the DATABASE_URL in the .env file to point to your existing database. If your database has no tables yet, read https://pris.ly/d/getting-started
2. Set the provider of the datasource block in schema.prisma to match your database: postgresql, mysql, sqlite, sqlserver, mongodb or cockroachdb.
3. Run prisma db pull to turn your database schema into a Prisma schema.
4. Run prisma generate to generate the Prisma Client. You can then start querying your database.
## Criando modelos no Prisma

### Models comuns
- Exemplo:
```js
model User {
  id String @id @default(uuid())
  email String @unique
  name String
  created_at DateTime @default(now())
  updated_at DateTime @updatedAt

  @@map("users")
}

model Movie {
  id           String      @id @default(uuid())
  title        String      @unique
  duration     Int
  release_date DateTime
  MovieRent    MovieRent[] @relation("movie")

  @@map("movies")
}
```

### Model de relacionamento

```js

model MovieRent {
  user    User   @relation("user", fields: [userId], references: [id], onUpdate: Cascade, onDelete: Cascade)
  userId  String
  movie   Movie  @relation("movie", fields: [movieId], references: [id], onUpdate: Cascade, onDelete: Cascade)
  movieId String

  @@id([userId, movieId])
  @@map("movieRent")
}
```

## Gerando os modelos para o banco
- Necessário fazer uso das migrações
```
npx prisma migrate dev --name init
```

- Para acompanhar o prisma studio 

```
npx prisma studio
```

## Estruturação do Projeto

![DataBaseApiFilmesEstrutura](https://user-images.githubusercontent.com/35776840/229016693-cb6e3dcb-26eb-42fa-8e37-b89c2a8429ef.png)

- modules
- models
- dtos (Tipagens do Data Transfer Objects)
- useCase
- Controllers
- dtos 


Para esse tipo de estrutura de projeto, utiliza-se a validação de informações com DTO's bem como caso de usos específico, como por exemplo a criação de um caso de uso CreateUser, atentando para validação da existência ou não de um usuário com as mesmas informações email.

Além disso, faz-se necessário a validação das informações dentro de cada caso de uso de acordo com a regra de negócio para isso irá ser gerado um erro genérico no servidor. 

```js

import "express-async-errors";
import express, { NextFunction, Request, Response } from 'express';
import { routes } from './routes';
import { AppError } from "./errors/AppError";
const app = express();

app.use(express.json()); //cors
app.use(routes);
app.use((err: Error, request: Request, response: Response, next: NextFunction) => {
  if(err instanceof AppError){
    return response.status(err.statusCode).json({
      status: "error",
      message: err.message
    });
  }
  return response.status(500).json({
    status: "error",
    message: `Internal server - ${err.message}`
  })
})
app.listen(3333, () => console.log("Servidor online na porta 3333"));

```

```js

import { User } from "@prisma/client";
import { AppError } from "../../../../errors/AppError";
import { prisma } from "../../../../prisma/client";
import { CreateUserDTO } from "../../dtos/CreateUserDTO";

export class CreateUserUseCase {
  async execute({name, email}: CreateUserDTO): Promise<User>{
  // Verifica se o usuário já existe
  const userAlreadyExists = await prisma.user.findUnique({
    where : {
      email: email
    }
  });
  if(userAlreadyExists){
    // Erro
    throw new AppError("User already exists!");
  }
  // Cria o user
  const user = await prisma.user.create({
    data:{
      name,email
    }
  });
  return user;  
  }
}
```

- Criação do Controller 

Criar o controller com a utilização da lib express e validação do corpo.

```js
import {Request, Response} from "express";
import { CreateUserUseCase } from "./CreateUserUseCase";
export class CreateUserController {
  async handle(req: Request, res: Response){
    const {name, email} = req.body;
    // Instanciando o caso de uso do create user
    const createUserCase = new CreateUserUseCase();
    // Armazenar resultado executando o caso de uso
    const result = await createUserCase.execute({name, email});
    return res.status(201).json(result);
  }
}
```

- Criação do Routes do User

```js
import { Router } from "express";
import { CreateUserController } from "../modules/users/useCases/createUser/CreateUserController";

// Instanciando o controller
const createUserController = new CreateUserController();
// Instanciando o Router
const userRoutes = Router();
// Criando o Post
userRoutes.post("/", createUserController.handle);

export {userRoutes};
```
- Criação do arquivo de index.ts(routes)

Esse arquivo é responsável por centralizar todas as rotas de cada modelo de acordo com seu arquivo de Routes

```js

import { Router } from "express";
import { movieRoutes } from "./movie.routes";
import { userRoutes } from "./user.routes";

const routes = Router();
routes.use("/users", userRoutes);
routes.use("/movies",movieRoutes);

export { routes }

```

- Criação do Routes do User

```js
import { Router } from "express";
import { CreateUserController } from "../modules/users/useCases/createUser/CreateUserController";
import { GetAllUsersController } from "../modules/users/useCases/getUsers/GetAllUsersController";


const createUserController = new CreateUserController();
const getAllUsersController = new GetAllUsersController();

const userRoutes = Router();

userRoutes.post("/", createUserController.handle);
userRoutes.get("/", getAllUsersController.handle);

export {userRoutes};

```

## Manipulando relacionamentos 

Uma vez que foi definido relação entre um ou mais model, pode-se fazer uso das relações para retornar dados filtrados ou ordenado etc... Um exemplo é disposto abaixo, onde no retorno de um filme, utiliza-se do relacionamento dessa tabela com outra e o retorno é construído mostrando dados relacionados.

```js
import { Movie } from "@prisma/client";
import { prisma } from "../../../../prisma/client";

export class GetMoviesByReleaseDateUseCase {
  async execute(): Promise<Movie[]>{
    const movies = await prisma.movie.findMany({
      orderBy: {
        release_date: "desc"
      }, 
      include: {
        MovieRent :{
          select :{
            user: {
              select: {
                name: true,
                email: true,
              }
            },
          }
        }
      }
    });
    return movies;
  }
}

```

- Saída do Get 

![InsomniaRelacioanmento](https://user-images.githubusercontent.com/35776840/229020525-b40e5e4a-a630-43a0-906f-ef71e3341e9d.png)

## Documentando a API com Swagger

![fig2](https://user-images.githubusercontent.com/35776840/230360600-5e7db4d4-8f71-4fdc-946e-0336cdd47928.png)

Para a documentação do swagger, faremos uso da lib swagger-ui no node.js, bem como seus tipos respectivos.

```
npm i swagger-ui-express
```
```
npm i @types/swagger-ui-express -D
```

- Adicionando rota para documentação

```js
import swaggerUI from "swagger-ui-express";
import swaggerDocument from "../swagger.json"

app.use('/api-docs', swaggerUI.serve, swaggerUI.setup(swaggerDocument))

```

Observa-se que faremos uso de um docuemtno json, para gerar a documentação, o arquivo json terá todas as configurações.

```js
{
  "openapi": "3.0.1",
  "info":{
    "title": "API-Filmes(Prisma ORM)",
    "description":"Documentação para API filmes criada com Prisma, Typescript, Express e Clean Architecture",
    "version": "1.0.0",
    "contact":{
      "email": "manoeleric59@gmail.com"
    }
  },
  "tags":[
    {
      "name": "Users",
      "description": "Info about users"
    },{
      "name": "Movies",
      "description":"Info about movies"
    }
  ],
  "servers":[
    {
      "url": "http://localhost:3333/",
      "description": "API de test"
    }
  ],
  "paths":{
    "/users":{
      "post": {
        "summary":"Register of a user",
        "description": "Route for register new users ",
        "tags": ["Users"],
        "requestBody":{
          "content":{
            "application/json":{
              "schema":{
                "$ref": "#/components/schemas/User"
              },
              "examples":{
                "user":{
                  "value":{
                    "name": "Name of user",
                    "email": "example@example.com.br"
                  }
                }
              }
            }
          }
        },
        "responses": {
          "400": {
            "description": "User already exists !"
          },
          "500": {
            "description": "Internal server Error"
          },
          "201": {
            "description": "OK",
            "content":{
              "application/json":{
                "schema":{
                  "type": "object",
                  "$ref": "#/components/schemas/User"
                }
              }
            }
          }
        }
      },
      "get":{
        "summary": "Obtain users registers",
        "description": "Route for obatin all users register",
        "tags":["Users"],
        "responses":{
          "200": {
            "description": "Ok"
          },
          "204": {
            "description": "Does not exists users registers in database"
          },
          "500":{
            "description": "Internal server error"
          }
        }
      }
    },
    "/movies":{
      "get":{
        "summary": "Obtain movies registers",
        "description": "Route for obatin all movies register",
        "tags":["Movies"],
        "responses":{
          "200": {
            "description": "Ok"
          },
          "500":{
            "description": "Internal server error"
          }
        }
      },
      "post":{
        "summary":"Register of a movie",
        "description": "Route for register new movies ",
        "tags": ["Movies"],
        "requestBody":{
          "content":{
            "application/json":{
              "schema":{
                "$ref": "#/components/schemas/Movie"
              },
              "examples":{
                "movie":{
                  "value":{
                    "title": "Name of user",
                    "duration": 123123,
                    "release_date": "2021-03-28T04:04:20.700Z"
                  }
                }
              }
            }
          }
        },
        "responses": {
          "400": {
            "description": "Movie already exists !"
          },
          "500": {
            "description": "Internal server Error"
          },
          "201": {
            "description": "OK",
            "content":{
              "application/json":{
                "schema":{
                  "type": "object",
                  "$ref": "#/components/schemas/Movie"
                }
              }
            }
          }
        }

      }
      },
      "/movies/{id}":{
        "patch":{
          "summary": "Alter values of movie object",
          "description": "Route for modify values of movie object",
          "tags":["Movies"],
          "parameters":[
            {
              "name": "id",
              "in": "path",
              "description": "Modify movie by Id",
              "required": true
            }
          ],
          "requestBody":{
            "content":{
              "application/json":{
                "schema":{
                  "$ref": "#/components/schemas/Movie"
                },
                "examples":{
                  "movie":{
                    "value":{
                      "title": "Name of user",
                      "duration": 123123,
                      "release_date": "2021-03-28T04:04:20.700Z"
                    }
                  }
                }
              }
            }
          },
          "responses":{
            "202": {
              "description": "Ok"
            },
            "400":{
              "description": "MovieId does not exists !"
            },
            "500":{
              "description": "Internal server error"
            }
          }
        },
        "delete":{
          "summary": "Delete a movie",
          "description": "Route for delete values of movie object",
          "tags":["Movies"],
          "parameters":[
            {
              "name": "id",
              "in": "path",
              "description": "Modify movie by Id",
              "required": true
            }
          ],
          "responses":{
            "202":{
              "description": "Movie deleted !"
            },
            "400":{
              "description": "MovieId does not exists !"
            },
            "500":{
              "description":"Internal server error"
            }
          }
        }
      },
      "/movies/release":{
        "get":{
          "summary": "Obtain movies registers order by release date and relationship with user",
          "description": "Route for obatin all movies register orders by release date and relationship with user",
          "tags":["Movies"],
          "responses":{
            "200": {
              "description": "Ok"
            },
            "500":{
              "description": "Internal server error"
            }
          }
        }
      },
      "movies/rent":{
        "post":{
          "summary":"Register of a movieRent",
          "description": "Route for register new movieRent ",
          "tags": ["Movies"],
          "requestBody":{
            "content":{
              "application/json":{
                "schema":{
                  "$ref": "#/components/schemas/MovieRent"
                },
                "examples":{
                  "movieRent":{
                    "value":{
                      "movieId": "64a8d933-affb-45c6-ae1f-7c7f4098856d",
                      "userId": "b178ded0-31cf-4ce3-bfd1-28e3d674dadf"
                    }
                  }
                }
              }
            }
          },
          "responses": {
            "400": {
              "description": "Movie does not exists or movie associate another user!"
            },
            "500": {
              "description": "Internal server Error"
            },
            "201": {
              "description": "OK",
              "content":{
                "application/json":{
                  "schema":{
                    "type": "object",
                    "$ref": "#/components/schemas/MovieRent"
                  }
                }
              }
            }
          }
  
        }
        }
    },
   
  "components":{
    "schemas":{
      "User":{
        "type":"object",
        "properties":{
          "name":{
            "type":"string"
          },
          "email":{
            "type": "string"
          }
        }
      },
      "Movie":{
        "type": "object",
        "properties":{
          "title": {
            "type": "string"
          },
          "duration": {
            "type": "number"
          },
          "release_date": {
            "type": "string"
          }
        }
      },
      "MovieRent":{
        "type": "object",
        "properties":{
          "movieId":{
            "type": "string"
          },
          "userId": {
            "type": "string"
          }
        }
      }
    }
  }
}

```
