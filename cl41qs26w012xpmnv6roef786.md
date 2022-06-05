## SEETF 2022 Flag Portal Writeup (unintended solution)

> This is a writeup for [SEETF 2022](https://play.seetf.sg/) which I participated in as a member of [DistributedLivelock](https://ctftime.org/team/187094) team. You can find my other writeups for this CTF [here](https://blog.opliko.dev/series/seetf-2022)

# Introduction

Flag Portal is an interesting challange that was meant to be based on request smuggling through a reverse proxy, but due to misconfiguration of said proxy turned out to be quite a bit easier and a fixed version of the challange was released later.

#### Challenge description:
> Welcome to the flag portal! Only admins can view the flag for now. Too bad you're behind a reverse proxy ¯\(ツ)/¯

> Note: There are two flags for this challenge.

> http://flagportal.chall.seetf.sg:10001

#### Challenge author: zeyu2001

# What are we looking at

The first thing we see after visiting the flag portal is a simple website that makes a request to an api to get the number of flags and displays it as text without even any styling:


![index page](https://cdn.hashnode.com/res/hashnode/image/upload/v1654459299087/0oVMxH1n0.png align="center")

If we go into the sources we can see we have 3 services in total, and the docker-compose file clarifies that only the proxy is public, flagportal is a server that contains the first flag and has some admin key, and backend contains the second flag and presumably the same admin key:
```yaml
version: "3"
services:
  proxy:
    build: ./proxy
    restart: always
    ports:
        - 8000:8080
  flagportal:
    build: ./flagportal
    restart: always
    environment:
      - ADMIN_KEY=FAKE_KEY
      - FIRST_FLAG=SEE{FAKE_FLAG}
  backend:
    build: ./backend
    restart: always
    environment:
      - ADMIN_KEY=FAKE_KEY
      - SECOND_FLAG=SEE{FAKE_FLAG}
```

# Frontend - the flagportal

The source for flagportal is a simple ruby web server based on rack and puma, and even without knowing ruby it's quite simple to see what we have to do to get the first flag:
```ruby
elsif path == '/admin'
            params = req.params
            flagApi = params.fetch("backend", false) ? params.fetch("backend") : "http://backend/flag-plz"
            target = "https://bit.ly/3jzERNa"

            uri = URI(flagApi)
            req = Net::HTTP::Post.new(uri)
                req['Admin-Key'] = ENV.fetch("ADMIN_KEY")
                req['First-Flag'] = ENV.fetch("FIRST_FLAG")
                req.set_form_data('target' => target)

                res = Net::HTTP.start(uri.hostname, uri.port) {|http|
                http.request(req)
            }

            resp = res.body

            return [200, {"Content-Type" => "text/html"}, [resp]]
```

We can see that it makes some request to the backend that includes the first flag, the admin key we saw in the docker environment earlier and a URL for a "target". Then it returns the body of the response.

Since usually flags are ordered from the easier to the harder ones, let's just start by exfiltrating the first one: we can see that the backend URL is actually determined by a parameter in the request, and as such we can point this function to an endpoint we control and the flagportal will just happily send us the flag.

However, when we try that, the endpoint simply returns a 403 error with the following note:

![unsuccessful attempt to go to /admin](https://cdn.hashnode.com/res/hashnode/image/upload/v1654459275975/5_wv4hkJ-.png align="center")

Well then, how about the api server, perhaps there is a way through there?

# Backend

On the backend we have a simple flask app with 4 endpoints, but it's quite easy to see that `/flag-plz` (or `/api/flag-plz`) is the endpoint we were looking for, and it's the one we saw the flagserver make requests to before.

```python
@app.route('/flag-plz', methods=['POST'])
def flag():
    if request.headers.get('ADMIN_KEY') == os.environ['ADMIN_KEY']:
        if 'target' not in request.form:
            return 'Missing URL'

        requests.post(request.form['target'], data={
            'flag': os.environ['SECOND_FLAG'],
            'congrats': 'Thanks for playing!'
        })

        return 'OK, flag has been securely sent!'
            
    else:
        return 'Access denied'
```

It takes a `POST` request and needs an `ADMIN_KEY` in the headers. It also needs us to send form data with a `target` property that it will then send the flag to in the form of another `POST` request. 

But if we try to make a request there, we don't even get the  `Access denied` message, but a 405 `Method Not Allowed` error. 

![attempt at sending POST to /api/flag-plz resulting with Method Not Allowed error](https://cdn.hashnode.com/res/hashnode/image/upload/v1654459394735/PNUviBtz9.png align="center")

Huh, what if we use `GET` then:

![GET on /api/flag-plz showing Forbidden as the response](https://cdn.hashnode.com/res/hashnode/image/upload/v1654459436726/RbFgtEmZX.png align="center")

Ah, that explains it. It's blocked somewhere. To get to the bottom of this we probably need to look at the proxy server then...

# Reverse proxy - Traffic Server

Apache Traffic Server is an HTTP caching proxy server. In this challenge all it does is act as a reverse proxy and maps endpoints to specific services that aren't exposed. The `remap.config` file is quite easy to read:
```config
map /api/flag-plz   http://backend/forbidden
map /api            http://backend/
map /admin          http://flagportal/forbidden
map /               http://flagportal/
```

So we can see that the index and API are routed properly, but any attempt to get at the admin or the flag-specific API endpoint will redirect to `/forbidden`.
There is one thing that you may notice here, that's even more obvious if you remember the code for the index page that fetches the flag count:
```js
fetch('/api/flag-count').then(resp => resp.text()).then(data => document.getElementById('count').innerText = data)
```
The paths here map everything that's under them too. So `/api` really means `/api*`, while `/` is `/*`. And we can even test that it doesn't require slashes between paths:

`http://flagportal.chall.seetf.sg:10001/shoulderror` shows us the Not Found error from the flagserver
![404 on /shoulderror](https://cdn.hashnode.com/res/hashnode/image/upload/v1654459517155/7LnVQ0__i.png align="center")


But `http://flagportal.chall.seetf.sg:10001/apishoulderror` gives a different looking Not Found message from the backend.
![/apishoulderror showing a longer Not Found message](https://cdn.hashnode.com/res/hashnode/image/upload/v1654459553012/-_aGEj91Z.png align="center")

Well then, if we assume that Traffic Server is just doing a dumb comparison and doesn't get how paths work, we can try to add something that should be meaningless, like another slash, and see if it still redirects us. For example we can check if `http://flagportal.chall.seetf.sg:10001/api//flag-count` still returns our flag count:
![GET /api//flag-count returning 2 as it should](https://cdn.hashnode.com/res/hashnode/image/upload/v1654459616666/HWE6IDeeg.png align="center")

Well, look at that - we have an error message and as we checked earlier it seems to come from the flagserver and not the backend. So what if we just go to `//admin`? Will the ruby server also mess up the path?
![GET //admin returning "OK, flag has been securely sent!" message](https://cdn.hashnode.com/res/hashnode/image/upload/v1654459653928/ci_emvGEa.png align="center")

Doesn't seem like it! So now all that's left is to get the servers to send us what we need.

# Exploiting the flaw

We'll need some publicly routable IP/domain on which we can listen to requests on - witch we can easily get by either using some virtual server or something like [`ngrok`]( https://ngrok.com/) that give us a tunnel with a public endpoint so that we can expose a service on our own PC. For the CTF I used the former, but since `ngrok` has a nice request inspector built in, let's use it here. Just run `ngrok http --scheme=http 0` (use `--scheme=http` because the CTF server doesn't speak TLS and upgrades break it too :) to get the endpoint and we can use the web GUI to inspect incoming requests. 

So let's send it:

![obraz.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654458443608/zH6nwnY67.png align="center")

We get an ngrok error back, but that's just because we aren't actually hosting anything. If we now check the web interface we can see the request the server sent:

![obraz.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654458750740/BucQUg1z-.png align="center")

So now we have the first flag and the admin key that the backend server needs. Let's just craft another request using the same trick then:

![obraz.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654458885151/ROQ5JFGwk.png align="center")

And check back on ngrok:
![obraz.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1654458912728/0UB3xXH-D.png align="center")

Now just to decode the URL-encoded `%7` and `%21` to `{` and `}` respectively and we have our second flag!

> This is a writeup for [SEETF 2022](https://play.seetf.sg/) which I participated in as a member of [DistributedLivelock](https://ctftime.org/team/187094) team. You can find my other writeups for this CTF [here](https://blog.opliko.dev/series/seetf-2022)
