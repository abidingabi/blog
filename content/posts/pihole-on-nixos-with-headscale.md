---
title: "Pihole on NixOS With Headscale"
date: 2023-08-20T03:21:50-04:00
---

I decided to try and set up a pihole instance, using NixOS. Why? Honestly, I’m not sure. I think I was more curious to play with NixOS’s `virtualisation.oci-containers` options more than anything else.

The setup covers every[^fn:1] device in a headscale tailnet, with the pihole only being accessible through tailscale. This is done by exposing a pihole instance in a docker container’s DNS settings over tailscale, end then configuring headscale to use that IP for DNS. A similar thing [can be done using Tailscale proper](https://tailscale.com/kb/1114/pi-hole/), and the pihole segment can largely be used without tailscale at all. Though, make sure not to expose a DNS server to the public internet, as they can be used for a DNS amplification attack.

# The Configuration

To cut to the chase, here’s the configuration, which was significantly inspired by this [NixOS Discourse thread](https://discourse.nixos.org/t/why-did-pihole-container-stop-working/19066):

```nix
{ ... }:

# tailscale needs to have --accept-dns=false set on the host
let
  tailscaleIp = "[the ip of the pihole host on tailscale]";
  ip = "127.0.0.1";
in {
  virtualisation.oci-containers.containers.pihole = {
    # 2023.05.2, linux/amd64
    image =
      "docker.io/pihole/pihole@sha256:06790d4a2409ef65bd2415e0422deb3a242119a43e33f4c9e8b97ca586017743";
    ports =
      [ "${tailscaleIp}:53:53/tcp" "${tailscaleIp}:53:53/udp" "${ip}:4002:80" ];
    volumes = [ "etc-pihole:/etc/pihole/" "etc-dnsmasq.d:/etc/dnsmasq.d/" ];
    environment = {
      WEBPASSWORD = "";
    };
  };

  services.headscale.settings.dns_config = {
    override_local_dns = true;
    nameservers = [ tailscaleIp ];

    extra_records = [{
      name = "pihole.example.com";
      type = "A";
      value = ip;
    }];
  };

  services.caddy.enable = true;
  services.caddy.extraConfig = ''
    http://pihole.example.com {
      rewrite / /admin
      reverse_proxy ${ip}:4002
    }
  '';
}
```

# Breaking it Down

`tailscaleIp` is the tailscale ip of the pihole host, while `ip` is the ip at which the pihole admin UI will be hosted behind the reverse proxy. While you could set `ip` to `tailscaleIp` and access the admin UI through `${tailscaleIp}:4002/admin`, this is a tad more convenient.

In my case, since all these services are being run on the same host, `ip` is set to be localhost. If Caddy or headscale are running on a different host as compared to pihole,  either the configuration would need to be split up, or module options would need to be added for each bit of the configuration.

## Pihole

In `nixpkgs`, `pihole` neither has a NixOS module, nor is it packaged in `nixpkgs` to begin with. However, it does have an official Docker container, so I used `virtualisation.oci-containers` to run it. This uses `podman` by default. 

The `image` attribute could be as simple as `"pihole/pihole:latest"`, but I prefer to use a hash, or at least use a specific version for reproducibility. Using the latest tag may also not update the container how you would want it to, as the NixOS configuration does not change just because a new container version is released.

The way the `ports` are bound allows any device on the tailnet to use the pihole as a DNS server, and exposes the admin UI at `ip` on port 4002 (chosen arbitrarily).

The `volumes` persistently save the needed information from pihole, and can be managed through the `podman` command.

The `WEBPASSWORD` environment variable is set to the empty string, as this allows connecting to the pihole admin UI without even presenting a password field. **This is only safe because I trust everyone on my tailnet** (i.e., me).

## Headscale

The two critical lines here are enabling `override_local_dns` and setting `nameservers` to include `tailscaleIp`. This tells headscale to both use the pihole instance we just set up, and to get the devices connected to it to use it too.

The `extra_records` block provides the DNS record to access the pihole admin UI. While this could be done as a custom record through pihole, I have other services using this approach.

## Caddy

This is not critical, but in conjunction with the `extra_records` entry in the headscale configuration, provides the nicer access point to the pihole admin UI being accessible at an actual URL. The rewrite from `/` to `/admin` is because pihole does not automatically redirect to the admin UI.

# Notes

I would only run this on a system you trust to have good uptime. If something goes wrong, DNS will stop working unless you stop accepting DNS from tailscale[^fn:1] or disconnect from it entirely.

---

The system on which `pihole` is running needs to have had `tailscale set --accept-dns=false` for DNS to work. I would have expected this to work okay because the Docker container gets its own explicit DNS settings. There was a diagnostic message in the `pihole` console about a request from a non-local network, and the `tailscaleIp` address, so maybe allowing pihole to allow requests from any network would have worked?

---

While deploying to `lily` (the VPS this was hosted on), `nixos-rebuild` would occasionally hang, as headscale would hang while restarting, and checking `journalctl -u headscale`  revealed:

```
Jun 25 07:57:00 lily headscale-start[1042]: 2023-06-25T07:57:00-04:00 ERR Error listing users error="sql: database is closed"
Jun 25 07:57:05 lily headscale-start[1042]: 2023-06-25T07:57:05-04:00 ERR error getting routes error="sql: database is closed"
```

Honestly, I do not know why that was happening, and apparently this has been happening a lot. Since blank, there have been 823,173 lines in headscale’s journalctl, and 711,678 of them are of this form. Running `pkill headscale` allowed the deploy to continue, and while that is probably bad form, it was good enough for me.

---

In the end, I realized: I don’t actually need the pihole instance, and its just another thing to potentially break :). The typical use case is adblocking on mobile, but I use uBlock Origin in Fennec. So, shortly after setting up the instance, I [reverted the commits](https://github.com/abidingabi/dotfiles/commit/55032a4234473b0d6858118e8858f89209891064) that did so (and started to write this post anyways; if you read this far, hopefully you found this interesting or useful).

[^fn:1]: devices can opt-out of accepting the DNS configuration from tailscale with `tailscale set --accept-dns=false`
