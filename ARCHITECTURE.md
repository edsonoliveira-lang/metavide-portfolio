# METAVIDE — Arquitetura do Sistema

> Plataforma de leitura bíblica e crescimento espiritual, estruturada em torno do
> **Ciclo da Videira**: 22 cultivos diários, cada um com 7 metas, medindo progresso
> de leitura, compreensão via quiz e evolução espiritual através do **IQE (Índice
> de Qualidade Espiritual)**.

---

## 1. Visão geral

```
┌─────────────┐      HTTPS       ┌──────────────────┐      JDBC       ┌────────────┐
│   Frontend   │ ───────────────▶ │   Backend (API)   │ ───────────────▶ │ PostgreSQL │
│ React + Vite │ ◀─────────────── │ Spring Boot (Java)│ ◀─────────────── │            │
└─────────────┘   cookie httpOnly └──────────────────┘                  └────────────┘
                                          │
                                          │ chamada externa (com fallback)
                                          ▼
                                  ┌──────────────────┐
                                  │   Gemini API      │
                                  │ (gemini-3.5-flash) │
                                  └──────────────────┘
```

- **Frontend**: React + Vite, consumindo a API via cliente HTTP (ver débito técnico
  em [§7](#7-débitos-técnicos-conhecidos) sobre duplicidade de clientes).
- **Backend**: Java Spring Boot, expõe API REST autenticada.
- **Banco de dados**: PostgreSQL, schema gerenciado por **Flyway** (migrations
  versionadas — reconstruído após período em que o schema era gerenciado apenas
  pelo Hibernate).
- **IA generativa**: Google Gemini API (`gemini-3.5-flash-lite`), usada em dois
  pontos do sistema, sempre com **fallback determinístico** (ver [§4](#4-arquitetura-de-ia-llm-com-fallback)).
- **Deploy alvo**: Railway (backend + PostgreSQL) e Vercel (frontend).

---

## 2. Autenticação e segurança

| Item | Implementação |
|---|---|
| Sessão | Cookie `httpOnly` + `Secure` + `SameSite`, substituindo token em `localStorage` |
| Proteção contra CSRF | Habilitada em conjunto com o cookie de sessão |
| Autorização | Spring Security, papéis `ALUNO` / `ADMIN` |
| Filtro de autenticação | `JwtFilter`, aplicado a todos os controllers |
| Troca de senha | Endpoint dedicado + tela no frontend |
| Acesso pago | Bloqueio HTTP 402 até confirmação manual de pagamento Pix por um admin |

### Correção relevante
Cinco controllers foram identificados driblando o `JwtFilter` e lendo o header
`Authorization` diretamente — corrigidos para passar pelo filtro central. Esse é
um padrão recorrente a observar em novos controllers (checklist antes de merge).

### Bug crítico corrigido
Falha de login silencioso: senha incorreta não informava o erro ao usuário porque
o `authStore.js` no frontend não validava `res.data.success` antes de setar o
estado de sessão como autenticado.

---

## 3. IQE — Índice de Qualidade Espiritual

Métrica central de acompanhamento do aluno, calculada a partir de:

- **Frequência** de acesso e uso
- **Compreensão** real, aferida por quiz (não estimada)
- **Retenção** ao longo do ciclo
- **Constância** de progresso
- **Reflexão**, registrada via "frutos do espírito"

**Gatilho**: recálculo automático disparado a cada conclusão de meta ou cultivo,
implementado em `CultivoService` e `QuizService`.

**Correção histórica**: a fórmula tinha um bug de duplicação, e havia inclusive um
valor de IQE inserido manualmente no banco antes de o serviço de cálculo existir
— resquício de desenvolvimento ad-hoc no início do projeto. Ambos corrigidos.

**Visibilidade**: breakdown completo (frequência, compreensão, retenção,
constância, reflexão) exposto na tela do aluno, e por aluno individualmente no
painel administrativo.

---

## 4. Arquitetura de IA (LLM com fallback)

> **Importante**: o sistema hoje utiliza **IA generativa via LLM**, não RAG
> (Retrieval-Augmented Generation). Não há busca semântica ou base vetorial —
> essa é uma evolução planejada, não implementada.

### Padrão de fallback

Toda chamada de IA generativa segue o mesmo princípio: **degradar de forma
graciosa em vez de falhar**. Se a chamada ao Gemini falhar, expirar ou não
retornar, o sistema gera uma versão baseada em **template determinístico**, sem
interromper a experiência do aluno.

Esse padrão está aplicado em dois pontos:

1. **Relatório de evolução em PDF** — gerado automaticamente ao concluir o 7º
   cultivo. Pipeline: dados do aluno → prompt ao Gemini → texto gerado →
   renderização via **Thymeleaf** → PDF via **openhtmltopdf**. Em caso de falha
   da IA, o PDF é gerado com narrativa padrão baseada nos dados brutos.
2. **Comentário interpretativo da Roda da Vida** — avaliação de satisfação em 12
   categorias (escala 1–10), com gráfico radar e comentário gerado por IA citando
   versículos da Bíblia King James Fiel. Mesmo mecanismo de fallback.

### Pontos de atenção para evolução futura
- **Idempotência**: validar se um mesmo evento (ex: concluir o 7º cultivo) pode
  disparar a geração duas vezes, gerando custo de API duplicado.
- **Observabilidade de custo**: ainda não há log/dashboard de quantas chamadas à
  API do Gemini saem por dia — relevante enquanto o projeto estiver na camada
  gratuita.
- **RAG (não implementado)**: evolução planejada é indexar o conteúdo dos 22
  cultivos como embeddings (pgvector no próprio PostgreSQL) para permitir busca
  semântica e respostas contextualizadas, em vez de depender só do conhecimento
  geral do LLM.

---

## 5. Módulo de Aulas (onboarding)

- 15 vídeos com **liberação sequencial** — só desbloqueia a próxima aula ao
  concluir a atual.
- **Aula 5 — Balanço da Colheita**: teste de leitura cronometrado, executado no
  servidor (não no cliente, para evitar manipulação), sobre o Livro de Tito (860
  palavras, 10 perguntas). Calcula PPM (palavras por minuto) e percentual de
  compreensão real via respostas do quiz.
- **Aula 6 — Roda da Vida**: avaliação de satisfação em 12 categorias, gráfico
  radar, comentário interpretativo por IA (ver [§4](#4-arquitetura-de-ia-llm-com-fallback)).
- Ao completar as 15 aulas, o aluno desbloqueia os 22 Cultivos (o produto
  principal).

---

## 6. Painel administrativo

- Dashboard com gráficos: funil de aulas, distribuição de progresso, engajamento
  geral da turma.
- Lista de alunos com telefone (WhatsApp), status de pagamento, aulas assistidas
  e badge de engajamento: `NUNCA_ACESSOU` / `PARADO` / `EM_DIA` / `CONCLUIU_AULAS`.
- Botão de contato direto via WhatsApp a partir da lista.
- Visão detalhada por aluno: grid dos 22 cultivos, breakdown completo do IQE,
  resultados dos testes.
- Admin tem acesso de prévia/teste aos cultivos sem as restrições do fluxo normal
  de aluno.

---

## 7. Débitos técnicos conhecidos

| Débito | Situação |
|---|---|
| Flyway desabilitado (schema só via Hibernate) | **Resolvido** — migrations reconstruídas |
| Casing de pacote (`com.metavide` vs `com.Metavide`) por case-insensitivity do WSL | Em aberto — exige atenção em novos arquivos |
| `core.fileMode=false` no Git (WSL), exigindo `git update-index --chmod=+x` manual para permissões de executável | Em aberto — processo manual |
| Clientes de API duplicados no frontend (`src/api/index.js` e `src/services/api.js`) | Em aberto |
| Credenciais em texto plano previamente commitadas em `application.yml`/`application.properties` | Refatorado para variáveis de ambiente — senha antiga deve ser considerada exposta e trocada |
| Ausência de RAG (busca semântica) | Planejado, não implementado |

---

## 8. Infraestrutura, testes e CI/CD

- **Testes**: Testcontainers para testes de integração contra PostgreSQL real.
- **CI**: GitHub Actions, rodando a cada Pull Request.
- **Deploy alvo**: Railway (backend + PostgreSQL gerenciado) + Vercel (frontend).
  Ainda pendente — bloqueador único para o lançamento piloto (grupo de ~50
  pessoas).

---

## 9. Pendências

- [ ] Deploy em produção (Railway + Vercel)
- [ ] Merge do PR de versículos da Roda da Vida
- [ ] Decisão de produto: público-alvo do Teste de Perfil do Leitor (livre,
      cadastrado ou pago)
- [ ] Implementação de RAG para os 22 cultivos
- [ ] Observabilidade de custo de chamadas à API do Gemini
- [ ] Validação de idempotência nos gatilhos de geração de IA

---

## 10. Stack resumida

| Camada | Tecnologia |
|---|---|
| Frontend | React, Vite |
| Backend | Java, Spring Boot, Spring Security |
| Banco de dados | PostgreSQL, Flyway, Hibernate |
| IA | Google Gemini API (`gemini-3.5-flash-lite`) |
| Geração de PDF | Thymeleaf + openhtmltopdf |
| Testes | Testcontainers |
| CI/CD | GitHub Actions |
| Deploy | Railway (backend/DB), Vercel (frontend) |
