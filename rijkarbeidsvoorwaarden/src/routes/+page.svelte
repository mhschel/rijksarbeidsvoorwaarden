<script lang="ts">
	// SVELTE CONCEPT: <script> tag contains your JavaScript/TypeScript
	// This runs when the component is created

	import { page } from '$app/stores';
	import { getAvailableSchalen, getAvailableTredes, getMonthlyCompensation } from '$lib/salaryData';

	// Get URL parameters for schaal, trede and hours (if provided)
	// Example: ?schaal=12&trede=3&uren=40
	const urlSchaal = $page.url.searchParams.get('schaal');
	const urlTrede = $page.url.searchParams.get('trede');
	const urlUren = $page.url.searchParams.get('uren');

	// SVELTE CONCEPT: These are "reactive variables"
	// When they change, Svelte automatically updates the UI
	// Use URL params if available, otherwise use defaults
	let selectedSchaal = urlSchaal ? parseInt(urlSchaal) : 11;  // Default to schaal 11
	let selectedTrede = urlTrede ? parseInt(urlTrede) : 5;    // Default to trede 5
	let selectedUren = urlUren ? parseInt(urlUren) : 36;      // Default to 36 hours

	// Available hours options
	const urenOptions = [27, 36, 38, 40];

	// SVELTE CONCEPT: $: makes this a "reactive statement"
	// It automatically recalculates when dependencies (selectedSchaal, selectedTrede) change
	$: monthlyCompensation = getMonthlyCompensation(selectedSchaal, selectedTrede);

	// Update URL when schaal, trede or uren changes (for easy sharing)
	$: {
		if (typeof window !== 'undefined') {
			const url = new URL(window.location.href);
			url.searchParams.set('schaal', selectedSchaal.toString());
			url.searchParams.set('trede', selectedTrede.toString());
			url.searchParams.set('uren', selectedUren.toString());
			window.history.replaceState({}, '', url);
		}
	}

	$: scaleFactor = 12 * (selectedUren / 36);

	$: yearlyPreTaxSalary = monthlyCompensation * scaleFactor;
	$: individualPensionContribution = (monthlyCompensation * 0.081) * scaleFactor;
	$: ikbBudget = (monthlyCompensation * 0.165) * scaleFactor;
	$: pretaxSalaryInclBenefits = yearlyPreTaxSalary + ikbBudget - individualPensionContribution;
	$: pensoenEmployer = (monthlyCompensation * (0.27 - 0.081)) * scaleFactor;
	$: pension = individualPensionContribution + pensoenEmployer;
	$: totalCompensation = pretaxSalaryInclBenefits + pension;

	// Get available options for dropdowns
	const schalen = getAvailableSchalen();
	const tredes = getAvailableTredes();

	// Format numbers as currency
	function formatCurrency(value: number): string {
		return new Intl.NumberFormat('nl-NL', {
			style: 'currency',
			currency: 'EUR',
			minimumFractionDigits: 0,
			maximumFractionDigits: 0
		}).format(value);
	}
</script>

