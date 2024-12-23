<script lang="ts">
	import { onMount } from 'svelte';
	import { writable, type Writable } from 'svelte/store';

	type Matrix = number[][];

	const numUsers = 5;
	const numItems = 5;
	const latentFactors = 3;
	const learningRate = 0.01;
	const numEpochs = 1000;

	let error: Writable<number> = writable(0);
	let currentEpoch: Writable<number> = writable(1);
	let progress: Writable<number> = writable(0);

	// Initialize matrices for user and item latent factors
	let P: Writable<Matrix> = writable(
		Array.from({ length: numUsers }, () =>
			Array(latentFactors)
				.fill(0)
				.map(() => Math.random())
		)
	);
	let Q: Writable<Matrix> = writable(
		Array.from({ length: numItems }, () =>
			Array(latentFactors)
				.fill(0)
				.map(() => Math.random())
		)
	);
	let R_hat: Writable<Matrix> = writable(
		Array(numUsers)
			.fill(0)
			.map(() => Array(numItems).fill(0))
	);

	// Original ratings matrix (R)
	const R: Matrix = [
		[5, 3, 0, 1, 4],
		[4, 0, 0, 1, 2],
		[1, 1, 0, 5, 3],
		[0, 0, 5, 4, 2],
		[3, 4, 4, 3, 0]
	];

	// Store each epoch's state for playback with slider
	let epochData: Array<{ P: Matrix; Q: Matrix; R_hat: Matrix; error: number }> = [];

	// Function to get color based on value for green-to-red gradient
	function getColor(value: number, min: number, max: number): string {
		const ratio = (value - min) / (max - min);
		const red = Math.round(255 * (1 - ratio));
		const green = Math.round(255 * ratio);
		return `rgb(${red}, ${green}, 0)`;
	}

	async function matrixFactorization(R: Matrix, epochs: number, learningRate: number) {
		let PVal = Array.from({ length: numUsers }, () =>
			Array(latentFactors)
				.fill(0)
				.map(() => Math.random())
		);
		let QVal = Array.from({ length: numItems }, () =>
			Array(latentFactors)
				.fill(0)
				.map(() => Math.random())
		);

		for (let epoch = 0; epoch < epochs; epoch++) {
			let errorSum = 0;

			for (let i = 0; i < R.length; i++) {
				for (let j = 0; j < R[i].length; j++) {
					const rating = R[i][j];
					if (rating > 0) {
						const prediction = PVal[i].reduce((sum, p_ik, k) => sum + p_ik * QVal[j][k], 0);
						const eij = rating - prediction;
						errorSum += eij ** 2;

						// Update P and Q using SGD
						for (let k = 0; k < latentFactors; k++) {
							PVal[i][k] += learningRate * (2 * eij * QVal[j][k]);
							QVal[j][k] += learningRate * (2 * eij * PVal[i][k]);
						}
					}
				}
			}

			// Calculate R_hat based on current P and Q
			const updatedRHat = PVal.map((userFactors, i) =>
				QVal.map((itemFactors, j) =>
					userFactors.reduce((sum, p_ik, k) => sum + p_ik * itemFactors[k], 0)
				)
			);

			// Save epoch data for slider control
			epochData.push({
				P: PVal.map((row) => [...row]),
				Q: QVal.map((row) => [...row]),
				R_hat: updatedRHat,
				error: errorSum
			});

			// Update the current epoch data for real-time display
			P.set(PVal.map((row) => [...row]));
			Q.set(QVal.map((row) => [...row]));
			R_hat.set(updatedRHat);
			error.set(errorSum);
			currentEpoch.set(epoch + 1);
			progress.set(((epoch + 1) / epochs) * 100);

			if (errorSum === 0) break; // Stop if error reaches zero

			await new Promise((resolve) => setTimeout(resolve, 10));
		}
	}

	function updateEpoch(epoch: number) {
		if (epoch >= 0 && epoch < epochData.length) {
			const data = epochData[epoch];
			P.set(data.P);
			Q.set(data.Q);
			R_hat.set(data.R_hat);
			error.set(data.error);
			currentEpoch.set(epoch + 1);
			progress.set(((epoch + 1) / numEpochs) * 100);
		}
	}

	onMount(() => {
		matrixFactorization(R, numEpochs, learningRate);
	});
</script>

<div class="py-6 text-center">
	<h1 class="mb-4 text-2xl font-bold">Matrix Factorization with Stochastic Gradient Descent</h1>
	<p class="mb-4">Epoch: {$currentEpoch} / {epochData.length}</p>
	<p class="mb-4">Error: {$error.toFixed(4)}</p>

	<!-- Slider for Epoch Control -->
	<div class="mb-4 flex items-center justify-center">
		<input
			type="range"
			min="1"
			max={epochData.length}
			bind:value={$currentEpoch}
			on:input={(e) => updateEpoch(parseInt((e.target as HTMLInputElement).value) - 1)}
		/>
	</div>

	<div class="relative mb-4 pt-1">
		<div class="mb-2 flex items-center justify-between">
			<span class="text-sm font-medium text-blue-700">Progress</span>
			<span class="text-sm font-medium text-blue-700">{$progress.toFixed(0)}%</span>
		</div>
		<div class="h-4 rounded-full bg-gray-200">
			<div class="h-4 rounded-full bg-blue-600" style="width: {$progress}%"></div>
		</div>
	</div>

	<div class="grid grid-cols-4 gap-4">
		<!-- Matrix R (Original Ratings) Table -->
		<div>
			<h2 class="font-bold">Matrix R (User-Item Ratings)</h2>
			<table class="w-full table-auto">
				<tbody>
					{#each R as row}
						<tr
							>{#each row as value}<td class="border p-2">{value}</td>{/each}</tr
						>
					{/each}
				</tbody>
			</table>
		</div>

		<!-- Matrix P Table -->
		<div>
			<h2 class="font-bold">Matrix P (Users x Factors)</h2>
			<table class="w-full table-auto">
				<tbody>
					{#each $P as row}
						<tr
							>{#each row as value}<td class="border p-2">{value.toFixed(2)}</td>{/each}</tr
						>
					{/each}
				</tbody>
			</table>
		</div>

		<!-- Matrix Q Table -->
		<div>
			<h2 class="font-bold">Matrix Q (Items x Factors)</h2>
			<table class="w-full table-auto">
				<tbody>
					{#each $Q as row}
						<tr
							>{#each row as value}<td class="border p-2">{value.toFixed(2)}</td>{/each}</tr
						>
					{/each}
				</tbody>
			</table>
		</div>

		<!-- Matrix R_hat (Approximation) Table with Color Gradient -->
		<div>
			<h2 class="font-bold">Matrix R_hat (Approximated Ratings)</h2>
			<table class="w-full table-auto">
				<tbody>
					{#each $R_hat as row}
						<tr
							>{#each row as value}
								<td class="border p-2" style="background-color: {getColor(value, 0, 5)}">
									{value.toFixed(2)}
								</td>
							{/each}</tr
						>
					{/each}
				</tbody>
			</table>
		</div>
	</div>
</div>
