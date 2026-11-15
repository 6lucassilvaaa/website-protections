---
name: website-protections
description: >-
  Audita e fortalece a segurança de sites, apps web, APIs e programas com base nos
  27 tópicos essenciais de proteção (autenticação, segredos, banco de dados/RLS,
  HTTPS, hash Argon2id, validação no servidor, XSS, logs/monitoramento, RBAC/admin,
  consultas parametrizadas, variáveis de ambiente, rate limiting, CORS, cabeçalhos de
  segurança, cookies seguros, CSRF, expiração de links de redefinição, upload de
  arquivos, preços/pagamentos, debug em produção, dependências vulneráveis, pacotes
  sem uso, arquivos expostos, histórico do Git, revogação de chaves, revisão completa
  e testes/relatório). Use sempre que o usuário mencionar "website protections",
  pedir para melhorar a segurança do site/app/programa, auditar vulnerabilidades ou
  proteger uma aplicação antes ou depois de publicar.
---

# Website Protections — Auditoria e Implementação de Segurança (27 Tópicos)

Esta skill audita e implementa proteções de segurança em sites, aplicativos web, APIs e programas em qualquer stack (Next.js, React/Vite, Node/Express, Python/FastAPI/Django, PHP, Go, etc.), seguindo integralmente os **27 tópicos de boas práticas de segurança**.

> **Compatibilidade universal:** Esta skill usa apenas instruções portáveis (leitura de código, busca de padrões e comandos padrão de terminal/Git), funcionando de forma idêntica em qualquer agente de código.

---

## Fluxo Obrigatório de Execução (2 Fases)

Salvo se o usuário pedir explicitamente para *"implementar tudo direto sem perguntar"*, execute **SEMPRE** em duas fases separadas:

### FASE 1 — Auditoria e Diagnóstico (Somente Leitura)

**NÃO modifique nenhum arquivo na Fase 1.**

1. **Carregue a referência dos 27 tópicos:**
   Leia [references/checklist-27-topicos.md](./references/checklist-27-topicos.md) por completo antes de inspecionar o projeto.
2. **Mapeie o projeto atual:**
   - Identifique a stack (linguagem, framework frontend/backend, banco/ORM, provedor de auth, gateway de pagamento, sistema de upload, deploy).
   - Localize arquivos-chave: rotas/endpoints/controllers, middlewares, schemas/modelos de banco, `.env*`, `.gitignore`, `package.json` / `requirements.txt` / `pyproject.toml`, configurações de servidor/headers, pastas públicas (`public/`, `static/`), e histórico Git.
3. **Verifique TODOS os 27 tópicos no projeto:**
   Para cada tópico de `01` a `27`, verifique evidências reais no código e classifique:
   - `✅ Protegido` — Já implementado adequadamente (cite arquivo/trecho observado).
   - `⚠️ Parcial` — Existe algo, mas incompleto ou com brecha (explique exatamente o que falta).
   - `❌ Faltando` — Ausente ou vulnerável (indique onde o risco aparece).
   - `➖ N/A` — Não aplicável ao projeto (explique o motivo; ex.: sem pagamentos ou sem upload de arquivos).
4. **Apresente o Diagnóstico ao usuário:**
   - Mostre primeiro um **Resumo Executivo** (quantos itens `✅`, `⚠️`, `❌`, `➖`).
   - Liste os **27 tópicos** organizados com status claro, evidência encontrada e o que precisa ser feito nos itens `⚠️` / `❌`.
   - Destaque **alertas urgentes** se houver (ex.: chaves de API no frontend, `.env` exposto ou segredos no histórico do Git que exigem revogação imediata).
5. **PARE E PERGUNTE antes de editar:**
   Finalize a Fase 1 com uma pergunta objetiva pedindo autorização para prosseguir:
   > *"Deseja que eu prossiga com a implementação das proteções faltantes/parciais? Você quer aplicar todas as correções recomendadas ou prefere escolher itens específicos?"*

---

### FASE 2 — Implementação, Revisão e Relatório Final

Execute **somente após a confirmação do usuário**:

1. **Consulte o guia de implementação:**
   Leia [references/guia-implementacao.md](./references/guia-implementacao.md) para aplicar padrões seguros e compatíveis com a stack do projeto.
2. **Implemente em ordem de prioridade:**
   - **Crítico:** Segredos expostos (mover para servidor, limpar prefixos públicos), autenticação e controle de acesso (`admin`/IDOR/RLS), consultas parametrizadas, validação de pagamentos no servidor.
   - **Alto:** Hash de senhas (`Argon2id`), validação de formulários no servidor, XSS, CSRF, cookies seguros (`HttpOnly`, `Secure`, `SameSite`), expiração de links de reset, uploads seguros, arquivos expostos, debug desligado em produção.
   - **Médio / Endurecimento:** Rate limiting (`429`), CORS restrito, cabeçalhos de segurança (`CSP`, `HSTS`, `X-Frame-Options`), HTTPS obrigatório, logs e monitoramento de atividades suspeitas, atualização de dependências vulneráveis e remoção de pacotes sem uso.
3. **Execute o Tópico 26 — Revisão completa de segurança:**
   Revise os 8 pilares integrados (**Acesso, Segredos, Dados, Rede, Pagamentos, Arquivos, Entradas e Pacotes**) garantindo que nenhuma proteção quebrou fluxos legítimos nem deixou brechas entre módulos (ex.: webhook sem verificação de assinatura).
4. **Execute o Tópico 27 — Teste as proteções e entregue o Relatório Final:**
   - Rode testes automatizados, build/typecheck e verificações práticas para confirmar que as proteções funcionam (ex.: acesso a `/admin` sem perfil retorna `403`, rate limit retorna `429`, variáveis privadas não aparecem no bundle do cliente, pacotes removidos não quebram o build).
   - Entregue o **Relatório de Segurança** seguindo o formato do guia, separando claramente:
     - **Falhas encontradas**
     - **Corrigidas e retestadas** (somente o que foi realmente testado após a edição final)
     - **Já estavam protegidas**
     - **Pendentes / Ação manual necessária** (ex.: revogar chaves comprometidas no painel do provedor, forçar HTTPS no provedor de DNS/hospedagem, limpar histórico Git caso o usuário deseje reescrever commits).
