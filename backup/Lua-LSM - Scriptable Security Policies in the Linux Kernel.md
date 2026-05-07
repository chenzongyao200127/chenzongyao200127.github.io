LSM (Linux Security Module) has been part of the kernel since 2.6. It's the framework behind SELinux, AppArmor, and Yama — hooks scattered across critical paths like file access, credential changes, and socket creation, giving security modules a chance to say "no" before something happens.

The catch is that writing an LSM module has always meant writing C, compiling it into the kernel, and living with the fact that you can't unload it. Every step has friction.

Lua-LSM removes most of that friction. You write a Lua script that returns a table of hook callbacks, push it into `/sys/kernel/security/lua/register`, and the kernel's embedded Lua engine compiles it, validates it, and attaches it to the relevant hooks. It runs in a sandbox. If you want it gone, you echo the module name into `unregister`. The whole cycle — from idea to enforcement to removal — happens without a reboot.

## What it looks like

Here's a complete policy that limits `/etc/shadow` to three opens, then blocks all further access regardless of privilege:

```lua
local kernel = require('kernel')

local function file_post_open(file, mask)
    local path = file:path()
    if path == "/etc/shadow" then
        local n = shared.dogs:incr('count')
        if n > 3 then
            kernel.printk(string.format('shadow_protect: %s deny %s',
                          current:comm(), path))
            return false
        end
    end
end

return {
    name = "shadow_protect",
    file_post_open = file_post_open
}
```

```shell
cat shadow_protect.lua > /sys/kernel/security/lua/register
```

That's it. Root gets denied too. `echo 'shadow_protect' > …/unregister` to remove it.

## Where this actually matters: CVE-2026-31431

The abstract use case for Lua-LSM is "hot-load security logic into the kernel." The concrete one is what happens when a CVE drops and you're staring at a multi-week window before the patch reaches your fleet.

CVE-2026-31431 ("Copy Fail") is a local privilege escalation disclosed in early 2026. The bug was introduced in 2017 during a performance optimization in `algif_aead` — the kernel crypto subsystem's AEAD socket interface. Under specific conditions, `splice()` on an `AF_ALG` socket can route page-cache pages to a writable kernel destination. The result is deterministic arbitrary file writes from an unprivileged user. No race, no ASLR bypass needed. A few seconds to root.

In multi-tenant environments it's worse — the page cache is host-wide. One compromised container means every container's isolation is gone.

The patch pipeline for something like this is not fast. Backport to LTS, distro QA, staged rollout. Weeks in the best case. The PoC was circulating before patches landed.

## 79 bytes

The entire exploit chain starts with `socket(AF_ALG, …)`. If that call fails, nothing else in the exploit works. So:

```lua
return{name="cf",socket_create=function(f) if f==38 then return false end end}
```

That's 79 bytes. `AF_ALG` is address family 38. The function hooks `socket_create`, checks the family, returns false. The kernel refuses the socket. Exploit dead at step one.

```bash
echo 'return{name="cf",socket_create=function(f) if f==38 then return false end end}' \
  > /sys/kernel/security/lua/register
```

One command. No compilation, no signing, no reboot. When the real patch arrives, `echo 'cf' > …/unregister` and you're back to normal.

The overhead is one integer comparison per `socket()` call — well below the noise floor of the syscall itself.

## What this is and isn't

Lua-LSM is not a replacement for SELinux or AppArmor. It doesn't ship with a policy language, a label system, or a type enforcement engine. It's a lower-level primitive: a way to get custom logic onto LSM hooks with minimal ceremony.

The sweet spot is reactive defense — something breaks, you need a mitigation now, and you'd rather write 3 lines of Lua than mass-deploy a kernel module. It's also useful for auditing (log who touches what, with full hook context), fine-grained access control that doesn't warrant a full MAC policy, and experimentation with LSM semantics without the compile-test-reboot cycle.

## Current status

Lua-LSM is open source under the OpenAnolis community. The code lives in `security/lua/` in the ANCK (Alibaba Cloud Kernel) 6.6 tree. It's merged into the development branch but hasn't shipped in a formal release yet — you can build from source today, but production use should wait for the release.

We went public now because CVE-2026-31431 happened to be the perfect demonstration of the design philosophy: defense doesn't need to be more complex than the attack. Sometimes it just needs to be in the right place.

## Links

- Source: https://gitee.com/anolis/cloud-kernel (`security/lua/`)
- Docs: https://github.com/openanolis/lua-lsm-kernel/wiki
- Contributions welcome — whether that's code, bug reports, or just useful policy scripts.

<!-- obsidian-publish-source: System Security 系统安全团队/「内部」lua-lsm/对外宣发/外文社区/HackNews/Lua-LSM - Scriptable Security Policies in the Linux Kernel.md -->