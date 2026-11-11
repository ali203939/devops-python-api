# API Python para demonstrar DevOps

Esta aplicação é uma API REST em Python, construída com FastAPI, para demonstrar práticas de integração contínua, entrega contínua e deploy em container em ambientes Linux na nuvem. O objetivo principal é mostrar, de forma didática, como código, testes, qualidade, imagens Docker e implantação em EC2 podem funcionar em um fluxo real de desenvolvimento e operação.

O projeto mantém os dados em memória para priorizar o aprendizado do pipeline e da automação, sem introduzir dependências extras de banco de dados.

## Descrição da aplicação e objetivo

A API expõe endpoints para gerenciar tarefas e verificar o estado geral do serviço. Em termos práticos, ela simula um cenário comum de software em produção:

- desenvolvimento local;
- validação automatizada em CI;
- construção de imagem Docker;
- publicação da imagem em um registry;
- deploy em uma instância EC2;
- verificação via endpoint de saúde.

A aplicação é um laboratório para demonstrar como a mesma mudança de código passa por validação automatizada antes de ser implantada.

## Tecnologias utilizadas

- Python 3.13
- FastAPI
- Pydantic
- Uvicorn
- Pytest
- Ruff
- Docker
- Docker Compose
- GitHub Actions
- Amazon ECR
- Amazon EC2
- AWS CLI

## Estrutura do repositório

```text
.
├── .github/workflows/
│   ├── ci.yml            # pipeline de integração contínua
│   └── deploy.yml        # pipeline de entrega contínua
├── app/
│   ├── __init__.py
│   └── main.py           # API FastAPI
├── docs/
│   ├── API.md
│   ├── DOCKER.md
│   ├── GITHUB-ACTIONS.md
│   └── ROTEIRO-AULA.md
├── tests/
│   └── test_api.py       # testes automatizados
├── .dockerignore
├── .python-version
├── compose.yaml
├── Dockerfile
├── pyproject.toml        # configuração de pytest, cobertura e Ruff
├── requirements.txt       # dependências de execução
├── requirements-dev.txt   # dependências de desenvolvimento e CI
├── .gitignore
└── README.md
```

## Instruções para execução local

Pré-requisitos: Git e Python 3.13.

### Linux e macOS

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --requirement requirements-dev.txt
uvicorn app.main:app --reload
```

### Windows PowerShell

```powershell
py -3.13 -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --requirement requirements-dev.txt
uvicorn app.main:app --reload
```

Acesse:

- API: <http://localhost:8000>
- documentação Swagger: <http://localhost:8000/docs>
- verificação de saúde: <http://localhost:8000/health>

### Exemplos de uso

```bash
curl http://localhost:8000/health

curl -X POST http://localhost:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Estudar integração contínua"}'

curl http://localhost:8000/tasks

curl -X PATCH http://localhost:8000/tasks/1/done
```

Endpoints principais:

| Método | Caminho | Objetivo | Resposta esperada |
|---|---|---|---|
| GET | `/` | Apresentar a API | `200` |
| GET | `/health` | Informar saúde e versão | `200` |
| GET | `/tasks` | Listar tarefas | `200` |
| POST | `/tasks` | Criar uma tarefa | `201` |
| PATCH | `/tasks/{id}/done` | Concluir uma tarefa | `200` ou `404` |

## Procedimento de construção e execução da imagem Docker

### Construir a imagem

```bash
docker build -t devops-python-api:v1 .
```

### Executar a imagem localmente

```bash
docker run --rm -p 8000:8000 --name devops-python-api devops-python-api:v1
```

### Verificar a API no container

```bash
curl http://localhost:8000/health
docker ps
docker logs devops-python-api
```

### Com Docker Compose

```bash
docker compose up --build
docker compose down
```

O Dockerfile define o ambiente de execução da aplicação e expõe a porta 8000 para acesso externo ao serviço.

## Pipeline de integração contínua

O workflow de CI está em [.github/workflows/ci.yml](.github/workflows/ci.yml). Ele é acionado em:

- push para qualquer branch;
- pull request;
- execução manual via `workflow_dispatch`.

A pipeline realiza os seguintes passos:

1. checkout do código;
2. configuração do Python;
3. instalação das dependências de desenvolvimento;
4. análise estática com Ruff;
5. verificação de formatação com Ruff;
6. execução de testes com Pytest;
7. geração de relatórios de cobertura e JUnit;
8. upload de artefatos como evidência;
9. build da imagem Docker e smoke test do container.

A etapa de smoke test garante que a aplicação esteja respondendo em `/health` antes de considerar o build como válido.

### Verificação local equivalente à CI

```bash
ruff check .
ruff format --check .
pytest --cov=app --cov-report=term-missing
```

Se essas etapas falharem localmente, elas também falham no GitHub Actions, mantendo a mesma regra em todas as etapas do desenvolvimento.

## Pipeline de entrega contínua

O workflow de CD está em [.github/workflows/deploy.yml](.github/workflows/deploy.yml). Ele é acionado em:

- push na branch `main`;
- execução manual via `workflow_dispatch`.

O fluxo de deploy realiza os seguintes passos:

1. autenticação no GitHub com o código do repositório;
2. configuração das credenciais AWS;
3. login no Amazon ECR;
4. build da imagem Docker;
5. push da imagem para o repositório ECR;
6. conexão SSH com a EC2;
7. autenticação no ECR pela instância;
8. pull da imagem mais recente;
9. remoção do container anterior;
10. execução do novo container em background na porta 8000.

Essa sequência garante que a atualização do código possa ser entregue sem intervenção manual direta no host da instância.

## Arquitetura de implantação envolvendo GitHub, ECR e EC2

O fluxo de implantação do projeto segue a seguinte arquitetura:

```text
GitHub Repository
      │
      ├─ push / pull request
      │
      ▼
