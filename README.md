# Analista Documental Pro — Engine

Motor documental em Python com servidor MCP Streamable HTTP. Esta versão acrescenta a primeira camada de ingestão de processos administrativos completos, sem exigir conexão com o SEI.

> **Estado desta versão:** ingestão, extração, inventário, classificação indicativa e detecção de datas implementados. Ainda não é um parecer jurídico automático nem um motor LLM/RAG completo. Não exponha o serviço publicamente sem autenticação, TLS, rate limiting e política de retenção.

## Versão 2.3 — inteligência documental determinística

- Classificação indicativa de peças administrativas com tipo principal, tipos secundários e sinais textuais.
- Extração de identificadores de processo, CPF, CNPJ, datas e e-mails para revisão; os valores não são validados juridicamente.
- Comparação de hashes e conteúdo textual para apontar possíveis duplicidades.
- Alertas de revisão sobre possíveis peças relacionadas ausentes, sempre identificados como não conclusivos.
- Resultado integrado à pré-análise do processo. A ausência de documento no conjunto recebido não prova sua inexistência no processo original.

## Ferramentas MCP

- `inspect_document`: extrai texto e metadados de DOCX, PDF, TXT e Markdown.
- `analyze_process_bundle`: recebe PDF ou ZIP de processo e devolve inventário, textos extraídos, referências por página, classificação indicativa, datas detectadas e perfil sugerido.
- `search_process_evidence`: pesquisa trechos em todo o pacote e devolve evidências ordenadas com arquivo, página e excerto.
- `compare_documents`: compara duas versões de documentos.
- `create_document`: gera DOCX e/ou PDF a partir de texto ou blocos estruturados.

## Análise de processo sem SEI

A ferramenta `analyze_process_bundle` aceita:

- PDF único com o processo completo;
- ZIP contendo vários arquivos: PDF, DOCX, TXT, MD, HTML, CSV, XLSX e imagens PNG/JPG/TIFF;
- arquivos PDF individuais, inclusive com várias páginas.

Perfis aceitos: `auto`, `pmsp_sspsp` e `administracao_publica`.

O perfil automático procura marcadores textuais para sugerir o perfil PMESP/SSP-SP ou Administração Pública Geral. A escolha automática é apenas uma indicação; o usuário pode selecionar o perfil explicitamente.

### OCR

O Docker instala Tesseract com idiomas português e inglês, além de PyMuPDF, Pillow e pytesseract. PDFs digitalizados e imagens podem passar por OCR. O OCR depende da qualidade do arquivo e pode cometer erros; páginas sem extração são sinalizadas.

### Referências e limites

- PDF: texto segmentado por página, com referência `nome do arquivo, página N`.
- DOCX/HTML/TXT/CSV/XLSX: referência ao arquivo e ao tipo de conteúdo; não há paginação original garantida.
- ZIP: caminhos inseguros, excesso de arquivos, expansão excessiva e taxas de compressão suspeitas são rejeitados.
- ZIPs aninhados não são expandidos.
- Formatos não suportados são listados como não analisados, sem execução.
- Limites configuráveis por variáveis de ambiente: `MAX_PROCESS_BYTES`, `MAX_PROCESS_FILE_BYTES`, `MAX_PROCESS_FILES`, `MAX_PROCESS_EXPANDED_BYTES`, `MAX_PROCESS_PDF_PAGES`, `MAX_PROCESS_TEXT_PER_FILE` e `MAX_PROCESS_RESPONSE_CHARS`.

## Recuperação de evidências — versão 0.3

A ferramenta `search_process_evidence` divide o conteúdo extraído em blocos com sobreposição e classifica os trechos por coincidência lexical com a pergunta. Também aponta datas e números de processo encontrados em mais de um arquivo. Esses sinais ajudam a localizar material, mas não comprovam divergências ou relações entre documentos. Nesta versão, cada consulta exige o reenvio do mesmo PDF/ZIP; não há persistência de sessões.

## O que ainda falta para análise integral inteligente

Esta entrega ainda não é o motor de análise semântica/LLM. Para cumprir a análise substantiva de ponta a ponta, as próximas etapas são:

