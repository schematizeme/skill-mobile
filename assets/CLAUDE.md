<!-- cross-skill: iam.md -> schematize-engineering -->
# CLAUDE.md — Engenharia de Apps Mobile da Casa (sempre on)

> Copie para a **raiz do repositório** do app e ajuste `<project>`. Fica pinado no contexto de
> toda tarefa e garante o piso mesmo quando a skill `schematize-mobile` não dispara sozinha. Em
> repo multi-skill (ex.: monorepo com backend), use **junto** com os `CLAUDE.md` das skills de
> engenharia e da linguagem do backend (rode `/mobile-claude` que mescla, sem sobrescrever os
> outros blocos).

## Regra mestre

App mobile é software da casa como qualquer outro: **o piso — segurança, IAM, testes, ops, DoD
(§35), archive (§28), índice (§39) — é o MESMO**. O mobile só muda o **como**, sob duas verdades:
o dispositivo é **cliente hostil** e a rede é **offline por padrão**. Em conflito entre "é só um
app, relaxa" e este piso, **o piso vence**. Consulte o reference antes de agir — não trabalhe de
memória.

## Pisos inegociáveis (VETADO — sem exceção)

1. **NUNCA segredo no bundle.** API key privada, `client_secret`, chave de assinatura, credencial
   — nada no `.ipa`/`.apk`, código, plist/`strings.xml`, `BuildConfig` ou var "pública". O app é
   **public client**; segredo que precisa existir fica no **servidor** atrás de um BFF. Achou
   segredo no rastro → **rotação** + remoção.
2. **Auth delegada ao IdP da casa, no servidor.** O app **não** implementa login: **OIDC/OAuth 2.1
   + PKCE** contra `auth.<domain>`, retorno por **Universal/App Links verificados**, login no
   **navegador do sistema (não WebView)**. **Authz sempre no servidor** (deny-default, token fino,
   ReBAC) — esconder botão é UX, não autorização.
3. **Secure storage e zero vazamento.** Refresh/chaves em **Keychain/Keystore** (Secure Enclave/
   StrongBox); **nunca** em store em claro; **nunca** token/PII em log, crash report ou analytics.
   Dado sensível e a outbox de sync **cifrados em repouso**.
4. **Passkeys+biometria no núcleo — biometria ≠ autorização.** Passkeys de plataforma são o fator
   forte; **biometria desbloqueia local** a chave, **não** substitui **step-up server-side** pra
   op sensível. Fallback via Email OTP always-on do IdP. **Logout irreversível** (revoga refresh+
   família no IdP, desassocia push).
5. **Offline-first sem perder escrita.** A UI **lê do local**; escrita é otimista + **outbox
   durável** com **idempotência**; sync é **delta** com tombstones; **conflito é resolvido por
   política explícita** (ADR) e o **servidor é a fonte de verdade**. Descartar escrita do usuário
   em silêncio é vetado.
6. **Release com rollout gradual + gate.** Build **assinado no CI** (chaves em cofre),
   **staged/phased rollout** com gate de **crash-free/ANR** e **halt** na regressão (rollback é
   hotfix pra frente). OTA só de JS/config, **dentro das regras da loja**. Crash/ANR com símbolos,
   telemetria **sem PII**.
7. **Device é território hostil — defesa em profundidade, verdade no servidor.** TLS + **SPKI
   pinning** (com backup/rotação); root/jailbreak e **attestation** como **sinal** pro risk engine
   (não bloqueio ingênuo); **ofuscação sensata** eleva custo mas **não** guarda segredo; deep link/
   IPC são entrada hostil (valide). A decisão de segurança nunca migra pro cliente.
