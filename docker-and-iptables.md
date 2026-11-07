# Docker, NAT, and Forwarding Rules
## NFTables
There are 7 Linux Netfilter hooks in Linux 5.16+:
1. INGRESS
2. PREROUTING
3. INPUT
4. FORWARD
5. OUTPUT
6. POSTROUTING
7. EGRESS

Which are evaluated in exactly that order. This is fairly straightforward when you have control over exactly every single underlying Netfilter rule in your machine, but when you're using Docker, you kind of don't.

When Docker starts, it writes a bunch of Netfilter rules into both the `inet` and `nat` tables, the latter being necessary for Docker to run on the `172.17.0.0/16` subnet. As for the `inet` table, one of the features enabled by this is container port publishing, which allows external services to directly address Docker container's ports through your host machine.

The problem is that these are abstracted from any configuration files (or at least ones that I'd expect to find on a Linux machine anyway).

## NFTables
The default `/etc/nftables.conf` file comes with the command `flush ruleset` at the start. Changing your `nftables.conf` in production and executing `nft -f /etc/nftables.conf` **will** break Docker's network module. To prevent this, I ensure:
- [ ] `flush ruleset` is replaced with `destroy table TABLE_NAME` for every relevant table.
- [ ] If the damage has been done, then simply restart the Docker daemon with `systemctl restart docker`.

## Docker's SNAT/DNAT
Docker performs destination NAT (DNAT) and source NAT (SNAT) to interact through the host machine with external services or hosts. This is what makes it very smooth when you use a client like TablePlus to interact with a particular container in your host machine or in your VM.

However, once firewall rules are involved, it is important to understand the underlying `iptables` rules that Docker daemon adds upon startup. In particular, its SNAT/DNAT rules can make it a little tricky to understand how to block packets to certain container ports. For instance, if your machine listens on port `6380:6379` for a Redis service, then doing the following is insufficient to block access to that container:

> ```nft
> table inet filter {
>     ...
>     chain forward {
>         type filter hook forward priority filter;
> 
>         tcp dport 6380 drop
>     }
> }
> ```

Of course, you could just do the above at the `INGRESS` layer instead and drop every packet that's headed for `6379`, but you would then lose some legitimate use case for allowing communication between the machine and the Redis container, such as maintenance.

The main issue here is that Docker has a DNAT rule in the `PREROUTING` layer that changes the published port to the target port. Since this layer is before both `INPUT` and `FORWARD`, the destination port gets changed from `6380` to `6379`. This can actually be very convenient, since we can now just drop all packets headed for `6379` to block access to **every** Redis container that uses the (default) internal Redis port.

On the other hand, Docker also has an SNAT rule in the `OUTPUT` layer that masquerades its IP as the host machine's, which is necessary for external services to be able to respond to each container. This is less likely to cause an issue, however, as outbound firewall rules typically don't clash with this (i.e. filtering specific outbound hostnames usually involve a hostname-aware proxy).

If needs be, each container can be targeted through the virtual interface created by Docker. Creating a network using `docker network create [NAME]` and adding a container to it using `docker network add [NETWORK] [CONTAINER]` will place that container behind a renamable unique interface. 

Alternatively, we can use the default Linux connection to the container with the `veth` interfaces. However, it may be smoother to place them into Docker networks as they support custom names for the interfaces using the `--opt com.docker.network.bridge.name` flag upon network creation, which will allow for a more user-friendly firewall rule. The `veth` interfaces can also be renamed, but it is generally not managed by Docker and may cause issues.

For filtering connections to **specific** hostnames and services, it's best to use a proxy that can resolve hostnames, as Netfilter does not have access to the application layer.