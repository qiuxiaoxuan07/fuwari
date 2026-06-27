<script lang="ts">
import Icon from "@iconify/svelte";
import { url } from "@utils/url-utils.ts";
import { onMount } from "svelte";

// Network Diagnostic States
let { ...props }: { [key: string]: unknown } = $props();
let ipV4 = $state("未检测");
let ipV6 = $state("未检测");
let ipV4Geo = $state("未检测");
let isTestingV4 = $state(false);
let isTestingV6 = $state(false);
let isTestingV4Geo = $state(false);

// NAT / P2P Diagnostic States
let natState = $state<"idle" | "testing" | "srflx" | "host" | "failed">("idle");
let natType = $state("未检测");
let detailedNatType = $state("未检测");

// HTTP Latency Nodes
let latencies = $state([
	{
		name: "博客本站",
		url: "/",
		ms: null as number | null,
		state: "idle" as "idle" | "testing" | "done" | "error",
	},
	{
		name: "Cloudflare CDN",
		url: "https://cloudflare.com/cdn-cgi/trace",
		ms: null as number | null,
		state: "idle" as "idle" | "testing" | "done" | "error",
	},
	{
		name: "GitHub",
		url: "https://github.com/favicon.ico",
		ms: null as number | null,
		state: "idle" as "idle" | "testing" | "done" | "error",
	},
	{
		name: "Bilibili",
		url: "https://www.bilibili.com/favicon.ico",
		ms: null as number | null,
		state: "idle" as "idle" | "testing" | "done" | "error",
	},
]);

// Custom fetch helper with abort controller for timeout
async function fetchWithTimeout(
	url: string,
	options: RequestInit = {},
	timeout = 3000,
) {
	const controller = new AbortController();
	const id = setTimeout(() => controller.abort(), timeout);
	try {
		const response = await fetch(url, {
			...options,
			signal: controller.signal,
		});
		return response;
	} finally {
		clearTimeout(id);
	}
}

async function detectIPv4() {
	isTestingV4 = true;
	isTestingV4Geo = true;
	ipV4 = "获取中...";
	ipV4Geo = "获取中...";
	try {
		// Try API 1: myip.ipip.net/json (CORS-friendly, domestic direct connection)
		try {
			const res = await fetchWithTimeout(
				"https://myip.ipip.net/json",
				{},
				2500,
			);
			const data = await res.json();
			if (data.ret === "ok" && data.data && data.data.ip && !data.data.ip.includes(":")) {
				ipV4 = data.data.ip;
				const location = data.data.location || [];
				// location order: [country, region, city, district, isp]
				const geoParts = location.slice(0, 4).filter(Boolean);
				const isp = location[4];
				const ispPart = isp ? `(${isp})` : "";
				ipV4Geo =
					geoParts.length > 0
						? `${geoParts.join(" ")} ${ispPart}`.trim()
						: "未知";
				return;
			}
		} catch (e) {}

		// Try API 2: api-ipv4.ip.sb (forces IPv4 and supports CORS/HTTPS)
		try {
			const res = await fetchWithTimeout(
				"https://api-ipv4.ip.sb/geoip",
				{},
				2500,
			);
			const data = await res.json();
			if (data.ip && !data.ip.includes(":")) {
				ipV4 = data.ip;
				const geoParts = [data.country, data.region, data.city].filter(Boolean);
				const ispPart =
					data.isp || data.organization
						? `(${data.isp || data.organization})`
						: "";
				ipV4Geo =
					geoParts.length > 0
						? `${geoParts.join(" ")} ${ispPart}`.trim()
						: "未知";
				return;
			}
		} catch (e) {}

		// Try API 3: ipify (IPv4 only) + freeipapi (passing IPv4)
		try {
			const ipRes = await fetchWithTimeout(
				"https://api.ipify.org?format=json",
				{},
				2500,
			);
			const ipData = await ipRes.json();
			if (ipData.ip && !ipData.ip.includes(":")) {
				ipV4 = ipData.ip;

				// Fetch location using the retrieved IPv4
				try {
					const geoRes = await fetchWithTimeout(
						`https://free.freeipapi.com/api/json/${ipV4}`,
						{},
						2500,
					);
					const geoData = await geoRes.json();
					const geoParts = [
						geoData.countryName,
						geoData.regionName,
						geoData.cityName,
					].filter(Boolean);
					const ispPart = geoData.asnOrganization
						? `(${geoData.asnOrganization})`
						: "";
					ipV4Geo =
						geoParts.length > 0
							? `${geoParts.join(" ")} ${ispPart}`.trim()
							: "未知";
				} catch (geoErr) {
					ipV4Geo = "位置获取失败";
				}
				return;
			}
		} catch (e) {}

		ipV4 = "获取失败";
		ipV4Geo = "位置获取失败";
	} finally {
		isTestingV4 = false;
		isTestingV4Geo = false;
	}
}

