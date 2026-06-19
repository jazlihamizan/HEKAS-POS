<script lang="ts">
	import { goto } from '$app/navigation';
	import { fade, scale } from 'svelte/transition';
	import { cubicOut } from 'svelte/easing';

	// ─── State ──────────────────────────────────────────────────────────────────
	let activeCategory = $state('all');
	let search = $state('');
	let barcodeInput = $state('');
	let cart = $state<CartItem[]>([]);
	let member = $state<Member | null>(null);
	let memberSearch = $state('');
	let modal = $state<ModalState>('none');
	let payMethod = $state<PaymentMethod>('tunai');
	let cashInput = $state('');
	let globalDisc = $state(0);
	let discInput = $state('');
	let numpadTarget = $state<number | null>(null);
	let numpadValue = $state('');
	let now = $state(new Date());

	// ─── Modal focus management ─────────────────────────────────────────────
	$effect(() => {
		if (modal === 'none') return;
		// Defer to next tick so modal is rendered
		const id = requestAnimationFrame(() => {
			const modalEl = document.querySelector('[role="dialog"]');
			const focusable = modalEl?.querySelector<HTMLElement>(
				'input, button, select, textarea, [tabindex]:not([tabindex="-1"])'
			);
			focusable?.focus();
		});
		return () => cancelAnimationFrame(id);
	});

	// ─── Global keyboard handlers ──────────────────────────────────────────────
	function onKeydown(e: KeyboardEvent) {
		const target = e.target as HTMLElement;
		const isInInput = target.tagName === 'INPUT' || target.tagName === 'TEXTAREA' || target.isContentEditable;

		// Escape closes modal (always)
		if (e.key === 'Escape' && modal !== 'none') {
			modal = 'none';
			lastTriggerEl?.focus();
			return;
		}

		// '/' focuses search (like in GitHub, Notion) — only when not in input
		if (e.key === '/' && !isInInput) {
			e.preventDefault();
			searchInputEl?.focus();
			searchInputEl?.select();
			return;
		}

		// F2 opens discount modal (only when cart has items)
		if (e.key === 'F2' && cart.length > 0 && modal === 'none') {
			e.preventDefault();
			lastTriggerEl = target;
			modal = 'discount';
			return;
		}

		// F4 / Enter on Bayar button triggers payment (only when cart has items)
		if ((e.key === 'F4' || (e.key === 'Enter' && !isInInput)) && cart.length > 0 && modal === 'none') {
			e.preventDefault();
			lastTriggerEl = target;
			modal = 'payment';
			return;
		}

		// Delete clears cart (only when not in input and cart has items)
		if ((e.key === 'Delete' || (e.key === 'Backspace' && (e.ctrlKey || e.metaKey))) && cart.length > 0 && !isInInput) {
			e.preventDefault();
			if (confirm('Kosongkan keranjang?')) clearCart();
			return;
		}
	}

	// ─── Types ──────────────────────────────────────────────────────────────────
	interface Product {
		id: number;
		name: string;
		price: number;
		category: string;
		sku: string;
		barcode: string;
		stock: number;
		unit: string;
		image: string;
	}

	interface CartItem extends Product {
		qty: number;
		disc: number;
	}

	interface Member {
		id: string;
		name: string;
		phone: string;
		points: number;
		tier: 'Silver' | 'Gold' | 'Platinum';
	}

	type PaymentMethod = 'tunai' | 'qris' | 'debit';
	type ModalState = 'none' | 'payment' | 'hold' | 'receipt' | 'member' | 'discount' | 'numpad';

	// ─── Data ──────────────────────────────────────────────────────────────────
	const CATEGORIES = [
		{ id: 'all', label: 'Semua' },
		{ id: 'minuman', label: 'Minuman' },
		{ id: 'snack', label: 'Snack' },
		{ id: 'sembako', label: 'Sembako' },
		{ id: 'frozen', label: 'Frozen' },
		{ id: 'rokok', label: 'Rokok' },
		{ id: 'lainnya', label: 'Lainnya' },
	];

	const PRODUCTS: Product[] = [
		{ id: 1, name: 'Aqua 600ml', price: 4000, category: 'minuman', sku: 'MNM001', barcode: '8996001300050', stock: 144, unit: 'btl', image: '💧' },
		{ id: 2, name: 'Teh Botol Sosro 350ml', price: 5000, category: 'minuman', sku: 'MNM002', barcode: '8992388100060', stock: 72, unit: 'btl', image: '🍵' },
		{ id: 3, name: 'Pocari Sweat 330ml', price: 8500, category: 'minuman', sku: 'MNM003', barcode: '4901085088700', stock: 36, unit: 'btl', image: '🥤' },
		{ id: 4, name: 'Milo 3in1 Sachet', price: 3000, category: 'minuman', sku: 'MNM004', barcode: '4800361001204', stock: 240, unit: 'pcs', image: '🥛' },
		{ id: 5, name: 'Chitato Sapi Panggang', price: 14000, category: 'snack', sku: 'SNK001', barcode: '8992752210011', stock: 48, unit: 'pcs', image: '🥨' },
		{ id: 6, name: 'Qtela Singkong BBQ', price: 9500, category: 'snack', sku: 'SNK002', barcode: '8993022000120', stock: 60, unit: 'pcs', image: '🍿' },
		{ id: 7, name: 'Richeese Nabati', price: 5500, category: 'snack', sku: 'SNK003', barcode: '8992979200050', stock: 84, unit: 'pcs', image: '🧀' },
		{ id: 8, name: 'Oreo Vanilla 119g', price: 12500, category: 'snack', sku: 'SNK004', barcode: '7622200672070', stock: 30, unit: 'pcs', image: '🍪' },
		{ id: 9, name: 'Indomie Goreng', price: 3500, category: 'sembako', sku: 'SBK001', barcode: '8992388101038', stock: 200, unit: 'pcs', image: '🍜' },
		{ id: 21, name: 'Biskuit Roma Kelapa', price: 8500, category: 'snack', sku: 'SNK005', barcode: '8991002300011', stock: 0, unit: 'pcs', image: '🥥' },
		{ id: 10, name: 'Beras Cap Ayam 5kg', price: 68000, category: 'sembako', sku: 'SBK002', barcode: '8996001000025', stock: 20, unit: 'kg', image: '🌾' },
		{ id: 11, name: 'Minyak Goreng Tropical 1L', price: 18500, category: 'sembako', sku: 'SBK003', barcode: '8999999011111', stock: 35, unit: 'ltr', image: '🫙' },
		{ id: 12, name: 'Gula Pasir 1kg', price: 15000, category: 'sembako', sku: 'SBK004', barcode: '8999999022222', stock: 45, unit: 'kg', image: '🍬' },
		{ id: 13, name: 'Sosis So Nice 375g', price: 24000, category: 'frozen', sku: 'FRZ001', barcode: '8993351000130', stock: 24, unit: 'pcs', image: '🌭' },
		{ id: 14, name: 'Nugget So Good 500g', price: 38000, category: 'frozen', sku: 'FRZ002', barcode: '8993351000147', stock: 18, unit: 'pcs', image: '🍗' },
		{ id: 15, name: 'Sampoerna Mild 16', price: 32000, category: 'rokok', sku: 'ROK001', barcode: '8990009000010', stock: 60, unit: 'bks', image: '🚬' },
		{ id: 16, name: 'Dji Sam Soe Filter', price: 28000, category: 'rokok', sku: 'ROK002', barcode: '8990009000027', stock: 40, unit: 'bks', image: '🚬' },
		{ id: 17, name: 'Sabun Lifebuoy 90g', price: 5500, category: 'lainnya', sku: 'LNY001', barcode: '8851932091111', stock: 90, unit: 'pcs', image: '🧼' },
		{ id: 18, name: 'Shampoo Sunsilk 170ml', price: 22000, category: 'lainnya', sku: 'LNY002', barcode: '8851932092222', stock: 55, unit: 'btl', image: '🧴' },
		{ id: 19, name: 'Pasta Gigi Pepsodent', price: 12000, category: 'lainnya', sku: 'LNY003', barcode: '8851932093333', stock: 70, unit: 'pcs', image: '🪥' },
		{ id: 20, name: 'Kecap Bango 135ml', price: 9000, category: 'lainnya', sku: 'LNY004', barcode: '8887290011223', stock: 65, unit: 'btl', image: '🫙' },
	];

	const MEMBERS: Member[] = [
		{ id: 'M001', name: 'Siti Rahayu', phone: '081234567890', points: 1250, tier: 'Gold' },
		{ id: 'M002', name: 'Budi Setiawan', phone: '082345678901', points: 320, tier: 'Silver' },
		{ id: 'M003', name: 'Dewi Lestari', phone: '083456789012', points: 5800, tier: 'Platinum' },
		{ id: 'M004', name: 'Andi Rahman', phone: '085678901234', points: 90, tier: 'Silver' },
	];

	const HELD: { id: string; items: number; total: number; time: string }[] = [
		{ id: 'H-001', items: 3, total: 47500, time: '10:22' },
		{ id: 'H-002', items: 7, total: 124000, time: '10:38' },
	];

	// ─── Helpers ────────────────────────────────────────────────────────────────
	const fmt = (n: number) => 'Rp ' + n.toLocaleString('id-ID');

	const tierColor: Record<Member['tier'], { bg: string; fg: string }> = {
		Silver: { bg: '#F1F5F9', fg: '#64748B' },
		Gold: { bg: '#FEF3C7', fg: '#92400E' },
		Platinum: { bg: '#EDE9FE', fg: '#5B21B6' },
	};

	// ─── Category color palette (for product tile backgrounds) ───────────
	const categoryColor: Record<string, { from: string; to: string }> = {
		minuman: { from: '#DBEAFE', to: '#BFDBFE' },
		snack: { from: '#FED7AA', to: '#FDBA74' },
		sembako: { from: '#FEF3C7', to: '#FDE68A' },
		frozen: { from: '#CFFAFE', to: '#A5F3FC' },
		rokok: { from: '#E0E7FF', to: '#C7D2FE' },
		lainnya: { from: '#F1F5F9', to: '#E2E8F0' },
		default: { from: '#F1F5F9', to: '#E2E8F0' }
	};

	// ─── Clock ─────────────────────────────────────────────────────────────────
	$effect(() => {
		const t = setInterval(() => { now = new Date(); }, 1000);
		return () => clearInterval(t);
	});

	const timeStr = $derived(now.toLocaleTimeString('id-ID', { hour: '2-digit', minute: '2-digit', second: '2-digit' }));

	// ─── Filtering ───────────────────────────────────────────────────────────────
	const filtered = $derived(PRODUCTS.filter(
		(p) =>
			(activeCategory === 'all' || p.category === activeCategory) &&
			(p.name.toLowerCase().includes(search.toLowerCase()) || p.sku.toLowerCase().includes(search.toLowerCase()))
	));

	// ─── Cart mutations ─────────────────────────────────────────────────────────
	function addProduct(product: Product) {
		addingProductId = product.id;
		const ex = cart.find((i) => i.id === product.id);
		if (ex) {
			cart = cart.map((i) => i.id === product.id ? { ...i, qty: i.qty + 1 } : i);
		} else {
			cart = [...cart, { ...product, qty: 1, disc: 0 }];
		}
		// Audio feedback (Web Audio API - no asset required)
		try {
			audioCtx ??= new (window.AudioContext || (window as any).webkitAudioContext)();
			const osc = audioCtx.createOscillator();
			const gain = audioCtx.createGain();
			osc.connect(gain);
			gain.connect(audioCtx.destination);
			osc.frequency.value = 880;
			osc.type = 'sine';
			gain.gain.setValueAtTime(0.05, audioCtx.currentTime);
			gain.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 0.08);
			osc.start();
			osc.stop(audioCtx.currentTime + 0.08);
		} catch { /* audio not available, silent fail */ }
		// Visual loading feedback (300ms)
		setTimeout(() => { addingProductId = null; }, 300);
	}

	function setQty(id: number, qty: number) {
		if (qty <= 0) { cart = cart.filter((i) => i.id !== id); return; }
		cart = cart.map((i) => i.id === id ? { ...i, qty } : i);
	}

	function removeItem(id: number) {
		cart = cart.filter((i) => i.id !== id);
	}

	function clearCart() {
		cart = [];
		member = null;
		globalDisc = 0;
		cashInput = '';
	}

	// ─── Barcode ───────────────────────────────────────────────────────────────
	function handleBarcodeKey(e: KeyboardEvent) {
		if (e.key === 'Enter') {
			const found = PRODUCTS.find((p) => p.barcode === barcodeInput.trim());
			if (found) addProduct(found);
			barcodeInput = '';
		}
	}

	// ─── Configurable settings ──────────────────────────────────────────────────
	const ppnRate = $state(11); // editable, default 11%
	let cartDrawerOpen = $state(false); // mobile cart drawer state

	// ─── UI state (loading, focus refs) ────────────────────────────────────────
	let addingProductId = $state<number | null>(null);
	let searchInputEl: HTMLInputElement | null = $state(null);
	let lastTriggerEl: HTMLElement | null = null; // for focus restoration on modal close
	let audioCtx: AudioContext | null = null;

	// ─── Totals ─────────────────────────────────────────────────────────────────
	const subtotal = $derived(cart.reduce((s, i) => s + i.price * i.qty - i.disc, 0));
	const discAmt = $derived(Math.round((subtotal * globalDisc) / 100));
	const afterDisc = $derived(subtotal - discAmt);
	const tax = $derived(Math.round(afterDisc * (ppnRate / 100)));
	const total = $derived(afterDisc + tax);
	const change = $derived(Number(cashInput.replace(/\D/g, '')) - total);
	const totalQty = $derived(cart.reduce((s, i) => s + i.qty, 0));

	// ─── Member search ─────────────────────────────────────────────────────────
	const memberResults = $derived(
		MEMBERS.filter(
			(m) =>
				memberSearch.length > 1 &&
				(m.name.toLowerCase().includes(memberSearch.toLowerCase()) || m.phone.includes(memberSearch))
		)
	);

	// ─── Numpad ────────────────────────────────────────────────────────────────
	function numpadPress(k: string) {
		if (k === '⌫') { numpadValue = numpadValue.slice(0, -1); return; }
		if (k === 'C') { numpadValue = ''; return; }
		numpadValue = (numpadValue + k).slice(0, 9);
	}

	function numpadConfirm() {
		if (numpadTarget !== null) setQty(numpadTarget, Number(numpadValue) || 1);
		numpadTarget = null;
		numpadValue = '';
		modal = 'none';
	}

	function openNumpad(itemId: number, currentQty: number) {
		numpadTarget = itemId;
		numpadValue = String(currentQty);
		modal = 'numpad';
	}

	// ─── Nav items ──────────────────────────────────────────────────────────────
	const navItems = [
		{ label: 'POS', active: true },
		{ label: 'Order', active: false },
		{ label: 'Produk', active: false },
		{ label: 'Pelanggan', active: false },
		{ label: 'Shift', active: false },
		{ label: 'Laporan', active: false },
		{ label: 'Lainnya', active: false },
		{ label: 'Setting', active: false },
	];
