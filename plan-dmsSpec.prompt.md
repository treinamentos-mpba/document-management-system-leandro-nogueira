## Plan: Especificação DMS

Produzir `docs/specs/dms-spec.md` usando `docs/specs/spec-template.md` como base, detalhando requisitos funcionais e não funcionais, modelo de metadados, contratos HTTP, decisões arquiteturais e um plano de execução futuro em marcos. O trabalho desta tarefa limita-se ao documento; nenhum arquivo de aplicação será criado ou alterado.

**Steps**
1. Manter as seções do modelo e detalhar o objetivo e o escopo, incluindo explicitamente as exclusões de armazenamento externo/nuvem, versionamento e autenticação real nesta fase.
2. Especificar requisitos funcionais verificáveis para upload, listagem filtrada pelo owner e download por identificador, junto dos casos de entrada inválida e recurso ausente.
3. Especificar requisitos não funcionais para Multer com `diskStorage` em `backend/storage`, metadados voláteis em memória, limite de upload padrão de 10 MiB configurável por ambiente, tipos livres, nomes de armazenamento controlados pela aplicação, segurança contra traversal e configuração 12-Factor.
4. Documentar o modelo público do documento (`id`, `originalName`, `size`, `uploadedAt`, `owner`) e registrar que qualquer chave/caminho de armazenamento é detalhe interno, não campo da API.
5. Definir contratos para `POST /upload`, `GET /documents` e `GET /documents/:id/download`: arquivo multipart no campo `file`; owner obrigatório via `X-User-Id`; listagem e download restritos ao owner declarado; sucesso com códigos 201/200; erros com JSON uniforme `{ error: { code, message } }` e estados para entrada ausente/inválida, tamanho excedido, documento inexistente ou não pertencente ao owner e falha de I/O. Distinguir os caminhos backend previstos e o prefixo `/api` usado pelo frontend/proxy.
6. Registrar decisões da Clean Architecture simples (`routes -> controllers -> services -> repositories`), responsabilidades em alto nível e uso de storage local + repositório de metadados em memória. Explicitar que `X-User-Id` é identificação declarada pelo cliente, não autenticação nem garantia de isolamento de segurança.
7. Incluir no documento um plano de execução posterior em etapas de produto (contratos/validações, persistência local e metadados, serviços e fluxo HTTP, integração de interface, testes e validação), sem indicar criação/edição de arquivos backend ou frontend nesta tarefa.

**Relevant files**
- `docs/specs/spec-template.md` — estrutura e requisitos base a preservar e detalhar.
- `.github/copilot-instructions.md` — restrição de armazenamento local e Clean Architecture a respeitar.
- `docs/specs/dms-spec.md` — único artefato a criar nesta tarefa.

**Verification**
1. Conferir que `dms-spec.md` contém as oito seções do modelo, todas as rotas/modelos de sucesso e erro, e critérios verificáveis para as decisões acordadas.
2. Conferir que o plano dentro da spec é apenas uma sequência de marcos futuros, sem executar ou prescrever alterações nos arquivos de aplicação nesta tarefa.
3. Confirmar que o único arquivo criado/alterado é `docs/specs/dms-spec.md`.

**Decisions**
- Sem autenticação nesta fase; `X-User-Id` identifica o owner declarado pelo consumidor e não deve ser apresentado como mecanismo de segurança.
- Tipos de arquivo livres; limite padrão de 10 MiB, ajustável por variável de ambiente.
- `GET /documents` retorna lista simples, sem paginação; erros seguem um envelope JSON uniforme.
- Metadados permanecem em memória e podem ser perdidos ao reiniciar; arquivos permanecem no filesystem local, sem serviço externo.
- Nenhuma implementação de backend/frontend nesta tarefa; somente o documento da especificação será produzido.
