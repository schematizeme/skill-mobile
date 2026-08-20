# Changelog — schematize-mobile

Formato: [Keep a Changelog](https://keepachangelog.com/pt-BR/). Versionamento semântico.

## [0.2.0] — 2026-08-20

Propaga pro recorte mobile o piso **"efeito externo NUNCA sai de não-produção"** — normativo em
`schematize-engineering` → `references/efeitos-externos.md`. No mobile o dano é maior: um build de
teste que aponta pra produção dispara efeito **real a partir de milhares de devices de testadores**,
e o app **já está distribuído** — não existe rollback de binário instalado.

### Adicionado
- **SKILL.md** — piso inegociável **9**: build de dev/QA/TestFlight/internal track/App Distribution
  **nunca** aponta pro backend de prd nem pro provedor real (ambiente por **build configuration /
  flavor / scheme**, resolvido no build, **fail-closed**); **push em sandbox** (APNs
  `aps-environment: development` + **projeto FCM de teste**); conta de teste/review em domínio de
  **ROTA NULA**; **E2E de Email OTP lido no sink**. Linha nova no mapa de references e menção na
  `description`.
- **references/entrega-lojas.md §7** (novo) — "build de teste NUNCA fala com produção": ambiente é
  build configuration e não `if` em runtime (**vetado** toggle de ambiente em runtime/config
  remota), **gate no CI sobre o artefato assinado** (URL de prd, plist/`google-services.json` de
  produção, `aps-environment` errado, chave de prd), **tabela trilho → backend/push/e-mail**, push
  em sandbox com o caso do **TestFlight** (perfil de distribuição não tem APNs sandbox → o
  isolamento passa pro **remetente**, e o backend de prd recusa token de build de não-prd), conta de
  teste/persona/fixture/**review de loja** em `test.<domain>` de rota nula, e o **multiplicador do
  device farm** (cap por execução conta a matriz inteira).
- **references/iam-mobile.md §8** (novo) — identidade: toda conta não-humana em domínio de rota nula
  (**vetado** `@gmail.com`, e-mail do testador/da equipe, domínio de prd); **Email OTP always-on
  exercido contra o SINK** (Mailpit, API HTTP) no E2E de cadastro/login; guard deny-by-default no
  provider do IdP + cap; SMS/voz com magic numbers; conta de review com **OTP pré-provisionado
  escopado + ADR** ou demo mode, nunca caixa real do time.
- **DoD/checklists** — 3 itens novos na DoD de release (`entrega-lojas.md`, agora §8) e 2 no
  checklist de IAM mobile, todos com prova exigida.
- **assets/CLAUDE.md** — piso sempre-on **8** com o mesmo veto, o "por quê" (reputação queimada
  derruba o **OTP de login** de produção) e o ponteiro pra normativa.

### Mudado
- `references/entrega-lojas.md`: a **DoD de release** passou de §7 para **§8** (o §7 agora é efeito
  externo); §5 (review) remete ao §7 pro endereço da conta de review.
- `references/performance.md` §4: registro de push token carrega `env`/`build_id` e o backend de prd
  **recusa** token de build de não-prd.
- `/mobile-release`: novo bloco de auditoria de efeito externo e referência da DoD atualizada (§8);
  build de não-prd apontando pra produção **não fecha** o release.

## [0.1.0] — 2026-08-15

Primeira versão da skill de **engenharia de apps mobile** da casa — o mesmo piso da
`schematize-engineering` (segurança, IAM, testes, ops, DoD §35, archive §28, índice §39) aplicado
ao recorte mobile (iOS/Android/cross), sob as duas verdades novas: o dispositivo é **cliente
hostil** e a rede é **offline por padrão**. Muda o "como", não o "o quê".

### Adicionado
- **SKILL.md** com 8 pisos inegociáveis (piso da casa é o mesmo; nunca segredo no bundle; auth
  delegada ao IdP no servidor; secure storage sem vazamento; passkeys+biometria no núcleo com
  biometria≠autorização e logout irreversível; offline-first com outbox e conflito explícito;
  release com rollout gradual + gate; device é território hostil com defesa em profundidade) +
  mapa de references + relação com engineering/backend/web/pentest/audit.
- **references/**:
  - `plataforma.md` — rol de plataformas sancionadas (Swift/Kotlin nativo, KMP/Flutter/RN cross),
    escolha por **fit + ADR** (espelha `linguagens.md`), arquitetura interna em camadas, app como
    cliente do backend/IdP da casa.
  - `offline-sync.md` — offline-first (UI lê do local), **outbox** durável de mutações,
    idempotência, delta sync, tombstones, **resolução de conflito explícita** (LWW/merge/CRDT/
    servidor-autoritativo), UX de estado de rede, testes de reconciliação.
  - `iam-mobile.md` — IAM mobile **casado com o `iam.md`**: public client OIDC/PKCE ao
    `auth.<domain>`, deep link verificado (Universal/App Links, não WebView), passkeys+biometria,
    secure storage (Keychain/Keystore), logout irreversível, authz no servidor, checklist DoD.
  - `seguranca-mobile.md` — cliente hostil: **nunca segredo no bundle**, TLS+SPKI pinning com
    backup, root/jailbreak+attestation como **sinal**, ofuscação sensata, dado cifrado em repouso,
    deep link/IPC como entrada hostil, SDKs de terceiro, permissões mínimas.
  - `entrega-lojas.md` — assinatura em cofre, build assinado no CI, trilhos de release,
    **staged/phased rollout** com gate de crash-free/ANR e **halt**, OTA de JS/config dentro das
    regras, review guidelines, observabilidade/crash sem PII, DoD de release.
  - `performance.md` — cold start/main thread, bateria (WorkManager/BGTaskScheduler), rede (delta/
    coalescing/cache), **push notifications** (APNs/FCM, token atado à sessão, payload é gatilho),
    tamanho do app, perfilar antes de otimizar.
- **assets/commands/**: `/mobile-help`, `/mobile-load`, `/mobile-auth`, `/mobile-offline`,
  `/mobile-release`, `/mobile-claude`, `/mobile-cc`, `/mobile-handoff`.
- **assets/CLAUDE.md** — regra sempre-on: o piso da casa é o mesmo; nunca segredo no bundle; auth
  delegada ao IdP; secure storage; offline-first sem perder escrita; rollout gradual com gate.