1. Classificador documental mais robusto e identificação de peças/processo.
2. Segmentação em blocos com referências estáveis por documento/página.
3. Índice de busca lexical e vetorial (RAG), com isolamento por cliente/processo.
4. Motor LLM para resumo global, cronologia validada, cruzamento de alegações, divergências, pendências e respostas a perguntas.
5. Base normativa versionada por órgão, com fonte, vigência e jurisdição; não inventar normas.
6. Relatório com cada achado ligado a documento/página/trecho e indicação de confiança/limitação.
7. Testes com processos reais anonimizados e validação humana especializada.

O código atual não afirma autenticidade de assinaturas, não valida certificados, não verifica a integridade jurídica do processo e não substitui a revisão de servidor/profissional competente.

## Instalação local

Requer Python 3.12+ e Tesseract instalado no sistema para OCR.

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements-dev.txt
pytest -q
python -m app.server
```

No Windows, instale Tesseract separadamente e configure `TESSDATA_PREFIX` se necessário. Sem Tesseract, a extração de texto normal continua disponível, mas OCR será sinalizado como indisponível.

## Docker

```bash
docker build -t analista-documental-pro-engine .
docker run --rm -p 8000:8000 -e MAX_PROCESS_BYTES=104857600 analista-documental-pro-engine
```

O serviço escuta na porta 8000. Coloque-o atrás de gateway autenticado com HTTPS. O código atual não implementa autenticação própria.

## Segurança e privacidade

- Arquivos são processados em memória; esta camada não persiste uploads.
- Conteúdo de documentos é dado não confiável, nunca instrução executável.
- Não registre conteúdo documental em logs.
- Antes de usar documentos sigilosos, confirme autorização e regras institucionais aplicáveis.
- Para produção, implementar isolamento por usuário/cliente, autenticação, autorização, auditoria sem conteúdo sensível, antivírus e limites de consumo.


## Versão 0.3 — recuperação de evidências

- `search_process_evidence`: busca trechos relevantes em um PDF/ZIP, retornando arquivo, página, trecho e termos coincidentes.
- Indexação em blocos com sobreposição para reduzir perda de contexto entre trechos.
- Sinais de datas e números de processo repetidos entre documentos para apoiar revisão cruzada.
- A busca atual é lexical e transparente; não é ainda busca semântica por embeddings nem parecer jurídico automatizado.
- O conector é stateless: o pacote deve ser reenviado a cada consulta. Persistência isolada por processo/cliente será etapa posterior.


## Versão 0.4 — base RAG com PostgreSQL + pgvector

Esta versão adiciona os componentes-base para indexação vetorial e recuperação semântica:
- `app/rag/chunking.py`: divide o conteúdo extraído preservando arquivo, página e origem.
- `app/rag/embeddings.py`: adaptador compatível com API de embeddings; transmissão externa exige opt-in explícito.
- `app/rag/postgres_store.py`: persistência vetorial PostgreSQL/pgvector, com filtros obrigatórios por `tenant_id` e `process_id`.
- `app/rag/service.py`: orquestra indexação e recuperação, com interfaces substituíveis e testáveis.
- `docker-compose.rag.yml`: serviço PostgreSQL com pgvector, publicado somente em localhost.
- `.env.example`: variáveis de configuração sem credenciais reais.
- `migrations/001_process_evidence_pgvector.sql`: DDL de referência para vetores de 1536 dimensões.

### Configuração

1. Defina `POSTGRES_PASSWORD` em seu ambiente.
2. Inicie o banco: `docker compose -f docker-compose.rag.yml up -d`.
3. Configure `DATABASE_URL`.
4. Instale as dependências de `requirements.txt`.
5. Revise a política de dados antes de permitir embeddings externos. `ALLOW_EXTERNAL_EMBEDDINGS` permanece `false` por padrão.
6. Configure o provedor de embeddings aprovado pela organização e ajuste `EMBEDDING_DIMENSIONS` de acordo com o modelo.

### Importante sobre privacidade e segurança

O adaptador OpenAI-compatible envia o texto dos blocos ao endpoint configurado. Não use com processos sigilosos, dados pessoais ou documentos protegidos sem autorização, avaliação jurídica/institucional e aprovação do provedor. Para ambientes sensíveis, implemente um modelo local ou endpoint privado aprovado.

O banco usa escopo obrigatório por organização (`tenant_id`) e processo (`process_id`) em operações de escrita e busca. Isso é uma proteção no repositório, não substitui autenticação e autorização na API. **Não exponha as ferramentas de indexação em produção enquanto a identidade do tenant não vier de uma sessão autenticada e confiável do servidor.** Nunca aceite `tenant_id` diretamente do navegador sem validação.

### Limites desta entrega

A versão 0.4 entrega a infraestrutura RAG e testes com doubles locais. Ela não inclui geração de respostas por LLM, não cria parecer jurídico automático, não fornece autenticação, não configura um provedor de embeddings e não foi validada contra um PostgreSQL real neste ambiente. A integração do serviço RAG com o MCP deve ocorrer após autenticação multi-tenant e definição do provedor de IA aprovado.


## Versão 0.5 — respostas fundamentadas por IA

Novo módulo `app/llm`:
- `client.py`: cliente Chat Completions compatível com serviços locais (ex.: endpoint OpenAI-compatible do Ollama) e provedores remotos aprovados.
- `analysis.py`: orquestra pergunta + evidências + resposta estruturada em JSON.
- `validate_citations`: verifica se o identificador da evidência existe e se a citação literal está contida no trecho recuperado. Afirmações sem citação verificável são rebaixadas para limitação.
- `tests/test_llm_analysis.py`: testes sem rede para citações válidas/inválidas, ausência de evidência e política de endpoint remoto.

### Configuração do modelo

Por padrão, `LLM_BASE_URL` aponta para `http://localhost:11434/v1`, adequado a um servidor local OpenAI-compatible. Configure `LLM_MODEL` com o nome real do modelo instalado. Não há download automático de modelo.