</script>

<!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
<div
	class="flex h-screen overflow-hidden"
	style="font-family: 'Inter', sans-serif; background: #F0F4F8"
	onkeydown={onKeydown}
	role="application"
>
	<!-- Skip-to-content link for keyboard accessibility -->
	<a
		href="#main-content"
		class="skip-to-content"
	>
		Loncat ke konten utama
	</a>
	<!-- ── Left Rail (Cashier Cockpit) ──────────────────────────────────────── -->
	<aside
		class="w-[60px] flex flex-col items-center py-2.5 shrink-0 z-10 relative"
		style="background: #0F172A; box-shadow: 1px 0 0 0 rgba(255,255,255,0.06), 4px 0 24px rgba(0,0,0,0.15)"
	>
		<!-- Brand monogram (replaces logo to free up space) -->
		<div
			class="w-9 h-9 rounded-lg flex items-center justify-center mb-3 relative"
			style="background: linear-gradient(135deg, #2563EB 0%, #7C3AED 100%); box-shadow: 0 2px 8px rgba(37,99,235,0.4), inset 0 1px 0 rgba(255,255,255,0.2)"
		>
			<span style="color: #fff; font-size: 12; font-weight: 800; letter-spacing: -0.04em; font-family: 'Inter', system-ui">H</span>
			<!-- Live status dot (pulsing) -->
			<div
				class="absolute -bottom-0.5 -right-0.5 w-2.5 h-2.5 rounded-full"
				style="background: #10B981; box-shadow: 0 0 0 2px #0F172A, 0 0 6px rgba(16,185,129,0.6); animation: pulse 2s infinite"
			></div>
		</div>

		<!-- Section label -->
		<div class="w-7 h-px mb-2" style="background: rgba(255,255,255,0.08)"></div>

		<!-- Nav: 4 main + 2 secondary, dengan tooltip reveal on hover -->
		<nav class="flex flex-col items-center w-full px-1.5 gap-0.5" aria-label="Menu utama">
			{#each navItems as item}
				{@const isActive = item.active}
				<button
					aria-label={item.label}
					aria-current={isActive ? 'page' : undefined}
					class="rail-item relative w-10 h-10 rounded-md flex items-center justify-center transition-all"
					style="
						background: {isActive ? 'rgba(37, 99, 235, 0.2)' : 'transparent'};
						color: {isActive ? '#fff' : 'rgba(226,235,248,0.5)'};
					"
				>
					{#if item.label === 'POS'}
						<svg width="17" height="17" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width={isActive ? 2.4 : 1.7} stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
							<rect x="2" y="3" width="20" height="14" rx="2" />
							<line x1="8" y1="21" x2="16" y2="21" />
							<line x1="12" y1="17" x2="12" y2="21" />
						</svg>
					{:else if item.label === 'Order'}
						<svg width="17" height="17" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width={isActive ? 2.4 : 1.7} stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
							<path d="M14 2H6a2 2 0 00-2 2v16a2 2 0 002 2h12a2 2 0 002-2V8z" />
							<polyline points="14 2 14 8 20 8" />
							<line x1="9" y1="13" x2="15" y2="13" />
							<line x1="9" y1="17" x2="15" y2="17" />
						</svg>
					{:else if item.label === 'Produk'}
						<svg width="17" height="17" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width={isActive ? 2.4 : 1.7} stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
							<path d="M16.5 9.4l-9-5.19M21 16V8a2 2 0 00-1-1.73l-7-4a2 2 0 00-2 0l-7 4A2 2 0 003 8v8a2 2 0 001 1.73l7 4a2 2 0 002 0l7-4A2 2 0 0021 16z" />
							<polyline points="3.27 6.96 12 12.01 20.73 6.96" />
							<line x1="12" y1="22.08" x2="12" y2="12" />
						</svg>
					{:else if item.label === 'Pelanggan'}
						<svg width="17" height="17" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width={isActive ? 2.4 : 1.7} stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
							<path d="M17 21v-2a4 4 0 00-4-4H5a4 4 0 00-4 4v2" />
							<circle cx="9" cy="7" r="4" />
							<path d="M23 21v-2a4 4 0 00-3-3.87" />
							<path d="M16 3.13a4 4 0 010 7.75" />
						</svg>
					{:else if item.label === 'Shift'}
						<svg width="17" height="17" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width={isActive ? 2.4 : 1.7} stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
							<circle cx="12" cy="12" r="10" />
							<polyline points="12 6 12 12 16 14" />
						</svg>
					{:else if item.label === 'Laporan'}
						<svg width="17" height="17" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width={isActive ? 2.4 : 1.7} stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
							<line x1="18" y1="20" x2="18" y2="10" />
							<line x1="12" y1="20" x2="12" y2="4" />
							<line x1="6" y1="20" x2="6" y2="14" />
						</svg>
					{:else if item.label === 'Lainnya'}
						<svg width="17" height="17" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width={isActive ? 2.4 : 1.7} stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
							<circle cx="5" cy="12" r="1" />
							<circle cx="12" cy="12" r="1" />
							<circle cx="19" cy="12" r="1" />
						</svg>
					{:else if item.label === 'Setting'}
						<svg width="17" height="17" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width={isActive ? 2.4 : 1.7} stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
							<circle cx="12" cy="12" r="3" />
							<path d="M19.4 15a1.65 1.65 0 00.33 1.82l.06.06a2 2 0 11-2.83 2.83l-.06-.06a1.65 1.65 0 00-1.82-.33 1.65 1.65 0 00-1 1.51V21a2 2 0 11-4 0v-.09A1.65 1.65 0 009 19.4a1.65 1.65 0 00-1.82.33l-.06.06a2 2 0 11-2.83-2.83l.06-.06A1.65 1.65 0 004.68 15a1.65 1.65 0 00-1.51-1H3a2 2 0 110-4h.09A1.65 1.65 0 004.6 9a1.65 1.65 0 00-.33-1.82l-.06-.06a2 2 0 112.83-2.83l.06.06A1.65 1.65 0 009 4.68a1.65 1.65 0 001-1.51V3a2 2 0 114 0v.09a1.65 1.65 0 001 1.51 1.65 1.65 0 001.82-.33l.06-.06a2 2 0 112.83 2.83l-.06.06A1.65 1.65 0 0019.4 9a1.65 1.65 0 001 1.51H21a2 2 0 110 4h-.09a1.65 1.65 0 00-1.51 1z" />
						</svg>
					{/if}
					<!-- Active indicator (subtle left bar) -->
					{#if isActive}
						<div
							class="absolute -left-1.5 top-1/2 -translate-y-1/2 w-0.5 h-4 rounded-r"
							style="background: #60A5FA"
						></div>
					{/if}
				</button>
			{/each}
		</nav>

		<!-- Spacer -->
		<div class="flex-1"></div>

		<!-- Held transactions badge -->
		{#if HELD.length > 0}
			<button
				onclick={(e) => { lastTriggerEl = e.currentTarget; modal = 'hold'; }}
				aria-label="Lihat {HELD.length} transaksi yang ditahankan"
				class="group relative w-10 h-10 rounded-md flex items-center justify-center mb-1 transition-all"
				style="background: rgba(245,158,11,0.15); color: #F59E0B; border: 1px solid rgba(245,158,11,0.3)"
			>
				<svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
					<circle cx="12" cy="12" r="10" />
					<line x1="10" y1="15" x2="10" y2="9" />
					<line x1="14" y1="15" x2="14" y2="9" />
				</svg>
				<div class="absolute -top-1 -right-1 w-4 h-4 rounded-full flex items-center justify-center tabular-nums" style="background: #F59E0B; color: #fff; font-size: 9; font-weight: 800; box-shadow: 0 0 0 2px #0F172A">
					{HELD.length}
				</div>
			</button>
		{/if}

		<!-- Section label -->
		<div class="w-7 h-px my-2" style="background: rgba(255,255,255,0.08)"></div>

		<!-- User + Logout (icon-only) -->
		<div class="flex flex-col items-center gap-1 w-full px-1.5">
			<button
				aria-label="Ganti user kasir"
				class="rail-item w-10 h-10 rounded-md flex items-center justify-center transition-all"
				style="color: rgba(226,235,248,0.45)"
			>
				<svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
					<path d="M20 21v-2a4 4 0 00-4-4H8a4 4 0 00-4 4v2" />
					<circle cx="12" cy="7" r="4" />
				</svg>
			</button>
			<button
				onclick={() => goto('/login')}
				aria-label="Keluar dari aplikasi"
				class="rail-item w-10 h-10 rounded-md flex items-center justify-center transition-all"
				style="color: rgba(226,235,248,0.35)"
			>
				<svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
					<path d="M9 21H5a2 2 0 01-2-2V5a2 2 0 012-2h4" />
					<polyline points="16 17 21 12 16 7" />
					<line x1="21" y1="12" x2="9" y2="12" />
				</svg>
			</button>
		</div>
	</aside>

	<!-- ── Main workspace ───────────────────────────────────────────────── -->
	<div id="main-content" class="flex flex-col flex-1 min-w-0 overflow-hidden" style="background: linear-gradient(180deg, #F8FAFC 0%, #EFF1F5 100%)">
		<!-- ── Command Bar (3-zone cockpit) ──────────────────────────────── -->
		<div
			class="flex items-stretch shrink-0"
			style="background: #fff; border-bottom: 1px solid #E2E8F0; box-shadow: 0 1px 0 0 rgba(15,23,42,0.04), 0 4px 12px rgba(15,23,42,0.04); height: 64"
		>
			<!-- ZONE A: Shift context (kiri, fixed width) -->
			<div class="flex items-center gap-3 pl-5 pr-6 shrink-0" style="border-right: 1px solid #F1F5F9; min-width: 280px">
				<!-- Shift avatar -->
				<div
					class="w-10 h-10 rounded-full flex items-center justify-center text-xs font-bold shrink-0 tabular-nums"
					style="background: linear-gradient(135deg, #2563EB 0%, #1D4ED8 100%); color: #fff; font-size: 12; box-shadow: 0 2px 8px rgba(37,99,235,0.3), inset 0 1px 0 rgba(255,255,255,0.2)"
				>
					SW
				</div>
				<div class="flex flex-col min-w-0">
					<div class="flex items-center gap-1.5">
						<span style="font-size: 13.5; font-weight: 700; color: #0F172A; letter-spacing: -0.005em">Siti Wulandari</span>
						<span class="px-1.5 py-0.5 rounded" style="background: #DBEAFE; color: #1D4ED8; font-size: 9.5; font-weight: 700; letter-spacing: 0.06em">CASHIER</span>
					</div>
					<div class="flex items-center gap-1.5 tabular-nums" style="font-size: 11; color: #64748B; font-weight: 500">
						<span>Duamart Panjen</span>
						<span style="color: #CBD5E1">/</span>
						<span style="color: #1E40AF; font-weight: 600">Shift #12345</span>
					</div>
				</div>
			</div>

			<!-- ZONE B: Live metrics (tengah, fills space) -->
			<div class="flex-1 flex items-center justify-center gap-6 px-6 min-w-0">
				<!-- Sales today -->
				<div class="flex items-center gap-2.5">
					<div class="w-9 h-9 rounded-md flex items-center justify-center" style="background: #ECFDF5; color: #059669">
						<svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
							<polyline points="22 7 13.5 15.5 8.5 10.5 2 17" />
							<polyline points="16 7 22 7 22 13" />
						</svg>
					</div>
					<div class="flex flex-col">
						<span style="font-size: 9.5; color: #64748B; font-weight: 600; text-transform: uppercase; letter-spacing: 0.08em">Penjualan hari ini</span>
						<span class="tabular-nums" style="font-size: 15; font-weight: 800; color: #0F172A; letter-spacing: -0.01em">{fmt(847500)}</span>
					</div>
				</div>

				<!-- Vertical divider -->
				<div class="w-px h-9" style="background: #E2E8F0"></div>

				<!-- Transactions count -->
				<div class="flex items-center gap-2.5">
					<div class="w-9 h-9 rounded-md flex items-center justify-center" style="background: #EFF6FF; color: #2563EB">
						<svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
							<rect x="2" y="5" width="20" height="14" rx="2" />
							<line x1="2" y1="10" x2="22" y2="10" />
						</svg>
					</div>
					<div class="flex flex-col">
						<span style="font-size: 9.5; color: #64748B; font-weight: 600; text-transform: uppercase; letter-spacing: 0.08em">Transaksi</span>
						<span class="tabular-nums" style="font-size: 15; font-weight: 800; color: #0F172A; letter-spacing: -0.01em">23 <span style="font-size: 11; color: #94A3B8; font-weight: 600">/ 35 target</span></span>
					</div>
				</div>

				<!-- Vertical divider -->
				<div class="w-px h-9" style="background: #E2E8F0"></div>

				<!-- Shift timer -->
				<div class="flex items-center gap-2.5">
					<div class="w-9 h-9 rounded-md flex items-center justify-center relative" style="background: #FEF3C7; color: #D97706">
						<svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
							<circle cx="12" cy="12" r="10" />
							<polyline points="12 6 12 12 16 14" />
						</svg>
						<div class="absolute -top-0.5 -right-0.5 w-2 h-2 rounded-full" style="background: #10B981; box-shadow: 0 0 0 1.5px #FEF3C7; animation: pulse 2s infinite"></div>
					</div>
					<div class="flex flex-col">
						<span style="font-size: 9.5; color: #64748B; font-weight: 600; text-transform: uppercase; letter-spacing: 0.08em">Shift berjalan</span>
						<span class="tabular-nums" style="font-size: 15; font-weight: 800; color: #0F172A; letter-spacing: -0.01em">{timeStr}</span>
					</div>
				</div>
			</div>

			<!-- ZONE C: Hardware + actions (kanan) -->
			<div class="flex items-center gap-2 px-5 shrink-0" style="border-left: 1px solid #F1F5F9">
				<!-- Hardware status strip (compact horizontal) -->
				<div
					class="flex items-center gap-1.5 px-2 py-1.5 rounded-md mr-1"
					style="background: #F8FAFC; border: 1px solid #E2E8F0"
					title="Status perangkat keras"
					role="status"
					aria-label="Status perangkat keras"
				>
					<!-- Printer online -->
					<span class="hardware-pill inline-flex items-center gap-1 px-1.5 py-0.5 rounded" style="background: #ECFDF5; color: #059669; font-size: 10; font-weight: 600">
						<span class="w-1.5 h-1.5 rounded-full" style="background: #10B981; box-shadow: 0 0 4px rgba(16,185,129,0.6)"></span>
						PRN
					</span>
					<!-- Scanner online -->
					<span class="hardware-pill inline-flex items-center gap-1 px-1.5 py-0.5 rounded" style="background: #ECFDF5; color: #059669; font-size: 10; font-weight: 600">
						<span class="w-1.5 h-1.5 rounded-full" style="background: #10B981; box-shadow: 0 0 4px rgba(16,185,129,0.6)"></span>
						SCN
					</span>
					<!-- Drawer online -->
					<span class="hardware-pill inline-flex items-center gap-1 px-1.5 py-0.5 rounded" style="background: #ECFDF5; color: #059669; font-size: 10; font-weight: 600">
						<span class="w-1.5 h-1.5 rounded-full" style="background: #10B981; box-shadow: 0 0 4px rgba(16,185,129,0.6)"></span>
						CD
					</span>
					<!-- Customer display offline -->
					<span class="hardware-pill inline-flex items-center gap-1 px-1.5 py-0.5 rounded" style="background: #F1F5F9; color: #94A3B8; font-size: 10; font-weight: 600">
						<span class="w-1.5 h-1.5 rounded-full" style="background: #CBD5E1"></span>
						CDP
					</span>
				</div>

				<!-- Mobile/tablet cart FAB -->
				<button
					class="cart-fab items-center gap-1.5 px-2.5 py-1.5 rounded-md transition-all"
					style="background: #F1F5F9; color: #475569; font-size: 12; font-weight: 600; border: 1px solid #E2E8F0"
					aria-label="Buka keranjang belanja"
					aria-expanded={cartDrawerOpen}
					onclick={() => (cartDrawerOpen = !cartDrawerOpen)}
				>
					<svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
						<circle cx="9" cy="21" r="1" />
						<circle cx="20" cy="21" r="1" />
						<path d="M1 1h4l2.68 13.39a2 2 0 002 1.61h9.72a2 2 0 002-1.61L23 6H6" />
					</svg>
					<span class="tabular-nums">{totalQty}</span>
				</button>

				<!-- Vertical divider -->
				<div class="w-px h-7" style="background: #E2E8F0"></div>

				<!-- Help button -->
				<button
					class="w-9 h-9 rounded-md flex items-center justify-center transition-all"
					style="background: #F1F5F9; color: #64748B; border: 1px solid #E2E8F0"
					aria-label="Bantuan & pintasan keyboard"
				>
					<svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
						<circle cx="12" cy="12" r="10" />
						<path d="M9.09 9a3 3 0 015.83 1c0 2-3 3-3 3" />
						<line x1="12" y1="17" x2="12.01" y2="17" />
					</svg>
				</button>
			</div>
		</div>

		<!-- ── Content row ───────────────────────────────────────────────── -->
		<div class="flex flex-1 overflow-hidden">
			<!-- ── LEFT: product area ──────────────────────────────────────── -->
			<div class="flex flex-col flex-1 min-w-0 overflow-hidden">
				<!-- Search + Barcode row (Enterprise 2026) -->
				<div class="flex items-center gap-2 px-3 py-2.5 shrink-0" style="background: #fff; border-bottom: 1px solid var(--color-hekas-border)">
					<!-- Product search -->
					<div class="relative flex-1 group">
						<svg class="absolute left-3 top-1/2 -translate-y-1/2 transition-colors" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="#94A3B8" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" style="pointer-events: none">
							<circle cx="11" cy="11" r="8" />
							<line x1="21" y1="21" x2="16.65" y2="16.65" />
						</svg>
						<input
							bind:this={searchInputEl}
							bind:value={search}
							placeholder="Cari produk atau SKU...  (tekan / untuk fokus)"
							aria-label="Cari produk berdasarkan nama atau SKU, tekan / untuk fokus"
							class="search-input w-full pl-9 pr-20 py-2 rounded-lg border outline-none transition-all"
							style="background: var(--color-hekas-surface); border-color: var(--color-hekas-border); font-size: 13; color: var(--color-hekas-text)"
						/>
						<!-- Keyboard hint -->
						<kbd
							class="absolute right-3 top-1/2 -translate-y-1/2 hidden md:inline-flex items-center justify-center px-1.5 h-5 rounded font-mono"
							style="background: #fff; color: var(--color-hekas-text-muted); font-size: 10.5; font-weight: 600; border: 1px solid var(--color-hekas-border); line-height: 1"
						>/</kbd>
						{#if search}
							<button
								onclick={() => (search = '')}
								aria-label="Bersihkan pencarian"
								class="search-clear absolute right-9 top-1/2 -translate-y-1/2 w-5 h-5 rounded-full flex items-center justify-center transition-all"
								style="background: #CBD5E1; color: #fff"
							>
								<svg width="10" height="10" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="3" stroke-linecap="round" aria-hidden="true">
									<line x1="18" y1="6" x2="6" y2="18" />
									<line x1="6" y1="6" x2="18" y2="18" />
								</svg>
							</button>
						{/if}
					</div>

					<!-- Barcode input -->
					<div class="relative">
						<svg class="absolute left-3 top-1/2 -translate-y-1/2" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="var(--color-hekas-blue)" stroke-width="2.2" stroke-linecap="round" style="pointer-events: none" aria-hidden="true">
							<path d="M3 5v14" />
							<path d="M8 5v14" />
							<path d="M12 5v14" />
							<path d="M17 5v14" />
							<path d="M21 5v14" />
						</svg>
						<input
							bind:value={barcodeInput}
							onkeydown={handleBarcodeKey}
							placeholder="Scan barcode..."
							aria-label="Scan barcode produk"
							class="pl-9 pr-3 py-2 rounded-lg border outline-none w-44 transition-all"
							style="background: var(--color-hekas-blue-tint); border-color: var(--color-hekas-blue-pale); font-size: 13; caret-color: var(--color-hekas-blue); color: var(--color-hekas-text)"
						/>
					</div>
				</div>

				<!-- Category tabs -->
				<div class="flex items-center gap-1 px-3 py-2 overflow-x-auto shrink-0 scrollbar-thin" style="background: var(--color-hekas-surface); border-bottom: 1px solid var(--color-hekas-border)">
					{#each CATEGORIES as cat}
						{@const isActive = activeCategory === cat.id}
						<button
							onclick={() => activeCategory = cat.id}
							aria-pressed={isActive}
							class="category-tab px-3.5 py-1.5 rounded-full whitespace-nowrap transition-all shrink-0"
							style="
								background: {isActive ? 'var(--color-hekas-blue)' : '#fff'};
								color: {isActive ? '#fff' : 'var(--color-hekas-text-muted)'};
								font-size: 12.5;
								font-weight: {isActive ? 600 : 500};
								border: 1px solid {isActive ? 'var(--color-hekas-blue)' : 'var(--color-hekas-border)'};
							"
						>
							{cat.label}
						</button>
					{/each}
				</div>

				<!-- Product grid — 4 columns -->
				<div class="flex-1 overflow-y-auto p-3">
					<div class="grid" style="display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 8px; column-gap: 8px; row-gap: 8px">
						{#each filtered as product (product.id)}
							{@const inCart = cart.find((c) => c.id === product.id)}
							{@const lowStock = product.stock <= 10}
							{@const outStock = product.stock === 0}
							<button
								onclick={(e) => { lastTriggerEl = e.currentTarget; if (!outStock) addProduct(product); }}
								disabled={outStock || addingProductId === product.id}
								aria-label={`Tambah ${product.name} ${fmt(product.price)} ke keranjang, stok ${product.stock}`}
								class="product-tile group relative overflow-hidden text-left transition-all card-press rounded-xl"
								style="
									background: #fff;
									border: {inCart ? '2px solid var(--color-hekas-blue)' : '1px solid #E2E8F0'};
									box-shadow: {inCart ? '0 0 0 4px rgba(37,99,235,0.12), 0 4px 12px rgba(15,23,42,0.08)' : '0 1px 2px rgba(15,23,42,0.04)'};
									cursor: {outStock ? 'not-allowed' : 'pointer'};
									filter: {outStock ? 'grayscale(1)' : 'none'};
								"
							>
								<!-- Loading overlay -->
								{#if addingProductId === product.id}
									<div class="absolute inset-0 z-20 flex items-center justify-center" style="background: rgba(37,99,235,0.18); backdrop-filter: blur(2px)">
										<svg width="32" height="32" viewBox="0 0 24 24" fill="none" stroke="var(--color-hekas-blue)" stroke-width="2.5" stroke-linecap="round" style="animation: spin 0.6s linear infinite">
											<path d="M21 12a9 9 0 11-6.219-8.56" />
										</svg>
									</div>
								{/if}

								<!-- Image area (DOMINANT, 65% of card) -->
								<div
									class="relative flex items-center justify-center"
									style="
										height: 130;
										background: linear-gradient(135deg, {(categoryColor[product.category] || categoryColor.default).from} 0%, {(categoryColor[product.category] || categoryColor.default).to} 100%);
									"
								>
									<!-- Subtle pattern overlay -->
									<div
										class="absolute inset-0 opacity-30"
										style="background-image: radial-gradient(circle at 70% 30%, rgba(255,255,255,0.4) 0%, transparent 60%);"
									></div>

									<!-- Product emoji/image -->
									<span class="relative" style="font-size: 56px; line-height: 1; filter: drop-shadow(0 2px 4px rgba(0,0,0,0.15)); font-family: 'Apple Color Emoji', 'Segoe UI Emoji', 'Noto Color Emoji', system-ui">
										{product.image}
									</span>

									<!-- Stok badge bottom-left (overlay on image) -->
									<div
										class="absolute bottom-1.5 left-1.5 px-1.5 py-0.5 rounded-md flex items-center gap-0.5 tabular-nums"
										style="
											background: rgba(15, 23, 42, 0.75);
											color: #fff;
											font-size: 10;
											font-weight: 700;
											backdrop-filter: blur(4px);
										"
									>
										<svg width="8" height="8" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="3" stroke-linecap="round" aria-hidden="true">
											<rect x="3" y="3" width="18" height="18" rx="2" />
										</svg>
										{outStock ? '0' : product.stock}
									</div>

									<!-- Qty badge top-right (when in cart) -->
									{#if inCart && !outStock}
										<div
											class="absolute top-2 right-2 w-7 h-7 rounded-full flex items-center justify-center tabular-nums z-10"
											style="background: var(--color-hekas-blue); color: #fff; font-size: 13; font-weight: 800; box-shadow: 0 2px 8px rgba(37,99,235,0.4), 0 0 0 2px #fff"
										>
											{inCart.qty}
										</div>
									{/if}

									<!-- HABIS stamp top-right (out of stock) -->
									{#if outStock}
										<div
											class="absolute top-2 right-2 px-2 py-0.5 rounded-md z-10"
											style="background: var(--color-hekas-danger); color: #fff; font-size: 10; font-weight: 800; letter-spacing: 0.08em; box-shadow: 0 2px 8px rgba(220,38,38,0.4)"
										>
											HABIS
										</div>
									{/if}
								</div>

								<!-- Info area -->
								<div class="p-2.5">
									<div
										class="line-clamp-2 leading-tight mb-1"
										style="font-size: 13; font-weight: 600; color: {outStock ? 'var(--color-hekas-text-muted)' : '#0F172A'}; min-height: 32; letter-spacing: -0.005em; {outStock ? 'text-decoration: line-through; text-decoration-color: var(--color-hekas-danger); text-decoration-thickness: 1.5px' : ''}"
									>
										{product.name}
									</div>
									<div class="flex items-center justify-between">
										<div class="tabular-nums" style="font-size: 10.5; color: #94A3B8; font-family: 'JetBrains Mono', ui-monospace, monospace; letter-spacing: 0.02em">
											{product.sku}
										</div>
										<div
											class="tabular-nums"
											style="font-size: 14.5; font-weight: 800; color: {outStock ? 'var(--color-hekas-text-muted)' : 'var(--color-hekas-blue)'}; letter-spacing: -0.015em"
										>
											{fmt(product.price)}
										</div>
									</div>
								</div>
							</button>
						{/each}
					</div>

					{#if filtered.length === 0}
						<div class="flex flex-col items-center justify-center py-16 gap-2" style="color: #94A3B8">
							<svg width="36" height="36" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" style="opacity: 0.25">
								<path d="M16.5 9.4l-9-5.19M21 16V8a2 2 0 00-1-1.73l-7-4a2 2 0 00-2 0l-7 4A2 2 0 003 8v8a2 2 0 001 1.73l7 4a2 2 0 002 0l7-4A2 2 0 0021 16z" />
							</svg>
							<span style="font-size: 13">Produk tidak ditemukan</span>
						</div>
					{/if}
				</div>
			</div>

			<!-- ── RIGHT: Cart (Cockpit) ───────────────────────────────────── -->
			<aside
				class="cart-panel {cartDrawerOpen ? 'open' : ''} flex flex-col shrink-0 overflow-hidden"
				style="width: 340; background: #fff; box-shadow: -4px 0 24px rgba(15,23,42,0.06)"
				aria-label="Panel keranjang belanja"
			>
				<!-- Cart header: STICKY TOTAL hero -->
				<div class="px-4 pt-3 pb-2.5 shrink-0" style="background: linear-gradient(180deg, #F8FAFC 0%, #FFFFFF 100%); border-bottom: 1px solid #F1F5F9">
					<div class="flex items-baseline justify-between mb-1.5">
						<div class="flex items-center gap-2">
							<svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="#2563EB" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
								<circle cx="9" cy="21" r="1" />
								<circle cx="20" cy="21" r="1" />
								<path d="M1 1h4l2.68 13.39a2 2 0 002 1.61h9.72a2 2 0 002-1.61L23 6H6" />
							</svg>
							<span style="font-size: 11; font-weight: 800; letter-spacing: 0.1em; color: #64748B; text-transform: uppercase">Keranjang</span>
						</div>
						{#if cart.length > 0}
							<button
								onclick={clearCart}
								class="flex items-center gap-1 px-1.5 py-0.5 rounded transition-all tabular-nums"
								style="background: #FEF2F2; color: #DC2626; font-size: 10; font-weight: 700; border: 1px solid #FECACA"
								aria-label="Hapus semua item di keranjang"
							>
								<svg width="9" height="9" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="3" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
									<polyline points="1 4 1 10 7 10" />
									<path d="M3.51 15a9 9 0 102.13-9.36L1 10" />
								</svg>
								Reset
							</button>
						{/if}
					</div>

					<!-- TOTAL HERO (sticky) -->
					<div class="flex items-baseline justify-between mt-1">
						<div class="flex flex-col">
							<span style="font-size: 10; color: #94A3B8; font-weight: 600; letter-spacing: 0.08em; text-transform: uppercase">Total</span>
							<span class="tabular-nums" aria-live="polite" aria-atomic="true" style="font-size: 28; font-weight: 800; color: #0F172A; letter-spacing: -0.025em; line-height: 1.1">
								{cart.length === 0 ? 'Rp 0' : fmt(total)}
							</span>
						</div>
						<div class="flex flex-col items-end">
							<span style="font-size: 10; color: #94A3B8; font-weight: 600; letter-spacing: 0.08em; text-transform: uppercase">Item</span>
							<span class="tabular-nums" style="font-size: 22; font-weight: 800; color: #2563EB; letter-spacing: -0.02em; line-height: 1.1">
								{totalQty}
							</span>
						</div>
					</div>
				</div>

				<!-- Member section -->
				<div class="px-3 py-2.5 shrink-0" style="border-bottom: 1px solid #F1F5F9; background: #F8FAFC">
					{#if member}
						<div class="flex items-center gap-2.5">
							<div class="w-9 h-9 rounded-full flex items-center justify-center text-xs font-bold shrink-0" style="background: var(--color-hekas-blue-pale); color: var(--color-hekas-blue-deep)">
								{member.name.split(' ').map((n: string) => n[0]).join('').slice(0, 2)}
							</div>
							<div class="flex-1 min-w-0">
								<div class="truncate" style="font-size: 12.5; font-weight: 700; color: var(--color-hekas-text)">{member.name}</div>
								<div class="tabular-nums" style="font-size: 10.5; color: var(--color-hekas-text-muted)">{member.phone} · {member.points.toLocaleString('id-ID')} poin</div>
							</div>
							<span class="px-1.5 py-0.5 rounded shrink-0" style="background: {tierColor[member.tier].bg}; color: {tierColor[member.tier].fg}; font-size: 9.5; font-weight: 700; letter-spacing: 0.04em">
								{member.tier}
							</span>
							<button onclick={() => member = null} style="color: var(--color-hekas-text-muted); padding: 4px" aria-label="Hapus member">
								<svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round">
									<line x1="18" y1="6" x2="6" y2="18" />
									<line x1="6" y1="6" x2="18" y2="18" />
								</svg>
							</button>
						</div>
					{:else}
						<div class="relative">
							<svg class="absolute left-3 top-1/2 -translate-y-1/2" width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="var(--color-hekas-text-subtle)" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" style="pointer-events: none" aria-hidden="true">
								<path d="M20 21v-2a4 4 0 00-4-4H8a4 4 0 00-4 4v2" />
								<circle cx="12" cy="7" r="4" />
							</svg>
							<input
								bind:value={memberSearch}
								placeholder="Cari member / no. HP..."
								aria-label="Cari member berdasarkan nama atau nomor HP"
								class="member-search w-full pl-8 pr-9 py-1.5 rounded-md border outline-none transition-all"
								style="background: #fff; border-color: #E2E8F0; font-size: 12.5; color: var(--color-hekas-text)"
							/>
							<button onclick={(e) => { lastTriggerEl = e.currentTarget; modal = 'member'; }} class="absolute right-1.5 top-1/2 -translate-y-1/2 p-1 rounded transition-all" style="color: var(--color-hekas-blue)" aria-label="Tambah member baru">
								<svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
									<path d="M16 21v-2a4 4 0 00-4-4H5a4 4 0 00-4 4v2" />
									<circle cx="8.5" cy="7" r="4" />
									<line x1="20" y1="8" x2="20" y2="14" />
									<line x1="23" y1="11" x2="17" y2="11" />
								</svg>
							</button>
							<!-- Dropdown results -->
							{#if memberResults.length > 0 || (memberSearch.length > 1 && memberResults.length === 0)}
								<div class="absolute left-0 right-0 top-full mt-1 rounded-md overflow-hidden shadow-lg z-20" style="background: #fff; border: 1px solid #E2E8F0">
									{#if memberResults.length > 0}
										{#each memberResults as m}
											<button
												onclick={() => { member = m; memberSearch = ''; }}
												class="w-full flex items-center gap-2.5 px-3 py-2 text-left transition-colors"
												style="background: #fff"
												onmouseover={(e) => (e.currentTarget.style.background = '#F8FAFC')}
												onmouseout={(e) => (e.currentTarget.style.background = '#fff')}
											>
												<div class="w-7 h-7 rounded-full flex items-center justify-center text-xs font-bold shrink-0" style="background: var(--color-hekas-blue-pale); color: var(--color-hekas-blue-deep)">
													{m.name.split(' ').map((n: string) => n[0]).join('').slice(0, 2)}
												</div>
												<div class="flex-1 min-w-0">
													<div class="truncate" style="font-size: 12.5; font-weight: 600; color: var(--color-hekas-text)">{m.name}</div>
													<div class="tabular-nums" style="font-size: 10.5; color: var(--color-hekas-text-muted)">{m.phone}</div>
												</div>
												<span class="px-1.5 py-0.5 rounded shrink-0" style="background: {tierColor[m.tier].bg}; color: {tierColor[m.tier].fg}; font-size: 10; font-weight: 700; letter-spacing: 0.02em">
													{m.tier}
												</span>
											</button>
										{/each}
									{:else}
										<div class="flex flex-col items-center justify-center py-5 px-3 gap-1">
											<svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="#94A3B8" stroke-width="1.5" style="opacity: 0.5">
												<circle cx="11" cy="11" r="8" />
												<line x1="21" y1="21" x2="16.65" y2="16.65" />
											</svg>
											<div style="font-size: 12; color: #64748B; font-weight: 600">Member tidak ditemukan</div>
											<div style="font-size: 10; color: #94A3B8; text-align: center">Coba kata kunci lain atau tambah member baru</div>
										</div>
									{/if}
								</div>
							{/if}
						</div>
					{/if}
				</div>

				<!-- Cart items -->
				<div class="flex-1 overflow-y-auto px-2 py-1" role="region" aria-label="Daftar item di keranjang" aria-live="polite">
					{#if cart.length === 0}
						<div class="flex flex-col items-center justify-center h-full gap-2 px-4" style="color: #94A3B8">
							<svg width="56" height="56" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" style="opacity: 0.45">
								<circle cx="9" cy="21" r="1" />
								<circle cx="20" cy="21" r="1" />
								<path d="M1 1h4l2.68 13.39a2 2 0 002 1.61h9.72a2 2 0 002-1.61L23 6H6" />
							</svg>
							<span style="font-size: 14.5; font-weight: 700; color: var(--color-hekas-text); letter-spacing: -0.01em">Belum ada item</span>
							<span style="font-size: 12; color: var(--color-hekas-text-muted); text-align: center; line-height: 1.5">Tap produk di sebelah kiri,<br/>atau scan barcode</span>
							<div class="mt-1.5 px-3 py-1.5 rounded-lg inline-flex items-center gap-1.5" style="background: var(--color-hekas-blue-tint); color: var(--color-hekas-blue-deep); font-size: 11; font-weight: 600; border: 1px solid var(--color-hekas-blue-pale)">
								Tekan
								<kbd style="background: #fff; padding: 1px 6px; border-radius: 4px; border: 1px solid var(--color-hekas-blue-pale); font-family: 'JetBrains Mono', monospace; font-size: 11; font-weight: 700">/</kbd>
								untuk cari produk
							</div>
						</div>
					{:else}
						{#each cart as item, idx}
							<div class="flex items-start gap-2 py-2" style="border-bottom: {idx < cart.length - 1 ? '1px solid #E2EBF4' : 'transparent'}">
								<!-- Emoji icon -->
								<div class="w-8 h-8 rounded-lg flex items-center justify-center text-base shrink-0 mt-0.5" style="background: #F0F4F8">
									{item.image}
								</div>

								<div class="flex-1 min-w-0">
									<div class="truncate" style="font-size: 12; font-weight: 500">{item.name}</div>
									<div style="font-size: 11; color: #2563EB; font-weight: 600">{fmt(item.price)}</div>

									<!-- Qty row -->
									<div class="flex items-center gap-1 mt-1.5">
										<button
											onclick={() => setQty(item.id, item.qty - 1)}
											class="w-6 h-6 rounded-md flex items-center justify-center border"
											style="border-color: #E2EBF4"
										>
											<svg width="10" height="10" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
												<line x1="5" y1="12" x2="19" y2="12" />
											</svg>
										</button>

										<!-- Qty — tap to open numpad -->
										<button
											onclick={() => openNumpad(item.id, item.qty)}
											class="w-8 h-6 rounded-md flex items-center justify-center border"
											style="background: #F8FAFC; border-color: #E2EBF4; font-size: 12; font-weight: 700"
										>
											{item.qty}
										</button>

										<button
											onclick={() => setQty(item.id, item.qty + 1)}
											class="w-6 h-6 rounded-md flex items-center justify-center"
											style="background: #2563EB; color: #fff"
										>
											<svg width="10" height="10" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="3">
												<line x1="12" y1="5" x2="12" y2="19" />
												<line x1="5" y1="12" x2="19" y2="12" />
											</svg>
										</button>

										<span class="ml-auto" style="font-size: 12; font-weight: 700; color: #0F172A">
											{fmt(item.price * item.qty - item.disc)}
										</span>
									</div>
								</div>

								<button
									onclick={() => removeItem(item.id)}
									class="w-6 h-6 rounded-lg flex items-center justify-center shrink-0 mt-0.5"
									style="color: #94A3B8"
								>
									<svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
										<line x1="18" y1="6" x2="6" y2="18" />
										<line x1="6" y1="6" x2="18" y2="18" />
									</svg>
								</button>
							</div>
						{/each}
					{/if}
				</div>

				<!-- ═══ CHECKOUT LANE (Payment + Summary + Action) ═══════════════ -->
				{#if cart.length > 0}
					{@const qrisActive = payMethod === 'qris'}
					{@const tunaiActive = payMethod === 'tunai'}
					{@const kartuActive = payMethod === 'debit'}
					<div class="shrink-0" style="background: #0F172A; color: #fff">
						<!-- Payment Method: QRIS primary full-width, others compact -->
						<div class="px-3 pt-3 pb-2.5">
							<div class="flex items-center justify-between mb-2">
								<span style="font-size: 10; font-weight: 800; letter-spacing: 0.12em; color: rgba(226,235,248,0.6); text-transform: uppercase">Metode Pembayaran</span>
								<span class="tabular-nums" style="font-size: 10; color: rgba(226,235,248,0.4); font-weight: 600">F4</span>
							</div>
							<div class="flex gap-1.5">
								<!-- QRIS primary: 60% width -->
								<button
									onclick={() => payMethod = 'qris'}
									aria-label="Metode bayar QRIS{qrisActive ? ' (aktif)' : ''}"
									aria-pressed={qrisActive}
									class="method-primary relative overflow-hidden flex-1 flex items-center gap-2.5 py-2.5 px-3 rounded-lg transition-all"
									style="
										background: {qrisActive ? 'linear-gradient(135deg, #2563EB 0%, #1D4ED8 100%)' : 'rgba(255,255,255,0.06)'};
										border: 1.5px solid {qrisActive ? '#60A5FA' : 'rgba(255,255,255,0.1)'};
										box-shadow: {qrisActive ? '0 4px 16px rgba(37,99,235,0.4), inset 0 1px 0 rgba(255,255,255,0.2)' : 'none'};
									"
								>
									<!-- QRIS mini-grid icon -->
									<div
										class="w-8 h-8 rounded flex items-center justify-center shrink-0"
										style="background: {qrisActive ? 'rgba(255,255,255,0.2)' : 'rgba(255,255,255,0.08)'}"
									>
										<svg width="18" height="18" viewBox="0 0 24 24" fill="white" aria-hidden="true">
											<rect x="2" y="2" width="8" height="8" rx="1.5" />
											<rect x="14" y="2" width="8" height="8" rx="1.5" />
											<rect x="2" y="14" width="8" height="8" rx="1.5" />
											<rect x="15" y="15" width="3" height="3" />
											<rect x="19" y="19" width="3" height="3" />
											<rect x="15" y="19" width="3" height="3" />
										</svg>
									</div>
									<div class="flex flex-col items-start">
										<span style="font-size: 13.5; font-weight: 800; color: #fff; letter-spacing: -0.01em; line-height: 1">QRIS</span>
										<span style="font-size: 9.5; color: {qrisActive ? 'rgba(255,255,255,0.7)' : 'rgba(255,255,255,0.4)'}; font-weight: 600; letter-spacing: 0.04em">ALL E-WALLET</span>
									</div>
								</button>

								<!-- Tunai + Kartu: 40% split -->
								<div class="flex gap-1.5" style="flex: 0 0 auto">
									<button
										onclick={() => payMethod = 'tunai'}
										aria-label="Metode bayar Tunai{tunaiActive ? ' (aktif)' : ''}"
										aria-pressed={tunaiActive}
										class="method-secondary w-16 flex flex-col items-center justify-center gap-0.5 py-2.5 rounded-lg transition-all"
										style="
											background: {tunaiActive ? 'rgba(37,99,235,0.3)' : 'rgba(255,255,255,0.06)'};
											border: 1.5px solid {tunaiActive ? '#60A5FA' : 'rgba(255,255,255,0.1)'};
										"
									>
										<svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="#fff" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
											<rect x="2" y="6" width="20" height="12" rx="1" />
											<circle cx="12" cy="12" r="2.5" />
											<path d="M6 12h.01M18 12h.01" />
										</svg>
										<span style="font-size: 9.5; font-weight: 700; color: #fff; letter-spacing: 0.04em">TUNAI</span>
									</button>
									<button
										onclick={() => payMethod = 'debit'}
										aria-label="Metode bayar Kartu{kartuActive ? ' (aktif)' : ''}"
										aria-pressed={kartuActive}
										class="method-secondary w-16 flex flex-col items-center justify-center gap-0.5 py-2.5 rounded-lg transition-all"
										style="
											background: {kartuActive ? 'rgba(37,99,235,0.3)' : 'rgba(255,255,255,0.06)'};
											border: 1.5px solid {kartuActive ? '#60A5FA' : 'rgba(255,255,255,0.1)'};
										"
									>
										<svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="#fff" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
											<rect x="2" y="5" width="20" height="14" rx="2" />
											<line x1="2" y1="10" x2="22" y2="10" />
										</svg>
										<span style="font-size: 9.5; font-weight: 700; color: #fff; letter-spacing: 0.04em">KARTU</span>
									</button>
								</div>
							</div>
						</div>

						<!-- Summary breakdown (compact inline) -->
						<div class="px-3 py-2 space-y-1" style="border-top: 1px solid rgba(255,255,255,0.08)">
							<button onclick={(e) => { lastTriggerEl = e.currentTarget; modal = 'discount'; }} class="w-full flex justify-between items-center group" aria-label="Tambah atau ubah diskon transaksi">
								<span style="font-size: 11; color: rgba(226,235,248,0.6); font-weight: 500">Subtotal {cart.length} item</span>
								<span class="tabular-nums" style="font-size: 11.5; color: rgba(226,235,248,0.9); font-weight: 600">{fmt(subtotal)}</span>
							</button>
							<button onclick={(e) => { lastTriggerEl = e.currentTarget; modal = 'discount'; }} class="w-full flex justify-between items-center" aria-label="Tambah atau ubah diskon transaksi">
								<span style="font-size: 11; color: {globalDisc > 0 ? '#34D399' : 'rgba(226,235,248,0.6)'}; font-weight: 500">
									Diskon {globalDisc > 0 ? `(${globalDisc}%)` : ''}
									{#if globalDisc === 0}
										<svg class="inline ml-1" width="9" height="9" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round"><line x1="12" y1="5" x2="12" y2="19"/><line x1="5" y1="12" x2="19" y2="12"/></svg>
									{/if}
								</span>
								<span class="tabular-nums" style="font-size: 11.5; color: {globalDisc > 0 ? '#34D399' : 'rgba(226,235,248,0.4)'}; font-weight: 600">
									{globalDisc > 0 ? `- ${fmt(discAmt)}` : '—'}
								</span>
							</button>
							<div class="flex justify-between items-center">
								<span style="font-size: 11; color: rgba(226,235,248,0.6); font-weight: 500">PPN {ppnRate}%</span>
								<span class="tabular-nums" style="font-size: 11.5; color: rgba(226,235,248,0.9); font-weight: 600">{fmt(tax)}</span>
							</div>
						</div>

						<!-- BAYAR HERO BUTTON -->
						<div class="px-3 pb-3 pt-2" style="border-top: 1px solid rgba(255,255,255,0.08)">
							<button
								onclick={(e) => { lastTriggerEl = e.currentTarget; modal = 'payment'; }}
								aria-label={`Bayar sekarang total ${fmt(total)} (tekan F4)`}
								class="bayar-hero w-full flex items-center justify-between gap-2 px-4 py-3.5 rounded-lg transition-all"
								style="
									background: linear-gradient(135deg, #10B981 0%, #059669 100%);
									box-shadow: 0 4px 20px rgba(16,185,129,0.4), inset 0 1px 0 rgba(255,255,255,0.2);
								"
							>
								<div class="flex items-center gap-2">
									<svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="#fff" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
										<rect x="2" y="5" width="20" height="14" rx="2" />
										<line x1="2" y1="10" x2="22" y2="10" />
									</svg>
									<span style="font-size: 14px; font-weight: 800; color: #fff; letter-spacing: 0.02em; text-transform: uppercase">Bayar Sekarang</span>
								</div>
								<span class="tabular-nums" style="font-size: 17px; font-weight: 800; color: #fff; letter-spacing: -0.015em">
									{fmt(total)}
								</span>
							</button>

							<!-- Secondary actions: Tahan + Batal -->
							<div class="flex gap-1.5 mt-1.5">
								<button
									onclick={(e) => { lastTriggerEl = e.currentTarget; modal = 'hold'; }}
									aria-label="Tahan transaksi (F2)"
									class="flex-1 flex items-center justify-center gap-1 py-2 rounded-md transition-all"
									style="background: rgba(255,255,255,0.06); color: rgba(226,235,248,0.7); font-size: 10.5; font-weight: 600; border: 1px solid rgba(255,255,255,0.08)"
								>
									<svg width="11" height="11" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
										<circle cx="12" cy="12" r="10" />
										<line x1="10" y1="15" x2="10" y2="9" />
										<line x1="14" y1="15" x2="14" y2="9" />
									</svg>
									Tahan (F2)
								</button>
								<button
									onclick={clearCart}
									aria-label="Batalkan transaksi"
									class="flex items-center justify-center gap-1 px-3 py-2 rounded-md transition-all"
									style="background: rgba(220,38,38,0.15); color: #FCA5A5; font-size: 10.5; font-weight: 600; border: 1px solid rgba(220,38,38,0.2)"
								>
									Batal
								</button>
							</div>
						</div>
					</div>
				{:else}
					<!-- Empty cart state: subtle prompt -->
					<div class="px-4 py-3 shrink-0" style="background: #F8FAFC; border-top: 1px solid #F1F5F9">
						<div class="flex items-center gap-2" style="color: #94A3B8; font-size: 11">
							<svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
								<circle cx="12" cy="12" r="10" />
								<line x1="12" y1="8" x2="12" y2="12" />
								<line x1="12" y1="16" x2="12.01" y2="16" />
							</svg>
							<span>Pilih produk untuk memulai transaksi baru</span>
						</div>
					</div>
				{/if}
			</aside>
		</div>
	</div>

	<!-- ─────────────────────────── MODALS ─────────────────────────────────── -->

	<!-- Payment Modal -->
	{#if modal === 'payment'}
		<!-- svelte-ignore a11y_click_events_have_key_events -->
		<!-- svelte-ignore a11y_no_static_element_interactions -->
		<div
			class="fixed inset-0 flex items-center justify-center z-50 p-4"
			style="background: rgba(15,23,41,0.55); backdrop-filter: blur(2px)"
			onclick={(e) => { if (e.target === e.currentTarget) { modal = 'none'; lastTriggerEl?.focus(); } }}
			transition:fade={{ duration: 150 }}
			role="dialog"
			aria-modal="true"
			aria-labelledby="payment-title"
		>
			<div
				class="rounded-3xl shadow-2xl overflow-hidden"
				style="background: #fff; width: 420; max-width: 100%"
				transition:scale={{ duration: 200, start: 0.96, easing: cubicOut }}
			>
				<div class="flex items-center justify-between px-5 py-4" style="background: #1E3A5F">
					<span id="payment-title" style="color: #fff; font-size: 15; font-weight: 700">Pembayaran</span>
					<button onclick={() => modal = 'none'}>
						<svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="rgba(255,255,255,0.6)" stroke-width="2">
							<line x1="18" y1="6" x2="6" y2="18" />
							<line x1="6" y1="6" x2="18" y2="18" />
						</svg>
					</button>
				</div>
				<div class="p-5">
					<!-- Total -->
					<div class="text-center py-4 mb-4" style="background: #F0F7FF; border-radius: 12">
						<div style="font-size: 12; color: #64748B; margin-bottom: 4">Total Tagihan</div>
						<div style="font-size: 32; font-weight: 900; color: #2563EB; letter-spacing: -0.02em">{fmt(total)}</div>
						{#if member}
							<div style="font-size: 11; color: #059669; margin-top: 4">Member: {member.name} • {member.points.toLocaleString('id-ID')} poin</div>
						{/if}
					</div>

					<!-- Method -->
					<div class="flex gap-2 mb-4">
						{#each [
							{ id: 'tunai' as PaymentMethod, label: 'Tunai' },
							{ id: 'qris' as PaymentMethod, label: 'QRIS' },
							{ id: 'debit' as PaymentMethod, label: 'Kartu' },
						] as m}
							<button
								onclick={() => payMethod = m.id}
								class="flex-1 py-3 rounded-xl border-2 flex flex-col items-center gap-1 transition-all"
								style="
									border-color: {payMethod === m.id ? '#2563EB' : '#E2EBF4'};
									background: {payMethod === m.id ? '#EFF6FF' : '#FAFAFA'};
									color: {payMethod === m.id ? '#2563EB' : '#64748B'};
								"
							>
								{#if m.id === 'tunai'}
									<svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
										<rect x="1" y="4" width="22" height="16" rx="2" ry="2" />
										<line x1="1" y1="10" x2="23" y2="10" />
									</svg>
								{:else if m.id === 'qris'}
									<svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
										<rect x="3" y="3" width="7" height="7" />
										<rect x="14" y="3" width="7" height="7" />
										<rect x="14" y="14" width="7" height="7" />
										<rect x="3" y="14" width="7" height="7" />
									</svg>
								{:else}
									<svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
										<rect x="1" y="4" width="22" height="16" rx="2" ry="2" />
										<line x1="1" y1="10" x2="23" y2="10" />
									</svg>
								{/if}
								<span style="font-size: 12; font-weight: 600">{m.label}</span>
							</button>
						{/each}
					</div>

					<!-- Tunai: quick amounts + input -->
					{#if payMethod === 'tunai'}
						<div class="mb-4">
							<div class="mb-2" style="font-size: 12; font-weight: 600; color: #64748B">Uang Diterima</div>
							<input
								value={cashInput ? fmt(Number(cashInput.replace(/\D/g, ''))) : ''}
								oninput={(e) => cashInput = (e.target as HTMLInputElement).value.replace(/\D/g, '')}
								placeholder={fmt(total)}
								class="w-full px-4 py-3 rounded-xl border text-center outline-none mb-2"
								style="font-size: 22; font-weight: 800; border-color: #E2EBF4; background: #F8FAFC"
							/>
							<!-- Quick amounts -->
							<div class="flex gap-1.5">
								{#each [
									total,
									total + (5000 - (total % 5000) || 5000),
									Math.ceil(total / 50000) * 50000,
									Math.ceil(total / 100000) * 100000,
								].filter((v, i, a) => a.indexOf(v) === i).slice(0, 4) as amt}
									<button
										onclick={() => cashInput = String(amt)}
										class="flex-1 py-2 rounded-xl border text-center transition-all"
										style="
											background: {Number(cashInput) === amt ? '#DBEAFE' : '#F8FAFC'};
											border-color: {Number(cashInput) === amt ? '#2563EB' : '#E2EBF4'};
											color: {Number(cashInput) === amt ? '#2563EB' : '#0F172A'};
											font-size: 12;
											font-weight: 600;
										"
									>
										{fmt(amt)}
									</button>
								{/each}
							</div>
							{#if Number(cashInput) >= total}
								<div class="mt-3 flex justify-between px-4 py-2.5 rounded-xl" style="background: #D1FAE5">
									<span style="font-size: 13; font-weight: 600; color: #065F46">Kembalian</span>
									<span style="font-size: 16; font-weight: 800; color: #059669">{fmt(change)}</span>
								</div>
							{/if}
						</div>
					{/if}

					<!-- QRIS -->
					{#if payMethod === 'qris'}
						<div class="flex flex-col items-center py-4 mb-4 gap-3">
							<div class="w-36 h-36 rounded-2xl flex items-center justify-center" style="background: #F1F5F9; border: 2px dashed #E2EBF4">
								<div class="text-center">
									<svg width="36" height="36" viewBox="0 0 24 24" fill="none" stroke="#94A3B8" stroke-width="1.5" style="opacity: 0.4; margin: 0 auto 6px">
										<rect x="5" y="2" width="6" height="6" rx="1" />
										<rect x="13" y="2" width="6" height="6" rx="1" />
										<rect x="5" y="16" width="6" height="6" rx="1" />
									</svg>
									<div style="font-size: 11; color: #64748B">QR Code</div>
								</div>
							</div>
							<p style="font-size: 12; color: #64748B; text-align: center">Minta pelanggan scan QR dengan aplikasi dompet digital</p>
						</div>
					{/if}

					<!-- Debit -->
					{#if payMethod === 'debit'}
						<div class="flex items-center gap-3 p-4 rounded-xl mb-4" style="background: #F8FAFC; border: 1.5px solid #E2EBF4">
							<svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="#64748B" stroke-width="2">
								<rect x="1" y="4" width="22" height="16" rx="2" ry="2" />
								<line x1="1" y1="10" x2="23" y2="10" />
							</svg>
							<div>
								<div style="font-size: 13; font-weight: 600">Tempelkan / Gesek Kartu</div>
								<div style="font-size: 11; color: #64748B">EDC BRI · BCA · Mandiri · BNI</div>
							</div>
						</div>
					{/if}

					<button
						onclick={(e) => { lastTriggerEl = e.currentTarget; modal = 'receipt'; clearCart(); }}
						class="w-full py-4 rounded-2xl flex items-center justify-center gap-2 transition-all"
						style="
							background: linear-gradient(135deg, #2563EB 0%, #1D4ED8 100%);
							color: #fff;
							font-size: 16;
							font-weight: 800;
							box-shadow: 0 6px 20px rgba(37,99,235,0.35);
						"
					>
						<svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
							<path d="M22 11.08V12a10 10 0 11-5.93-9.14" />
							<polyline points="22 4 12 14.01 9 11.01" />
						</svg>
						Proses Pembayaran
					</button>
				</div>
			</div>
		</div>
	{/if}

	<!-- Receipt Success -->
	{#if modal === 'receipt'}
		<!-- svelte-ignore a11y_click_events_have_key_events -->
		<!-- svelte-ignore a11y_no_static_element_interactions -->
		<div
			class="fixed inset-0 flex items-center justify-center z-50 p-4"
			style="background: rgba(15,23,41,0.55); backdrop-filter: blur(2px)"
			onclick={(e) => { if (e.target === e.currentTarget) { modal = 'none'; lastTriggerEl?.focus(); } }}
			transition:fade={{ duration: 150 }}
			role="dialog"
			aria-modal="true"
			aria-labelledby="receipt-title"
		>
			<div
				class="rounded-3xl p-8 text-center shadow-2xl"
				style="background: #fff; max-width: 320px; width: 100%"
				transition:scale={{ duration: 200, start: 0.92, easing: cubicOut }}
			>
				<div class="w-16 h-16 rounded-full flex items-center justify-center mx-auto mb-4" style="background: #D1FAE5">
					<svg width="32" height="32" viewBox="0 0 24 24" fill="none" stroke="#059669" stroke-width="2">
						<path d="M22 11.08V12a10 10 0 11-5.93-9.14" />
						<polyline points="22 4 12 14.01 9 11.01" />
					</svg>
				</div>
				<div id="receipt-title" style="font-size: 20; font-weight: 800; margin-bottom: 6">Pembayaran Berhasil</div>
				{#if payMethod === 'tunai' && change > 0}
					<div class="px-4 py-2 rounded-xl mb-4 mt-2" style="background: #D1FAE5">
						<div style="font-size: 11; color: #065F46">Kembalian</div>
						<div style="font-size: 22; font-weight: 900; color: #059669">{fmt(change)}</div>
					</div>
				{/if}
				<div class="flex items-center justify-center gap-2 px-4 py-2.5 rounded-xl mt-2" style="background: #EFF6FF; color: #2563EB; font-size: 13">
					<svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
						<polyline points="6 9 6 2 18 2 18 9" />
						<path d="M6 18H4a2 2 0 01-2-2v-5a2 2 0 012-2h16a2 2 0 012 2v5a2 2 0 01-2 2h-2" />
						<rect x="6" y="14" width="12" height="8" />
					</svg>
					Mencetak struk...
				</div>
				<button
					onclick={() => modal = 'none'}
					class="mt-4 w-full py-3 rounded-2xl"
					style="background: #2563EB; color: #fff; font-size: 14; font-weight: 700"
				>
					Transaksi Baru
				</button>
			</div>
		</div>
	{/if}

	<!-- Hold / Held Transactions -->
	{#if modal === 'hold'}
		<!-- svelte-ignore a11y_click_events_have_key_events -->
		<!-- svelte-ignore a11y_no_static_element_interactions -->
		<div
			class="fixed inset-0 flex items-center justify-center z-50 p-4"
			style="background: rgba(15,23,41,0.55); backdrop-filter: blur(2px)"
			onclick={(e) => { if (e.target === e.currentTarget) { modal = 'none'; lastTriggerEl?.focus(); } }}
			transition:fade={{ duration: 150 }}
			role="dialog"
			aria-modal="true"
			aria-labelledby="hold-title"
		>
			<div
				class="rounded-3xl shadow-2xl overflow-hidden"
				style="background: #fff; width: 360; max-width: 100%"
				transition:scale={{ duration: 200, start: 0.96, easing: cubicOut }}
			>
				<div class="flex items-center justify-between px-5 py-4" style="background: #1E3A5F">
					<span id="hold-title" style="color: #fff; font-size: 15; font-weight: 700">Transaksi Ditahan</span>
					<button onclick={() => modal = 'none'}>
						<svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="rgba(255,255,255,0.6)" stroke-width="2">
							<line x1="18" y1="6" x2="6" y2="18" />
							<line x1="6" y1="6" x2="18" y2="18" />
						</svg>
					</button>
				</div>
				<div class="p-5">
					{#if cart.length > 0}
						<button class="w-full flex items-center gap-2 px-4 py-3 rounded-xl mb-3 border" style="background: #FEF3C7; border-color: #FCD34D; color: #92400E; font-size: 13; font-weight: 600">
							<svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
								<circle cx="12" cy="12" r="10" />
								<line x1="10" y1="15" x2="10" y2="9" />
								<line x1="14" y1="15" x2="14" y2="9" />
							</svg>
							Tahan Transaksi Saat Ini ({totalQty} item • {fmt(total)})
						</button>
					{/if}
					<div class="mb-2" style="font-size: 11; font-weight: 600; color: #64748B; text-transform: uppercase">Ditahan Sebelumnya</div>
					{#each HELD as h}
						<div class="flex items-center gap-3 px-3 py-3 rounded-xl border mb-2" style="border-color: #E2EBF4">
							<div class="w-9 h-9 rounded-xl flex items-center justify-center" style="background: #FEF3C7">
								<svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="#F59E0B" stroke-width="2">
									<circle cx="12" cy="12" r="10" />
									<line x1="10" y1="15" x2="10" y2="9" />
									<line x1="14" y1="15" x2="14" y2="9" />
								</svg>
							</div>
							<div class="flex-1">
								<div style="font-size: 13; font-weight: 700">{h.id}</div>
								<div style="font-size: 11; color: #64748B">{h.items} item • {h.time}</div>
							</div>
							<div style="font-size: 13; font-weight: 700">{fmt(h.total)}</div>
							<button class="px-3 py-1.5 rounded-lg" style="background: #2563EB; color: #fff; font-size: 12; font-weight: 600">Lanjut</button>
						</div>
					{/each}
					{#if HELD.length === 0 && cart.length === 0}
						<div class="py-8 text-center" style="color: #64748B; font-size: 13">Tidak ada transaksi yang ditahan</div>
					{/if}
				</div>
			</div>
		</div>
	{/if}

	<!-- Discount Modal -->
	{#if modal === 'discount'}
		<!-- svelte-ignore a11y_click_events_have_key_events -->
		<!-- svelte-ignore a11y_no_static_element_interactions -->
		<div
			class="fixed inset-0 flex items-center justify-center z-50 p-4"
			style="background: rgba(15,23,41,0.55); backdrop-filter: blur(2px)"
			onclick={(e) => { if (e.target === e.currentTarget) { modal = 'none'; lastTriggerEl?.focus(); } }}
			transition:fade={{ duration: 150 }}
			role="dialog"
			aria-modal="true"
			aria-labelledby="discount-title"
		>
			<div
				class="rounded-3xl shadow-2xl overflow-hidden"
				style="background: #fff; width: 360; max-width: 100%"
				transition:scale={{ duration: 200, start: 0.96, easing: cubicOut }}
			>
				<div class="flex items-center justify-between px-5 py-4" style="background: #1E3A5F">
					<span id="discount-title" style="color: #fff; font-size: 15; font-weight: 700">Tambah Diskon</span>
					<button onclick={() => modal = 'none'}>
						<svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="rgba(255,255,255,0.6)" stroke-width="2">
							<line x1="18" y1="6" x2="6" y2="18" />
							<line x1="6" y1="6" x2="18" y2="18" />
						</svg>
					</button>
				</div>
				<div class="p-5">
					<div class="mb-4" style="font-size: 12; color: #64748B">Diskon berlaku untuk seluruh transaksi</div>
					<div class="grid grid-cols-3 gap-2 mb-4">
						{#each [5, 10, 15, 20, 25, 30] as pct}
							<button
								onclick={() => discInput = String(pct)}
								class="py-3 rounded-xl border-2 transition-all"
								style="
									border-color: {discInput === String(pct) ? '#2563EB' : '#E2EBF4'};
									background: {discInput === String(pct) ? '#EFF6FF' : '#FAFAFA'};
									color: {discInput === String(pct) ? '#2563EB' : '#0F172A'};
									font-size: 15;
									font-weight: 700;
								"
							>
								{pct}%
							</button>
						{/each}
					</div>
					<div class="relative mb-4">
						<input
							bind:value={discInput}
							oninput={(e) => discInput = (e.target as HTMLInputElement).value.replace(/\D/g, '')}
							placeholder="Masukkan % diskon"
							class="w-full px-4 py-3 rounded-xl border text-center outline-none"
							style="font-size: 18; font-weight: 700; border-color: #E2EBF4; background: #F8FAFC"
						/>
						<span class="absolute right-4 top-1/2 -translate-y-1/2" style="font-size: 16; font-weight: 700">%</span>
					</div>
					{#if discInput && Number(discInput) > 0}
						<div class="flex justify-between px-4 py-2 rounded-xl mb-4" style="background: #F0FDF4; border: 1px solid #BBF7D0">
							<span style="font-size: 13">Hemat</span>
							<span style="font-size: 14; font-weight: 700; color: #059669">- {fmt(Math.round(subtotal * Number(discInput) / 100))}</span>
						</div>
					{/if}
					<div class="flex gap-2">
						<button
							onclick={() => { globalDisc = 0; discInput = ''; modal = 'none'; }}
							class="flex-1 py-2.5 rounded-xl border"
							style="border-color: #E2EBF4; font-size: 13; font-weight: 600"
						>
							Hapus Diskon
						</button>
						<button
							onclick={() => { globalDisc = Number(discInput) || 0; modal = 'none'; }}
							class="flex-1 py-2.5 rounded-xl"
							style="background: #2563EB; color: #fff; font-size: 13; font-weight: 700"
						>
							Terapkan
						</button>
					</div>
				</div>
			</div>
		</div>
	{/if}

	<!-- Numpad Modal -->
	{#if modal === 'numpad'}
		<!-- svelte-ignore a11y_click_events_have_key_events -->
		<!-- svelte-ignore a11y_no_static_element_interactions -->
		<div
			class="fixed inset-0 flex items-center justify-center z-50 p-4"
			style="background: rgba(15,23,41,0.55); backdrop-filter: blur(2px)"
			onclick={(e) => { if (e.target === e.currentTarget) { modal = 'none'; lastTriggerEl?.focus(); } }}
			transition:fade={{ duration: 150 }}
			role="dialog"
			aria-modal="true"
			aria-labelledby="numpad-title"
		>
			<div
				class="rounded-3xl shadow-2xl overflow-hidden"
				style="background: #fff; width: 360; max-width: 100%"
				transition:scale={{ duration: 200, start: 0.96, easing: cubicOut }}
			>
				<div class="flex items-center justify-between px-5 py-4" style="background: #1E3A5F">
					<span id="numpad-title" style="color: #fff; font-size: 15; font-weight: 700">Ubah Jumlah</span>
					<button onclick={() => modal = 'none'}>
						<svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="rgba(255,255,255,0.6)" stroke-width="2">
							<line x1="18" y1="6" x2="6" y2="18" />
							<line x1="6" y1="6" x2="18" y2="18" />
						</svg>
					</button>
				</div>
				<div class="p-5">
					<div class="flex items-center justify-center py-3 mb-4 rounded-xl" style="background: #F8FAFC; font-size: 28; font-weight: 900">
						{numpadValue || '0'}
					</div>
					<div class="grid grid-cols-3 gap-2 mb-3">
						{#each ['1','2','3','4','5','6','7','8','9','C','0','⌫'] as k}
							<button
								onclick={() => numpadPress(k)}
								class="py-4 rounded-xl transition-all"
								style="
									background: {k === '⌫' || k === 'C' ? '#FEF2F2' : '#F8FAFC'};
									color: {k === '⌫' || k === 'C' ? '#DC2626' : '#0F172A'};
									font-size: 18;
									font-weight: 700;
									border: 1.5px solid #E2EBF4;
								"
							>
								{k}
							</button>
						{/each}
					</div>
					<button
						onclick={numpadConfirm}
						class="w-full py-3.5 rounded-xl flex items-center justify-center gap-2"
						style="background: #2563EB; color: #fff; font-size: 15; font-weight: 700"
					>
						<svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
							<polyline points="20 6 9 17 4 12" />
						</svg>
						Konfirmasi
					</button>
				</div>
			</div>
		</div>
	{/if}

	<!-- New Member Modal -->
	{#if modal === 'member'}
		<!-- svelte-ignore a11y_click_events_have_key_events -->
		<!-- svelte-ignore a11y_no_static_element_interactions -->
		<div
		class="fixed inset-0 flex items-center justify-center z-50 p-4"
		style="background: rgba(15,23,41,0.55); backdrop-filter: blur(2px)"
		onclick={(e) => { if (e.target === e.currentTarget) { modal = 'none'; lastTriggerEl?.focus(); } }}
		transition:fade={{ duration: 150 }}
		role="dialog"
		aria-modal="true"
		aria-labelledby="member-title"
		>
			<div
			class="rounded-3xl shadow-2xl overflow-hidden"
			style="background: #fff; width: 360; max-width: 100%"
			transition:scale={{ duration: 200, start: 0.96, easing: cubicOut }}
			>
			<div class="flex items-center justify-between px-5 py-4" style="background: #1E3A5F">
				<span id="member-title" style="color: #fff; font-size: 15; font-weight: 700">Tambah Member Baru</span>
				<button onclick={() => modal = 'none'}>
					<svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="rgba(255,255,255,0.6)" stroke-width="2">
						<line x1="18" y1="6" x2="6" y2="18" />
						<line x1="6" y1="6" x2="18" y2="18" />
					</svg>
				</button>
			</div>
			<div class="p-5">
				{#each [
					{ label: 'Nama Lengkap', placeholder: 'Nama pelanggan' },
					{ label: 'No. HP', placeholder: '08xxxxxxxxxx' },
					{ label: 'Email (opsional)', placeholder: 'email@domain.com' },
				] as f}
					<div class="mb-3">
						<label style="font-size: 12; font-weight: 600; display: block; margin-bottom: 4">{f.label}</label>
						<input
							placeholder={f.placeholder}
							class="w-full px-3 py-2.5 rounded-xl border outline-none"
							style="background: #F8FAFC; border-color: #E2EBF4; font-size: 13"
						/>
					</div>
				{/each}
				<button
					onclick={() => modal = 'none'}
					class="w-full mt-2 py-3 rounded-xl"
					style="background: #2563EB; color: #fff; font-size: 14; font-weight: 700"
				>
					Daftarkan Member
				</button>
				</div>
				</div>
		</div>
	{/if}
</div>

<style>
/* ── Product card: subtle hover lift + active press ───────────────────── */
.product-card:not(:disabled):hover {
	transform: translateY(-1px);
	box-shadow: 0 4px 12px rgba(15, 23, 42, 0.08), 0 0 0 1px var(--color-hekas-blue-pale) !important;
	border-color: var(--color-hekas-blue-light) !important;
}
.card-press:not(:disabled):active {
	transform: scale(0.97);
}

/* ── Search input: focus state ─────────────────────────────────────────── */
.search-input:focus {
	border-color: var(--color-hekas-blue) !important;
	background: #fff !important;
	box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.12) !important;
}
.search-clear:hover {
	background: #94A3B8 !important;
}

/* ── Category tabs: hover state ─────────────────────────────────────────── */
.category-tab:hover:not([aria-pressed="true"]) {
	background: var(--color-hekas-blue-tint) !important;
	color: var(--color-hekas-blue-deep) !important;
	border-color: var(--color-hekas-blue-pale) !important;
}

/* ── Member search: focus ─────────────────────────────────────────────── */
.member-search:focus {
	border-color: var(--color-hekas-blue) !important;
	box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.12) !important;
}

/* ── Method button: hover ─────────────────────────────────────────────── */
.method-btn:hover:not([aria-pressed="true"]) {
	background: var(--color-hekas-blue-tint) !important;
	color: var(--color-hekas-blue-deep) !important;
	border-color: var(--color-hekas-blue-pale) !important;
}

/* ── Sidebar nav: hover reveal ────────────────────────────────────────── */
.sidebar-nav-item:hover {
	background: rgba(255, 255, 255, 0.06) !important;
	color: #fff !important;
}
.sidebar-nav-item:hover .sidebar-tooltip {
	opacity: 1;
	transform: translate(0, -50%) translateX(0);
}
.sidebar-tooltip {
	opacity: 0;
	transform: translate(0, -50%) translateX(-4px);
	transition: opacity 150ms ease, transform 150ms ease;
}
.sidebar-nav-item:focus-visible {
	background: rgba(37, 99, 235, 0.18) !important;
	color: #fff !important;
}

/* ── Hardware status dot: pulse on hover ──────────────────────────────── */
.hardware-dot:hover {
	background: rgba(255, 255, 255, 0.08);
}

/* ── Global focus-visible ring for keyboard accessibility ──────────────── */
:global(button:focus-visible),
:global(input:focus-visible) {
	outline: 2px solid var(--color-hekas-blue);
	outline-offset: 2px;
}

/* ── Sidebar nav active scale on click ───────────────────────────────── */
:global(aside button:active) {
	transform: scale(0.92);
	transition: transform 80ms ease;
}

/* ── Reduce motion for accessibility ─────────────────────────────────── */
@media (prefers-reduced-motion: reduce) {
:global(*),
:global(*::before),
:global(*::after) {
animation-duration: 0.01ms !important;
animation-iteration-count: 1 !important;
transition-duration: 0.01ms !important;
}
}

/* Skip-to-content link — hidden until focused via keyboard */
.skip-to-content {
	position: absolute;
	left: -9999px;
	top: 0;
	z-index: 100;
	padding: 12px 18px;
	background: #2563EB;
	color: #fff;
	font-size: 13px;
	font-weight: 700;
	border-radius: 0 0 8px 0;
	text-decoration: none;
	box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
}
.skip-to-content:focus {
	left: 0;
}

/* Sidebar tooltip — visible on hover/focus */
.sidebar-tooltip {
	opacity: 0;
	transform: translateX(-8px) translateY(-50%);
	transition: opacity 150ms ease, transform 150ms ease;
}
button:hover .sidebar-tooltip,
button:focus-visible .sidebar-tooltip,
.group:hover .sidebar-tooltip {
	opacity: 1;
	transform: translateX(0) translateY(-50%);
}

/* Spin animation for loading indicator */
@keyframes spin {
	from { transform: rotate(0deg); }
	to { transform: rotate(360deg); }
}

@keyframes pulse {
	0%, 100% { opacity: 1; transform: scale(1); }
	50% { opacity: 0.6; transform: scale(1.15); }
}

@keyframes shimmer {
	0% { background-position: -200% 0; }
	100% { background-position: 200% 0; }
}

/* Responsive: collapse cart panel on tablet portrait (1024px) */
@media (max-width: 1024px) {
	:global(aside.cart-panel) {
		position: fixed;
		right: 0;
		top: 44px;
		bottom: 0;
		z-index: 40;
		transform: translateX(100%);
		transition: transform 200ms ease;
		box-shadow: -4px 0 16px rgba(0, 0, 0, 0.1);
	}
	:global(aside.cart-panel.open) {
		transform: translateX(0);
	}
	:global(.cart-fab) {
		display: flex !important;
	}
}
@media (min-width: 1025px) {
	:global(.cart-fab) {
		display: none !important;
	}
}

/* Larger font for production cashiers (accessibility) */
@media (min-resolution: 1dppx) and (max-width: 1280px) {
	:global(.product-name) {
		font-size: 13px;
	}
}
</style>