async function detectIPv6() {
	isTestingV6 = true;
	// Only set to loading if we haven't already got a detailed IPv6 from STUN candidate
	if (!ipV6.includes(":") || ipV6.includes("DNS限制")) {
		ipV6 = "获取中...";
	}
	try {
		// Try API 1: api6.ipify.org (standard hostname detection)
		try {
			const res = await fetchWithTimeout(
				"https://api6.ipify.org?format=json",
				{},
				2500,
			);
			const data = await res.json();
			if (data.ip && data.ip.includes(":")) {
				ipV6 = data.ip;
				return;
			}
		} catch (e) {}

		// Try API 2: Direct connection to Cloudflare IPv6 IP (verify if IPv6 routing works when DNS fails)
		try {
			await fetchWithTimeout(
				"https://[2606:4700:4700::1111]/cdn-cgi/trace",
				{ mode: "no-cors" },
				2500,
			);
			// If we got here, IPv6 routing is definitely working!
			if (!ipV6.includes(":") || ipV6.includes("DNS限制")) {
				ipV6 = "已开启 (DNS限制，无法获取具体地址)";
			}
			return;
		} catch (e) {}

		if (!ipV6.includes(":") || ipV6.includes("DNS限制")) {
			ipV6 = "未检测到 / 未启用";
		}
	} finally {
		isTestingV6 = false;
	}
}

function isPrivateIP(ip: string) {
	if (ip.endsWith(".local")) {
		return true;
	}
	if (ip.includes(":")) {
		// IPv6 private check
		return (
			ip.startsWith("fe80:") ||
			ip.startsWith("::") ||
			ip.startsWith("fc00:") ||
			ip.startsWith("fd00:")
		);
	}
	// IPv4 private check
	const parts = ip.split(".").map(Number);
	if (parts.length !== 4 || parts.some(Number.isNaN)) {
		return true; // Treat invalid/mDNS hostnames as private/local
	}
	return (
		parts[0] === 10 ||
		(parts[0] === 172 && parts[1] >= 16 && parts[1] <= 31) ||
		(parts[0] === 192 && parts[1] === 168) ||
		parts[0] === 127 ||
		(parts[0] === 169 && parts[1] === 254)
	);
}

