# Infraestrutura

Documentação do repositório de infraestrutura do sistema de gerenciamento de petshop (trabalho de Microsserviços). Versão 1.0.

## Sumário

1. [Identificação do repositório](#1-identificação-do-repositório)
2. [Pré-requisitos](#2-pré-requisitos)
3. [Configuração e execução](#3-configuração-e-execução)
4. [Estrutura do projeto](#4-estrutura-do-projeto)
5. [Serviços e portas](#5-serviços-e-portas)
6. [Integração entre os serviços](#6-integração-entre-os-serviços)

---

## 1. Identificação do repositório

| Item | Descrição |
|---|---|
| Nome | petshop_infra |
| Finalidade | Subir os cinco microsserviços do petshop de forma integrada, com Docker Compose, para execução e testes locais |
| Depende de | Repositórios dos cinco microsserviços, clonados na mesma pasta (seção 3.1) |
| Conteúdo | `docker-compose.yml` |

**Responsabilidades**

- Construir a imagem Docker de cada microsserviço a partir do `Dockerfile` do respectivo repositório.
- Subir os cinco serviços em uma rede comum, na qual cada um é acessado pelo nome do serviço.
- Configurar, por variáveis de ambiente, as URLs usadas nas integrações entre os serviços.
- Expor cada serviço em uma porta local para testes.

**Fora do escopo:** código e regras de negócio (ficam no repositório de cada microsserviço) e publicação no Render (cada serviço é publicado a partir do próprio repositório e `Dockerfile`).

**Tecnologias utilizadas**

| Tecnologia | Uso |
|---|---|
| Docker | Construção das imagens e execução dos containers |
| Docker Compose | Orquestração local dos cinco serviços |
| `mcr.microsoft.com/dotnet/sdk:10.0` | Imagem de build (compilação com o SDK do .NET 10) |
| `mcr.microsoft.com/dotnet/aspnet:10.0` | Imagem de execução (somente o runtime do ASP.NET Core) |

---

## 2. Pré-requisitos

| Ferramenta | Versão | Observação |
|---|---|---|
| Docker | Recente | Confira com `docker --version`. No Windows, use o Docker Desktop |
| Docker Compose | v2 ou superior | Incluso no Docker Desktop. Confira com `docker compose version` |
| Git | Recente | Para clonar os repositórios |
| .NET SDK | Não é necessário | A compilação acontece dentro do container |

---

## 3. Configuração e execução

### 3.1 Clonagem

O `docker-compose.yml` referencia os microsserviços por caminho relativo (`../petshop_usuarios`, `../petshop_estoque` etc.), então os seis repositórios devem ficar lado a lado na mesma pasta:

```powershell
mkdir petshop-microservicos
cd petshop-microservicos
git clone https://github.com/petshop-microservicos/petshop_infra.git
git clone https://github.com/petshop-microservicos/petshop_usuarios.git
git clone https://github.com/petshop-microservicos/petshop_animais_tutores.git
git clone https://github.com/petshop-microservicos/petshop_agendamentos.git
git clone https://github.com/petshop-microservicos/petshop_estoque.git
git clone https://github.com/petshop-microservicos/petshop_comercial.git
```

Resultado esperado:

```
petshop-microservicos/
├── petshop_infra/
├── petshop_usuarios/
├── petshop_animais_tutores/
├── petshop_agendamentos/
├── petshop_estoque/
└── petshop_comercial/
```

### 3.2 Variáveis de ambiente e configuração

As variáveis são definidas no `docker-compose.yml` e sobrescrevem as chaves do `appsettings.json` de cada serviço (em variável de ambiente, `__` equivale a `:`).

| Serviço | Variável | Valor | Descrição |
|---|---|---|---|
| Todos | `ASPNETCORE_ENVIRONMENT` | `Development` | Habilita a documentação OpenAPI, Swagger e Scalar |
| Todos (no `Dockerfile`) | `ASPNETCORE_HTTP_PORTS` | `8080` | Porta da API dentro do container |
| agendamentos | `Servicos__AnimaisTutores__BaseUrl` | `http://animais-tutores:8080/` | URL do serviço AnimaisTutores. **A barra final é obrigatória** |
| comercial | `Servicos__Estoque__BaseUrl` | `http://estoque:8080/` | URL do serviço Estoque. **A barra final é obrigatória** |

Dentro da rede do Docker, um serviço acessa o outro pelo nome definido no `docker-compose.yml` (por exemplo, `estoque`), e não por `localhost`.

### 3.3 Inicialização

```powershell
cd petshop_infra
docker compose up --build -d
```

Na primeira execução, o Docker baixa as imagens do .NET e compila os cinco serviços, o que leva alguns minutos. Nas seguintes, só o que mudou é recompilado.

| Comando | Descrição |
|---|---|
| `docker compose ps` | Lista os containers e o estado de cada um |
| `docker compose logs -f <serviço>` | Acompanha os logs de um serviço (ex.: `docker compose logs -f agendamentos`) |
| `docker compose up --build -d <serviço>` | Recompila e reinicia apenas um serviço |
| `docker compose stop` | Para os containers, mantendo os dados |
| `docker compose down` | Para e remove os containers, apagando os dados |

### 3.4 Testando as integrações

Com os containers no ar, use os arquivos `.http` de cada repositório, trocando o `@host` pela porta local da seção 5. Ordem sugerida:

1. **AnimaisTutores:** criar um tutor e um animal.
2. **Agendamentos:** criar serviço, profissional, vínculo e disponibilidade; depois, criar um agendamento para o animal do passo 1.
3. **Estoque:** criar um produto e registrar uma entrada manual.
4. **Comercial:** registrar uma venda do produto do passo 3 e conferir no Estoque que a quantidade diminuiu.

### 3.5 Problemas comuns

| Sintoma | Causa e solução |
|---|---|
| `port is already allocated` | Outra aplicação usa a porta. Altere a porta da esquerda em `ports` no `docker-compose.yml` (ex.: `"6002:8080"`) |
| `unable to prepare context` ou `path not found` | Os repositórios não estão lado a lado. Veja a seção 3.1 |
| Agendamento ou venda retorna 503 | O serviço chamado não está no ar ou ainda está iniciando. Confira com `docker compose ps` e `docker compose logs` |
| 500 com `no such table` | O banco foi criado por uma versão anterior do schema. Recrie com `docker compose down` e `docker compose up --build -d` |
| Alteração no código não aparece | Faltou o `--build` no `docker compose up` |
| `permission denied` ao rodar `docker` no Linux | O usuário não está no grupo `docker`. Rode `sudo usermod -aG docker $USER` e abra uma nova sessão |

---

## 4. Estrutura do projeto

```
petshop_infra/
├── docker-compose.yml
└── README.md
```

Cada repositório de microsserviço contém os arquivos usados na construção da sua imagem:

```
petshop_<serviço>/
├── Dockerfile
└── .dockerignore
```

| Arquivo | Responsabilidade |
|---|---|
| `docker-compose.yml` | Define os cinco serviços, as portas, as variáveis de ambiente e a ordem de inicialização (`depends_on`) |
| `Dockerfile` | Build em duas etapas: compila e publica com a imagem `sdk:10.0` e executa com a imagem `aspnet:10.0`, que é menor. A API escuta na porta 8080 |
| `.dockerignore` | Impede que `bin/`, `obj/`, `.git/` e arquivos `.db` sejam copiados para a imagem |

O mesmo `Dockerfile` é usado na publicação de cada serviço no Render.

---

## 5. Serviços e portas

| Serviço no compose | Repositório | Porta local | Documentação |
|---|---|---|---|
| `usuarios` | petshop_usuarios | 5001 | `http://localhost:5001/openapi/v1.json` |
| `animais-tutores` | petshop_animais_tutores | 5002 | `http://localhost:5002/scalar` |
| `agendamentos` | petshop_agendamentos | 5003 | `http://localhost:5003/openapi/v1.json` |
| `estoque` | petshop_estoque | 5004 | `http://localhost:5004/swagger` |
| `comercial` | petshop_comercial | 5005 | `http://localhost:5005/openapi/v1.json` |

### 5.1 Persistência

- Cada serviço usa seu próprio banco SQLite, em arquivo dentro do container (`/app/<nome>.db`).
- O banco e as tabelas são criados automaticamente na inicialização do serviço.
- `docker compose stop` mantém os dados; `docker compose down` e a recriação dos containers apagam os dados.

---

## 6. Integração entre os serviços

| Origem | Destino | Endpoint chamado | Finalidade | Variável |
|---|---|---|---|---|
| Agendamentos | AnimaisTutores | `GET /api/animais/{id}` | Validar o animal ao criar um agendamento | `Servicos__AnimaisTutores__BaseUrl` |
| Comercial | Estoque | `GET /api/produtos/{id}` | Consultar o produto e o preço | `Servicos__Estoque__BaseUrl` |
| Comercial | Estoque | `POST /api/estoque/movimentacoes/lote` | Dar saída no estoque na venda e entrada na compra confirmada | `Servicos__Estoque__BaseUrl` |

- O serviço Usuários não é chamado pelos demais (RN21).
- O `depends_on` garante apenas a ordem de inicialização dos containers, e não que a API já esteja pronta. Logo após subir, a primeira chamada entre serviços pode retornar 503; basta repetir.
