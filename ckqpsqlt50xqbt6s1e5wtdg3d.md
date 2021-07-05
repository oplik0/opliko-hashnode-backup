## Polish ESCS Qualifications 2021 - login system

While unfortunately I haven't managed to do a lot during the CTF (having different plans for the weekend didn't help when I picked apparently the second hardest challenge [judging by number of solves] after the few easiest ones :), I thought it's a good opportunity to start this blog. So this will be the first in probably 3 posts - the others covering *Back to school* and *NFC Init* challenges. So let's start.

# About the task

This is the first (not counting the sanity check) challenge of the CTF and as such it was definitely the most solved one - probably due to the placement and relatively low difficulty. You can find the challenge under https://hack.cert.pl/challenge/login-system

It's description is also short and simple:
> Just a login system. Nothing extraordinary.

# From the beginning
When we open the page we see that the description was honest and it really is just a login screen with some basic bootstrap styling.

![index.html](https://cdn.hashnode.com/res/hashnode/image/upload/v1625434190089/QXx4MiFma.png)

You may have noticed the weird version by now, but we'll come back to it later. First some basics - let's try a few credentials.

`admin:admin` doesn't work, but we see that we're redirected to `/user` page. Probably that's our target then.

![403 forbidden on /user](https://cdn.hashnode.com/res/hashnode/image/upload/v1625434334390/EE0lva-C8.png)

If we try a few more we might find that `user:user` seems to work and confirms the suspicion, but we still need admin credentials:

![login as user](https://cdn.hashnode.com/res/hashnode/image/upload/v1625434676672/FWdBl2can.png)

We could try some SQLi in the user inputs, but it won't work - and besides, we already have something to investigate.

# Weird version and a bit about `.git`

So what's up with the `Version: 8124c1fa36f29d23c70593c5f7cc9ee54244269c` string? This doesn't seem like some normal versioning scheme, but rather like a hash or something. Well, let's open the devtools and find if there are some hints:

![HTML source](https://cdn.hashnode.com/res/hashnode/image/upload/v1625434859857/tT0bY3-Ho.png)

Interesting, it seems that the version isn't set manually, but instead this short script fetches some endpoint to get it. The URL itself is more interesting though. We might have a git source code exposure here.

Unfortunately, it isn't as simple as going to `/.git/` - seems like we can't see folders directly.

![forbidden on .git](https://cdn.hashnode.com/res/hashnode/image/upload/v1625434974266/dJvXWKZiT.png)

Files in this folder work fine though - attempting to downlaod `/.git/HEAD` works fine
![HEAD download](https://cdn.hashnode.com/res/hashnode/image/upload/v1625435014177/nh1DXGzPN.png)

Now, if it was some random folder it'd be a problem, as we'd have to guess the filenames somehow. But fortunately `.git` folder has a easy to predict structure - due to the nature of git these files need to be linked together in a way that'd allow to find them after all.

We could now read the docs and figure out how git works, but fortunately people have done that for us already. Meet [git-dumper](https://github.com/arthaud/git-dumper) - a simple python tool that can extract a repository from an exposed .git folder. After installing it we can simply run it on the required url:
```bash
$ git-dumper https://login-system.ecsc21.hack.cert.pl/.git/ login-system
```

![git dumper](https://cdn.hashnode.com/res/hashnode/image/upload/v1625435619760/1NrgWEwlP.png)

And we get a full git repository out - with the code and its history. Let's open the folder in VS Code then and look at what we have.

# Analyzing the code

![all of the code](https://cdn.hashnode.com/res/hashnode/image/upload/v1625435784145/JkXHwrA9U.png)
Well, it seems that we were totally right - if we successfully log in as `admin` we will get to see contents of `flag.txt`. The file is in `.gitignore`, so we can't see it in the repository and the URL also doesn't work, so it seems like we don't have another choice.

It seems that the passwords are hardcoded, so the obvious route here would be to crack the admin password, but I haven't managed to find it in any online reverse lookup, so it can be probably ruled out - besides, what fun would it be to just wait for hashcat to chew through this sha256? There must be a more interesting solution.

If we look through commit history we can see the initial commit and two commits titled `fix`.


![git history](https://cdn.hashnode.com/res/hashnode/image/upload/v1625436246670/Yb0I0Y9jf.png)

The first `fix` appears to add the actual CTF mechanics to a previous even simpler app. That is - it adds the flag to the `user.html` template, modifies `app.py` to pass it to the template and adds the version script to `index.html`

![fix #1](https://cdn.hashnode.com/res/hashnode/image/upload/v1625436396981/CyLBBSVVA.png)

The second commit contains only a small change to a `SECRET_KEY` Flask variable and a new comment

![fix #2](https://cdn.hashnode.com/res/hashnode/image/upload/v1625436459196/xdmrbYTqq.png)

That is our biggest hint though. If the last change was adding a TODO, then surely it hasn't yet been done, so we can probably assume `Flask['SECRET_KEY']` is still set to `\xCC` repeated 16 times.

The name immediately suggests that we learned something that we shouldn't - so now we just need to know what `SECRET_KEY` is in Flask and why it needs to be secret, or in other words what can we do if we have it.

# Demystifying secrets

Fortunately a quick search can take us to [this StackOverflow question titled `demystify Flask app.secret_key`](https://stackoverflow.com/questions/22463939/demystify-flask-app-secret-key) where we have two helpful responses. It seems that Flask is using it to sign the session cookie. The cookie is not encrypted (which we can confirm in [quickstart docs](https://flask.palletsprojects.com/en/2.0.x/quickstart/#sessions) - "user could look at the contents of your cookie but not modify it"). The first answer also mentions the library Flask uses - [`itsdangerous`](https://itsdangerous.palletsprojects.com/en/2.0.x/), and more specifically the [`URLSafeTimedSerializer` class](https://itsdangerous.palletsprojects.com/en/2.0.x/url_safe/#itsdangerous.url_safe.URLSafeTimedSerializer).

![itsdangerous docs](https://cdn.hashnode.com/res/hashnode/image/upload/v1625437024292/QB_TUU_6_.png)

From the docs we can see that to create an object of this class we need `secret_key` (which we have), but can also add `salt` (which we don't yet know if is changed from the default in Flask), `serializer`, `serializer_kwargs`, `signer` and `signer_kwargs`. We can probably safely ignore the fallbacks.

We could try using the defaults, but Flask is fortunately open source so we don't have to.  We can quickly find its [GitHub repository](https://github.com/pallets/flask) and from there go to the [`sessions.py` file](https://github.com/pallets/flask/blob/main/src/flask/sessions.py). After a quick search for `URLSafeTimedSerializer` we can find it in [line 343](https://github.com/pallets/flask/blob/main/src/flask/sessions.py#L343),  inside the `SecureCookieSessionInterface` class where we can find all of the default parameters:
```python3
class SecureCookieSessionInterface(SessionInterface):
    """The default session interface that stores sessions in signed cookies
    through the :mod:`itsdangerous` module.
    """
    #: the salt that should be applied on top of the secret key for the
    #: signing of cookie based sessions.
    salt = "cookie-session"
    #: the hash function to use for the signature.  The default is sha1
    digest_method = staticmethod(hashlib.sha1)
    #: the name of the itsdangerous supported key derivation.  The default
    #: is hmac.
    key_derivation = "hmac"
    #: A python serializer for the payload.  The default is a compact
    #: JSON derived serializer with support for some extra Python types
    #: such as datetime objects or tuples.
    serializer = session_json_serializer
    session_class = SecureCookieSession
    def get_signing_serializer(
        self, app: "Flask"
    ) -> t.Optional[URLSafeTimedSerializer]:
        if not app.secret_key:
            return None
        signer_kwargs = dict(
            key_derivation=self.key_derivation,
            digest_method=self.digest_method
        )
        return URLSafeTimedSerializer(
            app.secret_key,
            salt=self.salt,
            serializer=self.serializer,
            signer_kwargs=signer_kwargs,
        )
```

So it seems we need to set `salt` to `cookie-session`, `signer_kwargs` to `{"key_derivation": "hmac", "digest_method": hashlib.sha1}` and `serializer` to... what exactly? The value in class references a global variable that's fortunately just above it:
```python
session_json_serializer = TaggedJSONSerializer()
```
After looking at the imports we find one that matches this class:
```python
from .json.tag import TaggedJSONSerializer
```
So we'll need to import this class from `Flask.json.tag` too.

# Creating a cookie

Let's put it all together then:
```python
import hashlib
from itsdangerous.url_safe import URLSafeTimedSerializer
from flask.json.tag import TaggedJSONSerializer

session_json_serializer = TaggedJSONSerializer()

# stolen secret
secret =  b"\xCC" * 16

serializer = URLSafeTimedSerializer(
                        secret, 
                        salt="cookie-session", 
                        serializer=session_json_serializer, 
                        signer_kwargs=dict(
                                                         key_derivation="hmac",
                                                         digest_method=hashlib.sha1
                                                         )
                        )
```

Finally we can use our `serializer` object to generate a signed cookie value with the data we need by just passing a dictionary to its `dumps` method (like we'd do with JSON or any other serializer). We wanted to become the `admin` user, so we can find in `app.py` that the key we need to set is `user` and the value is just the username:
```python
serializer.dumps({"user": "admin"})
```

![result](https://cdn.hashnode.com/res/hashnode/image/upload/v1625438332707/LI4s-RpLb.png)

And we got a value! We could now craft a request with a proper cookie, and we would have to if we didn't find user credentials (which can also be easily found by running the hash we found for `user` in any online sha256 reverse lookup), but while it's simple, we just don't have to bother with doing it manually and can just use the devtools modern browsers give us and change the value from there:

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1625438549501/4IUofNBVR.png)

# and finally, success

After refreshing we get to our flag (redacted, you have to do at least some work yourself to get it :)

![success](https://cdn.hashnode.com/res/hashnode/image/upload/v1625438764312/-gdARzHVE.png)

