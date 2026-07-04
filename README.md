# Site `ordina-app.fr` — Universal Links

Source du site statique servi sur `https://ordina-app.fr`. Son rôle premier :
héberger le fichier **`apple-app-site-association`** (AASA) qui associe le
domaine à l'app iOS (Universal Links), et les **pages relais** `/auth/*` qui
retransmettent les callbacks d'authentification Supabase vers l'app.

## Contenu

```
.well-known/apple-app-site-association   # preuve d'association domaine ↔ app (Team ID + bundle ID)
auth/confirm-callback/index.html         # relais confirmation d'inscription
auth/recovery-callback/index.html        # relais réinitialisation mot de passe
auth/invite-callback/index.html          # relais invitation d'équipe (V1.1+)
confidentialite/index.html               # politique de confidentialité (reprise du repo ordina-legal)
index.html                               # accueil provisoire (site vitrine à venir)
CNAME                                    # domaine personnalisé GitHub Pages
```

## Pourquoi des pages relais ?

Les Universal Links **ne se déclenchent pas sur une redirection serveur** :
quand Safari suit le 302 de Supabase (`…supabase.co/auth/v1/verify` →
`https://ordina-app.fr/auth/…`), iOS n'ouvre PAS l'app. La page relais comble
ce trou : elle retransmet le `?code=` (PKCE) à l'app via `ordina://…`
(tentative automatique + bouton visible). Sécurité conservée : le code ne
suffit pas sans le *code verifier* stocké dans la vraie app (PKCE), et l'app
n'accepte que les callbacks de son allowlist (SEC-10). L'association AASA
protège en plus tout lien `https://ordina-app.fr/auth/*` **tapé directement**
(Notes, Messages, mail affichant l'URL finale) : lui ouvre l'app sans détour
et n'est pas revendicable par une app tierce.

## Checklist d'activation (dans cet ordre)

1. **Déployer ce dossier** en réutilisant le repo existant `Hallusic/ordina-legal`
   (déjà branché sur GitHub Pages ; un domaine personnalisé ne peut appartenir
   qu'à UN repo, et c'est celui-ci qui portera aussi le futur site vitrine) :
   - cloner le repo à part, remplacer son contenu par le CONTENU de ce dossier
     (la politique de confidentialité y est reprise sous `confidentialite/`), pousser ;
   - Settings → Pages → Custom domain : `ordina-app.fr` (le fichier `CNAME` est
     déjà là) + cocher **Enforce HTTPS** (après émission du certificat) ;
   - l'ancienne URL `hallusic.github.io/ordina-legal/` continuera de fonctionner
     (GitHub redirige vers le domaine personnalisé) ; la politique vit désormais
     à `https://ordina-app.fr/confidentialite/` — **c'est cette URL** à utiliser
     pour App Store Connect (Privacy Policy URL), le Site URL Supabase PROD et
     les liens in-app (`PaywallView.privacyPolicy`, Réglages → Confidentialité),
     à mettre à jour à l'activation ;
   - (optionnel, cosmétique) renommer le repo en `ordina-app.fr`.
2. **DNS chez OVH** (zone `ordina-app.fr`) — ne PAS toucher aux enregistrements MX/TXT de Resend :
   - 4 enregistrements `A` sur l'apex → `185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153` ;
   - (optionnel) `CNAME www` → `hallusic.github.io`.
3. **Vérifier l'AASA** (⚠️ le CDN d'Apple met en cache — compter jusqu'à 24 h) :
   ```bash
   curl -i https://ordina-app.fr/.well-known/apple-app-site-association
   # attendu : 200, JSON, pas de redirection
   ```
   Si GitHub Pages sert un mauvais Content-Type et que l'association échoue,
   basculer sur Cloudflare Pages/Netlify (mêmes fichiers).
4. **Supabase Dashboard** (DEV `wxdw…` **et** PROD `jlaz…`) → Auth →
   URL Configuration → **Redirect URLs** : ajouter
   `https://ordina-app.fr/auth/confirm-callback`,
   `https://ordina-app.fr/auth/recovery-callback`,
   `https://ordina-app.fr/auth/invite-callback`
   (conserver les `ordina://…` existants — l'app garde le double support).
5. **App iOS** : passer `AppConfig.universalLinksEnabled` à `true` (les
   `redirectTo` des e-mails passent en `https://…`). L'entitlement
   `applinks:ordina-app.fr` est déjà dans `Ordina/Ordina.entitlements`.
6. **Edge fn `invite_member`** (inerte en V1, gestion d'équipe masquée) :
   basculer `INVITE_REDIRECT_URL` sur
   `https://ordina-app.fr/auth/invite-callback` et redéployer (DEV+PROD).
7. **Tester sur iPhone** : app installée → coller
   `https://ordina-app.fr/auth/confirm-callback` dans Notes → **appui long**
   → « Ouvrir dans Ordina » doit apparaître. Puis un vrai parcours
   inscription (PROD) et mot de passe oublié de bout en bout.

Astuce dev : pour court-circuiter le cache CDN d'Apple pendant les essais,
ajouter temporairement `applinks:ordina-app.fr?mode=developer` dans
l'entitlement (builds de développement uniquement) + activer le mode
développeur Associated Domains sur l'iPhone (Réglages → Développeur).

## Rappels

- L'AASA doit rester servi **sans extension**, en HTTPS, **sans redirection**.
- Toute modification d'appID (Team ID / bundle ID) doit être répercutée ici
  ET dans l'entitlement.
- PKCE : le lien d'un e-mail d'auth doit être ouvert sur **l'appareil qui a
  initié** l'inscription/la réinitialisation (le *code verifier* est local).
