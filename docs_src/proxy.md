# Using Traefik as a reverse proxy
Traefik is awesome... It's documented much better elsewhere but, from my perspective it covers off the following requirements:

* Reverse proxy capabilities (i.e. what you might have used nginx for previously). Routing public facing `http://app.mydomain.com` to the equivalent of `http://localhost:8000`
* Automatic SSL certificate generation via LetsEncrypt
* Ability to "watch" the docker daemon and, when container changes are detected, it can automatically add their routes and generate new certificates as required.

We're going to sit it on ports 80/443, generate a LetsEncrypt trusted certificate and automatically forward all traffic to https.

ssh into your CoreOS host `ssh core@172.17.0.1` and lets spin it up to illustrate some concepts quickly.

```bash
# pull the latest image
docker pull traefik

# now run it with default settings and expose the dashboard
# (temporary only, not recommended in production)
docker run -d --name traefik -p 80:80 -p 443:443 -p 8080:8080 traefik --api
```

You should now be able to navigate to http://ghost.algebraic.ninja:8080 and see a nice looking (if blank) dashboard. There's nothing in it which is rubbish as we know there's already a container running for our OpenVPN server.

We'll kill this instance and change the commandline to mount the docker unix socket from the host into the traefik container to give it access to and context of other containers.

```bash
# kill the current instance
docker rm -f traefik

# now append a volume mount
docker run -d --name traefik \
    -p 80:80 -p 443:443 -p 8080:8080 \
    -v /var/run/docker.sock:/var/run/docker.sock \
    traefik --api --docker --docker.watch \
    --docker.endpoint="unix:///var/run/docker.sock" \
    --docker.exposedbydefault \
    --loglevel=debug
```

But this still isn't enough?? Looking through the logs you'll spot the following:

```bash hl_lines="4 7"
level=debug msg="Provider connection established with docker 18.06.3-ce (API 1.38)"
level=debug msg="originLabelsmap[org.opencontainers.image.vendor:Containous org.opencontainers.image.version:v1.7.11 org.opencontainers.image.description:A modern reverse-proxy org.opencontainers.image.documentation:https://docs.traefik.io org.opencontainers.image.title:Traefik org.opencontainers.image.url:https://traefik.io]"
level=debug msg="allLabelsmap[:map[]]"
level=debug msg="Filtering container with empty frontend rule /traefik "
level=debug msg="originLabelsmap[maintainer:Kyle Manna <kyle@kylemanna.com>]"
level=debug msg="allLabelsmap[:map[]]"
level=debug msg="Filtering container with empty frontend rule /ovpn-ghost "
level=debug msg="Configuration received from provider docker: {}"
level=info msg="Server configuration reloaded on :8080"
level=info msg="Server configuration reloaded on :80"
```

What this is telling us is that traefik is detecting the containers but is ignoring them for the moment because we haven't told it how to present them externally using a defined [frontend](https://docs.traefik.io/basics/#frontends).

We need to do this via custom attributes which can be added to those containers at runtime with the following syntax (basic example):

* `traefik.frontend.rule`: tell traefik how to know to route incoming traffick to this container.
* `traefik.port`: if the rule above is matched, which port on the container to send the matched traffick to.

An example would therefore look like this:

```
- "traefik.frontend.rule=Host:ghost.algebraic.ninja"
- "traefik.port=80"
```

This tells Traefik that any incoming connections to a host of `ghost.algebraic.ninja` should be routed to port `80` on our container. Let's test this with a basic container.

```
docker run -d --name whoami --label traefik.frontend.rule=Host:ghost.algebraic.ninja containous/whoami
```

Now, with the `traefik` container your logs should show the following...