GitHub Actions
      │
      ├─ CI: qualidade + testes + build + smoke test
      │
      └─ CD: build da imagem + push para ECR + deploy via SSH na EC2
                        │
                        ▼
                     Amazon ECR
                        │
                        ▼
                 EC2 instance
                        │
                        ▼
                 Docker container
                        │
                        ▼
                   52.5.179.245:8000
```

Em outras palavras:

- o código é versionado no GitHub;
- o GitHub Actions executa a validação; 
- a imagem Docker é publicada no Amazon ECR;
- a EC2 recebe a imagem e sobe o container na porta 8000.

## URL ou endereço de acesso à API publicada

O endereço de acesso da API depende do ambiente provisionado na EC2. Em geral, a URL pública será da forma:

```text
http://52.5.179.245:8000
```

Também é possível verificar o endpoint de saúde diretamente:

```bash
curl 52.5.179.245:8000/health
```

## Evidências da execução das pipelines

As evidências podem ser consultadas na aba **Actions** do GitHub:

- logs da execução do workflow;
- status do job CI;
- artefatos gerados pelo pytest e pela cobertura;
- logs do build e do smoke test do container;
- logs da execução do deploy na EC2.

Os artefatos normalmente incluem:

- `coverage.xml`;
- `junit.xml`;
- logs do container;
- relatórios da execução da pipeline.

Esses arquivos servem como evidência documental da validação e da entrega da aplicação.

## Instruções para verificar a versão implantada

Para confirmar a versão em execução na EC2, execute:

```bash
curl http://52.5.179.245:8000/health
```

A resposta esperada inclui a versão da aplicação, por exemplo:

```json
{"status": "healthy", "version": "1.0.0"}
```

Também é possível verificar diretamente a imagem implantada em execução no host:

```bash
docker ps
docker logs devops-python-api
```

Se a versão do endpoint for alterada pela pipeline de deploy, a resposta em `/health` passa a refletir o novo valor implantado, permitindo confirmar que a entrega contínua realmente atingiu o ambiente de produção.

## Executar verificações locais

```bash
ruff check .
ruff format --check .
pytest --cov=app --cov-report=term-missing
```

Para aplicar a formatação automaticamente:

```bash
ruff format .
```

## Falhas intencionais para demonstrar

### Falha de teste

Em `app/main.py`, altere temporariamente a versão do health check para `2.0.0`. O teste espera `1.0.0`, então o pytest e o pipeline devem falhar.

### Falha de qualidade

Adicione um import não utilizado no início de `app/main.py`:

```python
import os
```

O step `ruff check .` localizará o problema.

### Falha de rede no container

No `Dockerfile`, troque `--host 0.0.0.0` por `--host 127.0.0.1`. O processo iniciará, mas o serviço não ficará acessível pela porta publicada no host.

## Próximos desafios

- Adicionar um endpoint `DELETE /tasks/{id}` e seus testes.
- Executar os testes em uma matriz com Python 3.12, 3.13 e 3.14.
- Publicar a imagem em um registry somente após a aprovação dos testes.
- Substituir o armazenamento em memória por um banco de dados.

## Referências oficiais

- [GitHub Actions: criar e testar Python](https://docs.github.com/pt/actions/tutorials/build-and-test-code/building-and-testing-python)
- [actions/checkout](https://github.com/actions/checkout)
- [actions/setup-python](https://github.com/actions/setup-python)
- [actions/upload-artifact](https://github.com/actions/upload-artifact)
- [FastAPI](https://fastapi.tiangolo.com/)
- [Docker para Python](https://docs.docker.com/language/python/)
