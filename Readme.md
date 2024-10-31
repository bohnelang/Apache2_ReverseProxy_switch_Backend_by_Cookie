# Switching reverse proxy backend by client cookie

## Introcution
Imagine you are testing a new layout and want to demonstrate this your leading group. You want to keep it simpley and it should look like the old one. 
Maybe you URL is: `https://www.mycompany.test/start/` and Backend CMS1 and CMS2

```
             +------------+   
             |   Reverse  +--------------- CMS1 (default)
User  >------+    Proxy   + 
             |            +--------------- CMS2 (test system 1)
             +------------+   
			 
         https://www.mycompany.test        https://cms1.mycompany.test
                                           https://cms2.mycompany.test
```

Now you have a presantation and want to switch by one click from the current to the new one. 

## Material and Methods
Now you have different ways:
* Switch by protocol: `http://www.mycompany.test/start/` -> Problems with data from forms
* Switch by host: `https://ww2.mycompany.test/start/` -> Maybe hard encoded URLs will crash demo
* Switch by port: `https://www.mycompany.test:8888/start/` -> Maybe hard encoded URLs will crash demo
* Switch by path: `https://www.mycompany.test/demo/start/` -> Maybe hard encoded URLs will crash demo / does not look like the original
* Switch by client's IP: From IP A: `http://www.mycompany.test/start/` directs to CMS1 and from IP B: `http://www.mycompany.test/start/` directs to CMS2 -> Need some configuration time
* Switch by user agent of the client: Needs special plug-ins


* **Switch by cookie**: e.g.: Cookie=1 `http://www.mycompany.test/start/` directs to CMS1 with Cookie=2 `http://www.mycompany.test/start/` directs to CMS2
```
             +------------+  Cookie not set
             |            +--------------- CMS1 (default)
User  >------+   Proxy    +
             |            +--------------- CMS2 (test system 1) 
             +--+---------+  Coockie=2
                | 
                +--- Web page for setting the Cookie OR RewriteRule to set cookie
```

This examplel ueses an dedicated web page for setting the cookies. Later the Rewrite rules of the mod_rewrite in combination with proxy function can switch requests by proxy.

**First** the web page where the user can select the backend: 

settestsyste.html:
```
<!DOCTYPE html>
<html>
<head>
<title> Choose System </title>
<script>
function setCookie(cname, cvalue, exdays) {
  if( exdays <0) {
        document.cookie =  cname + "=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/;";
  } else {
        var d = new Date();
        d.setTime(d.getTime() + (exdays*24*60*60*1000));
        var expires = "expires="+ d.toUTCString();
        document.cookie = cname + "=" + cvalue + ";" + expires + ";path=/";
  }
 location.href="https://www.mycompany.test/start/";
}

</script>
</head>

<body>
<form>
<input  type=button name="Default CMS1"      value="Default CMS1"       onclick="setCookie('ummsys', '1', -1) ;      return true">
<input  type=button name="System 1"          value="Test System 1"      onclick="setCookie('ummsys', '2',  1) ;      return true">
<input  type=button name="System 2"          value="Test System 2"      onclick="setCookie('ummsys', '3',  1) ;      return true">
</form>
</body>
</html>
```


**Second** you need a bunch of rewriting rules. In the first step you can define where the web page for setting the cookies is.
```
<VirtualHost _default_:443>
[...]

RewriteEngine On

#
# Enable the switching web page (in theis case the page is on the proxy itselve). It could be a web page on an other system as well. 
# 
RewriteRule ^/switch_backend$                                    /settestsyste.html      [PT,L]

# OR set cookie by rewrite (not tested)
# RewriteRule   ^/switch_backend$        https://www.mycompany.test/start/ 	[R,L,CO=ummsys:INVALID:;:-1]
# RewriteRule   ^/switch_backend/(.*)$   https://www.mycompany.test/start/ 	[R,L,CO=ummsys:$1:.mycompany.test:0:/;secure;httponly;samesite]
```

Secondly you define in a variable the URL of the backend.
```
#
# Defining the backends 
#

# Setting default CMS target
RewriteRule .* - [E=CMSHOST:https://cms1.mycompany.test]

# Selecting CMS2 as new default with cookie ummsys=2
RewriteCond %{HTTP_COOKIE} ummsys=2 [NC]
RewriteRule .* - [E=CMSHOST:https://cms2.mycompany.test]

# Selecting CMS3 as new default with cookie ummsys=3
RewriteCond %{HTTP_COOKIE} ummsys=3 [NC]
RewriteRule .* - [E=CMSHOST:https://cms3.mycompany.test]
```

And last not least you deligate the request to the defined backend by proxy function
```
#
# Redirecting pathes to selected backend 
#

# Redirect you path to backend 
RewriteCond            %{REQUEST_URI}          ^/start/(.*)$                                             
RewriteRule            ^(.*)$                  %{ENV:CMSHOST}/$1          [P,QSA,NE,L]
 
RewriteCond            %{REQUEST_URI}          ^/(([0-9]{1,6})|typo3conf|typo3temp|fileadmin|fileadmin.medma|uploads)/(.*)$
RewriteRule            ^(.*)$                  %{ENV:CMSHOST}/$1          [P,QSA,NE,L]
 
</VirtualHost>
``` 



## Result:
1. Visit webpage `http://www.mycompany.test/switch_backend` and select the backend you want to use
2. After selecting the CMS system you will be redirected to start page
3. All requestes are directed to selected CMS. The cookie will timeout after 1 day. 

4. Switch to default or other CMS start with 1



## Dicussion:
You are using the proxy function of the Apache2 mod_rewrite module (Rewrite Engine). You cannot do this by ProxyPass and ProxyPassRevers from the Apache mod_proxy module (Proxy). 
At some points you maybe need to filter stream and replace hard encoded links or use header directive to manipulate Location: value of the header. 

