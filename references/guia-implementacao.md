# Guia de Implementação e Templates — Website Protections

Use este guia na **Fase 2** (após confirmação do usuário) para implementar as correções de forma limpa e adequada à stack, e para padronizar os outputs das Fases 1 e 2.

---

## Princípios de Implementação por Camada

### 1. Autenticação, Senhas e Sessão (Tópicos 01, 05, 15, 16, 17)
- **Hash de senhas (05):**
  - Node.js: usar `argon2` (`argon2.argon2id`) ou `@node-rs/argon2`; se ambiente serverless puro sem binário nativo, usar Scrypt/Argon2 WASM ou `bcryptjs` com custo >= 12 (preferindo Argon2id sempre que suportado).
  - Python: usar `argon2-cffi` (`PasswordHasher()` usa Argon2id por padrão) ou `pwdlib[argon2]`.
  - Go: `golang.org/x/crypto/argon2` (`argon2.IDKey`).
- **Verificação de conta (01):** Checar `user.active && !user.locked` no servidor além da senha. Usar comparação em tempo constante para tokens/hashes.
- **Cookies de sessão (15):** Sempre definir `HttpOnly: true`, `Secure: process.env.NODE_ENV === 'production'`, `SameSite: 'lax'` (ou `'strict'` quando não depender de redirecionamento externo OAuth), `Path: '/'`, e `MaxAge`/`Expires` explícito. Nunca gravar token de sessão em `localStorage`.
- **Proteção CSRF (16):**
  - Para formulários/endpoints autenticados por cookie: validar token CSRF por sessão/double-submit cookie e verificar `Origin`/`Referer` contra a origem permitida em `POST/PUT/PATCH/DELETE`.
  - Se o framework já possui proteção nativa (ex.: Django `CsrfViewMiddleware`, Rails `protect_from_forgery`, Next.js Server Actions com validação de Origin), confirmar que está ativa e não foi desabilitada.
- **Links de redefinição de senha (17):**
  - Gerar token com CSPRNG (`crypto.randomBytes(32).toString('hex')` / `secrets.token_urlsafe(32)`).
  - Salvar apenas o **hash SHA-256** do token no banco junto com `expires_at = now + 15 min` e `used_at = null`.
  - No consumo: validar existência, `!used_at` e `expires_at > now` em transação atômica, marcando `used_at` imediatamente e invalidando sessões antigas do usuário.

### 2. Autorização, Banco de Dados e Rotas Admin (Tópicos 03, 09, 10)
- **Anti-IDOR / RLS (03):**
  - Toda query a recurso privado deve incluir filtro explícito pelo usuário autenticado (`WHERE id = $1 AND user_id = $2`).
  - Se usar Supabase/PostgreSQL com acesso direto, ativar `ALTER TABLE ... ENABLE ROW LEVEL SECURITY;` e criar políticas usando `auth.uid() = user_id`.
- **Rotas Admin (09):**
  - Criar guard/middleware reutilizável (`requireAdmin`) aplicado no roteador/handler do servidor que retorna `401` se não autenticado e `403` se `role !== 'admin'`.
- **Consultas parametrizadas (10):**
  - Substituir toda string concatenada/interpolada por parâmetros vinculados (`db.query('SELECT * FROM orders WHERE name = $1', [name])` ou query builders tipados como Prisma/Drizzle/SQLAlchemy).

### 3. Segredos, `.env`, Arquivos Expostos e Git (Tópicos 02, 11, 23, 24, 25)
- **Esconder chaves de API (02) e Variáveis de Ambiente (11):**
  - Mover chamadas com chaves privadas do frontend para rotas backend (`/api/...`).
  - Remover prefixos públicos (`NEXT_PUBLIC_`, `VITE_`, `PUBLIC_`, `REACT_APP_`, `EXPO_PUBLIC_`) de qualquer segredo.
  - Garantir `.env*` no `.gitignore` (mantendo `.env.example` apenas com nomes e placeholders) e adicionar validação de variáveis obrigatórias no boot do servidor.
- **Arquivos expostos (23):**
  - Remover arquivos sensíveis de `public/`/`static/`.
  - Em configs de servidor (Nginx, Apache, Vercel, Express static), bloquear acesso a dotfiles (`/.env*`, `/.git*`), dumps (`.sql`, `.bak`, `.old`) e desabilitar listagem de diretórios.
- **Histórico do Git e Revogação (24, 25):**
  - Se encontrar segredos no histórico, **não reescreva o histórico do Git sem pedir autorização explícita ao usuário** (pois altera hashes de commit). Em vez disso, alerte o usuário com o hash do commit e arquivo exato, marque a chave como **comprometida** e instrua a **revogação imediata** no provedor + inserção da nova chave no `.env` / cofre de segredos do servidor.