8. **Efeito externo NUNCA sai de não-produção — e o build já está distribuído.** Build de
   **dev/QA/TestFlight/internal track/App Distribution NUNCA aponta pro backend de produção** nem
   pro provedor real: base URL, `client_id` e chaves são **por build configuration / flavor /
   scheme**, resolvidas **no build** e **fail-closed** (config ausente = **não-prd**); o **artefato
   assinado** é inspecionado no CI. **Push em SANDBOX** (APNs `aps-environment: development` +
   **projeto FCM de teste**; plist/`google-services.json` de **produção** em build de QA/TestFlight
   é **VETADO** — em TestFlight o isolamento é do **remetente**). **Conta de teste/review/persona/
   fixture só em domínio de ROTA NULA** (`test.<domain>` com null MX + SPF `-all` + DMARC
   `p=reject`, ou `.test`/`.invalid`/`.example`) — **VETADO** `@gmail.com`, e-mail do testador/da
   equipe e o domínio de prd. **E2E de Email OTP lê o código na API do SINK** (Mailpit), nunca em
   caixa real. **Cap por execução** (contando a matriz inteira do device farm) + **guard
   deny-by-default DENTRO do provider** valem inteiros. **Por quê:** build de teste que vaza pra
   prd dispara efeito externo **real a partir de milhares de devices** — e não existe rollback de
   binário instalado; bounce/complaint em massa queima IP/domínio e derruba o **OTP de login** de
   produção. Normativa: `schematize-engineering` → `references/efeitos-externos.md`; recorte em
   `entrega-lojas.md` §7 e `iam-mobile.md` §8.
9. **Orquestrador não desenvolve; subagent barato executa** (`schematize-engineering` → `references/orquestracao.md` §9). O agent principal (o que fala com o humano, modelo padrão da sessão) **só planeja, decompõe, despacha, supervisiona e revisa** — não escreve código de entrega. Toda ação onerosa é quebrada em **micro-tasks/micro-funções** executáveis por agent barato. Subagents rodam em **`sonnet` por padrão**; falhou → o **mesmo subagent corrige** (até 2 rodadas) → re-decompõe → só então **`opus`**, com motivo registrado no checkpoint. O principal revisa toda entrega (diff + gate) e **só corrige com a própria mão se necessário**. No **overdev**, cada item do checklist é executado por subagent `sonnet` e revisado pelo principal antes do `- [x]`.

## Definition of Done

Nada é "pronto" sem: unit + UI/instrumentado verdes, fluxo crítico com e2e, **nenhum efeito externo
real fora de produção** (build de dev/QA/TestFlight nunca aponta pro backend nem pro provedor de
`prd`; push em sandbox APNs / projeto FCM de teste; conta de teste/review em domínio de **rota
nula**; Email OTP lido do **sink** — gate em `scripts/check-external-effects.sh`), crash-free acima
do gate de rollout, sem segredo no bundle, **índice atualizado**, **archive commitado**, CI verde e
review aprovado. A DoD (§35) é a da base, `schematize-engineering` → `references/entrega.md`; o
recorte mobile está em `references/entrega-lojas.md`.

## Como se decide aqui

- **Plataforma (`/mobile-load`, `plataforma.md`):** nativo (Swift/Kotlin) vs cross (KMP/Flutter/RN)
  por **fit + ADR (§27)** — como a política de linguagem da casa. Registre o porquê.
- **IAM mobile (`/mobile-auth`, `iam-mobile.md`):** casa com o `iam.md`; audita OIDC/PKCE, deep
  link verificado, passkeys+biometria, secure storage, logout irreversível, authz no servidor.
- **Offline/sync (`/mobile-offline`, `offline-sync.md`):** outbox, idempotência, delta sync,
  conflito explícito; roteiro de QA 100% offline.
- **Release (`/mobile-release`, `entrega-lojas.md`):** assinatura, build no CI, staged rollout+gate,
  OTA nas regras, review, crash com símbolos, **build de não-prd que não fala com produção** (§7).

## Relação com as outras skills

- **schematize-engineering** — a base: IAM (`iam.md`), DoD (§35), archive (§28), índice (§39).
- **backend do rol** (go/rust/elixir/csharp/zig/ruby) — o serviço que o app consome (authz/regra/
  segredo moram lá).
- **schematize-web** — mesmo IdP/IAM; "segredo nunca no cliente" dos dois lados.
- **schematize-pentest** — o app é superfície de ataque (BOLA/BFLA, bypass de step-up, deep link).

## Gestão de contexto (sessões longas)

Ao se aproximar do teto de contexto: **PARE e** gere o handoff em `<project>_archive/context/`
(estado + FEITO vs EM ABERTO) **antes** de compactar (`/mobile-cc`).
