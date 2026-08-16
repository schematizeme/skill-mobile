# Entrega nas lojas — assinatura, rollout, OTA, review, observabilidade

> A entrega da casa (`entrega.md`, DoD §35, archive §28) vale inteira no mobile; o que muda é que
> **entre o seu commit e o usuário existe uma loja** (App Store / Play) com **revisão humana,
> assinatura criptográfica e rollout gradual**. Publicar app é operação de **ops** com gate, não
> "sobe e reza". E o binário na mão do usuário **não dá rollback instantâneo** — por isso rollout
> gradual e observabilidade são inegociáveis.

## 1. Assinatura e credenciais de publicação — segredo de ops

- **Chaves de assinatura são segredo crítico (`seguranca-mobile.md`, `ops.md`):** a **upload/app
  signing key (Android)** e os **certificados/perfis (iOS)** vivem em cofre (não no repo, não no
  laptop do dev). Perder a chave Android = não conseguir mais atualizar o app.
- **Play App Signing** (Google gerencia a chave de assinatura) e **assinatura gerenciada da Apple**
  quando possível — reduz o raio de perda. Credenciais de CI (App Store Connect API key, service
  account) em secret manager, com rotação.
- **Build de release reprodutível e assinado no CI**, não na máquina de alguém. O artefato que vai
  pra loja é o que o CI produziu e registrou (rastreável ao commit — casa com o archive §28).

## 2. Versionamento e trilhos de release

- **SemVer visível** (versão de marketing) **+ build number monotônico** (sobe sempre, nunca
  reusa). Loja rejeita build number repetido.
- **Trilhos/tracks:** `internal → closed/TestFlight → open beta → production`. Nada vai direto pra
  produção sem passar por um trilho de teste com usuários reais de teste.
- **Cada release tem changelog e archive:** o que mudou, qual commit, qual resultado do rollout —
  no `_archive` (§28). Release sem rastro não aconteceu.

## 3. Staged/phased rollout — nunca 100% de uma vez

- **Rollout gradual obrigatório:** Play **staged rollout** (1% → 5% → 20% → 50% → 100%) e iOS
  **phased release** (7 dias). Você **observa crash/ANR/erro** entre os degraus e **segura/halt** se
  a métrica piorar.
- **Gate de promoção:** só sobe o degrau se crash-free e métricas de negócio estão sãos vs. a versão
  anterior. Regressão → **halt rollout** (Play) — para novos usuários, contém o dano.
- **Rollback é caro no mobile:** você **não** "volta a versão" no device do usuário; você **halta** o
  rollout e publica um **hotfix pra frente** (nova versão). Por isso o gradual é a rede de segurança
  — o estrago fica contido em % pequeno.

## 4. OTA (over-the-air) — quando aplicável e dentro das regras

- **OTA de conteúdo/config sempre:** feature flags, config remota, kill-switch de feature, conteúdo
  — atualizáveis sem release de loja. **Kill-switch** de feature nova é piso: liga/desliga sem
  esperar review.
- **OTA de código JS (RN — CodePush/EAS Update)** é legítimo **para JS bundle**, respeitando as
  **regras das lojas**: OTA **não** pode mudar propósito/comportamento aprovado do app nem burlar a
  review (Apple 3.3.1 / Google). Correção de bug e ajuste de UI: ok. "App novo por OTA": violação.
- **OTA assinado e versionado:** o pacote OTA é assinado e casado a uma faixa de versão nativa
  (bundle novo em binário antigo pode crashar) — teste de compatibilidade antes de empurrar.
- **Native precisa de review:** mudança de código nativo, permissão nova, SDK novo → passa pela loja.
  Não existe OTA de binário nativo.

## 5. Review guidelines — projete pra passar

- **Conheça as regras antes de construir:** App Store Review Guidelines e Google Play Policies são
  requisito, não surpresa. Login obrigatório sem valor demonstrável, **Sign in with Apple** quando há
  login social de terceiro, pagamento de bem digital **fora** do IAP, permissão sem justificativa,
  privacy label errada, conteúdo gerado sem moderação → **rejeição**.
- **Privacy é gate de loja:** App Privacy (Apple) / Data safety (Google) preenchidos com a verdade;
  **ATT (App Tracking Transparency)** no iOS quando há tracking; peça permissão **just-in-time** com
  purpose string honesta (`seguranca-mobile.md` §6).
- **Conta de teste e demo:** forneça credenciais/roteiro pro revisor (senão rejeita por "não
  consegui testar"). Guest mode ou deep link de demo ajuda.

## 6. Observabilidade e crash — o app está longe, você precisa enxergar

- **Crash + ANR reporting obrigatório** (Crashlytics/Sentry/nativo): stack simbolicada (suba
  **dSYM/mapping/ProGuard map** no release), crash-free users/sessions como métrica de gate do
  rollout (§3).
- **Sem PII/segredo na telemetria:** scrub de token, email, dado de cliente de logs, breadcrumbs e
  crash reports (`iam-mobile.md` §4). Telemetria é dado que sai do device — trate como export.
- **Métricas de UX/negócio:** cold start, tempo até interativo, taxa de sucesso de sync, falhas de
  rede, funil — casam com a observabilidade LGTM+ da casa (`observabilidade.md`).
- **Consentimento e privacidade:** analytics respeita opt-out/ATT; mínimo necessário; nada de
  fingerprinting escondido (é rejeição de loja e violação de confiança).

## 7. DoD de release mobile (soma à DoD §35)

- [ ] Build de release **assinado no CI**, rastreável ao commit; chaves em cofre.
- [ ] Version/build number corretos; changelog + archive (§28) do release.
- [ ] Passou por **trilho de teste** (TestFlight/closed track) antes de produção.
- [ ] **Staged/phased rollout** configurado com gate de crash-free/ANR e plano de **halt**.
- [ ] **Kill-switch/flags** da feature nova; OTA (se houver) dentro das regras da loja e
  compatível com a faixa de versão nativa.
- [ ] **Crash/ANR** com símbolos subidos; telemetria **sem PII/segredo**; privacy labels/ATT/permissões corretas.
- [ ] Conta/roteiro de **review** preparados; regras da loja verificadas.
