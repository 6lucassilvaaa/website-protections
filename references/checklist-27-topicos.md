# Checklist Completo — Os 27 Tópicos de Proteção

Cada tópico abaixo corresponde exatamente aos **27 pontos** apresentados no vídeo de referência. Na Fase 1, verifique todos os 27 itens contra o código real do projeto.

---

### 01/27 — Configure uma autenticação adequada
- **Princípio:** Nunca confiar apenas no frontend para decidir se o usuário está autenticado. Toda verificação ocorre no servidor/cofre de segredos.
- **O que verificar:**
  - Fluxo de login valida credenciais no servidor (hash da senha + status da conta ativa/não bloqueada/verificada).
  - Sessões ou tokens são validados no servidor em cada rota protegida/API.
  - Não há bypass de autenticação em rotas de API, Server Actions ou endpoints internos.
- **Critério ✅:** Autenticação validada no servidor com checagem de conta ativa e sessão/token verificado em toda rota protegida.

### 02/27 — Esconda as chaves de API
- **Princípio:** Chaves secretas de terceiros (pagamentos, IA, clima, mapas privados, e-mail, etc.) jamais podem rodar no navegador.
- **O que verificar:**
  - Procurar chamadas `fetch`/SDK no código do cliente (`src/`, `app/`, `components/`, `public/`) contendo `sk_`, `apiKey`, `secret`, `Authorization: Bearer ...` hardcoded ou vindas de variáveis públicas.
  - Confirmar se integrações externas com chave privada passam por um endpoint proxy no servidor (`/api/...`).
- **Critério ✅:** Nenhuma chave secreta no código enviado ao navegador; chamadas sensíveis intermediadas pelo servidor.

### 03/27 — Proteja meu banco de dados (Autorização / RLS / Anti-IDOR)
- **Princípio:** Usuário autenticado só pode ler/alterar os próprios registros ("somente o dono"), salvo permissão explícita.
- **O que verificar:**
  - Queries de leitura, atualização e exclusão filtram por `user_id` / `owner_id` vindo da sessão autenticada (nunca apenas pelo ID enviado no body/URL).
  - Em bancos como Supabase/Postgres, verificar se **Row-Level Security (RLS)** está habilitado com políticas restritivas por usuário.
  - Verificar se credenciais do banco (`DATABASE_URL`, `service_role`) ficam exclusivamente no servidor.
- **Critério ✅:** Isolamento por dono validado no servidor e/ou políticas de RLS ativas para todas as tabelas sensíveis.

### 04/27 — Exija HTTPS
- **Princípio:** Todo tráfego (principalmente login, formulários e sessões) deve ser criptografado em trânsito.
- **O que verificar:**
  - URLs de API, webhooks, callbacks OAuth e links gerados usam `https://` em produção.
  - Redirecionamento HTTP → HTTPS configurado no servidor/proxy/plataforma, além de HSTS (`Strict-Transport-Security`).
  - Cookies e links não trafegam em `http://` fora de `localhost`.
- **Critério ✅:** HTTPS exigido em produção, URLs configuradas com `https://` e redirecionamento/HSTS presentes.

### 05/27 — Armazene senhas com hash (Argon2id)
- **Princípio:** Senhas nunca são armazenadas em texto puro ou criptografia reversível ("sem caminho de volta").
- **O que verificar:**
  - Verificar como senhas são salvas no cadastro e comparadas no login.
  - Preferir **Argon2id** (recomendação direta do vídeo); se o projeto já usar `bcrypt` ou `scrypt` com custo adequado, sinalizar como aceitável ou recomendar migração para Argon2id se estiver fraco/legado (MD5, SHA-1, SHA-256 simples ou texto puro = `❌ Faltando`).
  - Garantir que `password_hash` nunca é retornado em respostas da API.
- **Critério ✅:** Hashing forte com Argon2id (ou bcrypt/scrypt robusto com salt) e nunca exposição do hash nas respostas.

### 06/27 — Valide meus formulários no servidor
- **Princípio:** Validação no HTML/JavaScript do navegador é só UX; qualquer requisição pode ser adulterada.
- **O que verificar:**
  - Todos os endpoints POST/PUT/PATCH e Server Actions validam tipos, formatos, tamanhos e regras de negócio no servidor (ex.: `quantity: 1 to 10`, proibindo quantidades negativas como `-3`).
  - Presença de validação por schema (Zod, Valibot, Joi, Pydantic, Bean Validation, etc.) no backend.
- **Critério ✅:** Todos os formulários e payloads de API têm validação rigorosa de schema e regras no servidor.

