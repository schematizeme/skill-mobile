# IAM no mobile — o mesmo piso da casa, no cliente hostil

> Este reference **não inventa** um IAM de mobile: ele **casa com o `iam.md`** da
> `schematize-engineering` (o piso agnóstico) e diz **como** aquele piso se realiza num app, onde
> o dispositivo é **território hostil**. Auth continua sendo uma **aplicação separada**
> (`auth.<domain>`, o IdP da casa); o app é só **mais um cliente OIDC**. A força continua sendo
> **≥2 fatores**, identidade ≠ email, ReBAC multi-tenant, enforcement **no servidor**. O que muda
> é o **como**: deep-link OIDC, secure storage, biometria, passkeys de plataforma.

## 1. O app delega ao IdP da casa — nunca implementa auth

- **VETADO** o app ter login próprio, guardar hash de senha, ou falar com o banco de usuários.
  Ele **redireciona** ao `auth.<domain>` por **OIDC/OAuth 2.1 + PKCE** e recebe tokens de volta,
  exatamente como o front web (`iam.md` §1). O IdP é o mesmo pra web, app e serviços.
- **Authorization Code + PKCE obrigatório** (nunca implicit flow, nunca ROPC/senha direta no app).
  O `code_verifier` é gerado e guardado no device só durante o fluxo.
- **Segredo de client NÃO existe no app.** App é **public client** (não confidencial): não há
  `client_secret` no bundle — extrair o bundle é trivial. Segurança vem do PKCE + redirect
  registrado, não de um segredo embutido (`seguranca-mobile.md`, §"nunca segredo no bundle").

## 2. O retorno do fluxo — deep link seguro

- **Redirect por deep link:** use **Universal Links (iOS) / App Links (Android)** verificados por
  domínio (`apple-app-site-association` / `assetlinks.json` no servidor) — não `custom scheme`
  (`meuapp://`) sozinho, que **qualquer app pode sequestrar**. O link verificado prova posse do
  domínio e evita interceptação do `code`.
- **Fluxo no navegador do sistema:** use **ASWebAuthenticationSession (iOS) / Custom Tabs
  (Android)**, nunca uma `WebView` embutida — WebView permite ao app ler as credenciais digitadas
  e quebra SSO/passkeys. WebView de login é **vetado**.
- **`state` + PKCE** validados no retorno (anti-CSRF/anti-injeção de code).

## 3. Passkeys/WebAuthn + biometria — o núcleo (não roadmap)

Casa com `iam.md` §3 ("passkey é núcleo, já é 2 fatores num"):

- **Passkeys de plataforma:** **Passkeys (iOS, AuthenticationServices) / Credential Manager +
  Passkeys (Android)** são o fator forte de primeira classe — phishing-resistant, sincronizados
  pelo keychain da plataforma. O app oferece passkey no cadastro/login e no step-up.
- **Biometria (Face ID/Touch ID/BiometricPrompt) protege a chave, não substitui o servidor.** A
  biometria **desbloqueia localmente** a passkey/refresh no secure storage; ela é *gate local*,
  não um "logou no servidor". Autorização sensível ainda exige **step-up de verdade no IdP**
  (AAL alto, `iam.md` §3).
- **Nada de "biometria == autenticado" ingênuo:** biometria falsa/ausente cai no fluxo de fallback
  do IdP (Email OTP always-on, `iam.md`), nunca num bypass. E biometria **local** nunca autoriza,
  sozinha, ação sensível cross-tenant/billing — isso é step-up server-side.

## 4. Onde moram os tokens — secure storage, sempre

- **Refresh token e chaves em Keychain (iOS) / Keystore + EncryptedSharedPreferences (Android)** —
  respaldados por Secure Enclave/StrongBox quando disponível. **Nunca** em `UserDefaults`/
  `SharedPreferences` em claro, arquivo, log, ou `AsyncStorage`/`localStorage` de WebView.
- **Access token curto** (ex.: 15 min) + **refresh rotativo com detecção de reuso** (`iam.md` §6);
  o app renova silenciosamente — sessão longa (7d/90d device confiável) sem re-login constante,
  igual ao piso da casa.
- **Token nunca em log/crash report/analytics.** Scrub obrigatório do pipeline de observabilidade
  (`entrega-lojas.md`) — segredo em crash dump é vazamento.
- **Cifra em repouso:** dado sensível e a outbox (`offline-sync.md`) cifrados; chave no
  Keychain/Keystore, não hardcoded.

## 5. Logout, sessão e multi-dispositivo

- **Logout irreversível (`iam.md` §6):** não basta apagar o token local — **revoga o refresh
  (e a família) no IdP**, apaga a sessão server-side, e **desassocia o push token** do device.
  Depois do logout, replay do token guardado não reativa nada.
- **O device é uma sessão de 1ª classe:** aparece na view de dispositivos do IdP (rótulo, IP/geo,
  último uso), revogável remotamente ("sair deste aparelho"). Roubo de device → o dono revoga do
  web.
- **Push token atado à sessão:** ao deslogar/trocar de usuário, o push token é rotacionado — senão
  notificações do usuário A chegam no device agora do usuário B.

## 6. Authz continua no servidor — o app não é PDP

- **Deny-by-default no servidor; o app não decide permissão.** Esconder um botão no app é **UX**,
  não autorização. Todo endpoint valida a permissão no PEP server-side (ReBAC, `iam.md` §5). App
  que "confia" no que a tela mostrou é BOLA/BFLA esperando acontecer (`schematize-pentest`).
- **Token fino:** carrega `sub`/tenant/AAL/sessão, **não** a lista de permissões — authz é
  consultada/cacheada no servidor com TTL curto (evita authz stale num token longo de app).
- **Multi-tenant:** troca de tenant no app é troca de contexto **verificada no servidor** a cada
  request; o app nunca "vira admin" mexendo em estado local.

## 7. Risco e transversais (herda do `iam.md` §9)

- O **risk engine** do IdP vale pro app: device novo, geovelocidade impossível, jailbreak/root
  detectado (`seguranca-mobile.md`) alimentam o score → step-up/deceção server-side.
- **Notifica canais verificados** em login novo/troca de credencial; "não fui eu" revoga a sessão
  daquele device.
- **Migração de legado (prioridade 0):** app com login caseiro antigo → estrangula pro IdP da casa
  (re-hash preguiçoso, revoga sessões legadas, ativa Email OTP baseline) — `iam.md` §7.

## Checklist mobile (entra na DoD quando o app tem auth)

- [ ] App é **public client OIDC/PKCE** delegando ao `auth.<domain>` — **sem** login próprio nem
  `client_secret` no bundle.
- [ ] Retorno por **Universal/App Links verificados** (não custom scheme cru); login no navegador
  do sistema (**não WebView**).
- [ ] **Passkeys de plataforma no núcleo**; **biometria desbloqueia local**, não substitui step-up
  server-side; fallback via Email OTP always-on.
- [ ] Refresh/chaves em **Keychain/Keystore** (Secure Enclave/StrongBox); **nada** em store em
  claro; token nunca em log/crash/analytics.
- [ ] **Logout irreversível** (revoga refresh+família no IdP, desassocia push); device é sessão
  revogável remotamente; push token rotacionado no logout/troca de usuário.
- [ ] **Authz sempre no servidor** (deny-default, token fino, ReBAC multi-tenant); esconder botão ≠
  autorizar.
- [ ] Jailbreak/root alimenta o **risk engine**; testes de abuso de fluxo no IdP (schematize-pentest).
</content>