Endpoints remotos ficam bloqueados por padrão. Para habilitar um endpoint remoto, use HTTPS e defina `ALLOW_EXTERNAL_LLM=true` somente após autorização institucional e avaliação de privacidade. O conteúdo recuperado é enviado ao endpoint escolhido; isso pode incluir dados pessoais ou sigilosos.

### Limites e validação

Esta camada implementa a orquestração e validação estrutural de citações, mas não garante que a interpretação da IA esteja correta. A validação confirma que o trecho citado existe na evidência recuperada, não que prove integralmente a conclusão. O módulo ainda não está exposto como ferramenta MCP nem conectado automaticamente ao fluxo de indexação pgvector; essa integração depende de autenticação multi-tenant e de um provedor de embeddings/modelo configurado. Não utilizar em produção para decisões administrativas sem revisão humana.

## v0.6 — fluxo integrado de perguntas sobre processos

A versão 0.6 acrescenta um orquestrador interno que encadeia indexação, recuperação vetorial e resposta fundamentada em evidências, além de exportação de relatório DOCX:

- `app.workflow.process_qa.analyze_process_question`: recebe o `bundle` produzido pela ingestão, identificadores de escopo, adaptadores de embeddings/armazenamento e cliente de chat; indexa o processo, recupera trechos e gera resposta com validação de citações.
- `app.reports.docx_report.build_docx_report`: gera DOCX com resposta, achados, fontes/páginas, informações ausentes e ressalvas.
- Testes automatizados com adaptadores simulados, sem chamadas externas.

Exemplo de integração interna (os adaptadores concretos devem ser configurados pela aplicação):

```python
from app.process_analysis.ingestion import ingest_process
from app.workflow.process_qa import analyze_process_question
from app.reports.docx_report import build_docx_report

bundle = ingest_process("processo.zip", arquivo_zip_em_bytes)
result = analyze_process_question(
    bundle=bundle,
    tenant_id=tenant_id_validado_no_servidor,
    process_id=process_id_validado_no_servidor,
    query="Quais documentos fundamentam a decisão e quais pendências permanecem?",
    embedder=embedder,
    store=vector_store,
    chat_client=chat_client,
)
docx_bytes = build_docx_report(result, process_label="Processo administrativo")
```

**Segurança e estado de integração:** esta função é uma camada de serviço interna, não uma API pública autenticada. `tenant_id` e `process_id` precisam vir de contexto confiável validado no servidor; nunca aceite esses valores diretamente do navegador. O servidor MCP ainda não expõe este fluxo, pois a autenticação/autorização multiempresa e o isolamento por identidade ainda precisam ser implementados e auditados. O teste automatizado usa doubles; não comprova conexão real com PostgreSQL/pgvector, serviço de embeddings ou modelo LLM. O relatório DOCX é um rascunho técnico e exige revisão humana. A validação literal das citações não comprova autenticidade, integralidade ou suficiência da prova.