### 07/27 — Proteja contra XSS (Cross-Site Scripting)
- **Princípio:** Dados não confiáveis enviados por usuários (comentários, perfis, buscas) devem ser codificados/sanitizados na saída para não executarem `<script>`.
- **O que verificar:**
  - Uso de `dangerouslySetInnerHTML`, `innerHTML`, `outerHTML`, `document.write`, `v-html`, `{@html}`, `eval()`, `new Function()`, ou templates sem escape automático.
  - Renderização de HTML rico sem sanitizador confiável (ex.: `DOMPurify`).
  - URLs vindas do usuário injetadas em `href`/`src` sem bloquear `javascript:`.
- **Critério ✅:** Saída escapada por padrão, zero sinks perigosos sem sanitização rigorosa e bloqueio de `javascript:` URLs.

### 08/27 — Registre e monitore atividades suspeitas
- **Princípio:** Registrar eventos de segurança ao vivo e alertar comportamentos anômalos (ex.: 5 falhas de login em 2 segundos).
- **O que verificar:**
  - Logs estruturados para logins bem-sucedidos/falhos, bloqueios 401/403, redefinição de senha, ações administrativas e eventos de pagamento.
  - Detecção/alerta para picos de falhas (brute force, varredura, abuso).
  - Confirmar que logs **não** gravam senhas, tokens de sessão ou chaves secretas em texto puro.
- **Critério ✅:** Logging de eventos críticos implementado sem vazamento de segredos, com mecanismo de detecção/alerta para anomalias.

### 09/27 — Restrinja minhas rotas de administrador
- **Princípio:** Verificar o perfil/cargo (`admin`) no servidor a cada requisição; esconder link no menu não protege nada.
- **O que verificar:**
  - Rotas `/admin`, painéis administrativos, endpoints de usuários/reembolsos/configurações exigem sessão válida **e** verificação explícita de role/permissão no servidor, retornando `403` quando usuário comum tenta acessar.
- **Critério ✅:** Middleware/guard no servidor valida role de administrador em todas as rotas e ações administrativas, retornando `403` para não-admins.

### 10/27 — Use consultas parametrizadas
- **Princípio:** Manter estrutura fixa da query para impedir SQL/NoSQL Injection.
- **O que verificar:**
  - Procurar concatenação ou interpolação de strings em SQL/queries (`"SELECT ... " + input`, `` `SELECT ... ${input}` ``, `f"SELECT ... {input}"`, `$where`, queries raw).
  - Confirmar uso de placeholders (`$1`, `?`, `:param`) ou ORM seguro sem raw strings interpoladas.
- **Critério ✅:** Todas as consultas ao banco usam placeholders parametrizados ou métodos seguros do ORM.

### 11/27 — Confira minhas variáveis de ambiente
- **Princípio:** Separar estritamente o que vai para o pacote do navegador (`PUBLIC_*`, `NEXT_PUBLIC_*`, `VITE_*`, `REACT_APP_*`, `EXPO_PUBLIC_*`) do que é somente servidor.
- **O que verificar:**
  - Auditar `.env`, `.env.local`, `.env.example` e usos de `process.env` / `import.meta.env`.
  - Garantir que nenhum segredo (`PAY_SECRET`, `DATABASE_URL`, `JWT_SECRET`, `API_KEY`, `WEBHOOK_SECRET`, `SERVICE_ROLE`) tenha prefixo público.
  - Verificar se `.env*` reais estão no `.gitignore` (exceto `.env.example` sem segredos).
- **Critério ✅:** Nenhuma variável sensível exposta com prefixo público ou versionada no repositório; variáveis críticas validadas no startup do servidor.

### 12/27 — Adicione limites de requisições (Rate Limiting)
- **Princípio:** Bloquear rajadas abusivas retornando `429 Too Many Requests`.
- **O que verificar:**
  - Presença de rate limiter em rotas de autenticação (`/login`, `/register`, `/forgot-password`, OTP), APIs públicas e endpoints custosos (IA, e-mail, upload, checkout).
  - Configuração de janela e limite por IP/usuário com resposta `429`.
- **Critério ✅:** Rate limiting ativo nas rotas críticas e públicas com resposta `429` adequada.

### 13/27 — Restrinja as configurações de CORS
- **Princípio:** Permitir apenas origens conhecidas e autorizadas; bloquear origens desconhecidas.
- **O que verificar:**
  - Procurar `Access-Control-Allow-Origin: *` ou `origin: true` / reflexão cega do cabeçalho `Origin` junto com cookies/credenciais.
  - Verificar lista explícita de origens permitidas (allowlist) controlada por configuração de ambiente, métodos e headers restritos.
- **Critério ✅:** Política CORS usa allowlist explícita de origens confiáveis (sem wildcard permissivo com credenciais).

