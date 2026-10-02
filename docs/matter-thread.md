# Matter over Thread with Home Assistant

This chart runs Home Assistant as a Kubernetes container, so it cannot install Home Assistant add-ons. It deploys OpenThread Border Router (OTBR) and the Matter.js Server as separate host-networked workloads instead. Home Assistant documents custom Matter Server containers as unsupported and at your own risk; Home Assistant OS with its official Matter Server app remains the supported installation.

The new Connect ZBT-2 is dedicated to Thread. The existing ZBT-2 used by Zigbee2MQTT is not changed.

## Before deploying

### 1. Verify the EG node and network interface

On a machine with `kubectl` configured:

```sh
kubectl get nodes -l floor=eg -o wide
```

Confirm the node with internal IP `192.168.5.123` is the node selected by `floor=eg`. The chart uses that label to schedule workloads that access the USB radio.

SSH to the EG Pi and find the interface that owns `192.168.5.123`:

```sh
ip -o -4 addr show | grep '192.168.5.123'
```

The example below assumes it is `eth0`. Substitute the actual interface name everywhere `eth0` appears below and set the matching Helm value during deployment.

Confirm the radio path and TUN device exist:

```sh
ls -l /dev/serial/by-id/usb-Nabu_Casa_ZBT-2_1CDBD45F0928-if00 /dev/net/tun
```

### 2. Enable host forwarding for OTBR

The OTBR container uses the Pi's host network. Thread routing requires IPv4/IPv6 forwarding and accepting IPv6 router advertisements on the infrastructure interface. Configure these settings persistently on the EG Pi:

```sh
sudo tee /etc/sysctl.d/60-otbr-accept-ra.conf >/dev/null <<'EOF'
net.ipv6.conf.eth0.accept_ra = 2
net.ipv6.conf.eth0.accept_ra_rt_info_max_plen = 64
EOF

sudo tee /etc/sysctl.d/60-otbr-ip-forward.conf >/dev/null <<'EOF'
net.ipv6.conf.all.forwarding = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

Verify the values were applied:

```sh
sysctl net.ipv6.conf.eth0.accept_ra \
  net.ipv6.conf.eth0.accept_ra_rt_info_max_plen \
  net.ipv6.conf.all.forwarding \
  net.ipv4.ip_forward
```

If `/dev/net/tun` does not exist, enable the Linux `tun` module on the Pi before deploying.

### 3. Flash the new ZBT-2 to Thread firmware

The ZBT-2 ships with Zigbee firmware. Flash the new adapter on the Pi before deploying OTBR; the existing Zigbee2MQTT adapter is not involved.

Install `universal-silabs-flasher` on the Pi (for example, install `pipx` from the OS package manager, then run `pipx install universal-silabs-flasher`). Download the stable ZBT-2 OpenThread RCP firmware and verify its checksum:

```sh
cd /tmp
curl -fL -o zbt2_openthread_rcp_2.7.2.0_GitHub-fb0446f53_gsdk_2025.6.2.gbl \
  https://github.com/NabuCasa/silabs-firmware-builder/releases/download/v2026.02.23/zbt2_openthread_rcp_2.7.2.0_GitHub-fb0446f53_gsdk_2025.6.2.gbl
echo '431500fd4399a45c8a7910262762562fa13ae02fe18b546531a3c2d14fc39d96  zbt2_openthread_rcp_2.7.2.0_GitHub-fb0446f53_gsdk_2025.6.2.gbl' | sha256sum --check -
```

Flash the adapter:

```sh
universal-silabs-flasher \
  --device /dev/serial/by-id/usb-Nabu_Casa_ZBT-2_1CDBD45F0928-if00 \
  flash \
  --profile zbt2 \
  --firmware /tmp/zbt2_openthread_rcp_2.7.2.0_GitHub-fb0446f53_gsdk_2025.6.2.gbl