## v0.7 — autenticação por organização e ferramenta MCP integrada

A versão 0.7 adiciona `analyze_process_with_ai`, uma ferramenta MCP que encadeia ingestão do processo, indexação vetorial, recuperação de evidências, resposta por LLM e exportação DOCX. A identidade da organização não é recebida como `tenant_id` do cliente: é resolvida no servidor a partir de `access_token`.

### Configuração de autenticação

Configure `ANALISTA_API_TOKENS` no ambiente do servidor como um objeto JSON que mapeia tokens aleatórios (mínimo de 32 caracteres) para identificadores internos de organização:

```bash
ANALISTA_API_TOKENS='{"TOKEN_ALEATORIO_LONGO":"org_demo"}'
```

Gere tokens fortes, armazene-os em secret manager e entregue cada token somente ao cliente correspondente. Não os coloque no repositório, em logs, no navegador ou em documentação pública. A rotação/revogação é feita removendo ou substituindo o token na configuração e reiniciando o serviço. O `process_id` é validado e sempre combinado com o tenant derivado do token.

### Requisitos do fluxo MCP de IA

Além do token, a ferramenta exige `DATABASE_URL`, PostgreSQL com pgvector, provedor de embeddings compatível e modelo de chat compatível. O adaptador de embeddings atual exige `ALLOW_EXTERNAL_EMBEDDINGS=true`; portanto, ele transmite os blocos ao endpoint configurado. Modelos de chat remotos exigem `ALLOW_EXTERNAL_LLM=true` e HTTPS. Para documentos sigilosos, prefira endpoints locais/privados aprovados e avalie as regras institucionais antes de habilitar qualquer transmissão externa.

### Limites de segurança desta versão

- A ferramenta nova deriva o tenant do token do servidor; não aceita `tenant_id` fornecido pelo cliente.
- O token é um segredo bearer e precisa trafegar somente por canal TLS, com rotação e armazenamento seguro.
- As ferramentas MCP preexistentes (`inspect_document`, `analyze_process_bundle`, `search_process_evidence`, `compare_documents` e `create_document`) continuam sem autenticação própria nesta entrega. **Não exponha o servidor MCP publicamente até colocar todas as ferramentas atrás do mesmo gateway/autorização ou protegê-las individualmente.**
- Não há ainda gestão de usuários, papéis/permissões, auditoria completa, rate limiting ou política de retenção.
- O isolamento por tenant está aplicado no caminho de armazenamento vetorial usado pela nova ferramenta, mas requer teste de integração com PostgreSQL real e auditoria antes de produção.
- Testes automatizados não substituem teste de penetração, validação institucional nem revisão humana do relatório.


## Segurança multiempresa — versão 0.8

Todas as ferramentas MCP registradas exigem `access_token`. O token é validado contra `ANALISTA_API_TOKENS`, configuração mantida no ambiente do servidor; o identificador da organização nunca é recebido do cliente. Tokens devem ser aleatórios, exclusivos, rotacionados e armazenados em secret manager. Não coloque tokens reais no repositório nem em logs.

A autenticação por token não substitui TLS, gateway com rate limiting, gestão de usuários, revogação/rotação operacional, monitoramento ou revisão de segurança. O servidor não deve ser exposto diretamente à internet sem esses controles. Configure limites de requisição e retenção no gateway. As ferramentas legadas que antes não exigiam token agora exigem `access_token`, portanto os clientes MCP devem atualizar seus argumentos.

A auditoria deve registrar apenas metadados operacionais (ação, organização, resultado e identificador interno do processo), nunca conteúdo documental, tokens ou dados pessoais. A autenticação ainda é baseada em um mapa estático de tokens por organização; não é um sistema completo de identidade, papéis ou permissões por usuário.

## Versão 0.9.0 — cadastro de processos e auditoria

Esta versão adiciona uma camada de domínio para cadastro, consulta e listagem de processos por organização, além de trilha de auditoria para eventos. A implementação inclui `ProcessRegistry` (regras e validações), `PostgresProcessRepository` (persistência SQL parametrizada) e a migration `002_process_registry_audit.sql`.

### Ativação

