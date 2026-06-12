 ## Reminder for me, about the basics of Oauth 2.0 and it's core security weak points

Okay so basically, here let's dump the basic stuff first

### Basic concepts
- Idp == Authorization Server == e.g. Google,

- Resources Server  (e.g. google drive, google photos etc...)

- Client  (the app that uses Oauth)


###  Oauth 2  Login flow
1)
```
$url = "https://accounts.google.com/o/oauth2/auth";
 
// build the HTTP GET query
$params = array(
    "response_type" => "code",
    "client_id" => $oauth2_client_id,
    "redirect_uri" => $oauth2_redirect,
    "scope" => "openid email profile"
    );
$request_to = $url . '?' . http_build_query($params);
 
// forward the user to the login access page on the OAuth 2 server
header("Location: " . $request_to);
```
2) This brings user to the Google Oauth page.
After user clicks on his account, google redirects to the specified redirect_uri get parameter value (which is also specified in your google console, to prevent malicious redirects,
however code stealing by universal param redirect can cause ATO if I got the logic correct)
With ?code=... 
3)  Now the last part, the page that should handle google redirect with code parameter 
```
    $code = $_GET['code'];
    $url = 'https://accounts.google.com/o/oauth2/token';
        // this will be our POST data to send back to the OAuth server in exchange
        // for an access token
    $params = array(
        "code" => $code,
        "client_id" => $oauth2_client_id,
        "client_secret" => $oauth2_secret,
        "redirect_uri" => $oauth2_redirect,
        "grant_type" => "authorization_code"
    );
```

This request basically gives us back access_token, id_token, refresh_token etc... 
NOTE: Refresh tokens can be non-expireable which is dangerous if exposed

**Quick notes:**
Now access_token and id_token are sensitive, and if they are ever exposed somewhere, then passed to backend
We as a pentester, should try access_token or id_token swapping, by our own access_token which we can get after authing via Oauth from our local instance.

So passing that to backend,
If there is a blind trust, that's a bug  -  basically attacker can create fake app, make user register with his Google Oauth,
then after getting the access_token  --> pass that to the app X, which has some /endpoint, where when you pass the access_token, it tries to get email/sub with that access_token,
and instantly passes user session after.

by default in code= case, the Idp (Google) handles the check that code should belong to client_id
So there is no spoofing possible in this part of flow.  (See again the 3-rd part of the flow)

4) But when app does
```
curl -H "Authorization: Bearer <TOKEN>" \
https://www.googleapis.com/oauth2/v3/userinfo
```
in backend, usually there might be no checks at all.
In Oauth implicit flow  -  where access_token is passed in fragment then postMessage-ed etc... 
This token swapping can be tested.

After this request,  app get's in JSON sub and email of the user.

It's recommended for apps to trust the `sub` (it's just numeric ID) during session initiation.
Cause trusting email can cause some account linking confusions, I also thought about homoglyph attacks here, small talk with AI reveals I just re-discovered a niche bug class,
well this happens time to time, not the first case nor last.

###  Oauth 2  Resources permission grant flow 

Same as Oauth login just in scope mention what you want, and then using `access_token` get whatever access you need, from the Oauth resource provider.


## Security angles overall

1) `access_token`, `id_token`  swapping with yours from local instance.

2) State not verified against session case,  during account attachment.
You see, there is also the case, where you need to attach SSO to your account login methods, after you registered in traditional way.

And this is also a juicy area, you can basically with **acc B** not finish your ?code=...&state=... request, and pass it to **acc A**

If he clicks, and there is no verification of state against session, the **account A** will now have attackers gmail attached to his account,
as an alternate login method.
Attacker can now simply login with his gmail to **acc A**, anytime.

3) I am writing this, inspired by my latest finding.
Same `client_id`  --> and considering cryptography is used against `sub` or `email`.
If anything as such is observed cross different domains of same company,
Or cross subdomains of same domain that have no shared login methods in general (else there may be intended one session cross subs, we don't need these cases).

We can try cookie or token of **site A acc A** on **site B acc A**,

If it works report it as Oauth 2 RFC violation 
```
RFC 6750 §5.2 — resource servers MUST validate tokens were issued for them
```
Cause it is basically, **ATO on site A acc A === ATO on site B acc A**.

P.S. This is theory rn in my mind, but shall work imho.
