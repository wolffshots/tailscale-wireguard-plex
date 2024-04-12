# tailscale-wireguard-plex

Instructions and scripts for getting Tailscale to work nicely with a Wireguard VPN on Linux and optionally getting Plex to bypass both.

The instructions are primarily aimed at my use case which is Ubuntu Server (so with `systemd`) but the basic idea and scripts should work with any Unix system that uses `iproute2` (or something similar).

The script included with this project (`split_routes`) can be used to add, delete and refresh `ip rule`s and `ip route`s even outside of this use case. You should be able to adapt it to work with things other than Plex.

## Wireguard changes
Run `sudo systemctl edit wg-quick@wg0.service` to edit the override file for the `wg-quick` service for profile `wg0.conf`.

```
# /etc/systemd/system/wg-quick@wg0.service.d/override.conf

[Unit]
Before=tailscaled.service plexmediaserver.service
```
This just tells the `wg-quick` service for profile `wg0.conf` that it should start before the `tailscaled` and `plexmediaserver` services. If you don't plan to use Plex then you can just leave out `plexmediaserver.service`.
This is important because the `wg-quick` script inserts ip rules _above_ whatever is currently in your rules. This means it takes precedence over Tailscale and our Plex rules if it starts _after_ them.

## Tailscale changes
Run `sudo systemctl edit tailscaled.service` to edit the override file for the `tailscaled` service

```
# /etc/systemd/system/tailscaled.service.d/override.conf

[Unit]
After=wg-quick@wg0.service

[Service]
ExecStartPre=-/usr/sbin/ip rule add pref 65 table 52
ExecStop=-/usr/sbin/ip rule del pref 65 table 52
```
This adds a rule to check table 52 and since it's preference if 65 it should happen before routing stuff through Wireguard. This allows traffic to check the Tailscale table by default and then route everything that doesn't match that through the Wireguard VPN.
If you're interested you can check the Tailscale table with `ip route show table 52` (while Tailscale is up). It should show the same IPs as are allocated to your devices on your Tailnet (as you should be able to see with `tailscale status`)

If you set the Tailscale device as an exit node then connecting to it should allow you to use the Wireguard VPN connection as well.

## Plex changes

You'll need to add some rules for Plex to your config and there are a couple ways of doing that. 
The first way is a little more robust in case IPs change but is has some security considerations because it allows the `plex` user to change rules and routes without a password.

### 1. Allowing the `plex` user permissions to `ip rule` and `ip route` 

1. Copy the `split_routes` script in this project to somewhere accessible to the `plex` user like `/usr/local/bin`

2. Run `sudo systemctl edit plexmediaserver.service` to edit the override file for the `plexmediaserver` service
    ```
    # /etc/systemd/system/plexmediaserver.service.d/override.conf

    [Unit]
    After=wg-quick@wg0.service tailscaled.service

    [Service]
    ExecStartPre=-/usr/local/bin/split_routes refresh plexroute 60 plex.tv app.plex.tv
    ExecStop=-/usr/local/bin/split_routes del plexroute 60 plex.tv app.plex.tv
    ```

3. Create a `sudoer` file for the `plex` user to use `sudo ip route` and `sudo ip rule` without a password for the script.
    `sudo vim /etc/sudoers.d/plex` 
    ```
    plex ALL=(ALL) NOPASSWD: /sbin/ip route add *, /sbin/ip route del *, /sbin/ip rule add to *, /sbin/ip rule del to *, /sbin/ip rule del from *
    ```

### 2. Runnning the script in a `cron` job

1. Edit the `crontab` for the `root` user with `sudo crontab -e`

    ```
    # macro                                         command
    @reboot                                         sleep 60 && /usr/local/bin/split_routes refresh plexroute 60 plex.tv app.plex.tv
    #       m       h       dom     mon     dow     command
            00      12      *       *       *       /usr/local/bin/split_routes refresh plexroute 60 plex.tv app.plex.tv
    ```

    This example runs the script to refresh the routes at noon every day and on boot. The `sleep` might not be necessary but the idea is to allow Wireguard and Tailscale a little extra time since we can't trigger the script explicitly after they start.