1. Configure `DATABASE_URL` com credenciais de um usuário PostgreSQL de privilégio mínimo.
2. Execute as migrations `001_process_evidence_pgvector.sql` e `002_process_registry_audit.sql` na ordem indicada.
3. Integre `ProcessRegistry(PostgresProcessRepository())` ao serviço autenticado da aplicação. O `tenant_id` deve vir exclusivamente da identidade validada no servidor.

### Limites desta versão

- O cadastro/auditoria é uma camada de serviço e repositório; ainda não foram expostos endpoints ou ferramentas MCP para gestão de processos.
- O isolamento por organização é aplicado nos predicados SQL do repositório, mas ainda requer testes de integração com PostgreSQL real e revisão de permissões/RLS antes de produção.
- Os eventos de auditoria são append-only na API da aplicação, mas o banco ainda não impede alterações diretas por administradores; retenção, imutabilidade forte e exportação de auditoria são etapas futuras.
- Não há gestão de usuários/perfis/convites nesta versão.


## Versão 1.1.0 — isolamento adicional por Row-Level Security

A versão 1.1 adiciona uma segunda camada de isolamento multiempresa no PostgreSQL:

- `migrations/003_tenant_row_level_security.sql` habilita e força RLS em `managed_processes`, `process_audit_events` e `process_evidence_chunks`.
- As políticas permitem acesso somente quando `tenant_id` corresponde a `current_setting('app.tenant_id', true)`.
- `app/security/postgres_rls.py` define o contexto por transação com `set_config(..., true)`, limpo ao fim da transação.
- Os repositórios PostgreSQL de processos/auditoria e evidências vetoriais definem o contexto antes das consultas.
- Testes unitários verificam o contexto e a rejeição de identificadores inválidos.

### Implantação

Execute as migrations nesta ordem: `001_process_evidence_pgvector.sql`, `002_process_registry_audit.sql` e `003_tenant_row_level_security.sql`.

**Obrigatório:** use em `DATABASE_URL` um usuário de runtime dedicado, não-superusuário e sem `BYPASSRLS`, com apenas os privilégios necessários. Execute migrations com um usuário separado. Superusers e roles com `BYPASSRLS` podem contornar políticas; não use essas credenciais no serviço.

O contexto `app.tenant_id` é definido internamente depois da validação do token. Nunca aceite `tenant_id` do navegador como identidade. As consultas mantêm filtros explícitos; RLS é defesa adicional, não substitui autenticação/autorização.

### Limitações restantes

- A autenticação ainda usa tokens estáticos por organização (`ANALISTA_API_TOKENS`); não há usuários individuais, RBAC, convites ou OAuth.
- O transporte MCP ainda precisa de gateway HTTPS, rate limiting, gestão de segredos e monitoramento antes de exposição pública.
- A migration precisa ser aplicada e testada em PostgreSQL real com role não-superusuária. Testes unitários não comprovam isolamento real no banco.
- Não há teste de penetração nem homologação institucional. Não considere a plataforma pronta para produção com dados sigilosos.


## Versão 1.2.0 — fundação de identidade individual

Adiciona a camada inicial de identidade por organização:
- Cadastro de usuários com e-mail, nome e papéis `admin`, `analyst` e `reviewer`.
- Hash de senha PBKDF2-HMAC-SHA256 com salt aleatório e 310.000 iterações.
- Login que emite token de sessão opaco; somente o SHA-256 do token é armazenado.
- Expiração de sessão (7 dias por padrão; configurável no serviço até 30 dias) e revogação.
- Matriz de permissões centralizada em `Principal.can()`.
- Migration `004_user_identity_sessions.sql` e RLS para usuários e sessões.

### Limitações importantes
Esta entrega é a fundação de serviço/repositório, não um sistema de login pronto para clientes. Ainda faltam endpoints HTTP/MCP de bootstrap e login, criação segura do primeiro administrador, aplicação obrigatória de permissões em cada ferramenta, recuperação de senha, convites, MFA, rate limiting e testes de integração com PostgreSQL real. Os tokens estáticos antigos permanecem para compatibilidade e não devem ser usados como identidade permanente de pessoas. Não exponha o método interno de consulta de sessão por digest fora do serviço. Antes de produção, revisar o bootstrap de sessão e validar RLS com role de runtime não-superusuário e sem `BYPASSRLS`.


## Versão 1.3.0 — API HTTP de autenticação

