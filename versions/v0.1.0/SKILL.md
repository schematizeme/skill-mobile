---
name: schematize-mobile
metadata:
  version: 0.1.0
description: Engenharia de apps mobile da casa — o mesmo piso (segurança, IAM, testes, ops, DoD, archive) da schematize-engineering, no cliente hostil que é o dispositivo. Cobre a escolha de plataforma (nativo iOS Swift / Android Kotlin vs cross KMP/Flutter/RN — por fit + ADR, como a política de linguagem da casa); offline-first & sync (UI lê do local, outbox durável de mutações, idempotência, delta sync, tombstones, resolução de conflito explícita — LWW/merge/CRDT/servidor-autoritativo, nenhuma escrita some em silêncio); IAM mobile CASANDO com o iam.md (app é public client OIDC/PKCE delegando ao auth.<domain>, NUNCA login próprio nem client_secret no bundle; passkeys de plataforma + biometria como núcleo, biometria desbloqueia LOCAL e não substitui step-up server-side; refresh/chaves em Keychain/Keystore/Secure Enclave, nunca em store em claro nem em log/crash; retorno por Universal/App Links verificados, login no navegador do sistema e não WebView; logout irreversível revoga refresh+família e desassocia push; authz sempre no servidor, token fino, ReBAC multi-tenant); push notifications (APNs/FCM, token atado à sessão, payload é gatilho não segredo, permissão just-in-time); performance/bateria/rede (cold start, main thread livre, WorkManager/BGTaskScheduler, delta/coalescing, medir não adivinhar); entrega nas lojas (assinatura em cofre, build assinado no CI, staged/phased rollout com gate de crash-free e halt, OTA de JS/config dentro das regras, review guidelines, kill-switch); observabilidade/crash (Crashlytics/Sentry com símbolos, telemetria sem PII); segurança (cert/SPKI pinning com backup, root/jailbreak + attestation como SINAL pro risk engine, ofuscação sensata que eleva custo mas não guarda segredo, dado cifrado em repouso, deep link/IPC como entrada hostil). Enfatiza: o piso da casa é o MESMO — muda o "como". Use SEMPRE que for projetar, gerar, revisar ou refatorar app iOS/Android/cross, decidir plataforma, desenhar auth/offline/sync/push de mobile, publicar em loja, ou tratar performance/segurança de app — mesmo sem citar "padrão". Pareia com schematize-engineering (a BASE: IAM/DoD §35/archive §28/índice §39), com o backend do rol (go/rust/elixir/c#/zig/ruby) que o app consome, e com schematize-pentest (o app é território hostil).
---

# Engenharia de apps mobile da casa (schematize-mobile)

Disciplina normativa para **apps mobile** — iOS, Android e cross-platform. A tese é uma só: o
**piso da casa não muda no mobile**. Segurança, IAM, testes, operação, Definition of Done (§35),
archive (§28), índice/MAPA (§39) são **os mesmos** de qualquer software da casa. O que o mobile
acrescenta é o **como**, sob duas verdades novas: o app **roda na máquina do adversário** (cliente
hostil) e **a rede falha o tempo todo** (offline por padrão). Esta skill especializa a
`schematize-engineering` (a base agnóstica) para esse recorte, sem afrouxar nenhum piso.

Um app não é uma ilha: é o **cliente** de uma arquitetura que já tem os pisos da casa — um backend
do **rol sancionado** (Go/Rust/Elixir/C#/Zig/Ruby) e um **IAM como app separada** em
`auth.<domain>` (o IdP da casa). O app **delega** authz e segredo ao servidor; ele é conveniência
de UX, nunca a fonte de verdade nem o guardião do segredo.

**Versão:** skill `schematize-mobile` v0.1.0. Changelog em `CHANGELOG.md`.

## Comandos (Claude Code)

Digite `/mobile-help` pra ver todos. Em resumo:

| Comando | O que faz |
|---|---|
| `/mobile-help` | lista todos os comandos do schematize-mobile |
| `/mobile-load` | carrega à força TODO o corpo normativo (plataforma, offline/sync, IAM mobile, segurança, entrega, performance) e passa a aplicá-lo |
| `/mobile-auth` | força/audita o IAM mobile casado com o `iam.md`: OIDC/PKCE delegando ao `auth.<domain>`, passkeys+biometria, secure storage, logout irreversível, sem segredo no bundle |
| `/mobile-offline` | audita/planeja o offline-first & sync: UI lê do local, outbox durável, idempotência, delta sync, resolução de conflito explícita; roteiro de QA 100% offline |
| `/mobile-release` | prepara/audita o release de loja: assinatura em cofre, build no CI, staged/phased rollout com gate de crash-free + halt, OTA dentro das regras, review guidelines, kill-switch |
| `/mobile-claude` | cria ou mescla o `CLAUDE.md` sempre-on de mobile na raiz do repo |
| `/mobile-cc` | context compact: gera handoff no archive e roda `/compact` |
| `/mobile-handoff` | gera o handoff (context.md + checklist.md) sem compactar |

Os comandos ficam em `assets/commands/` e são instalados em `.claude/commands/`.

## Como usar esta skill

1. **Decida a plataforma primeiro, com ADR** (`references/plataforma.md`): nativo (Swift/Kotlin)
   ou cross (KMP/Flutter/RN) por **fit + ADR (§27)**, como a política de linguagem da casa. A
   escolha é estrutural e vira registro, não gosto do dia.
2. **Desenhe offline-first e sync** (`references/offline-sync.md`): a UI lê do local, a escrita é
   otimista com **outbox durável**, e a reconciliação (conflito/fila/idempotência) é um problema
   de dados distribuídos com política **explícita** — nenhuma escrita do usuário some em silêncio.
3. **IAM mobile casando com o `iam.md`** (`references/iam-mobile.md`): o app é **public client
   OIDC/PKCE** delegando ao `auth.<domain>`; passkeys+biometria no núcleo; tokens em
   Keychain/Keystore; **nunca** segredo no bundle; logout irreversível; authz no servidor.
4. **Segurança do cliente hostil** (`references/seguranca-mobile.md`): TLS+pinning, root/jailbreak
   e attestation como **sinal**, ofuscação sensata — defesa em profundidade, com a verdade sempre
   no servidor.
5. **Entrega e operação** (`references/entrega-lojas.md`): assinatura em cofre, build no CI,
   **staged/phased rollout** com gate de crash-free, OTA dentro das regras, observabilidade/crash.
6. **Performance/bateria/rede/push** (`references/performance.md`): medir, não adivinhar.
7. **Não trabalhe de memória** — os pisos abaixo valem independentemente do reference carregado.

Mapa de references — leia o que casa com a tarefa:

| Tarefa | Reference |
|---|---|
| Escolher nativo vs cross por fit + ADR; arquitetura interna em camadas; app como cliente do backend/IdP da casa | `references/plataforma.md` |
| Offline-first, outbox de mutações, idempotência, delta sync, tombstones, resolução de conflito (LWW/merge/CRDT/servidor), testes de reconciliação | `references/offline-sync.md` |
| IAM mobile: OIDC/PKCE, deep link verificado, passkeys+biometria, secure storage, logout irreversível, authz no servidor — casado com o `iam.md` | `references/iam-mobile.md` |
| Segurança do cliente hostil: nunca segredo no bundle, cert/SPKI pinning, root/jailbreak+attestation, ofuscação, dado em repouso, deep link/IPC | `references/seguranca-mobile.md` |
| Entrega nas lojas: assinatura, build no CI, staged/phased rollout+halt, OTA/regras de loja, review, observabilidade/crash | `references/entrega-lojas.md` |
| Performance, bateria, rede, tamanho e push notifications; perfilar antes de otimizar | `references/performance.md` |

## Pisos inegociáveis (vetam o atalho)

Independente do reference, estes limites nunca são cruzados:

1. **O piso da casa é o MESMO — muda o "como".** Segurança, IAM, testes, ops, DoD (§35), archive
   (§28), índice (§39) valem inteiros. Mobile **não** é desculpa pra afrouxar nada; é o mesmo piso
   realizado num cliente hostil com rede ruim.
2. **NUNCA segredo no bundle.** API key privada, `client_secret`, chave de assinatura, credencial
   — nada no `.ipa`/`.apk`, código, plist/strings, `BuildConfig` ou var "pública". O app é **public
   client**; segredo que precisa existir fica no **servidor** atrás de um BFF. Extrair o bundle é
   trivial. (mesma regra do `NEXT_PUBLIC_` do `schematize-web`).
3. **Auth é delegada ao IdP da casa, no servidor.** O app **não** implementa login próprio: usa
   **OIDC/OAuth 2.1 + PKCE** contra `auth.<domain>`, retorno por **Universal/App Links verificados**
   e login no **navegador do sistema (não WebView)**. Authz **sempre no servidor** (deny-default,
   token fino, ReBAC) — esconder botão é UX, não autorização.
4. **Secure storage para token/chave — e nada de vazamento.** Refresh e chaves em
   **Keychain/Keystore** (Secure Enclave/StrongBox); **nunca** em store em claro, e **nunca** token/
   PII em log, crash report ou analytics. Dado sensível e a outbox **cifrados em repouso**.
5. **Passkeys+biometria no núcleo — biometria não é autorização.** Passkeys de plataforma são o
   fator forte; **biometria desbloqueia local** a chave, **não** substitui **step-up server-side**
   pra operação sensível. Fallback via Email OTP always-on do IdP (`iam.md`). **Logout irreversível**
   (revoga refresh+família no IdP, desassocia push).
6. **Offline-first: a UI lê do local e nenhuma escrita some.** Escrita é otimista + **outbox
   durável** com **idempotência**; sync é **delta** com tombstones; **resolução de conflito é
   explícita** (ADR) e o **servidor é a fonte de verdade**. Descartar escrita do usuário em silêncio
   é vetado.
7. **Release com rollout gradual + gate — nunca 100% de uma vez.** Build **assinado no CI** (chaves
   em cofre), **staged/phased rollout** com gate de **crash-free/ANR** e **halt** na regressão;
   rollback é hotfix pra frente. OTA só de JS/config, **dentro das regras da loja**. Crash/ANR com
   símbolos e telemetria **sem PII**.
8. **O dispositivo é território hostil — defesa em profundidade, verdade no servidor.** TLS+**SPKI
   pinning** (com backup/rotação), root/jailbreak e **attestation** como **sinal** pro risk engine
   (não bloqueio ingênuo), **ofuscação sensata** que eleva custo mas **não** guarda segredo. Deep
   link/IPC são entrada hostil — valide. A decisão de segurança nunca migra pro cliente.

## Relação com as outras skills

- **schematize-engineering** — a **BASE** agnóstica. Esta skill herda e não afrouxa: **IAM**
  (`iam.md`, o modelo que o `iam-mobile.md` realiza), **DoD (§35)**, **archive (§28)**, **índice/
  MAPA (§39)**, **ops (§ ops.md)**, **cadeia de suprimentos**, **observabilidade**, e o fluxo
  (scan/plan/refactor/overdev/auditoria). A escolha de plataforma espelha a **política de linguagem**
  (`linguagens.md`): rol sancionado + fit + ADR.
- **schematize-go / rust / elixir / c# / zig / ruby** — o **backend** que o app consome. O app fala
  com um serviço do rol sancionado; a authz, a regra de negócio e o segredo moram lá.
- **schematize-web** — o **IdP/BFF web** e o fronte de auth compartilham o mesmo IAM; PWA/web app é
  frontend governado por ela (vira "app de loja" só com requisito nativo real). O piso "segredo
  nunca no cliente" é o mesmo dos dois lados.
- **schematize-pentest** — o **oráculo do cliente hostil**: BOLA/BFLA por confiar no que a tela
  mostrou, bypass de step-up/biometria, abuso de deep link, token em store inseguro, authz no
  cliente. O app vira superfície de ataque testável.
- **schematize-audit** — fecha o loop: os checklists de mobile (IAM/offline/release) viram itens
  provados, não marcados na fé.
