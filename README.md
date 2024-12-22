# Introduction
This repo is a simplified setup of my current traefik and dns setup to provide internal aswell as external access to internal hosted services.

I use ovh as provider for the fictional domain `mydomain.com` in this example. But you can use any other provider you'd like. Just make sure ][traefik support](https://doc.traefik.io/traefik/https/acme/#providers) the dns challenge with this provider.

# Overview
<img src="https://github.com/cored0wn/traefik-tailscale-example/blob/main/diagram.svg?raw=true"/>

This setup requires an external reachable host (in the diagram above called VPS) and a domain provider (in the diagram above OVH).

You could replace the last requirement (dns provider) if you want to host your nameserver by yourself, but keep in mind that a self hosted dns server should be fully secured to prevent misuse and DoS attacks of your host.

### Tailscale ACL configuration
Both hosts (VPS and Homelab) need to be part of your tailscale network. The VPS must be granted access via [Tailscale ACLs](https://tailscale.com/kb/1337/acl-syntax#acls) to the ports 80, 443 (HTTP / HTTPS) and 53 (DNS).

<details>

<summary>Example yaml acl</summary>

```yaml
// Example ACLs
{
	[...]

	"hosts": {
		"vps": 		"100.72.231.100",
		"homelab":  "100.72.123.200",
	},

	// Define access control lists 
	"acls": [

		// HomeLab HTTP Access
		{
			"action": "accept",
			"src":    ["vps"],
			"dst": [
				"homelab:443",
				"homelab:80",
			],
		},

		// HomeLab DNS Access (also useful for other tailscale clients like phones etc therefore anybody can access)
		{
			"action": "accept",
			"src": [ "*" ],
			"dst": ["homelab:53"],
		},
	],

	[...]
}

```

</details>

### Tailscale DNS configuration
You have to configure your internal dns service / server as responsible dns server for your domain `mydomain.com` in your tailnet.
You should configure it as a [restricted nameserver](https://tailscale.com/kb/1054/dns#restricted-nameservers) so your configuration won't produce conflicts with other host dns configurations.

### Accept DNS configration on your VPS
Tailscale on the VPS must be configured to accept DNS (see --accept-dns flag in [cli documentation](https://tailscale.com/kb/1080/cli#set))

# Pitfalls / Headups
## Self signed certificates
If your internal service provides https only connections, you either have to create a trusted certificate (can be issued from a self hosted ca like [step-ca](https://smallstep.com/docs/step-ca/)) or you have to accept the untrusted certificate with your traefik. In the file [internal my_internal_service.yml](internal/traefik/config/my_internal_service.yml) I've added the necessary lines.

## Let's encrypt rate limits
Let's encrypt has rate limits (5 request for exact domain per 7 days) in place which can be reached if e.g. the certificate store of traefik isn't persistent.
For more information [see here](https://letsencrypt.org/docs/rate-limits/#new-certificates-per-exact-set-of-hostnames).