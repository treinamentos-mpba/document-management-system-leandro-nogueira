# Especificação - Document Management System

## 1. Objetivo

Entregar uma aplicação web que permita a um usuário enviar, listar e baixar seus documentos, mantendo os arquivos no filesystem local e os metadados em memória.

## 2. Escopo

### Dentro do escopo

- Upload de um documento por requisição.
- Listagem dos documentos associados ao identificador de usuário enviado pelo cliente.
- Download de um documento por identificador, limitado ao identificador de usuário da requisição.
- Metadados mantidos em memória durante a execução do processo.
- Interface web para envio, listagem e download de documentos.
- Armazenamento dos arquivos exclusivamente no filesystem local da aplicação, em `backend/storage`, usando Multer com `diskStorage`.

### Fora do escopo

- Autenticação, cadastro de usuários e garantia de identidade do usuário.
- Armazenamento externo ou em nuvem.
- Banco de dados ou persistência durável dos metadados.
- Versionamento de documentos, compartilhamento entre usuários, edição ou exclusão de documentos.
- Allowlist de extensões ou tipos MIME.
- Paginação, busca e filtros além do identificador de usuário.

## 3. Requisitos funcionais

| ID | Requisito |
| --- | --- |
| RF-01 | O usuário pode enviar um arquivo usando `multipart/form-data`, no campo `file`, informando o identificador do proprietário no header `X-User-Id`. |
| RF-02 | O sistema valida a presença de `X-User-Id` não vazio e de exatamente um arquivo antes de aceitar o upload. |
| RF-03 | O sistema grava o conteúdo do arquivo em `backend/storage` com um nome interno gerado pela aplicação, sem usar o nome original como caminho no filesystem. |
| RF-04 | Após um upload bem-sucedido, o sistema cria metadados com identificador único, nome original, tamanho em bytes, data/hora ISO 8601 e proprietário, e retorna esses metadados. |
| RF-05 | O usuário pode listar os metadados dos documentos cujo `owner` corresponda ao valor de `X-User-Id`. A lista é retornada em ordem decrescente de `uploadedAt`; empates podem ser ordenados por `id`. |
| RF-06 | O usuário pode baixar um documento por `id` quando o documento existe e seu `owner` corresponde a `X-User-Id`. O download é apresentado como anexo usando o nome original como nome sugerido. |
| RF-07 | Se `X-User-Id` estiver ausente ou vazio, a requisição é rejeitada. O identificador é comparado após remover espaços nas extremidades. |
| RF-08 | Se o arquivo solicitado não existir, não pertencer ao `X-User-Id` informado ou não puder ser localizado no storage, o sistema responde como documento não encontrado, sem revelar dados de outro proprietário. |
| RF-09 | Se um upload falhar após a gravação do arquivo e antes da criação dos metadados, o sistema remove o arquivo parcial quando possível e registra/trata a falha sem retornar sucesso. |
| RF-10 | A interface permite selecionar e enviar um arquivo, visualizar a lista retornada e iniciar o download de um documento. Ela apresenta estados de carregamento e mensagens de erro compreensíveis em português. |

## 4. Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| RNF-01 | Os arquivos são gravados somente no filesystem local, em `backend/storage`, usando Multer com `diskStorage`; não são usados provedores externos. |
| RNF-02 | Os metadados são mantidos em memória. Eles podem ser perdidos ao reiniciar o processo; os arquivos no disco não são removidos automaticamente por uma reinicialização. |
| RNF-03 | A configuração operacional deve usar variáveis de ambiente conforme o princípio 12-Factor. `PORT` define a porta do backend e `MAX_FILE_SIZE_BYTES` define o limite de upload. |
| RNF-04 | `MAX_FILE_SIZE_BYTES` tem valor padrão de `10485760` bytes (10 MiB). Tipos MIME e extensões não são restringidos nesta fase. |
| RNF-05 | O identificador e o nome interno do arquivo são gerados pelo sistema. Um nome original nunca pode ser interpretado como caminho de storage nem permitir traversal de diretório. |
| RNF-06 | O acesso aos metadados e ao arquivo é sempre limitado ao `owner` declarado em `X-User-Id`; a ausência de autenticação real deve ser considerada uma limitação de segurança conhecida. |
| RNF-07 | Erros HTTP são tratados na fronteira da API e retornados no formato JSON definido nesta especificação. Respostas de download bem-sucedidas são binárias, não JSON. |
| RNF-08 | A implementação deve manter as camadas separadas e as dependências no sentido `routes -> controllers -> services -> repositories`. |
| RNF-09 | A aplicação deve validar entradas e tratar falhas de leitura e gravação do filesystem sem expor caminhos internos, stack traces ou dados de outros usuários nas respostas HTTP. |

## 5. Modelo de dados (metadados do documento)

| Campo | Tipo | Descrição |
| --- | --- | --- |
| `id` | string | Identificador único, opaco e gerado pela aplicação. É usado na rota de download. |
| `originalName` | string | Nome original fornecido pelo cliente, preservado como metadado e usado como nome sugerido no download. Não é usado como caminho no filesystem. |
| `size` | number | Tamanho do arquivo em bytes. |
| `uploadedAt` | string | Data e hora do upload em formato ISO 8601. |
| `owner` | string | Valor normalizado de `X-User-Id` enviado pelo cliente. Não representa uma identidade autenticada. |

O objeto público de documento contém somente esses campos. O nome interno do arquivo, o caminho no filesystem e outros detalhes de persistência são dados internos do repositório e não são expostos pela API.

## 6. Contratos de API

### Convenções gerais