async function testNAT() {
	natState = "testing";
	natType = "正在测试 WebRTC P2P STUN 穿透与反射能力...";
	detailedNatType = "测试中...";
	try {
		const pc = new RTCPeerConnection({
			iceServers: [
				{ urls: "stun:stun.miwifi.com:3478" },
				{ urls: "stun:stun.qhimg.com:3478" },
				{ urls: "stun:stun.l.google.com:19302" },
				{ urls: "stun:stun.twilio.com:3478" },
				// IPv6 Literals (Bypasses IPv6 DNS resolution)
				{ urls: "stun:[2001:678:b28::118]:3478" },
				{ urls: "stun:[2a03:8600::89]:3478" },
				{ urls: "stun:[2a01:4f8:242:56ca::2]:3478" },
			],
		});

		let srflxCandidates: { ip: string; port: number; rport: number }[] = [];
		let hostCandidates: string[] = [];

		pc.createDataChannel("test-p2p");
		const offer = await pc.createOffer();
		await pc.setLocalDescription(offer);

		const candidatePromise = new Promise<void>((resolve) => {
			pc.onicecandidate = (event) => {
				if (event.candidate) {
					const cand = event.candidate.candidate;
					if (cand.includes("srflx")) {
						const parts = cand.split(" ");
						const ip = parts[4];
						if (ip) {
							if (ip.includes(":")) {
								// Found public IPv6 via raw STUN IP direct connection!
								if (ipV6 === "获取中..." || ipV6 === "未检测" || ipV6 === "未检测到 / 未启用" || ipV6.includes("DNS限制")) {
									ipV6 = ip;
								}
							} else {
								// Filter out IPv6 srflx candidates to avoid NAT 4 false positives on dual-stack
								const port = Number.parseInt(parts[5], 10);
								const rportIdx = parts.indexOf("rport");
								const rport =
									rportIdx !== -1 ? Number.parseInt(parts[rportIdx + 1], 10) : 0;
								srflxCandidates.push({ ip, port, rport });
							}
						}
					}
					if (cand.includes("host")) {
						const parts = cand.split(" ");
						const ip = parts[4];
						// Filter out IPv6 host candidates
						if (ip && !ip.includes(":")) {
							hostCandidates.push(ip);
						}
					}
				} else {
					resolve();
				}
			};
			setTimeout(() => resolve(), 3000);
		});

		await candidatePromise;
		pc.close();

		// Check for public IPv4 in host candidates first
		let hasPublicIPv4Host = false;
		for (const ip of hostCandidates) {
			if (!ip.includes(":") && !ip.endsWith(".local") && !isPrivateIP(ip)) {
				hasPublicIPv4Host = true;
				break;
			}
		}

		if (hasPublicIPv4Host) {
			natState = "host";
			detailedNatType = "公网 IP (No NAT)";
			natType =
				"您的设备直接处于公网，不受任何 NAT 限制影响，P2P 联机通道极度通畅。";
			return;
		}

		if (srflxCandidates.length > 0) {
			// Group mapping ports by local rport to detect Symmetric behavior
			const rports: { [key: number]: { ip: string; port: number }[] } = {};
			for (const cand of srflxCandidates) {
				if (!rports[cand.rport]) {
					rports[cand.rport] = [];
				}
				rports[cand.rport].push({ ip: cand.ip, port: cand.port });
			}

			let isSymmetric = false;
			for (const rport in rports) {
				const mappings = rports[rport];
				if (mappings.length > 1) {
					const first = mappings[0];
					const hasDifferent = mappings.some(
						(m) => m.ip !== first.ip || m.port !== first.port,
					);
					if (hasDifferent) {
						isSymmetric = true;
						break;
					}
				}
			}

			natState = "srflx";
			if (isSymmetric) {
				detailedNatType = "对称型 (Symmetric NAT / NAT 4)";
				natType =
					"您的网络环境对不同的连接使用不同的公网端口。这属于对称型 (NAT 4)。您与 NAT 1 和 NAT 2 用户可以正常打洞联机，但与 NAT 3 或同为 NAT 4 的用户无法直接打洞，此时需要通过中继 (TURN) 服务器进行连接。";
			} else {
				detailedNatType = "圆锥型 (Cone NAT / NAT 1/2/3)";
				natType =
					"您的网络环境端口映射规则固定，支持 P2P 直连与打洞。这通常对应 NAT 1 (Full Cone)、NAT 2 或 NAT 3。除了 NAT 3 用户遇到 NAT 4 (对称型) 无法直接打洞外，其余各种圆锥型 NAT 组合以及与公网的联机均可顺利连通，联机环境良好。";
			}
		} else {
			natState = "failed";
			detailedNatType = "受限 / 防火墙拦截";
			natType =
				"未能建立 STUN 通信。您的局域网防火墙或网络服务商可能彻底拦截了 UDP/WebRTC 打洞端口。";
		}
	} catch (e) {
		natState = "failed";
		detailedNatType = "不支持";
		natType = "测试出错，可能您的浏览器禁用了 WebRTC 的 ICE 候选查询逻辑。";
	}
}

async function testNodeLatency(node: (typeof latencies)[0]) {
	node.state = "testing";
	node.ms = null;
	const start = performance.now();
	try {
		await fetchWithTimeout(
			node.url,
			{
				mode: "no-cors",
				cache: "no-store",
			},
			3000,
		);
		node.ms = Math.round(performance.now() - start);
		node.state = "done";
	} catch (e) {
		node.ms = null;
		node.state = "error";
	}
}