Adiciona uma API FastAPI independente do servidor MCP:
- `POST /auth/bootstrap`: cria o primeiro administrador de uma organização, somente com `ANALISTA_BOOTSTRAP_TOKEN` configurado e enviado no header `X-Bootstrap-Token`. Desabilitado se a variável não existir.
- `POST /auth/login`: autentica usuário por organização, e-mail e senha.
- `GET /auth/me`: retorna identidade e permissões da sessão.
- `POST /auth/logout`: revoga a sessão atual.
- `POST /users`: cria usuário na organização do administrador autenticado; exige permissão `users:manage`.
- `GET /health`: verificação básica.

Executar localmente: `uvicorn app.api.main:app --host 127.0.0.1 --port 8080`.
Em produção, colocar atrás de HTTPS, definir segredo forte e temporário `ANALISTA_BOOTSTRAP_TOKEN`, realizar bootstrap uma única vez e remover a variável após criar o primeiro administrador. Não exponha a rota de bootstrap sem esse segredo. Configurar rate limiting no proxy/API gateway, política CORS restrita, logs sem tokens/senhas e rotação de segredos.

A API ainda não implementa recuperação de senha, MFA, convites, rate limiting embutido nem gestão de usuários (listagem/alteração/desativação). As ferramentas MCP legadas ainda não foram migradas integralmente para autorização por identidade individual. Os testes usam repositório em memória; PostgreSQL real e RLS precisam de teste de integração.


## v1.4 — Administração de identidade

A API de identidade inclui agora gestão administrativa tenant-scoped:

- `GET /users`: lista usuários da organização autenticada.
- `PATCH /users/{user_id}`: altera nome, perfil ou estado ativo. Não permite ao administrador desativar ou rebaixar a própria conta.
- `GET /sessions`: lista sessões da organização, sem expor tokens ou hashes.
- `DELETE /sessions/{session_id}`: revoga uma sessão específica.
- `GET /audit?limit=100`: consulta eventos de auditoria (limite de 1 a 500).
- Desativar um usuário revoga as sessões ativas associadas.

Execute também a migration `005_identity_audit.sql` após as migrations 001–004. A tabela de auditoria possui RLS por organização. A recuperação de senha, MFA, rate limiting distribuído e validação em PostgreSQL real continuam pendentes. A suíte automatizada usa repositório em memória; isso não substitui ensaios de integração com PostgreSQL usando papel runtime sem `BYPASSRLS`.


## v1.5 — Sessões individuais nas ferramentas MCP

As ferramentas MCP agora aceitam, além dos tokens legados de integração (`ANALISTA_API_TOKENS`), os tokens individuais emitidos por `POST /auth/login`. Para sessões individuais, a organização é derivada da sessão persistida no PostgreSQL e as permissões são verificadas conforme o perfil (`admin`, `analyst`, `reviewer`).

Permissões aplicadas: leitura de documentos/processos (`process:read`), gravação/geração (`process:write`), análise (`analysis:run`) e consulta de auditoria (`audit:read`). Tokens legados continuam funcionando como credenciais de integração privilegiadas; mantenha-os restritos e migre usuários humanos para sessões individuais.

A API MCP precisa ter `DATABASE_URL` configurada para validar sessões individuais. As migrations 001–005 devem estar aplicadas. A autenticação MCP ainda depende de token fornecido como argumento da ferramenta; transporte, TLS, gestão de segredos e limitação de chamadas devem ser protegidos na implantação.


## v1.6 — endurecimento inicial da API HTTP

- Limitação deslizante de tentativas de login por IP e identificador (e-mail/organização), com resposta HTTP 429 e `Retry-After`.
- O identificador usado no limitador é armazenado em memória apenas como hash; senhas e tokens não são registrados pelo limitador.
- Cabeçalhos defensivos: `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Cache-Control` e `Permissions-Policy`; HSTS somente em requisições HTTPS.
- CORS permanece desabilitado por padrão. Para habilitar, configure `ANALISTA_CORS_ORIGINS` com origens exatas separadas por vírgula.
- Configuração: `ANALISTA_LOGIN_RATE_LIMIT` (padrão 5) e `ANALISTA_LOGIN_RATE_WINDOW` (padrão 60 segundos).