- As rotas do backend são `/upload`, `/documents` e `/documents/:id/download`.
- Quando chamadas pelo frontend durante o desenvolvimento, as rotas são acessadas sob o prefixo `/api` (por exemplo, `/api/documents`), encaminhado pelo proxy do Vite ao backend.
- Todas as rotas exigem o header `X-User-Id`, com valor não vazio após a remoção de espaços nas extremidades.
- Erros JSON usam o formato `{ "error": { "code": "ERROR_CODE", "message": "Descrição em português" } }`.
- O valor de `message` é destinado ao consumidor da API e não inclui stack trace, caminhos locais ou detalhes internos.

### `POST /upload`

Envia um arquivo e cria os metadados do documento.

**Entrada**

- Header obrigatório: `X-User-Id: <identificador-do-usuário>`.
- Content-Type: `multipart/form-data`.
- Campo obrigatório: `file`, contendo um arquivo.
- Apenas um arquivo é aceito por requisição.
- Tamanho máximo: o valor de `MAX_FILE_SIZE_BYTES`, padrão de 10 MiB.
- Qualquer tipo MIME/extensão é aceito nesta fase.

**Sucesso: `201 Created`**

Content-Type: `application/json`.

```json
{
  "id": "doc_abc123",
  "originalName": "relatorio.pdf",
  "size": 24576,
  "uploadedAt": "2026-10-28T12:00:00.000Z",
  "owner": "user-123"
}
```

**Erros**

| HTTP | `code` | Condição |
| --- | --- | --- |
| 400 | `OWNER_REQUIRED` | Header `X-User-Id` ausente ou vazio. |
| 400 | `FILE_REQUIRED` | Campo `file` ausente ou sem arquivo. |
| 400 | `INVALID_UPLOAD` | Formulário inválido ou mais de um arquivo enviado. |
| 413 | `FILE_TOO_LARGE` | Arquivo excede `MAX_FILE_SIZE_BYTES`. |
| 500 | `STORAGE_ERROR` | Falha ao gravar o arquivo ou criar os metadados. |

### `GET /documents`

Lista os metadados dos documentos pertencentes ao usuário informado. Não há paginação nesta versão.

**Entrada**

- Header obrigatório: `X-User-Id: <identificador-do-usuário>`.

**Sucesso: `200 OK`**

Content-Type: `application/json`. A resposta é um array; quando não houver documentos, retorna `[]`.

```json
[
  {
    "id": "doc_abc123",
    "originalName": "relatorio.pdf",
    "size": 24576,
    "uploadedAt": "2026-10-28T12:00:00.000Z",
    "owner": "user-123"
  }
]
```

**Erros**

| HTTP | `code` | Condição |
| --- | --- | --- |
| 400 | `OWNER_REQUIRED` | Header `X-User-Id` ausente ou vazio. |
| 500 | `DOCUMENT_LIST_ERROR` | Falha ao consultar os metadados. |

### `GET /documents/:id/download`

Retorna o conteúdo binário de um documento pertencente ao usuário informado.

**Entrada**

- Header obrigatório: `X-User-Id: <identificador-do-usuário>`.
- Parâmetro `id`: identificador do documento.

**Sucesso: `200 OK`**

- Content-Type: `application/octet-stream`.
- Content-Disposition: `attachment`, com o nome original devidamente tratado para uso seguro no header.
- Corpo: bytes do arquivo armazenado.

**Erros**

| HTTP | `code` | Condição |
| --- | --- | --- |
| 400 | `OWNER_REQUIRED` | Header `X-User-Id` ausente ou vazio. |
| 404 | `DOCUMENT_NOT_FOUND` | ID inexistente, documento de outro owner ou arquivo ausente no filesystem. |
| 500 | `STORAGE_ERROR` | Falha de leitura do arquivo por motivo diferente de arquivo ausente. |

## 7. Decisões arquiteturais

- O backend segue Clean Architecture simples, com dependências em uma única direção: `routes -> controllers -> services -> repositories`.
- `routes/` declara os endpoints e encaminha as requisições aos controllers.
- `controllers/` adapta entrada e saída HTTP, valida parâmetros e headers e define status e formato das respostas.
- `services/` contém as regras de negócio, incluindo validação de limites, autorização baseada no owner declarado e coordenação do upload/download.
- `repositories/` concentra acesso aos metadados em memória e aos arquivos no filesystem local.
- O upload usa Multer com `diskStorage`, salvando em `backend/storage` com nome interno gerado pela aplicação.
- A persistência de metadados em memória é intencional nesta fase; após reiniciar o processo, os arquivos podem permanecer no disco sem metadados correspondentes e, portanto, não estarão disponíveis pela API.
- O frontend usa componentes React e `fetch` para chamar a API sob `/api`, conforme o proxy do Vite.
- `X-User-Id` é um identificador fornecido pelo consumidor. Como não há autenticação, um cliente pode declarar outro valor; a filtragem por owner organiza os dados, mas não constitui controle de acesso seguro para produção.

## 8. Plano de execução

As etapas abaixo descrevem uma implementação futura do produto. Nesta tarefa, somente este documento de especificação é produzido.

1. Revisar e aprovar os requisitos, os contratos HTTP, as decisões de ownership e os limites de upload.
2. Definir o fluxo de validação de entradas, erros padronizados e critérios para nomes e identificadores de armazenamento.
3. Implementar o armazenamento local dos arquivos e a manutenção em memória dos metadados, incluindo tratamento de falhas e limpeza de gravações incompletas.
4. Implementar os fluxos de upload, listagem por owner e download, respeitando as responsabilidades das camadas arquiteturais.
5. Integrar a interface de upload, listagem, estados de carregamento/erro e download com a API.
6. Validar os contratos, casos de erro, isolamento por owner declarado e comportamento do filesystem por testes automatizados.
7. Executar validação integrada da aplicação e revisar a documentação para garantir conformidade com esta especificação.