```

Keep the ZBT-2 connected to the Pi after flashing.

### 4. Create persistent data directories

The Matter.js Server container runs as UID/GID `1000`; OTBR runs with the permissions it needs to configure network interfaces.

```sh
sudo install -d -o root -g root /home/he/otbr
sudo install -d -o 1000 -g 1000 /home/he/matter-server
```

Do not remove these directories after commissioning. They contain the Thread dataset and Matter fabric data.

## Deploy the chart

Upgrade the existing release, retaining its current environment-specific values. Add the real infrastructure interface if it is not `eth0`:

```sh
helm upgrade home helm/ \
  --install \
  --namespace default \
  --reuse-values \
  --set thread.infraInterface=eth0
```

If your normal upgrade command supplies values that were not saved in the release, keep those existing `--set` arguments too. Do not put secrets into this guide or the chart defaults.

Check that both workloads start:

```sh
kubectl get pods -n default -o wide
kubectl logs -n default deploy/openthread-border-router
kubectl logs -n default deploy/matter-server
```

Both workloads use host networking on the `floor=eg` node. OTBR runs privileged so it can access the host serial device and configure Thread networking. Its local web listener uses port `8082` to avoid UniFi's host-networked port `8080`; the REST API listens on the node IP at port `8081` for Home Assistant. The Matter Server WebSocket and OTBR REST API are not exposed through ingress, but host-networked ports can be reachable on the LAN; restrict access with the Pi's firewall if needed.

## Configure Home Assistant

1. In **Settings > Devices & services**, add the **OpenThread Border Router** integration. When asked for its REST API URL, enter `http://openthread-border-router:8081`.
2. Add the **Matter** integration. Choose the custom/external Matter Server option and enter `ws://matter-server:5580/ws`.
3. Open **Settings > Connectivity > Thread**. With no other Thread networks present, Home Assistant should create a new network named `ha-thread-xxxx`. Confirm it appears under **Preferred network**. If it does not, check that the OTBR integration is connected and reload the Thread page.
4. Share the Thread credentials with the phone that will commission the bulbs:
   - **Android:** In the Companion App, go to **Settings > Companion app > Troubleshooting > Sync Thread credentials**.
   - **iPhone:** In Home Assistant's **Settings > Connectivity > Thread** page, select **Send credentials to phone**.
   Keep the phone on the same LAN and allow Bluetooth/local-network access when prompted.
5. Put each Kajplats bulb into pairing/reset mode using IKEA's instructions. In Home Assistant, choose **Add Matter device**, scan the bulb's Matter QR code (or enter its setup code), and complete commissioning. Repeat for each bulb.

## Troubleshooting and recovery

- If OTBR cannot open the radio, verify the new adapter still has the Thread RCP firmware, the by-id path exists on the EG Pi, and the OTBR pod is scheduled on that same node with its privileged security context.
- If `otbr-agent` fails during Spinel `Init()`, check that the UART baud rate matches the flashed firmware. The documented ZBT-2 OpenThread RCP firmware release uses `460800` baud.
- If OTBR reports that port `8080` is already in use, check that the updated chart is deployed; its local web listener is moved to port `8082` to avoid UniFi.
- If OTBR starts but Thread devices cannot join or route traffic, recheck the infrastructure interface and the four sysctl values above.
- If Home Assistant cannot connect to either service, check that the pods are running and that the Matter Server URL and OTBR REST URL use the service names and ports shown above.
- Back up `/home/he/otbr`, `/home/he/matter-server`, and the Home Assistant configuration before changing firmware or restoring the chart.

## References

- [Home Assistant Matter integration](https://www.home-assistant.io/integrations/matter/)
- [Home Assistant Thread integration](https://www.home-assistant.io/integrations/thread/)
- [Home Assistant OpenThread Border Router integration](https://www.home-assistant.io/integrations/otbr/)
- [OpenThread Border Router Docker guide](https://openthread.io/guides/border-router/build-docker)
- [Matter.js Server Docker guide](https://github.com/matter-js/matterjs-server/blob/main/docs/docker.md)
- [Nabu Casa ZBT-2 Thread setup guide](https://support.nabucasa.com/hc/en-us/articles/31347105826077-Forming-a-new-Thread-network-with-Home-Assistant-Connect-ZBT-2)
- [Nabu Casa ZBT-2 firmware releases](https://github.com/NabuCasa/silabs-firmware-builder/releases)
