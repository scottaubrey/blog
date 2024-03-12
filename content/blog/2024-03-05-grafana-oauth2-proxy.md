---
title: "Setting up grafana with oauth2-proxy authentication"
date: 2024-03-12T08:00:00
draft: false
---

It's been a while since I've posted here. Truth be told - I've partially written a few blog posts of the previous couple of years, and never finished them. Some of them were so far from completion that in the interim time period, I've lost the thread of what I was thinking at the time! I intend to remedy that by making a series of smaller posts, with little QA. Starting with this one.

I've deploy the excellent [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) Helm chart a number of times, including the [grafana helm chart](https://github.com/grafana/helm-charts/tree/main/charts/grafana) used by that one. Equally, I've deployed [oauth2-proxy](https://github.com/oauth2-proxy/oauth2-proxy) alongside ingress-nginx a number of times to give easy access to cluster services without the need to setting up a more heavy weight authenticaton regime like SAML and such. At work, this protects dev services in the cluster by granting access through[ our github org](https://oauth2-proxy.github.io/oauth2-proxy/configuration/providers/github) users.

This worked great by default for readable, cluster-controlled dashboards - everyone who visits and passes oauth2-proxy gets read access.

There are/were a number of limitations with this default config:

- read only means also not getting access to [explore the data](https://grafana.com/docs/grafana/latest/explore/) grafana is connected to.
- anything requiring debugging admin-related access required having and using a default user/password.

For our environment, we trust our devs and the cluster-driven config enough to grant admin access. In this post I'll show you what I did to evolve grafana helm chart values to first grant anonymous admin access, then data provided by oauth2-proxy to login as the actual user.

## First steps - ingress-nginx, oauth2-proxy

First step - deploy ingress-nginx. We used the helm chart, and it's fairly straight forward setup. For the purposes of this blog post, I won't detail the how's, but there are many excellent tutorials you can google. I recommend also making sure you can automate TLS certificate provisioning, either via a default wildcard, or using cert-manager, and letsencrypt. Again, plenty or resources can help you with this.

Next, deploy oauth2-proxy. There is [good doumentation available](https://oauth2-proxy.github.io/oauth2-proxy/installation) for doing so, with each [providers config detailed there too](https://oauth2-proxy.github.io/oauth2-proxy/configuration/providers/). As mentioned, we used github, but I've also used Google oauth2 provider. Both are fairly stright forward (as long as you can navigate GCE console, which I rarely do).

Once you have ingress (and optional but recommended TLS cert automation) and oauth2-provider deployed correctly. We can move onto the the grafana config.

## Configure oauth2-proxy for ingress

Following the steps in this [documentation for ingress-nginx](https://kubernetes.github.io/ingress-nginx/examples/auth/oauth-external-auth/), you can protect an ingress using non-standard annotations that ingress-nginx will read. For our example, we're definining an ingress via the grafana helm chart (via kube-prometheus-stack helm chart). The example values to pass to helm (via whatever means you're used to, we use fluxCD's `HelmRelease` objects):

```yaml
grafana:
    ingress:
        enabled: "true"
        hosts:
            - grafana.your-host.com
        annotations:
            nginx.ingress.kubernetes.io/auth-url: "https://oauth-proxy.your-host.com/oauth2/auth"
            nginx.ingress.kubernetes.io/auth-signin: "https://oauth-proxy.your-host.com/oauth2/start?rd=https%3A%2F%2F$host$request_uri"
```

While the documentation of what exactly those annotations do is lacking, My informed guess is that ingress-nginx attempts to forward the request to the auth service specified in `nginx.ingress.kubernetes.io/auth-url`. If that requests returns 200, the requests continues, if not the user request is redirected to `nginx.ingress.kubernetes.io/auth-signing`.

Once this is configured and deployed, only the users authorised by oauth2-proxy can visit the ingress routes.

However - grafana will still ask for login.

## Disable grafana auth and change the default permissions

Next step, let's configure grafana to accept all incoming anonymous requests as and Editor role, granting more access to the explore tab, but not admin by default.

To do so within the `kube-prometheus-stack` Helm chart (or the `grafana` helm chart), use the following config:

```yaml
grafana.ini:
    users:
        viewers_can_edit: true
    auth.anonymous:
        enabled: true
        org_role: Editor
```

Redeploy the helm chart, and you should not be able to log straight in with the correct role.

## Proxying information using oauth2-proxy headers

While this now protects the grafana dashboards and grants elevated access to those who visit, we can do better.

oauth2-proxy can add additional headers to pass information about the authenticated user. Using this, we can configure grafana to login as separate users.

To do so, first we configure oauth2-proxy to set additional headers with authentication details (this example is via a helm chart value):

```yaml
extraArgs:
    set-xauthrequest: true
```

Next, we need to enable additional config snippets with `ingress-nginx`. Set the following additional config in the ingress-nginx helm chart:

```yaml
controller:
    allowSnippetAnnotations: true
```

Then, we can reconfigure the grafana config to add additional config to the headers for the ingress:

```yaml
grafana:
    ingress:
        annotations:
            nginx.ingress.kubernetes.io/auth-url: "https://oauth-proxy.your-host.com/oauth2/auth"
            nginx.ingress.kubernetes.io/auth-signin: "https://oauth-proxy.your-host.com/oauth2/start?rd=https%3A%2F%2F$host$request_uri"
            nginx.ingress.kubernetes.io/configuration-snippet: |
                auth_request_set $user   $upstream_http_x_auth_request_user;
                auth_request_set $email  $upstream_http_x_auth_request_email;
                proxy_set_header X-User  $user;
                proxy_set_header X-Email $email;

```

With all those pieces in place, grafana will be receiving enough headers to create and login users on demand.

To configure grafana to read these and log the user request in, we can set the config thus:

```yaml
grafana.ini:
    auth:
        oauth_auto_login: true # turning on auto login
        signout_redirect_url: "https://oauth-proxy.your-host.com/oauth2/sign_out" # the oauth2-proxy logout URL
    auth.proxy:
        enabled: true # enable receiving login info via headers
        header_name: X-Email # The header name for the email of the user, must match the proxy_set_header in the ingress snippet
        header_property: email # Which user value is included in this header
        headers: Name:X-User # a map of additional user headers to pull from the request. Again, must match proxy_set_header above.
        enable_login_token: false # This runs the authentication check for all routes, instead of just the login route.
    users:
        allow_sign_up: false # Turn off the ability to sign up once logged in.
        auto_assign_org: true # auto assign to an org (we only use one org)
        auto_assign_org_role: Editor # The role to grant new users by default
```

Once all that's in place, all your users should be able to login automatically, with the Editor role.

That's all for now. See you next time.