???+ info "whoami"
    ```bash hl_lines="3 7 9"
    level=debug msg="allLabelsmap[:map[]]"
    level=debug msg="Filtering container with empty frontend rule /ovpn-ghost "
    level=debug msg="originLabelsmap[traefik.frontend.rule:Host:ghost.algebraic.ninja]"
    level=debug msg="allLabelsmap[:map[traefik.frontend.rule:Host:ghost.algebraic.ninja]]"
    level=debug msg="Backend backend-whoami: no load-balancer defined, fallback to 'wrr' method"
    level=debug msg="Configuration received from provider docker: {\"backends\":{\"backend-whoami\":{\"servers\":{\"server-whoami-6a97af237a8c4b33d3b3252f98e273c3\":{\"url\":\"http://172.17.0.3:80\",\"weight\":1}},\"loadBalancer\":{\"method\":\"wrr\"}}},\"frontends\":{\"frontend-Host-ghost-algebraic-ninja-0\":{\"entryPoints\":[\"http\"],\"backend\":\"backend-whoami\",\"routes\":{\"route-frontend-Host-ghost-algebraic-ninja-0\":{\"rule\":\"Host:ghost.algebraic.ninja\"}},\"passHostHeader\":true,\"priority\":0,\"basicAuth\":null}}}"
    level=debug msg="Wiring frontend frontend-Host-ghost-algebraic-ninja-0 to entryPoint http"
    level=debug msg="Creating backend backend-whoami"
    level=debug msg="Adding TLSClientHeaders middleware for frontend frontend-Host-ghost-algebraic-ninja-0"
    level=debug msg="Creating load-balancer wrr"
    level=debug msg="Creating server server-whoami-6a97af237a8c4b33d3b3252f98e273c3 at http://172.17.0.3:80 with weight 1"
    level=debug msg="Creating route route-frontend-Host-ghost-algebraic-ninja-0 Host:ghost.algebraic.ninja"
    ```

Navigating to `ghost.algebraic.ninja` should now show you a load of useful stats about your request and how it's been perceived by the webserver. In addition, navigating to the traefik dashboard that we still have exposed (for now) at `ghost.algebraic.ninja:8080` should show the container.

## Adding HTTPS
For the moment we've purely focused on proxying requests through to the containers. Let's now leverage Traefik's ability to enforce https across all traffick.

```bash
# kill the old container
docker rm -f traefik

# start a new one
docker run -d --name traefik \
    -p 80:80 -p 443:443 -p 8080:8080 \
    -v /var/run/docker.sock:/var/run/docker.sock \
    traefik --api --docker --docker.watch \
    --docker.endpoint="unix:///var/run/docker.sock" \
    --docker.exposedbydefault \
    --loglevel=debug \
    --defaultentrypoints="https" \
    --entryPoints="Name:http Address::80 Redirect.EntryPoint:https" \
    --entryPoints="Name:https Address::443 TLS"
```

The logs will now show the following:

```bash
...
level=debug msg="Creating entry point redirect http -> https"
...
```

Navigating to `ghost.algebraic.ninja` in your browser will display a warning message as it redirects you to a https page. The exact warning will vary by browser but it will be essentially saying it can't verify the authenticity of the webpage. This is because Traefik is serving it up using the default self-signed certificate.

???+ "logs"
    ```
    level=debug msg="Serving default cert for request: \"ghost.algebraic.ninja\""
    level=debug msg="http: TLS handshake error from 82.24.111.21:53644: EOF"
    ```

You can choose to ignore the warning and proceed. Interestingly though you'll see that attempting to access the Traefik dashboard via `ghost.algebraic.ninja:8080` does not get automatically upgraded to SSL. Add a couple of labels to the command to tell traefik to use the `/traefik` address like a virtual host directory to itself.

```bash
# kill the old container
docker rm -f traefik

# start a new one
docker run -d --name traefik \
    -p 80:80 -p 443:443 -p 8080:8080 \
    -v /var/run/docker.sock:/var/run/docker.sock \
    --label "traefik.frontend.rule=Host:ghost.algebraic.ninja; PathPrefixStrip: /traefik" \
    --label "traefik.port=8080" \
    traefik --api --docker --docker.watch \
    --docker.endpoint="unix:///var/run/docker.sock" \
    --docker.exposedbydefault \
    --loglevel=debug \
    --defaultentrypoints="https" \
    --entryPoints="Name:http Address::80 Redirect.EntryPoint:https" \
    --entryPoints="Name:https Address::443 TLS"
```

Navigating to `ghost.algebraic.ninja/traefik` will now take you to the dahboard and automatically upgrade the connection to SSL.

## Adding LetsEncrypt
So far we've used the self-signed certificate from Traefik to provide SSL capabilities. Whilst this is probably fine for testing and local development with an externally recognisable domain name we should be able to use the free service from LetsEncrypt to provide a fully-trusted 3rd party certificate for our web services. More information on this can be found in the [ACME](https://docs.traefik.io/configuration/acme/) section of the Traefik docs.

