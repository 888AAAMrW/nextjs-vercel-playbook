---
name: vercel-nextjs-deploy
description: Full workflow for Next.js + Vercel projects: custom domains, subdomain routing, DNS, cleanup. Use when deploying Next.js apps to Vercel with custom domains, especially for China accessibility.
---

# Next.js + Vercel Full Deployment Playbook

## Core Rules

1. **Always clean up after every change.** Deleted CSS classes → delete their keyframes. Changed imports → remove unused ones. No accumulation of cruft.
2. **Always clean old Vercel deployments.** After every `vercel --prod`, immediately remove the previous deployment. Never keep more than 1 deployment.
3. **`.vercel.app` domains are blocked on mobile networks in China.** Custom domains are mandatory for China accessibility.

---

## 1. Scaffold Cleanup

After `create-next-app`, delete unused boilerplate immediately:

```bash
rm public/*.svg   # file.svg globe.svg next.svg vercel.svg window.svg
```

Strip `globals.css` to bare minimum. Rewrite `README.md` with actual project info — never keep the CNA template text.

---

## 2. Custom Domain Setup

### Buying strategy
- `.cc` is the sweet spot: ~$5/year, no ICP required, recognized in China
- `.com` is a graveyard — virtually every decent name taken by squatters
- `.io` is $35+/year, premium feel but expensive
- One domain serves unlimited subdomains for future projects

### DNS (Alibaba Cloud / Cloudflare)
```
Type    Host    Value
CNAME   @       cname.vercel-dns.com
CNAME   hot     cname.vercel-dns.com
```

### Vercel binding
```bash
vercel domains add example.cc              # add to team
vercel domains add example.cc <project>    # bind to project
vercel domains add hot.example.cc <project>
vercel --prod --yes                        # deploy to activate
```

---

## 3. Subdomain Routing (Next.js 16 `proxy.ts`)

Next.js 16 renamed `middleware.ts` → `proxy.ts` and `export function middleware` → `export default function proxy`.

```ts
import { NextRequest, NextResponse } from "next/server";

export default function proxy(req: NextRequest) {
  const host = req.headers.get("host") || "";

  if (host.startsWith("hot.")) return NextResponse.next();

  if (host === "example.cc" || host === "www.example.cc") {
    const url = req.nextUrl.clone();
    if (url.pathname === "/") {
      url.pathname = "/nav";
      return NextResponse.rewrite(url);
    }
  }

  return NextResponse.next();
}

export const config = {
  matcher: ["/((?!api|_next|favicon.ico).*)"],
};
```

---

## 4. Page Titles (Metadata)

Root layout uses template pattern so sub-pages can override:

```tsx
// app/layout.tsx
export const metadata: Metadata = {
  title: { default: "Default Title", template: "%s — Site Name" },
};
```

Sub-pages add their own `layout.tsx`:

```tsx
// app/nav/layout.tsx
export const metadata: Metadata = { title: "Nav Title" };
```

---

## 5. Analytics

```bash
npm install @vercel/analytics @vercel/speed-insights
```

```tsx
import { Analytics } from "@vercel/analytics/react";
import { SpeedInsights } from "@vercel/speed-insights/next";

// End of body in root layout
<Analytics />
<SpeedInsights />
```

---

## 6. Deploy & Cleanup

```bash
# Commit + push + deploy in one shot
git add -A && git commit -m "msg" && git push origin master && vercel --prod --yes

# After deploy succeeds, remove the old one immediately
vercel ls                          # find old deployment URL
vercel rm <old-url> --yes          # delete it — keep only 1
```

---

## 7. Canvas Animation Performance

- **Mobile**: maximum 1500 particles. More = frozen phone.
- Use **time-based animation** (`elapsed / DURATION`), not frame-count based. Low FPS = slower but still completes.
- Add a **safety timeout** (6 seconds) that forces completion if the animation hangs.
- Minimize `createRadialGradient` calls — they're expensive on mobile GPU.

---

## 8. bfcache / Back Navigation Fix

When a user navigates away during an animation and returns via browser back/swipe, bfcache may restore a broken page state. Fix:

```tsx
// On the page that has animations
useEffect(() => {
  const onShow = (e: PageTransitionEvent) => {
    if (e.persisted) window.location.reload();
  };
  window.addEventListener("pageshow", onShow);
  return () => window.removeEventListener("pageshow", onShow);
}, []);
```

- Use `location.href` for navigation (preserves history), NOT `location.replace` (deletes history).

---

## 9. Debugging Traps

- **DNS hijacking**: `nslookup` on local machine may return fake IPs for non-existent domains (ISP interception). Verify with `dns.google/resolve?name=...` or WHOIS instead.
- **WHOIS rate limiting**: `who.is` returns 403 after a few queries. Rotate sources.
- **GitHub push TLS errors**: sporadic `SSL routines::unexpected eof` — just retry.
- **Vercel deploy `fetch failed`**: network hiccup, retry.
