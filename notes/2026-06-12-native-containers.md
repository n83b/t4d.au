# Native Container CLI in macOS 26

Apple's introduction of the native `container` command-line utility completely changes local development on macOS. By leveraging the updated `Virtualization.framework`, we no longer need heavy, non-native hypervisor applications or background sync utilities to run OCI-compliant containers.

## Core Advantages

1. **Zero Electron Overhead:** The engine runs as a pure Swift daemon. No high idling memory footprints.
2. **Native Bridged Networking:** Every single container gets assigned its own IP on a private local network interface (`192.168.64.0/24`). You can route traffic directly via Safari without setting up manual port mappings.
3. **Sub-second Initialization:** Micro-VMs boot instantly because they share memory spaces efficiently with the host kernel.

## Quick Reference Setup

To spin up a highly efficient PHP environment for testing isolated scripts:

```bash
# Start the background engine
container system start

# Pull and run the official Alpine-based PHP FPM image
container run --detach --name php-local php:8.3-fpm-alpine

# You can instantly check your background execution stats by reading the raw process list:
container ps