For the moment we will use a basic TLS verification where the authority just verifies that we have an accessible webserver they can handshake with (challenge/response) at the given address by dns. **This does not allow wildcard certs though (which we will look at later).**

```bash
# kill the old container
docker rm -f traefik

# create a storage file for holding the certificates
sudo mkdir -p /etc/traefik
sudo chown -R core:core /etc/traefik
touch /etc/traefik/acme.json
sudo chmod 600 /etc/traefik/*

# start a new one
docker run -d --name traefik \
    -p 80:80 -p 443:443 -p 8080:8080 \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v /etc/traefik:/etc/traefik \
    --label "traefik.frontend.rule=Host:ghost.algebraic.ninja; PathPrefixStrip: /traefik" \
    --label "traefik.port=8080" \
    traefik --api --docker --docker.watch \
    --docker.endpoint="unix:///var/run/docker.sock" \
    --docker.exposedbydefault \
    --loglevel=debug \
    --defaultentrypoints="https" \
    --entryPoints="Name:http Address::80 Redirect.EntryPoint:https" \
    --entryPoints="Name:https Address::443 TLS" \
    --acme \
    --acme.storage="/etc/traefik/acme.json" \
    --acme.email="admin@ghost.algebraic.ninja" \
    --acme.domains="ghost.algebraic.ninja" \
    --acme.entrypoint="https" \
    --acme.tlschallenge \
    --acme.onhostrule
```

The logs will indicate some additional activity now as Traefik identifies the new configuration and then obtains trusted certificates from LetsEncrypt.