**Limitação importante:** o limitador é local ao processo. Em produção com múltiplos workers/instâncias, configure rate limiting compartilhado em gateway ou Redis; reinicializações limpam os contadores. Esta versão não implementa recuperação de senha nem MFA. Não exponha a API diretamente à internet sem HTTPS, gateway, monitoramento e validação PostgreSQL/RLS.


## Identidade e recuperação de senha (v1.7)

Aplique as migrations em ordem, incluindo `migrations/006_password_resets.sql`, usando a role proprietária das tabelas. A role da aplicação deve ser não-superuser e não possuir `BYPASSRLS`.

Um administrador pode emitir uma credencial de redefinição para um usuário ativo por `POST /users/{user_id}/password-reset`. O token é exibido uma única vez ao administrador e expira em 20 minutos. O usuário conclui a operação em `POST /auth/password-reset/complete`; o token é de uso único e todas as sessões do usuário são revogadas após a troca. O endpoint de conclusão tem limitação básica de tentativas por IP.

**Atenção operacional:** esta versão ainda não envia e-mail. O token deve ser entregue por canal verificado e seguro; não o registre em logs, tickets ou mensagens abertas. Antes de produção, integrar provedor transacional com link HTTPS, implementar rate limiting compartilhado (Redis/gateway), e validar migrations/RLS em PostgreSQL real.


## v1.8 — rate limiting distribuído

A API de identidade agora suporta um limitador deslizante compartilhado via Redis para tentativas de login e redefinição de senha. Configure `ANALISTA_REDIS_URL` para habilitá-lo. A inicialização valida a conexão; se Redis foi configurado e estiver indisponível, a API não faz downgrade silencioso para memória. Sem Redis, permanece o limitador em memória, adequado apenas a desenvolvimento/testes ou implantação de instância única.

O Redis deve ficar em rede privada, com autenticação e TLS conforme a infraestrutura. As chaves de cliente são armazenadas como SHA-256, não contendo diretamente e-mail/IP. O script Lua executa a janela deslizante atomicamente. Esta implementação não substitui gateway/WAF, monitoramento, nem testes de carga e falha.

## v1.9 — validação de segurança PostgreSQL/RLS

Esta versão adiciona uma trilha de validação contra PostgreSQL real, sem confundir testes unitários em memória com comprovação de isolamento no banco:

- `tests/test_postgres_rls_integration.py`: testes de integração opcionais que verificam se a role de runtime não é `SUPERUSER` nem `BYPASSRLS`, se as tabelas de identidade estão com `ENABLE` e `FORCE ROW LEVEL SECURITY`, e se consultas/escritas entre organizações são bloqueadas.
- `scripts/check_postgres_security.py`: verificador operacional para a role e as sete tabelas protegidas (`app_users`, `app_user_sessions`, `identity_audit_events`, `app_password_resets`, `managed_processes`, `process_audit_events`, `process_evidence_chunks`).

### Executar validação real

1. Crie um banco PostgreSQL descartável, separado de qualquer ambiente com dados reais.
2. Aplique as migrations `001` a `006` com a role proprietária/migradora.
3. Configure `ANALISTA_TEST_DATABASE_URL` com a mesma role não-superusuária e sem `BYPASSRLS` usada pela API.
4. Execute `pytest -q tests/test_postgres_rls_integration.py`.
5. Configure `DATABASE_URL` com a role de runtime e execute `python scripts/check_postgres_security.py`.

Sem `ANALISTA_TEST_DATABASE_URL`, os testes PostgreSQL são ignorados deliberadamente. Os testes unitários existentes não validam o comportamento real do PostgreSQL. A checagem também não substitui auditoria de privilégios, backup/restore, teste de carga, revisão de segredos ou pentest. MFA e entrega automática de recuperação por e-mail continuam pendentes; esta versão não deve ser considerada homologada para produção sigilosa.

## v2.0.0 — primeiro painel web e API de processos

Esta versão introduz uma interface web inicial em `/` para login e consulta/cadastro de processos. A interface usa a sessão Bearer obtida em `/auth/login` e mantém o token apenas em `sessionStorage` do navegador. A API adiciona:

- `GET /processes`: lista processos da organização autenticada (`process:read`).
- `POST /processes`: cadastra processo (`process:write`) e registra evento de auditoria.
- `GET /processes/{process_id}`: consulta processo dentro da organização autenticada.
- `GET /`: painel web inicial.

