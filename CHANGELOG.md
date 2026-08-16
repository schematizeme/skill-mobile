# Changelog — schematize-mobile

Formato: [Keep a Changelog](https://keepachangelog.com/pt-BR/). Versionamento semântico.

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