??? info "ACME logs"
    ```bash hl_lines="1 11 12 13 14 15 16 17 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65"
    level=debug msg="Setting Acme Certificate store from Entrypoint: https"
    level=info msg="Preparing server traefik &{Address::8080 TLS:<nil> Redirect:<nil> Auth:<nil> WhitelistSourceRange:[] WhiteList:<nil> Compress:false ProxyProtocol:<nil> ForwardedHeaders:0xc00046d080} with readTimeout=0s writeTimeout=0s idleTimeout=3m0s"
    level=debug msg="Creating entry point redirect http -> https"
    level=info msg="Preparing server http &{Address::80 TLS:<nil> Redirect:0xc000481600 Auth:<nil> WhitelistSourceRange:[] WhiteList:<nil> Compress:false ProxyProtocol:<nil> ForwardedHeaders:0xc00046ce20} with readTimeout=0s writeTimeout=0s idleTimeout=3m0s"
    level=info msg="Preparing server https &{Address::443 TLS:0xc000466fc0 Redirect:<nil> Auth:<nil> WhitelistSourceRange:[] WhiteList:<nil> Compress:false ProxyProtocol:<nil> ForwardedHeaders:0xc00046cee0} with readTimeout=0s writeTimeout=0s idleTimeout=3m0s"
    level=info msg="Starting server on :8080"
    level=info msg="Starting server on :80"
    level=info msg="Starting provider configuration.ProviderAggregator {}"
    level=info msg="Starting server on :443"
    level=info msg="Starting provider *docker.Provider {\"Watch\":true,\"Filename\":\"\",\"Constraints\":null,\"Trace\":false,\"TemplateVersion\":2,\"DebugLogGeneratedTemplate\":false,\"Endpoint\":\"unix:///var/run/docker.sock\",\"Domain\":\"\",\"TLS\":null,\"ExposedByDefault\":true,\"UseBindPortIP\":false,\"SwarmMode\":false,\"Network\":\"\",\"SwarmModeRefreshSeconds\":15}"
    level=info msg="Starting provider *acme.Provider {\"Email\":\"admin@ghost.algebraic.ninja\",\"ACMELogging\":false,\"CAServer\":\"https://acme-v02.api.letsencrypt.org/directory\",\"Storage\":\"/etc/traefik/acme.json\",\"EntryPoint\":\"https\",\"KeyType\":\"\",\"OnHostRule\":true,\"OnDemand\":false,\"DNSChallenge\":null,\"HTTPChallenge\":null,\"TLSChallenge\":{},\"Domains\":[{\"Main\":\"ghost.algebraic.ninja\",\"SANs\":null}],\"Store\":{}}"
    level=info msg="Testing certificate renew..."
    level=debug msg="Looking for provided certificate(s) to validate [\"ghost.algebraic.ninja\"]..."
    level=debug msg="Domains [\"ghost.algebraic.ninja\"] need ACME certificates generation for domains \"ghost.algebraic.ninja\"."
    level=debug msg="Loading ACME certificates [ghost.algebraic.ninja]..."
    level=info msg="The key type is empty. Use default key type 4096."
    level=debug msg="Configuration received from provider ACME: {}"
    level=debug msg="Provider connection established with docker 18.06.3-ce (API 1.38)"
    level=debug msg="originLabelsmap[org.opencontainers.image.description:A modern reverse-proxy org.opencontainers.image.documentation:https://docs.traefik.io org.opencontainers.image.title:Traefik org.opencontainers.image.url:https://traefik.io org.opencontainers.image.vendor:Containous org.opencontainers.image.version:v1.7.11 traefik.frontend.rule:Host:ghost.algebraic.ninja; PathPrefixStrip: /traefik traefik.port:8080]"
    level=debug msg="allLabelsmap[:map[traefik.frontend.rule:Host:ghost.algebraic.ninja; PathPrefixStrip: /traefik traefik.port:8080]]"
    level=debug msg="originLabelsmap[traefik.frontend.rule:Host:ghost.algebraic.ninja]"
    level=debug msg="allLabelsmap[:map[traefik.frontend.rule:Host:ghost.algebraic.ninja]]"
    level=debug msg="originLabelsmap[maintainer:Kyle Manna <kyle@kylemanna.com>]"
    level=debug msg="allLabelsmap[:map[]]"
    level=debug msg="Filtering container with empty frontend rule /ovpn-ghost "
    level=debug msg="originLabelsmap[org.opencontainers.image.documentation:https://docs.traefik.io org.opencontainers.image.title:Traefik org.opencontainers.image.url:https://traefik.io org.opencontainers.image.vendor:Containous org.opencontainers.image.version:v1.7.11 traefik.frontend.rule:Host:ghost.algebraic.ninja; PathPrefixStrip: /traefik traefik.port:8080 org.opencontainers.image.description:A modern reverse-proxy]"
    level=debug msg="allLabelsmap[:map[traefik.frontend.rule:Host:ghost.algebraic.ninja; PathPrefixStrip: /traefik traefik.port:8080]]"
    level=debug msg="originLabelsmap[traefik.frontend.rule:Host:ghost.algebraic.ninja]"
    level=debug msg="allLabelsmap[:map[traefik.frontend.rule:Host:ghost.algebraic.ninja]]"
    level=debug msg="Backend backend-traefik: no load-balancer defined, fallback to 'wrr' method"
    level=debug msg="Backend backend-whoami: no load-balancer defined, fallback to 'wrr' method"
    level=debug msg="Configuration received from provider docker: {\"backends\":{\"backend-traefik\":{\"servers\":{\"server-traefik-1604bc8129985bbfcb9fcdde28e4d449\":{\"url\":\"http://172.17.0.4:8080\",\"weight\":1}},\"loadBalancer\":{\"method\":\"wrr\"}},\"backend-whoami\":{\"servers\":{\"server-whoami-6a97af237a8c4b33d3b3252f98e273c3\":{\"url\":\"http://172.17.0.3:80\",\"weight\":1}},\"loadBalancer\":{\"method\":\"wrr\"}}},\"frontends\":{\"frontend-Host-ghost-algebraic-ninja-1\":{\"entryPoints\":[\"https\"],\"backend\":\"backend-whoami\",\"routes\":{\"route-frontend-Host-ghost-algebraic-ninja-1\":{\"rule\":\"Host:ghost.algebraic.ninja\"}},\"passHostHeader\":true,\"priority\":0,\"basicAuth\":null},\"frontend-Host-ghost-algebraic-ninja-PathPrefixStrip-traefik-0\":{\"entryPoints\":[\"https\"],\"backend\":\"backend-traefik\",\"routes\":{\"route-frontend-Host-ghost-algebraic-ninja-PathPrefixStrip-traefik-0\":{\"rule\":\"Host:ghost.algebraic.ninja; PathPrefixStrip: /traefik\"}},\"passHostHeader\":true,\"priority\":0,\"basicAuth\":null}}}"
    level=info msg="Server configuration reloaded on :80"
    level=info msg="Server configuration reloaded on :443"
    level=info msg="Server configuration reloaded on :8080"
    level=debug msg="Wiring frontend frontend-Host-ghost-algebraic-ninja-1 to entryPoint https"
    level=debug msg="Creating backend backend-whoami"
    level=debug msg="Adding TLSClientHeaders middleware for frontend frontend-Host-ghost-algebraic-ninja-1"
    level=debug msg="Creating load-balancer wrr"
    level=debug msg="Creating server server-whoami-6a97af237a8c4b33d3b3252f98e273c3 at http://172.17.0.3:80 with weight 1"
    level=debug msg="Creating route route-frontend-Host-ghost-algebraic-ninja-1 Host:ghost.algebraic.ninja"
    level=debug msg="Wiring frontend frontend-Host-ghost-algebraic-ninja-PathPrefixStrip-traefik-0 to entryPoint https"
    level=debug msg="Creating backend backend-traefik"
    level=debug msg="Adding TLSClientHeaders middleware for frontend frontend-Host-ghost-algebraic-ninja-PathPrefixStrip-traefik-0"
    level=debug msg="Creating load-balancer wrr"
    level=debug msg="Creating server server-traefik-1604bc8129985bbfcb9fcdde28e4d449 at http://172.17.0.4:8080 with weight 1"
    level=debug msg="Creating route route-frontend-Host-ghost-algebraic-ninja-PathPrefixStrip-traefik-0 Host:ghost.algebraic.ninja; PathPrefixStrip: /traefik"
    level=info msg="Server configuration reloaded on :443"
    level=info msg="Server configuration reloaded on :8080"
    level=info msg="Server configuration reloaded on :80"
    level=debug msg="Try to challenge certificate for domain [ghost.algebraic.ninja] founded in Host rule"
    level=debug msg="Try to challenge certificate for domain [ghost.algebraic.ninja] founded in Host rule"
    level=debug msg="Looking for provided certificate(s) to validate [\"ghost.algebraic.ninja\"]..."
    level=debug msg="No ACME certificate generation required for domains [\"ghost.algebraic.ninja\"]."
    level=debug msg="Looking for provided certificate(s) to validate [\"ghost.algebraic.ninja\"]..."
    level=debug msg="No ACME certificate generation required for domains [\"ghost.algebraic.ninja\"]."
    level=debug msg="Building ACME client..."
    level=debug msg="https://acme-v02.api.letsencrypt.org/directory"
    level=info msg=Register...
    level=debug msg="Using TLS Challenge provider."
    level=debug msg="TLS Challenge Present temp certificate for ghost.algebraic.ninja"
    level=debug msg="TLS Challenge CleanUp temp certificate for ghost.algebraic.ninja"
    level=debug msg="Certificates obtained for domains [ghost.algebraic.ninja]"
    level=debug msg="Configuration received from provider ACME: {}"
    level=debug msg="Adding certificate for domain(s) ghost.algebraic.ninja"
    ```

Navigating to `ghost.algebraic.ninja/traefik` should now present itself with a green tick / trusted certificate signed by the `Lets Encrypt Authority`. As a result we can now remove the `-p 8080:8080` from our command as **it is no longer required to expose the ports directly to the outside world**.

As a result our final traefik command looks like this.

```bash
docker run -d --name traefik \
    -p 80:80 -p 443:443 \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v /etc/traefik:/etc/traefik \
    --label "traefik.frontend.rule=Host:ghost.algebraic.ninja; PathPrefixStrip: /traefik" \
    --label "traefik.port=8080" \
    traefik --api --docker --docker.watch \
    --docker.endpoint="unix:///var/run/docker.sock" \
    --docker.exposedbydefault \
    --loglevel=debug \
    --defaultentrypoints="https" \
    --entryPoints="Name:http Address::80 Redirect.EntryPoint:https" \
    --entryPoints="Name:https Address::443 TLS" \
    --acme \
    --acme.storage="/etc/traefik/acme.json" \
    --acme.email="admin@ghost.algebraic.ninja" \
    --acme.domains="ghost.algebraic.ninja" \
    --acme.entrypoint="https" \
    --acme.tlschallenge \
    --acme.onhostrule
```