### 14/27 — Adicione cabeçalhos de segurança
- **Princípio:** Instruir o navegador sobre o que pode carregar, exigir HTTPS e impedir enquadramento (clickjacking).
- **O que verificar:**
  - `Content-Security-Policy` (CSP) — controlando scripts, estilos, conexões e `frame-ancestors`.
  - `Strict-Transport-Security` (HSTS) — somente HTTPS.
  - `X-Frame-Options: DENY` (ou `SAMEORIGIN` quando estritamente necessário) — sem enquadramento em iframes de terceiros.
  - Complementos padrão recomendados: `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `Permissions-Policy`.
- **Critério ✅:** Cabeçalhos de segurança configurados nas respostas HTTP do app/servidor/proxy.

### 15/27 — Proteja os cookies de sessão
- **Princípio:** Cookies de sessão não podem ser lidos por scripts da página nem enviados sem criptografia ou de forma insegura entre sites.
- **O que verificar:**
  - Flags `HttpOnly` (sem acesso por `document.cookie`), `Secure` (somente HTTPS em produção) e `SameSite=Lax` ou `SameSite=Strict`.
  - Verificar se tokens de sessão/JWT estão sendo salvos em `localStorage` ou `sessionStorage` (sinalizar como vulnerável a XSS).
- **Critério ✅:** Sessões em cookies com `HttpOnly`, `Secure` (produção) e `SameSite=Lax`/`Strict`, nunca em `localStorage`.

### 16/27 — Adicione proteção contra CSRF
- **Princípio:** Impedir que formulários ocultos ou requisições forjadas de outros sites executem ações em nome do usuário autenticado.
- **O que verificar:**
  - Endpoints que alteram estado (POST, PUT, PATCH, DELETE) validam token anti-CSRF e/ou cabeçalhos `Origin`/`Referer` quando usam cookies de sessão (além de `SameSite`).
  - Exceções (como webhooks externos assinados) continuam protegidas por assinatura criptográfica própria.
- **Critério ✅:** Proteção CSRF ativa para todas as mutações autenticadas por cookie.

### 17/27 — Faça os links de redefinição de senha expirarem
- **Princípio:** Links de reset/recuperação devem ter validade curta (ex.: 15 minutos) e serem de uso único ("sem reutilização").
- **O que verificar:**
  - Tokens gerados com aleatoriedade criptográfica (`crypto.randomBytes` / `secrets.token_urlsafe`), armazenados com hash no banco.
  - Campo `expires_at` curto (ex.: 15 min) validado no servidor.
  - Invalidação imediata após o primeiro uso e ao gerar novo pedido.
- **Critério ✅:** Tokens criptográficos com expiração curta e uso único garantido no servidor.

### 18/27 — Restrinja os tipos e tamanhos dos arquivos enviados
- **Princípio:** Rejeitar executáveis ou arquivos gigantes validando tipo, tamanho e conteúdo real no servidor.
- **O que verificar:**
  - Allowlist explícita de extensões e MIME types permitidos (ex.: apenas imagens `.jpg`, `.png`, `.webp`).
  - Limite máximo de tamanho aplicado no servidor (ex.: máx. 10 MB).
  - Verificação do conteúdo real / magic bytes do arquivo (não confiar apenas na extensão ou `Content-Type` do cliente), sanitização do nome e armazenamento seguro fora de diretórios executáveis.
- **Critério ✅:** Validação server-side de extensão/MIME, magic bytes/conteúdo e tamanho máximo em todos os uploads (ou `➖ N/A` se o projeto não possui upload).

### 19/27 — Verifique preços e pagamentos
- **Princípio:** Nunca aceitar preço enviado pelo navegador nem considerar "tela de obrigado" como prova de pagamento.
- **O que verificar:**
  - Checkout busca preços, descontos e totais no catálogo/banco do servidor, ignorando ou rejeitando valores manipulados pelo cliente.
  - Confirmação de pedido pago ocorre somente via **webhook assinado** (verificação criptográfica da assinatura do provedor, ex.: Stripe/Mercado Pago/Pagar.me) ou consulta direta server-to-server na API do provedor.
- **Critério ✅:** Preços autoritativos no servidor e confirmação exclusivamente por webhook assinado / verificação server-side (ou `➖ N/A` se não há cobrança/pagamentos).

### 20/27 — Desative o modo de depuração em produção
- **Princípio:** Nunca vazar stack traces, arquivos internos, consultas SQL, variáveis ou caminhos de servidor para o usuário final.
- **O que verificar:**
  - Flags `DEBUG=True`, `NODE_ENV !== 'production'`, exposição de `err.stack` / `error.message` / SQL errors em respostas HTTP.
  - Tratamento global de erros retornando mensagem genérica + código/ID de referência para o usuário, enviando detalhes técnicos apenas para logs internos protegidos.
- **Critério ✅:** Modo debug desabilitado em produção e manipulador de erro global sem vazamento de stack traces ou dados internos.

### 21/27 — Atualize dependências vulneráveis
- **Princípio:** Corrigir bibliotecas com CVEs conhecidos e validar compatibilidade.
- **O que verificar:**
  - Executar auditoria de dependências da stack (`npm audit`, `pnpm audit`, `yarn audit`, `pip-audit` / `safety`, `cargo audit`, `composer audit`, `govulncheck`, etc.).
  - Identificar pacotes com vulnerabilidades conhecidas (críticas/altas/médias) e checar versões corrigidas disponíveis.
- **Critério ✅:** Sem vulnerabilidades conhecidas críticas/altas nas dependências (ou plano documentado quando correção exige breaking change).

### 22/27 — Remova pacotes que não utiliza
- **Princípio:** Menos dependências = menor superfície de ataque e menor risco de supply-chain.
- **O que verificar:**
  - Identificar pacotes declarados no manifesto (`package.json`, `requirements.txt`, etc.) sem nenhuma importação ou uso em código/scripts/configurações (ex.: `depcheck`, busca de imports no projeto).
- **Critério ✅:** Nenhuma dependência órfã ou não utilizada no projeto, mantendo build e testes funcionando.

### 23/27 — Verifique se há arquivos expostos
- **Princípio:** Arquivos sensíveis nunca podem ser servidos publicamente pela web (devem retornar 404 / serem bloqueados ou removidos).
- **O que verificar:**
  - Inspecionar pastas públicas (`public/`, `static/`, `dist/`, `build/`, raiz web) e regras de servidor/roteamento procurando `.env`, `.env.backup`, `*.sql`, `db-dump.sql`, `.git/`, backups `.zip`/`.bak`, logs, source maps em produção ou endpoints de debug/status expostos.
- **Critério ✅:** Nenhum arquivo sensível em diretórios públicos e bloqueio explícito para `.env*`, `.git/*`, dumps e backups.

### 24/27 — Procure segredos vazados no histórico do Git
- **Princípio:** Mesmo que o arquivo atual (`HEAD`) esteja limpo, commits antigos podem conter chaves, tokens ou senhas.
- **O que verificar:**
  - Verificar se é um repositório Git e inspecionar histórico (`git log -p`, buscas por `sk_`, `pk_live`, `AIza`, `ghp_`, `xoxb-`, `BEGIN PRIVATE KEY`, `API_KEY`, `SECRET`, `.env` comitados anteriormente, ou ferramentas como `gitleaks`/`trufflehog` se instaladas).
- **Critério ✅:** Histórico do Git verificado; zero segredos encontrados ou lista exata de commits/chaves comprometidas documentada para ação imediata.

### 25/27 — Revogue e substitua chaves comprometidas
- **Princípio:** Chave que vazou (no código cliente, logs ou histórico do Git) deve ser tratada como comprometida: revogar a antiga (passar a dar `401`) e gerar nova no cofre de segredos do servidor.
- **O que verificar:**
  - Cruzar achados dos itens `02`, `11`, `23` e `24`: qualquer chave exposta deve ser listada com urgência.
  - Na Fase 2, garantir que a nova chave seja referenciada apenas no servidor/variáveis privadas e orientar o usuário exatamente quais chaves revogar nos respectivos painéis externos.
- **Critério ✅:** Nenhuma chave comprometida ativa no projeto; todas as ocorrências identificadas têm instrução clara de revogação e substituição segura no servidor.

### 26/27 — Faça uma revisão completa de segurança
- **Princípio:** Revisão transversal após alterações individuais cobrindo os 8 domínios: **Acesso, Segredos, Dados, Rede, Pagamentos, Arquivos, Entradas e Pacotes**.
- **O que verificar:**
  - Conferir se não existem inconsistências entre camadas (ex.: rota protegida no frontend mas aberta na API; webhook sem verificação; CORS aberto em rota específica).
- **Critério ✅:** Revisão final dos 8 domínios concluída sem brechas cruzadas.

### 27/27 — Teste as proteções e entregue um relatório das falhas e das correções
- **Princípio:** Validar na prática que cada proteção funciona e documentar claramente o resultado.
- **O que verificar:**
  - Na Fase 1: verificar se há testes de segurança automatizados no projeto.
  - Na Fase 2: criar/executar testes de proteção (ex.: usuário comum recebe `403` em `/admin`, login repetido dispara `429`, formulário inválido é rejeitado pelo servidor, webhook sem assinatura válida é rejeitado, build e suíte de testes passam) e entregar o **Relatório de Segurança**.
- **Critério ✅:** Proteções testadas após a última edição e relatório completo entregue com falhas, correções retestadas e pendências.
