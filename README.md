![Security Labs](https://img.shields.io/badge/Security-Labs-blue) ![DVWA](https://img.shields.io/badge/Tool-DVWA-red) ![Burp Suite](https://img.shields.io/badge/Tool-BurpSuite-orange) ![PortSwigger](https://img.shields.io/badge/Platform-PortSwigger-purple)
# 🔐 Web Application Security Labs / Labs de Sécurité Web
### by Anas TAGUI

> **EN:** A practical walkthrough of real web security vulnerabilities , how I found them, exploited them, and what they taught me.  
> **FR:** Un compte-rendu pratique de vraies vulnérabilités web , comment je les ai trouvées, exploitées, et ce qu'elles m'ont appris.

**Tools / Outils:** DVWA · PortSwigger Web Security Academy · Burp Suite Community Edition  
**Topics / Sujets:** CSRF · Authentication · Session Management · Access Control · IDOR · JWT

---

## 📋 Table of Contents / Sommaire

- [Part 1 – CSRF (DVWA)](#part-1--csrf-dvwa)
- [Part 2 – Authentication & Sessions (PortSwigger)](#part-2--authentication--sessions)
- [Part 3 – Access Control (PortSwigger)](#part-3--access-control)
- [Key Takeaways / Ce que j'ai retenu](#key-takeaways--ce-que-jai-retenu)

---

## Part 1 – CSRF (DVWA)

### What's going on here? / C'est quoi le problème ?

**EN:** CSRF (Cross-Site Request Forgery) is one of those vulnerabilities that sounds complicated but is actually terrifying in how simple it is. The idea: trick a logged-in user's browser into making a request they never intended to make. The server sees a valid session cookie and just... does it.

**FR:** Le CSRF (Cross-Site Request Forgery) est une de ces vulnérabilités qui semble complexe mais qui est en réalité effrayante par sa simplicité. Le principe : tromper le navigateur d'un utilisateur connecté pour qu'il envoie une requête à son insu. Le serveur voit un cookie de session valide et... l'exécute.

---

### 🟢 Level Low / Niveau Faible

**EN:** At this level, there's literally zero protection. The password change happens over a GET request with the new password sitting right there in the URL. All I had to do was craft a link and get the victim to click it (or even just visit a page that auto-loads it).

**FR:** À ce niveau, il n'y a absolument aucune protection. Le changement de mot de passe se fait via une requête GET avec le nouveau mot de passe directement dans l'URL. Il suffit de créer un lien et de faire en sorte que la victime le visite.

```
http://localhost/vulnerabilities/csrf/?password_new=hacked&password_conf=hacked&Change=Change
```

**Result / Résultat:** Password changed instantly. No clicks, no confirmation, nothing.  
*Mot de passe changé instantanément. Aucun clic, aucune confirmation.*

> 💡 **Fix:** Never use GET for state-changing actions. Use POST + anti-CSRF tokens.  
> *Ne jamais utiliser GET pour des actions qui modifient l'état. Utiliser POST + jetons anti-CSRF.*

---

### 🟡 Level Medium / Niveau Moyen

**EN:** This level tries to protect itself by checking the `Referer` header , basically asking "where did this request come from?" The catch? It uses `stripos()` to check if the word `localhost` appears *anywhere* in the URL. So I just named my attack file `localhost.html` and served it from my own machine. The check passed.

**FR:** Ce niveau tente de se protéger en vérifiant l'en-tête `Referer` , en demandant "d'où vient cette requête ?". Le problème ? Il utilise `stripos()` pour vérifier si le mot `localhost` apparaît *quelque part* dans l'URL. J'ai donc simplement nommé mon fichier d'attaque `localhost.html` et l'ai servi depuis ma machine. Le filtre est passé.

**Attack file / Fichier d'attaque (`localhost.html`):**
```html
<html>
  <body onload="document.forms[0].submit()">
    <form action="http://localhost/vulnerabilities/csrf/" method="GET">
      <input type="hidden" name="password_new" value="mediumhacked">
      <input type="hidden" name="password_conf" value="mediumhacked">
      <input type="hidden" name="Change" value="Change">
    </form>
  </body>
</html>
```

Served with / Servi avec: `python3 -m http.server 8000`

> 💡 **Fix:** Verify the full origin, not just a keyword. Or better , use proper CSRF tokens.  
> *Vérifier l'origine complète, pas seulement un mot-clé. Ou mieux : utiliser de vrais jetons CSRF.*

---

### 🔴 Level High / Niveau Élevé

**EN:** Now we're talking. The app generates a random `user_token` on every page load. You can't guess it, you can't reuse an old one. A classic CSRF attack fails here... unless you combine it with XSS. The trick is to use JavaScript to silently fetch the CSRF page, steal the token from the HTML, then fire the forged request with the valid token included.

**FR:** Là ça se corse. L'application génère un `user_token` aléatoire à chaque chargement de page. Impossible à deviner, impossible à réutiliser. Une attaque CSRF classique échoue ici... sauf si on la combine avec du XSS. L'astuce : utiliser JavaScript pour récupérer silencieusement la page CSRF, extraire le token du HTML, puis envoyer la requête forgée avec ce token valide.

```javascript
// Fetch the page to get the token / Récupérer la page pour obtenir le token
var xhr = new XMLHttpRequest();
xhr.open("GET", "http://localhost/vulnerabilities/csrf/", false);
xhr.send(null);

// Extract the token / Extraire le token
var parser = new DOMParser();
var doc = parser.parseFromString(xhr.responseText, "text/html");
var token = doc.getElementsByName('user_token')[0].value;

// Fire the forged request / Envoyer la requête forgée
var req = new XMLHttpRequest();
req.open("GET",
  "http://localhost/vulnerabilities/csrf/?password_new=highhacked&password_conf=highhacked&Change=Change&user_token=" + token,
  false
);
req.send(null);
```

> 💡 **Fix:** CSRF tokens work great , but they rely on XSS being impossible. Fix XSS first. Also use `SameSite=Strict` cookies.  
> *Les jetons CSRF sont efficaces , mais supposent l'absence de XSS. Corriger d'abord le XSS. Utiliser aussi les cookies `SameSite=Strict`.*

---

## Part 2 – Authentication & Sessions

**Platform / Plateforme:** PortSwigger Web Security Academy  
**Tool / Outil:** Burp Suite Community Edition

---

### 1. Username Enumeration via Different Responses

**Lab:** [Username enumeration via different responses](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-different-responses)

**EN:** The login page always returned HTTP 200 , no matter what username you tried. Looked safe. But the response *length* was slightly different when a username actually existed. That tiny difference was enough to enumerate valid accounts using Burp Intruder.

**FR:** La page de connexion retournait toujours un HTTP 200 , peu importe le nom d'utilisateur. Ça semblait sûr. Mais la *longueur* de la réponse était légèrement différente quand un nom d'utilisateur existait vraiment. Cette petite différence suffisait à énumérer les comptes valides avec Burp Intruder.

**Steps / Étapes:**
1. Intercept `POST /login` → send to Intruder / *Intercepter `POST /login` → envoyer dans Intruder*
2. Fuzz the `username` field with the wordlist / *Fuzzer le champ `username` avec la wordlist*
3. Sort by response length , `arkansas` returned `2986` vs `2984` for all others / *Trier par longueur , `arkansas` retourne `2986` contre `2984` pour les autres*
4. Run the same attack on `password` → `matthew` returned a `302` redirect / *Même attaque sur `password` → `matthew` retourne une redirection `302`*

> 💡 **Fix:** Same error message, same response size, every time. Add rate limiting.  
> *Même message d'erreur, même taille de réponse, à chaque fois. Ajouter du rate limiting.*

---

### 2. 2FA Simple Bypass

**Lab:** [2FA simple bypass](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-simple-bypass)

**EN:** This one was almost too easy and that's what makes it scary. The app has a two-step login: password first, then a 2FA code. But after entering the password, nothing stops you from just... skipping the 2FA page and navigating directly to `/my-account`. The server never checked whether step 2 was actually completed.

**FR:** Celle-là était presque trop facile , et c'est justement ce qui la rend inquiétante. L'app a une connexion en deux étapes : d'abord le mot de passe, puis un code 2FA. Mais après avoir entré le mot de passe, rien n'empêche de... sauter la page 2FA et naviguer directement vers `/my-account`. Le serveur ne vérifiait jamais si l'étape 2 avait été complétée.

**Steps / Étapes:**
1. Log in as `carlos / montoya` / *Se connecter en tant que `carlos / montoya`*
2. Get redirected to `/login2` for the 2FA code / *Être redirigé vers `/login2` pour le code 2FA*
3. Manually type `/my-account` in the address bar / *Taper manuellement `/my-account` dans la barre d'adresse*
4. Full access granted / *Accès complet accordé*

> 💡 **Fix:** Track auth state server-side. Only grant access after all steps are confirmed complete.  
> *Suivre l'état d'authentification côté serveur. N'accorder l'accès qu'après confirmation de toutes les étapes.*

---

### 3. JWT Authentication Bypass via Unverified Signature

**Lab:** [JWT authentication bypass via unverified signature](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-unverified-signature)

**EN:** JWTs have three parts: header, payload, and a cryptographic signature that proves the token wasn't tampered with. This server decoded the payload and trusted it , but never actually verified the signature. So I just edited my username in the payload from `wiener` to `administrator`, left the signature untouched, and sent it. Admin access, no questions asked.

**FR:** Les JWT ont trois parties : en-tête, payload, et une signature cryptographique qui prouve que le token n'a pas été modifié. Ce serveur décodait le payload et lui faisait confiance , sans jamais vérifier la signature. J'ai donc modifié mon nom d'utilisateur de `wiener` à `administrator`, laissé la signature intacte, et envoyé le tout. Accès admin, sans aucune question.

**Before / Avant:** `"sub": "wiener"`  
**After / Après:** `"sub": "administrator"`

> 💡 **Fix:** Always verify JWT signatures server-side. Use a maintained library. Reject any token with an invalid or missing signature.  
> *Toujours vérifier les signatures JWT côté serveur. Utiliser une bibliothèque maintenue. Rejeter tout token avec une signature invalide ou absente.*

---

## Part 3 – Access Control

**Platform / Plateforme:** PortSwigger Web Security Academy

**EN:** This section was a reality check. Most of these vulnerabilities don't require any special tools or clever tricks , just the willingness to try things the developer assumed nobody would try.

**FR:** Cette section a été un vrai réveil. La plupart de ces vulnérabilités ne nécessitent aucun outil spécial ni astuce complexe , juste la volonté d'essayer des choses que le développeur supposait que personne ne tenterait.

---

### 1. User Role Can Be Modified in User Profile

**Lab:** [User role can be modified in user profile](https://portswigger.net/web-security/access-control/lab-user-role-can-be-modified-in-user-profile)  
**Vulnerability / Vulnérabilité:** Mass Assignment / Vertical Privilege Escalation

**EN:** The profile update endpoint accepted a JSON body. The UI only showed an email field. But I added `"roleid": 2` to the request body and the server happily processed it, upgrading my account to admin.

**FR:** L'endpoint de mise à jour du profil acceptait un corps JSON. L'interface n'affichait qu'un champ email. Mais j'ai ajouté `"roleid": 2` dans le corps de la requête et le serveur l'a traité sans broncher, me donnant les droits admin.

> 💡 **Fix:** Allowlist only the fields each endpoint is supposed to accept. Never trust the client to send only safe parameters.  
> *N'autoriser que les champs attendus pour chaque endpoint. Ne jamais faire confiance au client.*

---

### 2. User ID Controlled by Request Parameters (IDOR/GUID)

**Lab:** [User ID controlled by request parameters](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-unpredictable-user-ids)

**EN:** The app used GUIDs instead of sequential IDs , a smart move. But those GUIDs were visible in the blog's author profile URLs. Once I had Carlos's GUID, I swapped it into my own account URL and accessed his profile, including his API key.

**FR:** L'app utilisait des GUIDs au lieu d'IDs séquentiels , une bonne idée. Mais ces GUIDs étaient visibles dans les URLs des profils d'auteurs du blog. Une fois le GUID de Carlos obtenu, je l'ai remplacé dans l'URL de mon propre compte et j'ai accédé à son profil, clé API incluse.

> 💡 **Fix:** Hard-to-guess IDs aren't enough. Always check server-side that the requesting user owns the resource.  
> *Des IDs difficiles à deviner ne suffisent pas. Toujours vérifier côté serveur que l'utilisateur est propriétaire de la ressource.*

---

### 3. Unprotected Admin Functions

**Lab:** [Unprotected admin functionality](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality)

**EN:** The admin panel wasn't linked anywhere in the UI. The developers thought that was enough. It wasn't , the path `/administrator-panel` was sitting right there in `robots.txt`. Visiting the URL gave full admin access with zero authentication checks.

**FR:** Le panneau d'administration n'était lié nulle part dans l'interface. Les développeurs pensaient que c'était suffisant. Ce ne l'était pas , le chemin `/administrator-panel` était là, dans `robots.txt`. Visiter l'URL donnait un accès admin complet sans aucune vérification.

> 💡 **Fix:** Security through obscurity is not security. Every sensitive endpoint needs real authorization checks.  
> *La sécurité par l'obscurité n'est pas de la sécurité. Chaque endpoint sensible a besoin de vraies vérifications d'autorisation.*

---

### 4. Insecure Direct Object Referencing (IDOR on Static Files)

**Lab:** [Insecure Direct Object References](https://portswigger.net/web-security/access-control/lab-insecure-direct-object-references)

**EN:** The chat transcript download used a sequential filename in the URL (`2.txt`, `1.txt`, etc.). No check was made to verify who the file belonged to. I changed `2.txt` to `1.txt` and downloaded Carlos's conversation , which contained his plaintext password.

**FR:** Le téléchargement des transcripts de chat utilisait un nom de fichier séquentiel dans l'URL (`2.txt`, `1.txt`, etc.). Aucune vérification n'était faite pour savoir à qui appartenait le fichier. J'ai changé `2.txt` en `1.txt` et téléchargé la conversation de Carlos , qui contenait son mot de passe en clair.

> 💡 **Fix:** Use signed, user-specific access tokens for file downloads. Always verify ownership server-side.  
> *Utiliser des tokens d'accès signés et spécifiques à l'utilisateur. Toujours vérifier la propriété côté serveur.*

---

## Key Takeaways / Ce que j'ai retenu

**EN:** After going through all of these, one pattern became very clear: most of these vulnerabilities aren't sophisticated. They exist because of small assumptions , "nobody will type that URL", "the response looks the same", "we're using GUIDs so we're fine". Security isn't about making things hard to find. It's about making sure that even if someone finds it, they still can't do anything with it.

**FR:** Après avoir traversé tout ça, un schéma est devenu très clair : la plupart de ces vulnérabilités ne sont pas sophistiquées. Elles existent à cause de petites suppositions , "personne ne tapera cette URL", "la réponse a l'air identique", "on utilise des GUIDs donc on est tranquilles". La sécurité ne consiste pas à rendre les choses difficiles à trouver. C'est s'assurer que même si quelqu'un les trouve, il ne peut rien en faire.

| Vulnerability / Vulnérabilité | Root Cause / Cause | Fix / Correction |
|---|---|---|
| CSRF (Low) | No protection / Aucune protection | Anti-CSRF tokens + POST |
| CSRF (Medium) | Weak Referer check / Vérification Referer faible | Strict origin validation |
| CSRF (High) | Token stolen via XSS / Token volé via XSS | Fix XSS + SameSite cookies |
| Username Enumeration | Different response sizes / Tailles de réponse différentes | Uniform responses + rate limiting |
| 2FA Bypass | Missing state check / Vérification d'état manquante | Server-side auth state tracking |
| JWT Bypass | Signature not verified / Signature non vérifiée | Always verify signatures |
| Mass Assignment | No field filtering / Pas de filtrage des champs | Server-side allowlisting |
| IDOR / GUID | No ownership check / Pas de vérification de propriété | Server-side authorization |
| Admin Obscurity | No real access control / Pas de vrai contrôle d'accès | Auth checks on every endpoint |
| IDOR on Files | Sequential IDs, no auth / IDs séquentiels, pas d'auth | Signed tokens + ownership checks |

---

> ⚠️ **Disclaimer / Avertissement:** All exercises were performed in legal, isolated lab environments (DVWA and PortSwigger Web Security Academy). No real systems were targeted.  
> *Tous les exercices ont été réalisés dans des environnements de laboratoire légaux et isolés. Aucun système réel n'a été ciblé.*