<!-- SVELTE CONCEPT: Everything outside <script> and <style> is your HTML template -->
<div class="container">
	<h1>Arbeidsvoorwaarden Rijksoverheid Vergelijken</h1>
	<p class="subtitle">
		Bij de Rijksoverheid bestaat de beloning uit een aantal verschillende onderdelen.
		Hierdoor is het soms lastig om de beloning af te zetten tegen een bruto salaris in de private sector.
		Deze onofficiële website helpt met vergelijken. 
		Deze onderliggende informatie is afkomstig uit de <a href="https://ambtenarensalaris.nl/wp-content/uploads/2024/07/CAORijk2024-2025per1juli2024-versievastgesteld27juni2024.pdf">CAO Rijk 2024-2025</a>.
	</p>

	<!-- Input Section -->
	<div class="input-section">
		<div class="input-group">
			<label for="schaal">Schaal:</label>
			<!-- SVELTE CONCEPT: bind:value creates two-way binding -->
			<!-- When user changes select, selectedSchaal updates automatically -->
			<select id="schaal" bind:value={selectedSchaal}>
				{#each schalen as schaal}
					<option value={schaal}>{schaal}</option>
				{/each}
			</select>
		</div>

		<div class="input-group">
			<label for="trede">Trede:</label>
			<select id="trede" bind:value={selectedTrede}>
				<!-- SVELTE CONCEPT: {#each} is a loop in Svelte -->
				{#each tredes as trede}
					<option value={trede}>{trede}</option>
				{/each}
			</select>
		</div>

		<div class="input-group">
			<label for="uren">Uren:</label>
			<select id="uren" bind:value={selectedUren}>
				{#each urenOptions as uren}
					<option value={uren}>{uren}</option>
				{/each}
			</select>
		</div>
	</div>

	{#if yearlyPreTaxSalary + ikbBudget > 130000}
		<p class="warning-note">
			Voor inkomen boven de €137.000 gelden andere regels qua pensioenopbouw. De onderstaande bedragen nemen dat niet mee.
		</p>
	{/if}

	<!-- Section 1: Monthly CAO Amount (Highlighted) -->
	<div class="section">
		<h2>Maandbedrag CAO Rijk (geschaald naar aantal uren)<sup class="footnote-ref">1</sup></h2>
		<div class="value">{formatCurrency(monthlyCompensation * (selectedUren / 36))}</div>
	</div>

	<!-- Section 2: Bruto Salary including IKB and Vakantiegeld -->
	<div class="section">
		<h2>Bruto jaarsalaris (incl. IKB)<sup class="footnote-ref">2</sup></h2>
		<div class="section-content">
			<div class="result-item">
				<span class="label">Bruto jaarsalaris:</span>
				<span class="value">{formatCurrency(yearlyPreTaxSalary)}</span>
			</div>
			<div class="result-item">
				<span class="label">Verplicht Eigen Inleg Pensioen:</span>
				<span class="value">{formatCurrency(-1 * individualPensionContribution)}</span>
			</div>
			<div class="result-item">
				<span class="label">IKB:</span>
				<span class="value">{formatCurrency(ikbBudget)}</span>
			</div>
			<div class="result-item total">
				<span class="label">Totaal:</span>
				<span class="value">{formatCurrency(pretaxSalaryInclBenefits)}</span>
			</div>
		</div>
	</div>

	<div class="section">
		<h2>Pensioeninleg<sup class="footnote-ref">3</sup></h2>
		<div class="section-content">
			<div class="result-item">
				<span class="label">Verplicht Eigen Inleg Pensioen:</span>
				<span class="value">{formatCurrency(individualPensionContribution)}</span>
			</div>
		</div>
		<div class="section-content">
			<div class="result-item">
				<span class="label">Pensioeninleg Werkgever:</span>
				<span class="value">{formatCurrency(pensoenEmployer)}</span>
			</div>
		</div>
		<div class="section-content"></div>
		<div class="result-item total">
			<span class="label">Totaal:</span>
			<span class="value">{formatCurrency(pension)}</span>
		</div>
	</div>

	<!-- Section 4: Total Compensation -->
	<div class="section final">
		<h2>Totale beloning voor {selectedUren} uur per week</h2>
		<div class="section-content">
			<div class="result-item total">
				<span class="label">Totaal:</span>
				<span class="value">{formatCurrency(totalCompensation)}</span>
			</div>
			{#if selectedUren === 36}
				<div class="result-item total">
					<span class="label">Beloning geschaald van 4-daagse naar 5-daagse werkweek:</span>
					<span class="value">{formatCurrency(totalCompensation * (5 / 4))}</span>
				</div>
			{/if}
		</div>
	</div>

	<!-- Footnotes -->
	<div class="footnotes">
		<h3>Toelichting</h3>
		<ol>
			<li>
				Bij de Rijksoverheid zijn contracten doorgaans voor 36 uur per week. Werknemers hebben recht op '4x9' werken (4 dagen x 9 uur). Werknemers die 5x8 werken met een contract voor 36 uur bouwen compensatieverlof op, waardoor zij in totaal tot wel 11,5 weken per jaar verlof opbouwen. In overleg met de manager is het soms mogelijk om meer dan 36 uur te werken, namelijk 38 of 40 uur. Dan ontvangt de werknemer een hogere beloning en bouwt men geen compensatieverlof op.
				<a href="https://www.p-direkt.nl/informatie-rijkspersoneel-2020/vrije-dagen/vakantie/compensatie-uren/compensatie-uren-opbouwen">Meer info</a>
			</li>
			<li>
				Het IKB is 16,5% van je bruto maandsalaris en kan je bijvoorbeeld gebruiken voor extra verlof of uitbetaling.
				<a href="https://www.p-direkt.nl/informatie-rijkspersoneel-2020/financien/salaris/individueel-keuzebudget-ikb" target="_blank" rel="noopener">Meer info</a>
			</li>
			<li>
				27% van het totale salaris is pensioeninleg bij ABP. Twee-derde van de inleg is van de werkgever; een-derde is een verplichte eigen inleg uit het bruto salaris.
				<a href="https://www.abp.nl/pensioen-bij-abp/pensioenpremie-en-opbouw">Meer info</a>
			</li>
		</ol>
	</div>
</div>

<!-- SVELTE CONCEPT: <style> tag is scoped to this component only -->
<!-- These styles won't affect other components -->
<style>
	@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');

	.container {
		max-width: 800px;
		margin: 0 auto;
		padding: 2rem;
		font-family: 'Inter', system-ui, -apple-system, sans-serif;
		min-height: 100vh;
	}

	h1 {
		color: #01689b;
		margin-bottom: 1rem;
		font-size: clamp(1.75rem, 5vw, 2.5rem);
		font-weight: 700;
		letter-spacing: -0.025em;
		line-height: 1.2;
		word-wrap: break-word;
		hyphens: auto;
	}

	.subtitle {
		color: #334155;
		margin-bottom: 2.5rem;
		font-size: 1.125rem;
		line-height: 1.7;
		max-width: 700px;
	}

	.warning-note {
		background: #fef3c7;
		border: 1px solid #f59e0b;
		border-radius: 8px;
		padding: 1rem;
		margin-bottom: 1.5rem;
		color: #92400e;
		font-size: 0.875rem;
		line-height: 1.5;
	}

	.input-section {
		display: flex;
		gap: 1.5rem;
		margin-bottom: 2rem;
		padding: 1.5rem;
		background: white;
		border: 2px solid #e2e8f0;
		border-radius: 12px;
		box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
	}

	.input-group {
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
		flex: 1;
	}

	label {
		font-weight: 600;
		color: #334155;
		font-size: 0.875rem;
		text-transform: uppercase;
		letter-spacing: 0.05em;
	}

	select {
		padding: 0.75rem;
		border: 2px solid #cbd5e1;
		border-radius: 6px;
		font-size: 1.125rem;
		font-weight: 600;
		color: #01689b;
		background: white;
		cursor: pointer;
		transition: all 0.15s ease;
	}

	select:hover {
		border-color: #01689b;
	}

	select:focus {
		outline: none;
		border-color: #01689b;
		box-shadow: 0 0 0 3px rgba(1, 104, 155, 0.1);
	}

	/* Section Styles */
	.section {
		background: white;
		border: 2px solid #e2e8f0;
		border-radius: 12px;
		padding: 1.75rem;
		margin-bottom: 1.25rem;
		box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
	}

	.section.final {
		background: linear-gradient(to bottom, #ffffff 0%, #f1f5f9 100%);
		border: 2px solid #01689b;
		box-shadow: 0 4px 12px rgba(1, 104, 155, 0.15);
	}

	h2 {
		margin-top: 0;
		color: #01689b;
		font-size: 1rem;
		margin-bottom: 1.25rem;
		font-weight: 700;
		text-transform: uppercase;
		letter-spacing: 0.05em;
	}

	.section-content {
		display: flex;
		flex-direction: column;
		gap: 0;
	}

	.result-item {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 0.875rem 0;
		border-bottom: 1px solid #e2e8f0;
	}

	.result-item:last-child {
		border-bottom: none;
	}

	.result-item.total {
		border-top: 2px solid #01689b;
		border-bottom: none;
		font-weight: 700;
		margin-top: 0.75rem;
		padding: 1rem 0 0.5rem 0;
		font-size: 1.125rem;
		padding-left: 0.75rem;
		padding-right: 0.75rem;
		margin-left: -0.75rem;
		margin-right: -0.75rem;
		border-radius: 6px;
	}

	.result-item.equivalent {
		background: linear-gradient(135deg, #01689b 0%, #0284c7 100%);
		padding: 1.25rem;
		margin-top: 1rem;
		border-radius: 8px;
		font-weight: 700;
		font-size: 1.25rem;
		color: white;
		border: none;
		box-shadow: 0 4px 12px rgba(1, 104, 155, 0.25);
	}

	.label {
		color: #475569;
		font-size: 0.9375rem;
	}

	.result-item.total .label {
		color: #01689b;
	}

	.result-item.equivalent .label {
		color: white;
		opacity: 0.95;
	}

	.value {
		font-weight: 700;
		color: #01689b;
		font-size: 1.125rem;
		letter-spacing: -0.025em;
		font-variant-numeric: tabular-nums;
	}

	.result-item.total .value {
		font-size: 1.375rem;
	}

	.result-item.equivalent .value {
		color: white;
		font-size: 1.5rem;
	}

	/* Footnotes */
	.footnote-ref {
		color: #01689b;
		font-weight: 600;
		font-size: 0.75em;
		margin-left: 0.15em;
	}

	.footnotes {
		margin-top: 3rem;
		padding: 1.5rem;
		background: #f8fafc;
		border-radius: 8px;
		border-left: 4px solid #01689b;
	}

	.footnotes h3 {
		margin-top: 0;
		margin-bottom: 1rem;
		color: #01689b;
		font-size: 0.875rem;
		text-transform: uppercase;
		letter-spacing: 0.05em;
		font-weight: 700;
	}

	.footnotes ol {
		margin: 0;
		padding-left: 1.5rem;
	}

	.footnotes li {
		margin-bottom: 1rem;
		color: #475569;
		font-size: 0.875rem;
		line-height: 1.6;
	}

	.footnotes li:last-child {
		margin-bottom: 0;
	}

	.footnotes a {
		color: #01689b;
		text-decoration: none;
		font-weight: 600;
		margin-left: 0.25rem;
	}

	.footnotes a:hover {
		text-decoration: underline;
	}
</style>