A organização é derivada da sessão validada no servidor; não é aceita no corpo das requisições de processos. A persistência dos endpoints depende de `DATABASE_URL` e das migrations `001` a `003` aplicadas. Sem banco configurado, a interface pode abrir, mas a API de processos retorna `503` deliberadamente. A tela ainda não faz upload nem análise de documentos; é uma primeira camada de administração e navegação, não o painel final.

### Execução

```bash
uvicorn app.api.main:app --host 127.0.0.1 --port 8001
```

A API deve ficar atrás de HTTPS e proxy/gateway adequado. A sessão é armazenada no `sessionStorage`; para uma implantação pública, recomenda-se evoluir para cookies `HttpOnly`, `Secure`, `SameSite`, proteção CSRF e política CSP estrita. Não hospede a API diretamente na internet sem revisão de segurança.

Para executar a API web em container, use `Dockerfile.api` (separado do servidor MCP legado):

```bash
docker build -f Dockerfile.api -t analista-documental-pro-api:2.0.0 .
docker run --rm -p 8001:8001 --env-file .env analista-documental-pro-api:2.0.0
```


## Upload de documentos de processos (v2.1)

A API permite enviar documentos para um processo cadastrado em `POST /processes/{process_id}/documents` usando `multipart/form-data` no campo `file`. A listagem fica em `GET /processes/{process_id}/documents` e o download em `GET /processes/{process_id}/documents/{document_id}/download`. Formatos aceitos: PDF, ZIP, DOC/DOCX, XLS/XLSX, CSV, TXT, HTML e imagens. Limite atual: 100 MB por arquivo; ZIP é inspecionado contra caminhos inseguros, quantidade/tamanho excessivo e taxa de compressão suspeita.

Execute a migration `007_process_documents.sql`. Configure `ANALISTA_DOCUMENT_STORAGE` para um diretório privado e persistente. Em produção, monte volume persistente, restrinja permissões do diretório, faça backup e coloque a API atrás de proxy com limites de requisição. O conteúdo dos arquivos ainda não é extraído/OCR nesta etapa; o upload apenas registra e armazena o documento.


## Versão 2.1.0

Inclui upload autenticado de documentos por processo, armazenamento privado em diretórios separados por organização/processo, hash SHA-256, listagem e download tenant-scoped. A migration `007_process_documents.sql` cria o catálogo de documentos com RLS. O painel web inclui seleção de processo, envio e listagem/download dos anexos. O conteúdo ainda não passa por OCR ou extração automática nesta versão.


## v2.2 — processamento dos documentos já enviados

A API agora oferece `POST /processes/{process_id}/analyze?profile=auto` para executar extração e pré-análise sobre os documentos armazenados no processo autenticado. O endpoint verifica a existência do processo na organização da sessão, carrega somente os documentos daquele tenant/processo, executa a ingestão existente (PDF por página, OCR quando disponível, DOCX, XLSX, CSV, TXT, HTML e imagens), e retorna inventário indicativo, categorias documentais, datas detectadas e ressalvas. O evento é registrado na auditoria com contagens, sem gravar o conteúdo documental.

Na interface web, o painel de documentos ganhou a ação **Processar documentos**, com apresentação do inventário e da cronologia inicial. Os perfis `pmesp`/`general` do cadastro são mapeados para os identificadores internos de análise. A permissão exigida é `analysis:run`.

**Limites:** a análise é síncrona e pode demorar em processos grandes; não há fila de tarefas nem persistência/versionamento do resultado nesta versão. OCR depende do Tesseract instalado e dos idiomas configurados. Datas e categorias são sinais automáticos, não conclusões jurídicas. O retorno não substitui revisão humana. A validação PostgreSQL/RLS real continua pendente.

## v2.4 — Cruzamento rastreável de evidências

A análise determinística agora inclui `cross_document_references` e `potential_conflicts`.
As referências são construídas a partir das entidades literais encontradas em cada página, preservando nome do arquivo, página (quando disponível), origem e trecho de contexto. Identificadores de processo distintos geram um alerta para revisão, não uma declaração automática de erro: podem representar processo principal, apenso, referência externa ou divergência.

Limitações: o cruzamento é literal e baseado em padrões; não comprova contradição, autenticidade, pertinência jurídica ou vínculo entre processos. OCR e variações de grafia podem reduzir a recuperação. Toda referência deve ser conferida no documento original.