### 4. Validação de Entrada, XSS, Uploads e Pagamentos (Tópicos 06, 07, 18, 19)
- **Validação no servidor (06):** Validar todos os inputs com biblioteca de schema (Zod, Valibot, Pydantic, etc.) verificando tipos, limites numéricos (ex.: `min(1).max(10)`), tamanho máximo de strings e formatos antes de tocar no banco.
- **XSS (07):** Remover `innerHTML`/`dangerouslySetInnerHTML` sempre que possível. Quando HTML rico for inevitável, sanitizar com `DOMPurify` (isomórfico/server-side) e validar esquemas de URL permitindo apenas `http:`, `https:` e `mailto:`.
- **Uploads (18):**
  - Validar extensão + MIME type na allowlist.
  - Validar **magic bytes** (ex.: `file-type` no Node ou `python-magic`/assinatura de bytes) para confirmar o conteúdo real.
  - Aplicar limite de tamanho no parser/middleware (ex.: 10 MB) e renomear o arquivo com UUID no storage.
- **Preços e Pagamentos (19):**
  - Receber do cliente apenas `productId` e `quantity` (validada `>= 1`). Calcular preço unitário, frete e total **exclusivamente no servidor** consultando o catálogo no banco.
  - Verificar assinatura criptográfica do webhook do gateway (`stripe.webhooks.constructEvent`, HMAC SHA-256 do provedor, etc.) usando o raw body antes de marcar pedido como pago.

### 5. Rede, Cabeçalhos, Rate Limit, Debug e Logs (Tópicos 04, 08, 12, 13, 14, 20)
- **HTTPS (04):** Configurar redirecionamento para HTTPS em proxies/middlewares de produção e forçar `https://` em callbacks/URLs base.
- **Rate Limiting (12):** Aplicar limitador por IP/usuário em `/login`, `/register`, `/forgot-password`, `/api/*` retornando HTTP `429` com header `Retry-After`.
- **CORS (13):** Configurar `origin` com array explícito vindo de variável de ambiente (ex.: `ALLOWED_ORIGINS`), `credentials: true` somente quando necessário, nunca `*` combinado com credenciais.
- **Cabeçalhos de segurança (14):** Configurar `Content-Security-Policy`, `Strict-Transport-Security: max-age=31536000; includeSubDomains`, `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`.
- **Debug em produção (20) e Monitoramento (08):**
  - Garantir handler global de erro que gera um `errorId` único, loga detalhes completos internamente (redigindo senhas/tokens/chaves) e devolve ao cliente apenas `{ error: "Algo deu errado", reference: errorId }`.
  - Registrar logs de eventos de segurança (falhas consecutivas de login, acessos 403 em `/admin`, rejeições de webhook/CSRF) com alerta ou hook configurável.

### 6. Dependências e Pacotes Sem Uso (Tópicos 21, 22)
- Atualizar pacotes vulneráveis usando o gerenciador do projeto (`npm update` / `npm audit fix` sem `--force` destrutivo às cegas, ou atualização direcionada em `pyproject.toml`/`requirements.txt`).
- Confirmar 0 referências antes de remover pacotes sem uso, e rodar build + testes logo após remover.

---

## Template de Resposta — Fase 1 (Diagnóstico)

```markdown
# 🛡️ Auditoria de Segurança — Website Protections (27 Tópicos)

**Projeto analisado:** `<nome/caminho>` | **Stack detectada:** `<stack>`

## Resumo Executivo
- ✅ **Protegido:** X/27
- ⚠️ **Parcial:** Y/27
- ❌ **Faltando / Vulnerável:** Z/27
- ➖ **Não aplicável:** W/27

> 🚨 **Alertas Urgentes (se houver):** <chaves expostas, histórico git comprometido, falhas críticas>

## Diagnóstico Completo (01 a 27)
| # | Tópico | Status | O que foi encontrado | Ação recomendada |
|---|---|---|---|---|
| 01 | Autenticação adequada | ... | ... | ... |
...
| 27 | Testes e relatório | ... | ... | ... |

---
**Posso prosseguir com a implementação das correções faltantes/parciais?**
Você gostaria que eu aplique **todas as melhorias recomendadas** ou prefere priorizar/selecionar itens específicos?
```

---

## Template de Resposta — Fase 2 (Relatório Final — Tópico 27)

```markdown
# 📋 Relatório Final de Segurança — Website Protections

## Resumo dos Resultados
- **Falhas encontradas na auditoria:** X
- **Corrigidas e retestadas:** Y
- **Já protegidas inicialmente:** Z
- **Pendentes (ação externa/manual):** W

## Detalhamento das Correções e Testes
| # | Tópico | Status Final | O que foi implementado / Evidência do teste |
|---|---|---|---|
| ... | ... | ✅ Corrigido • Retestado | ... |

## Testes Executados
- Listar exatamente os comandos/testes executados após a última edição e seus resultados observados.

## Pendências Externas / Ações Manuais (se houver)
- Ex.: Revogar chave `sk_...` no painel do provedor e configurar nova chave no ambiente de produção; habilitar HTTPS/TLS no provedor de hospedagem.
```