function startAllDiagnostics() {
	detectIPv4();
	detectIPv6();
	testNAT();
	for (let node of latencies) {
		testNodeLatency(node);
	}
}

onMount(() => {
	startAllDiagnostics();
});
</script>

<div class="flex flex-col gap-6 max-w-2xl w-full mx-auto p-6 card-base backdrop-blur-md text-neutral-800 dark:text-neutral-200">
    <!-- Header with Action -->
    <div class="flex items-center justify-between border-b border-black/5 dark:border-white/5 pb-3">
        <div class="flex items-center gap-2.5">
            <Icon icon="material-symbols:network-check-rounded" class="text-2xl text-[var(--primary)]" />
            <h2 class="font-bold text-lg">网络状态与 P2P 联机打洞诊断</h2>
        </div>
        <button onclick={startAllDiagnostics} class="btn-regular rounded-xl px-4 h-9 font-bold text-xs flex items-center gap-1.5 active:scale-95 transition-all">
            <Icon icon="material-symbols:refresh-rounded" class="text-base {isTestingV4 || isTestingV6 || natState === 'testing' ? 'animate-spin' : ''}" />
            重新诊断
        </button>
    </div>

    <!-- Diagnostic Details Grid -->
    <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
        <!-- Card: IP Info -->
        <div class="bg-black/5 dark:bg-white/5 rounded-xl p-4 flex flex-col gap-3 border border-black/5 dark:border-white/5">
            <div class="flex items-center gap-2 border-b border-black/5 dark:border-white/5 pb-2">
                <Icon icon="material-symbols:vpn-lock-rounded" class="text-lg text-neutral-500 dark:text-neutral-400" />
                <span class="font-bold text-sm">地址检测</span>
            </div>
            
            <div class="flex flex-col gap-2.5 text-xs">
                <!-- IPv4 -->
                <div class="flex items-center justify-between">
                    <span class="text-neutral-500 dark:text-neutral-400 font-medium">IPv4 地址</span>
                    {#if isTestingV4}
                        <span class="text-neutral-400 animate-pulse">正在获取...</span>
                    {:else}
                        <span class="font-mono font-bold">{ipV4}</span>
                    {/if}
                </div>
                <div class="flex items-center justify-between border-b border-black/5 dark:border-white/5 pb-2">
                    <span class="text-neutral-500 dark:text-neutral-400 font-medium">IPv4 归属地/运营商</span>
                    {#if isTestingV4Geo}
                        <span class="text-neutral-400 animate-pulse">正在定位...</span>
                    {:else}
                        <span class="text-neutral-600 dark:text-neutral-300 font-bold truncate max-w-[70%]" title={ipV4Geo}>{ipV4Geo}</span>
                    {/if}
                </div>
                
                <!-- IPv6 -->
                <div class="flex items-center justify-between">
                    <span class="text-neutral-500 dark:text-neutral-400 font-medium">IPv6 支持状态</span>
                    <div class="flex items-center gap-1.5 max-w-[65%]">
                        {#if isTestingV6}
                            <span class="text-neutral-400 animate-pulse">正在获取...</span>
                        {:else if ipV6.includes(':')}
                            <span class="px-1.5 py-0.5 rounded text-[10px] bg-green-500/10 text-green-600 dark:text-green-400 font-bold border border-green-500/20 whitespace-nowrap">已开启</span>
                            <span class="font-mono font-bold truncate" title={ipV6}>{ipV6}</span>
                        {:else}
                            <span class="px-1.5 py-0.5 rounded text-[10px] bg-red-500/10 text-red-600 dark:text-red-400 font-bold border border-red-500/20 whitespace-nowrap">未启用</span>
                        {/if}
                    </div>
                </div>
            </div>
        </div>

        <!-- Card: NAT/P2P Info -->
        <div class="bg-black/5 dark:bg-white/5 rounded-xl p-4 flex flex-col gap-3 border border-black/5 dark:border-white/5">
            <div class="flex items-center gap-2 border-b border-black/5 dark:border-white/5 pb-2">
                <Icon icon="material-symbols:router-outline-rounded" class="text-lg text-neutral-500 dark:text-neutral-400" />
                <span class="font-bold text-sm">P2P 联机与打洞穿透诊断</span>
            </div>
            
            <div class="flex flex-col gap-1.5 text-xs">
                <div class="flex items-center justify-between">
                    <span class="text-neutral-500 dark:text-neutral-400 font-medium">STUN 连接测试</span>
                    {#if natState === 'testing'}
                        <span class="text-neutral-400 animate-pulse">检测中...</span>
                    {:else if natState === 'host'}
                        <span class="text-green-600 dark:text-green-400 font-bold">🟢 公网直连</span>
                    {:else if natState === 'srflx'}
                        <span class="text-green-600 dark:text-green-400 font-bold">🟢 通信正常</span>
                    {:else}
                        <span class="text-red-500 dark:text-red-400 font-bold">🔴 连接失败</span>
                    {/if}
                </div>
                <div class="flex items-center justify-between border-b border-black/5 dark:border-white/5 pb-2">
                    <span class="text-neutral-500 dark:text-neutral-400 font-medium">检测 NAT 类型</span>
                    {#if natState === 'testing'}
                        <span class="text-neutral-400 animate-pulse">分析中...</span>
                    {:else}
                        <span class="font-bold text-[var(--primary)]">{detailedNatType}</span>
                    {/if}
                </div>
                <p class="text-[10px] text-neutral-400 dark:text-neutral-500 leading-relaxed mt-1">
                    {natType}
                </p>
            </div>
        </div>
    </div>

    <!-- Section: Latency Grid -->
    <div class="flex flex-col gap-3">
        <div class="flex items-center gap-2 pl-1">
            <Icon icon="material-symbols:speed-outline" class="text-lg text-neutral-500 dark:text-neutral-400" />
            <span class="font-bold text-sm">网页级响应延迟测速 (HTTP)</span>
        </div>
        <div class="grid grid-cols-2 md:grid-cols-4 gap-3 text-xs">
            {#each latencies as node}
                <div class="bg-black/5 dark:bg-white/5 rounded-xl p-3 flex flex-col gap-2.5 border border-black/5 dark:border-white/5 hover:border-black/10 dark:hover:border-white/10 transition">
                    <span class="text-neutral-600 dark:text-neutral-300 font-bold truncate">{node.name}</span>
                    <div class="flex items-center justify-between">
                        {#if node.state === 'testing'}
                            <span class="text-[10px] text-neutral-400 animate-pulse">测速中...</span>
                        {:else if node.state === 'done'}
                            <span class="font-mono font-bold text-sm {node.ms !== null && node.ms < 80 ? 'text-green-600 dark:text-green-400' : node.ms !== null && node.ms < 200 ? 'text-yellow-600 dark:text-yellow-400' : 'text-red-500'}">
                                {node.ms} ms
                            </span>
                        {:else if node.state === 'error'}
                            <span class="text-[10px] text-red-500 font-semibold">连接超时</span>
                        {:else}
                            <span class="text-[10px] text-neutral-400">未测试</span>
                        {/if}
                    </div>
                </div>
            {/each}
        </div>
    </div>

    <!-- Related Articles -->
    <div class="border-t border-black/5 dark:border-white/5 pt-4 text-xs text-center text-neutral-500">
        诊断异常？可参考本站网络优化教程：
        <div class="flex justify-center gap-3 mt-2 font-bold">
            <a href={url('/posts/home-computer-enable-ipv6/')} class="text-[var(--primary)] hover:underline flex items-center gap-0.5">
                <Icon icon="material-symbols:article-outline" />
                家庭电脑开启 IPv6 教程
            </a>
            <span class="text-neutral-300 dark:text-neutral-700">|</span>
            <a href={url('/posts/understanding-nat-types-and-p2p-hole-punching/')} class="text-[var(--primary)] hover:underline flex items-center gap-0.5">
                <Icon icon="material-symbols:article-outline" />
                NAT 类型与 P2P 穿透详解
            </a>
        </div>
    </div>
</div>
